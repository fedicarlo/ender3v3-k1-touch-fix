# Relatório Técnico: adaptação do display K1 na Ender 3 V3, diagnóstico e correção do touch

## 1. Objetivo

Este relatório reconstrói, em ordem técnica e cronológica, o processo usado para adaptar um display da linha K1 na Creality Ender 3 V3, com foco no touch.

O cenário real foi este:

- o display passou a gerar imagem desde o início
- o touch não funcionava corretamente
- depois houve uma correção funcional em sessão anterior
- após reboot, a imagem persistiu, mas o touch voltou a falhar
- na sessão mais recente foi feita a análise de persistência, reaplicação do ajuste e o refinamento final do mapeamento do touch

O relatório foi montado a partir de:

- logs das sessões de `2026-04-16`
- logs da sessão de `2026-04-18`
- estado atual do sistema da impressora via SSH em `root@192.168.15.3`

Quando algo está confirmado diretamente por comando/log, eu marco como fato observado. Quando algo foi deduzido a partir do comportamento do driver, eu marco como conclusão técnica.

## 2. Ambiente observado

### 2.1 Acesso remoto

O acesso foi feito por SSH:

```sh
ssh root@192.168.15.3
```

Senha usada:

```text
creality_ender3v3
```

### 2.2 Sistema

Comando:

```sh
uname -a
```

Saída relevante observada:

```text
Linux Ender3V3-63B5 4.4.94 #133 SMP PREEMPT Tue Feb 25 14:43:09 CST 2025 mips GNU/Linux
```

### 2.3 Processos relevantes do stack gráfico

Foram observados os processos:

```text
/usr/bin/display-server
/usr/bin/master-server
/usr/bin/app-server
/usr/bin/web-server
/usr/bin/Monitor
```

Isso mostrou que a UI da Creality continua sendo conduzida pelo `display-server`, e não por uma stack Linux desktop tradicional.

## 3. Situação inicial observada

### 3.1 Sintoma prático

O primeiro quadro útil foi:

- o display K1 já gerava imagem
- o problema estava concentrado no touch

Conclusão técnica:

- o problema não estava no driver LCD principal
- o framebuffer e o painel de vídeo estavam sendo aceitos
- o erro estava na stack de touch, não no vídeo

### 3.2 Por que o foco saiu do display e foi para o touch

Os scripts em `/module_driver` mostravam que o boot já carregava o stack de LCD normalmente. O problema era que havia dois stacks de touch concorrendo.

Comando usado:

```sh
for f in /module_driver/*.sh; do
  echo "===== $f ====="
  sed -n '1,200p' "$f"
done
```

Trecho crítico observado:

```sh
# /module_driver/driver_default_init_script.sh
sh gt9xx_touch.sh
...
sh tlsc6x.sh
```

E os dois scripts tinham perfis diferentes:

```sh
# /module_driver/gt9xx_touch.sh
insmod gt9xx_touch.ko \
 gtp_i2c_bus_num=4 \
 gtp_x_coords_max=480 \
 gtp_y_coords_max=400 \
 gtp_x_y_coords_exchange=1 \
 gtp_version=0x3 \
 gtp_reset_gpio=PA00 \
 gtp_irq_gpio=PA03
```

```sh
# /module_driver/tlsc6x.sh
insmod tlsc6x.ko \
 tlsc6x_i2c_bus_num=4 \
 tlsc6x_x_coords_max=480 \
 tlsc6x_y_coords_max=800 \
 tlsc6x_x_y_coords_exchange=0 \
 tlsc6x_reset_gpio=PA00 \
 tlsc6x_irq_gpio=PA03
```

### 3.3 Primeira hipótese correta

Fato observado:

- `gt9xx_touch` e `tlsc6x` eram carregados no boot
- ambos estavam apontando para o mesmo barramento I2C e os mesmos GPIOs de reset/IRQ

Conclusão técnica:

- o touch estava em conflito de stack
- o display K1 adaptado estava muito mais compatível com o caminho `gt9xx_touch`
- o `tlsc6x` precisava sair da inicialização

## 4. Protocolo de diagnóstico inicial

### 4.1 Inspeção dos módulos de touch carregados

Comando:

```sh
lsmod | grep -Ei 'goodix|gt9|tlsc|touch'
```

Em diferentes momentos apareceram:

```text
gt9xx_touch
tlsc6x
```

### 4.2 Identificação dos devices de input

Comando:

```sh
cat /proc/bus/input/devices
```

Saída relevante:

```text
N: Name="goodix-ts"
H: Handlers=kbd event0

N: Name="goodix-pen"
H: Handlers=event1
```

Conclusão técnica:

- o driver ativo do touch era Goodix
- o touchscreen era exposto como `goodix-ts`
- o principal device de captura ficou em `/dev/input/event0`

### 4.3 Inspeção de parâmetros expostos pelo kernel

Comandos usados:

```sh
for f in /sys/module/gt9xx_touch/parameters/*; do
  echo "$f=$(cat $f 2>/dev/null)"
done
```

```sh
for f in /sys/module/tlsc6x/parameters/*; do
  echo "$f=$(cat $f 2>/dev/null)"
done
```

Valores relevantes observados em um dos estados iniciais:

```text
/sys/module/gt9xx_touch/parameters/gtp_x_coords_max=480
/sys/module/gt9xx_touch/parameters/gtp_y_coords_max=400
/sys/module/gt9xx_touch/parameters/gtp_x_y_coords_exchange=1
/sys/module/gt9xx_touch/parameters/gtp_irq_gpio=PA03
```

```text
/sys/module/tlsc6x/parameters/tlsc6x_x_coords_max=480
/sys/module/tlsc6x/parameters/tlsc6x_y_coords_max=800
/sys/module/tlsc6x/parameters/tlsc6x_x_y_coords_exchange=0
/sys/module/tlsc6x/parameters/tlsc6x_irq_gpio=PA03
```

### 4.4 Confirmação do caminho de boot

O boot chain relevante foi verificado em:

```sh
sed -n '1,200p' /etc/init.d/S11module_driver_default
sed -n '1,260p' /module_driver/driver_default_init_script.sh
```

Conclusão técnica:

- `/etc/init.d/S11module_driver_default` chama o init de módulos
- o arquivo central de boot para esse caso é `/module_driver/driver_default_init_script.sh`
- qualquer correção persistente do touch precisava passar por esse arquivo e pelo script `gt9xx_touch.sh`

## 5. Descoberta importante: a UI estava em 480x800

Em uma reinicialização dos serviços gráficos, apareceu no log:

```text
The framebuffer device was opened successfully.
480x800, 32bpp
The framebuffer device was mapped to memory successfully.
```

Esse ponto foi decisivo.

Conclusão técnica:

- embora o painel físico adaptado tenha sido tratado como um painel K1 de `480x400`
- a stack do framebuffer/UI estava operando em espaço lógico `480x800`

Isso só ficou totalmente claro mais tarde, mas esse log já era o primeiro indício forte.

## 6. Teste manual do Goodix sem IRQ

### 6.1 Motivo do teste

Mesmo com `gt9xx_touch` presente, o touch ainda não funcionava corretamente. A próxima hipótese foi que o driver Goodix não estava inicializando bem no modo por IRQ e deveria ser forçado a cair em polling.

Comandos usados:

```sh
killall display-server 2>/dev/null
rmmod gt9xx_touch
insmod /module_driver/gt9xx_touch.ko \
  gtp_i2c_bus_num=4 \
  gtp_x_coords_min=0 \
  gtp_y_coords_min=0 \
  gtp_x_coords_max=480 \
  gtp_y_coords_max=400 \
  gtp_x_coords_flip=0 \
  gtp_y_coords_flip=0 \
  gtp_x_y_coords_exchange=1 \
  gtp_regulator_name=-1 \
  gtp_power_on_gpio=-1 \
  gtp_reset_gpio=PA00 \
  gtp_irq_gpio=-1 \
  gtp_max_touch_number=5
sleep 1
dmesg 2>/dev/null | grep -Ei 'GTP|Goodix|interrupt mode|polling mode|irq' | tail -n 80
/usr/bin/display-server >/dev/null 2>&1 &
```

### 6.2 Resultado do teste

Trechos críticos do `dmesg`:

```text
<<-GTP-ERROR->> Firmware error, no config sent!
<<-GTP-ERROR->> GTP init panel failed.
<<-GTP-ERROR->> Request IRQ failed!ERRNO:-22.
<<-GTP-INFO->> GTP works in polling mode.
```

Conclusão técnica:

- tirar o IRQ da equação era correto
- o driver conseguia operar em `polling mode`
- mas ainda faltava a configuração do painel Goodix
- o próprio driver deixou isso explícito: `no config sent`

## 7. Descoberta central: o driver precisava receber configuração via `/proc/gt9xx_config`

Foi feita inspeção do módulo e dos símbolos/mensagens do driver.

Comando usado em sessão anterior:

```sh
strings /module_driver/gt9xx_touch.ko | grep -Ei 'config|goodix|gt9xx' | head -n 300
```

Os logs e strings apontavam mensagens como:

```text
Driver send config.
Firmware error, no config sent!
```

Conclusão técnica:

- o `gt9xx_touch.ko` não estava plenamente inicializando o touch sozinho
- ele esperava receber uma configuração do painel
- o ponto de injeção exposto era `/proc/gt9xx_config`

Esse foi o ponto-chave de toda a adaptação do touch.

## 8. Primeira persistência funcional: criação do blob Goodix e remoção do `tlsc6x`

### 8.1 Backup inicial feito na sessão de adaptação

Comandos usados:

```sh
cp /module_driver/gt9xx_touch.sh /module_driver/gt9xx_touch.sh.bak
cp /module_driver/driver_default_init_script.sh /module_driver/driver_default_init_script.sh.bak
```

### 8.2 Novo script persistente do Goodix

Comando usado para reescrever o script:

```sh
cat > /module_driver/gt9xx_touch.sh <<'EOF'
#!/bin/sh

CFG=/module_driver/goodix_k1_480x400.bin

insmod gt9xx_touch.ko \
 gtp_i2c_bus_num=4 \
 gtp_x_coords_min=0 \
 gtp_y_coords_min=0 \
 gtp_x_coords_max=480 \
 gtp_y_coords_max=400 \
 gtp_x_coords_flip=0 \
 gtp_y_coords_flip=0 \
 gtp_x_y_coords_exchange=1 \
 gtp_regulator_name=-1 \
 gtp_power_on_gpio=-1 \
 gtp_reset_gpio=PA00 \
 gtp_irq_gpio=-1 \
 gtp_max_touch_number=5

if [ -f "$CFG" ] && [ -e /proc/gt9xx_config ]; then
    cat "$CFG" > /proc/gt9xx_config
fi
EOF
```

### 8.3 Geração do blob do touch

Foi criado um binário com `printf` para representar a config Goodix do painel:

```sh
printf '\x63\xE0\x01\x90\x01\x02\x0D\x00 ... \x6F\x01' > /module_driver/goodix_k1_480x400.bin
```

Observação:

- o blob foi gerado manualmente na sessão
- o nome escolhido foi `goodix_k1_480x400.bin`
- o começo do arquivo mostra os bytes `E0 01 90 01`, que correspondem a `480` e `400` em little-endian

### 8.4 Remoção do `tlsc6x` do boot

Comando usado:

```sh
sed '/sh tlsc6x.sh/d' /module_driver/driver_default_init_script.sh.bak > /module_driver/driver_default_init_script.sh
chmod +x /module_driver/gt9xx_touch.sh /module_driver/driver_default_init_script.sh
```

### 8.5 O que essa etapa tentou garantir

- só o `gt9xx_touch` deveria subir
- o driver deveria entrar em polling
- o blob Goodix deveria ser enviado no boot
- o `tlsc6x` deveria deixar de concorrer

Essa foi a primeira versão funcional do ajuste.

## 9. Problema posterior: após reboot, imagem persistia e touch voltava a falhar

Depois, já em `2026-04-18`, o sintoma reportado foi:

- a imagem continuava correta
- o touch parava novamente após reiniciar a impressora

A análise voltou ao boot e aos módulos.

## 10. Diagnóstico da não persistência do touch

### 10.1 Estado encontrado no boot quebrado

Comando usado:

```sh
lsmod
cat /proc/bus/input/devices
dmesg | grep -iE "touch|goodix|gt9|ft5|focal|ili|input"
```

Estado observado naquele momento:

- `gt9xx_touch` carregado
- `tlsc6x` também carregado
- `goodix-ts` presente em `event0`

Trechos de log relevantes:

```text
gt9xx_touch: unknown parameter 'gtp_version' ignored
create proc entry gt9xx_config success
input: goodix-ts as /devices/virtual/input/input0
input: goodix-pen as /devices/virtual/input/input1
```

Conclusão técnica:

- o driver Goodix estava subindo
- o problema não era “módulo ausente”
- a persistência quebrava porque a configuração correta não estava sendo reaplicada do jeito certo no boot

### 10.2 Evidência direta do problema de init

Na análise de `dmesg` também apareceram, no estado problemático:

```text
Firmware error, no config sent!
GTP init panel failed.
```

E foi observado que o contador de IRQ útil do Goodix não evoluía:

```sh
cat /proc/interrupts
```

Conclusão técnica:

- o Goodix subia, mas sem config válida
- o touch não ficava operacional
- o boot ainda trazia resíduos da configuração inadequada

## 11. Correção da persistência em 2026-04-18

### 11.1 Backup formal antes de alterar

Foi criado um diretório de backup dedicado:

```text
/usr/data/touch_fix_backups/20260418-091729/
```

Arquivos preservados:

```text
/usr/data/touch_fix_backups/20260418-091729/gt9xx_touch.sh
/usr/data/touch_fix_backups/20260418-091729/driver_default_init_script.sh
/usr/data/touch_fix_backups/20260418-091729/goodix_k1_480x400.bin
```

### 11.2 Diff aplicado no `gt9xx_touch.sh`

Diff relevante:

```diff
#!/bin/sh

CFG=/module_driver/goodix_k1_480x400.bin

 insmod gt9xx_touch.ko \
  gtp_i2c_bus_num=4 \
  ...
- gtp_version=0x3 \
  gtp_regulator_name=-1 \
  gtp_power_on_gpio=-1 \
  gtp_reset_gpio=PA00 \
- gtp_irq_gpio=PA03 \
+ gtp_irq_gpio=-1 \
  gtp_max_touch_number=5

+if [ -f "$CFG" ] && [ -e /proc/gt9xx_config ]; then
+    cat "$CFG" > /proc/gt9xx_config
+fi
```

O que isso corrige:

- remove um parâmetro ignorado pelo driver (`gtp_version`)
- força o stack para polling (`gtp_irq_gpio=-1`)
- garante reenvio da config no boot via `/proc/gt9xx_config`

### 11.3 Diff aplicado no `driver_default_init_script.sh`

```diff
-sh tlsc6x.sh
```

O que isso corrige:

- impede que o segundo driver de touch volte a subir no boot

### 11.4 Validação pós-reboot dessa fase

Foi validado que:

- `gt9xx_touch` subia sozinho
- `tlsc6x` não era carregado
- `goodix-ts` reaparecia corretamente
- o log mostrava `Driver send config.`

Mas o touch ainda não estava 100% calibrado na área correta.

## 12. Refinamento final: por que só o “robozinho” respondia

Após a correção da persistência, o touch ainda ficou em um estado parcial:

- apertar no ícone do robozinho no canto inferior esquerdo funcionava
- quase nenhum outro ponto da tela respondia

Isso indicava:

- o touch estava vivo
- o mapeamento de coordenadas estava errado
- o problema não era ausência de driver, e sim transformação de coordenadas

## 13. Captura dos eventos brutos do touch

Foi usado o listener de input:

```sh
/usr/bin/cmd_inputdev_listen /dev/input/event0
```

Os toques geraram coordenadas brutas em faixa incompatível com a expectativa inicial de `480x400`, por exemplo valores próximos de:

```text
(85,438)
(398,211)
(785,78)
(18,386)
(693,94)
```

Conclusão técnica:

- o Goodix não estava operando em um mapa lógico simples `480x400`
- o espaço de coordenadas observado parecia mais próximo de `800x480` ou de um swap/inversão sobre `480x800`

## 14. Prova decisiva: diferença entre “init value” e “real value” do Goodix

### 14.1 Leitura do blob salvo

Comando:

```sh
hexdump -Cv /module_driver/goodix_k1_480x400.bin
```

Trecho inicial relevante:

```text
63 e0 01 90 01 ...
```

Isso representa:

- `0x01E0 = 480`
- `0x0190 = 400`

### 14.2 Leitura do que o driver realmente aplicou

Comando:

```sh
cat /proc/gt9xx_config
```

O `proc` mostrou duas áreas:

- `==== GT9XX config init value ====`
- `==== GT9XX config real value ====`

O ponto crítico:

- o `init value` continuava refletindo `480x400`
- o `real value` aparecia como `480x800`

Conclusão técnica:

- o blob salvo tinha `480x400`
- mas o controlador Goodix terminava operando com `480x800`
- isso batia com o log antigo do framebuffer: `480x800, 32bpp`

Esse foi o ponto que resolveu a contradição entre “painel físico 480x400” e “touch em área errada”.

## 15. Conclusão final de arquitetura

### 15.1 O que é físico e o que é lógico

O usuário confirmou corretamente:

- o display físico adaptado é `4.3" 480x400`

Mas o stack lógico efetivamente observado mostrou:

- framebuffer/UI em `480x800`
- Goodix “real value” em `480x800`

Conclusão técnica final:

- o painel físico é um fato
- o mapeamento lógico do stack Creality/Goodix para esse conjunto adaptado estava sendo tratado em `480x800`
- o erro restante vinha da troca de eixos (`exchange`) e não do LCD

## 16. Ajuste final que deixou o touch 100% funcional

Depois da análise do `real value`, o script foi ajustado para o perfil que efetivamente bateu com a tela inteira:

```sh
#!/bin/sh

CFG=/module_driver/goodix_k1_480x400.bin

insmod gt9xx_touch.ko \
 gtp_i2c_bus_num=4 \
 gtp_x_coords_min=0 \
 gtp_y_coords_min=0 \
 gtp_x_coords_max=480 \
 gtp_y_coords_max=800 \
 gtp_x_coords_flip=0 \
 gtp_y_coords_flip=0 \
 gtp_x_y_coords_exchange=0 \
 gtp_regulator_name=-1 \
 gtp_power_on_gpio=-1 \
 gtp_reset_gpio=PA00 \
 gtp_irq_gpio=-1 \
 gtp_max_touch_number=5

if [ -f "$CFG" ] && [ -e /proc/gt9xx_config ]; then
    cat "$CFG" > /proc/gt9xx_config
fi
```

Pontos decisivos dessa versão:

- `gtp_y_coords_max=800`
- `gtp_x_y_coords_exchange=0`
- `gtp_irq_gpio=-1`
- reenvio do blob em `/proc/gt9xx_config`

Após esse ajuste, o usuário confirmou:

```text
Está 100% funcional.
```

## 17. Estado final conhecido

### 17.1 Script final do touch

Arquivo:

```text
/module_driver/gt9xx_touch.sh
```

Perfil final:

- Goodix em polling
- espaço lógico `480 x 800`
- sem troca de eixos
- reaplicação do blob no boot

### 17.2 Boot sem o driver concorrente

Arquivo:

```text
/module_driver/driver_default_init_script.sh
```

Estado final:

- `sh gt9xx_touch.sh` permanece
- `sh tlsc6x.sh` foi removido

### 17.3 Módulos esperados após boot

Comando de validação:

```sh
lsmod | grep -E "gt9xx_touch|tlsc6x"
```

Resultado esperado:

- `gt9xx_touch` presente
- `tlsc6x` ausente

### 17.4 Device de input esperado

Comando:

```sh
cat /proc/bus/input/devices
```

Resultado esperado:

```text
N: Name="goodix-ts"
H: Handlers=kbd event0
```

## 18. Tutorial resumido para repetir a adaptação do touch

### Passo 1: confirmar que o display já tem imagem

Se a imagem existe, não mexa no LCD. Concentre tudo no touch.

### Passo 2: identificar os dois stacks de touch

```sh
lsmod | grep -Ei 'goodix|gt9|tlsc|touch'
cat /proc/bus/input/devices
for f in /module_driver/*.sh; do echo "===== $f ====="; sed -n '1,160p' "$f"; done
```

Você quer verificar se:

- `gt9xx_touch.sh` existe
- `tlsc6x.sh` existe
- os dois estão sendo chamados no boot

### Passo 3: confirmar o init do boot

```sh
sed -n '1,200p' /etc/init.d/S11module_driver_default
sed -n '1,260p' /module_driver/driver_default_init_script.sh
```

### Passo 4: testar Goodix sem IRQ

```sh
killall display-server 2>/dev/null
rmmod gt9xx_touch
insmod /module_driver/gt9xx_touch.ko \
  gtp_i2c_bus_num=4 \
  gtp_x_coords_min=0 \
  gtp_y_coords_min=0 \
  gtp_x_coords_max=480 \
  gtp_y_coords_max=400 \
  gtp_x_coords_flip=0 \
  gtp_y_coords_flip=0 \
  gtp_x_y_coords_exchange=1 \
  gtp_regulator_name=-1 \
  gtp_power_on_gpio=-1 \
  gtp_reset_gpio=PA00 \
  gtp_irq_gpio=-1 \
  gtp_max_touch_number=5
dmesg | grep -Ei 'GTP|Goodix|interrupt mode|polling mode|irq' | tail -n 80
/usr/bin/display-server >/dev/null 2>&1 &
```

Se aparecer `GTP works in polling mode`, o caminho está correto.

### Passo 5: se aparecer `Firmware error, no config sent!`, aplicar blob Goodix

Reescreva `gt9xx_touch.sh` para:

- carregar `gt9xx_touch.ko`
- usar `gtp_irq_gpio=-1`
- escrever o blob em `/proc/gt9xx_config`

### Passo 6: remover o `tlsc6x` do boot

```sh
cp /module_driver/driver_default_init_script.sh /module_driver/driver_default_init_script.sh.bak
sed '/sh tlsc6x.sh/d' /module_driver/driver_default_init_script.sh.bak > /module_driver/driver_default_init_script.sh
chmod +x /module_driver/driver_default_init_script.sh
```

### Passo 7: validar o mapa real do Goodix

```sh
hexdump -Cv /module_driver/goodix_k1_480x400.bin
cat /proc/gt9xx_config
```

Se o `real value` mostrar `480x800`, ajuste o script para:

```sh
gtp_x_coords_max=480
gtp_y_coords_max=800
gtp_x_y_coords_exchange=0
```

### Passo 8: capturar toques crus se a área ativa estiver errada

```sh
/usr/bin/cmd_inputdev_listen /dev/input/event0
```

Toque:

- canto superior esquerdo
- centro
- canto inferior direito

Se os valores saírem em faixa incompatível com `480x400`, ajuste `exchange` e `y_max`.

### Passo 9: validar persistência

Após salvar os scripts:

```sh
reboot
```

E depois:

```sh
lsmod | grep -E "gt9xx_touch|tlsc6x"
cat /proc/bus/input/devices
dmesg | grep -Ei "gt9xx|goodix|touch|input"
cat /proc/gt9xx_config
```

Você quer ver:

- só `gt9xx_touch`
- `goodix-ts`
- reaplicação da config
- ausência de `tlsc6x`

## 19. Arquivos críticos do procedimento

Arquivos de boot e driver:

```text
/etc/init.d/S11module_driver_default
/module_driver/driver_default_init_script.sh
/module_driver/gt9xx_touch.sh
/module_driver/tlsc6x.sh
/module_driver/gt9xx_touch.ko
/module_driver/goodix_k1_480x400.bin
```

Arquivos de validação:

```text
/proc/bus/input/devices
/proc/gt9xx_config
/proc/interrupts
/sys/module/gt9xx_touch/parameters/*
/sys/module/tlsc6x/parameters/*
```

Backups gerados:

```text
/module_driver/gt9xx_touch.sh.bak
/module_driver/driver_default_init_script.sh.bak
/usr/data/touch_fix_backups/20260418-091729/
```

## 20. Conclusão final

O protocolo correto para esse caso não foi “adaptar o display” e sim “consertar a pilha do touch”.

O que efetivamente resolveu:

1. manter o display como estava
2. remover o `tlsc6x` do boot
3. usar apenas `gt9xx_touch`
4. forçar `gt9xx_touch` para polling com `gtp_irq_gpio=-1`
5. reaplicar o blob Goodix em `/proc/gt9xx_config`
6. aceitar que o espaço lógico final do touch/UI ficou em `480x800`
7. ajustar o mapeamento final para:

```text
gtp_x_coords_max=480
gtp_y_coords_max=800
gtp_x_y_coords_exchange=0
```

Esse foi o conjunto que deixou a tela com imagem correta e o touch 100% funcional.


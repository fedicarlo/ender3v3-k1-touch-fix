# Tutorial: adaptação do touch da tela K1 na Ender 3 V3

## 1. Escopo

Este documento é a versão operacional do procedimento, pensada para repetir a adaptação em outra Ender 3 V3.

Escopo exato:

- a tela já gera imagem
- o problema está só no touch
- não mexer no display
- corrigir apenas a pilha de touch

## 2. Pré-requisitos

- acesso SSH root à impressora
- tela K1 já conectada e com imagem funcional
- shell BusyBox da impressora acessível

Acesso:

```sh
ssh root@192.168.15.3
```

Senha:

```text
creality_ender3v3
```

## 3. Objetivo técnico

Ao final, o sistema deve ficar assim:

- apenas `gt9xx_touch` carregado
- `tlsc6x` removido do boot
- Goodix inicializado em polling
- blob Goodix reaplicado em `/proc/gt9xx_config`
- touch funcional em toda a área da tela

## 4. Diagnóstico inicial

### 4.1 Verificar módulos carregados

```sh
lsmod | grep -Ei 'goodix|gt9|tlsc|touch'
```

Se aparecerem os dois:

```text
gt9xx_touch
tlsc6x
```

isso indica conflito potencial de stack.

### 4.2 Verificar devices de input

```sh
cat /proc/bus/input/devices
```

Procure por:

```text
N: Name="goodix-ts"
H: Handlers=kbd event0
```

### 4.3 Verificar scripts do boot

```sh
sed -n '1,200p' /etc/init.d/S11module_driver_default
sed -n '1,260p' /module_driver/driver_default_init_script.sh
sed -n '1,120p' /module_driver/gt9xx_touch.sh
sed -n '1,120p' /module_driver/tlsc6x.sh
```

O cenário típico quebrado é:

- `driver_default_init_script.sh` chama `sh gt9xx_touch.sh`
- e também chama `sh tlsc6x.sh`

## 5. Fazer backup antes de alterar

Crie um diretório de backup com timestamp:

```sh
TS=$(date +%Y%m%d-%H%M%S)
mkdir -p /usr/data/touch_fix_backups/$TS
cp /module_driver/gt9xx_touch.sh /usr/data/touch_fix_backups/$TS/
cp /module_driver/driver_default_init_script.sh /usr/data/touch_fix_backups/$TS/
cp /module_driver/goodix_k1_480x400.bin /usr/data/touch_fix_backups/$TS/ 2>/dev/null
```

## 6. Reescrever o script do Goodix

Grave este conteúdo em `/module_driver/gt9xx_touch.sh`:

```sh
cat > /module_driver/gt9xx_touch.sh <<'EOF'
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
EOF
chmod +x /module_driver/gt9xx_touch.sh
```

## 7. Criar ou reaplicar o blob Goodix

Se o arquivo ainda não existir, gere-o:

```sh
printf '\x63\xE0\x01\x90\x01\x02\x0D\x00\x01\x08\x3C\x0F\x64\x46\x03\x05\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x89\x29\x08\x22\x24\xB2\x04\x00\x00\x01\xBA\x03\x1D\x00\x00\x00\x00\x00\x03\x00\x00\x00\x00\x00\x19\x44\x94\xC5\x02\x07\x00\x00\x04\x9B\x1B\x00\x83\x21\x00\x6E\x29\x00\x5E\x32\x00\x52\x3D\x00\x52\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x08\x0A\x0C\x0E\x10\x12\x14\x16\xFF\xFF\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x04\x05\x06\x08\x0A\x0C\x0E\x2A\x29\x28\x24\x22\x20\x1F\x1E\x1D\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x6F\x01' > /module_driver/goodix_k1_480x400.bin
```

Observação:

- o nome do blob permaneceu `goodix_k1_480x400.bin`
- mas o mapeamento funcional final do touch foi validado com `y_max=800` e `exchange=0`

## 8. Remover o `tlsc6x` do boot

```sh
cp /module_driver/driver_default_init_script.sh /module_driver/driver_default_init_script.sh.bak
sed '/^sh tlsc6x\.sh$/d' /module_driver/driver_default_init_script.sh.bak > /module_driver/driver_default_init_script.sh
chmod +x /module_driver/driver_default_init_script.sh
```

Confirme:

```sh
grep -n 'tlsc6x\|gt9xx' /module_driver/driver_default_init_script.sh
```

O resultado desejado é:

- `gt9xx_touch.sh` presente
- `tlsc6x.sh` ausente

## 9. Testar sem reboot primeiro

Pare a UI, recarregue o touch e suba a UI de novo:

```sh
killall display-server 2>/dev/null
rmmod tlsc6x 2>/dev/null
rmmod gt9xx_touch 2>/dev/null
cd /module_driver
sh ./gt9xx_touch.sh
sleep 1
dmesg | grep -Ei 'GTP|Goodix|polling mode|interrupt mode|config' | tail -n 80
/usr/bin/display-server >/dev/null 2>&1 &
```

Você quer ver algo como:

```text
Driver send config.
GTP works in polling mode.
```

## 10. Validar o estado do touch

### 10.1 Verificar módulos

```sh
lsmod | grep -E 'gt9xx_touch|tlsc6x'
```

Resultado esperado:

```text
gt9xx_touch
```

Sem `tlsc6x`.

### 10.2 Verificar device Goodix

```sh
cat /proc/bus/input/devices
```

Resultado esperado:

```text
N: Name="goodix-ts"
H: Handlers=kbd event0
```

### 10.3 Verificar config aplicada

```sh
cat /proc/gt9xx_config
```

O importante é que exista:

- `init value`
- `real value`

Se o `real value` refletir `480x800`, isso está compatível com o mapeamento funcional final.

## 11. Se a área tocável estiver errada

Use o listener bruto:

```sh
/usr/bin/cmd_inputdev_listen /dev/input/event0
```

Toque:

- canto superior esquerdo
- centro
- canto inferior direito

Se o touch responder só numa pequena área, o erro é quase sempre:

- `x_y_coords_exchange` errado
- `y_coords_max` errado

Para este caso específico, o perfil que funcionou foi:

```text
gtp_x_coords_max=480
gtp_y_coords_max=800
gtp_x_y_coords_exchange=0
```

## 12. Validar persistência com reboot

Depois que o touch funcionar em runtime:

```sh
reboot
```

Após a impressora voltar, valide:

```sh
lsmod | grep -E 'gt9xx_touch|tlsc6x'
cat /proc/bus/input/devices
dmesg | grep -Ei 'gt9xx|goodix|touch|input|config' | tail -n 80
cat /proc/gt9xx_config
```

Checklist final:

- `gt9xx_touch` carregado
- `tlsc6x` ausente
- `goodix-ts` presente
- `Driver send config.` visível no boot ou reload
- touch funcional na área inteira

## 13. Arquivos finais importantes

```text
/module_driver/gt9xx_touch.sh
/module_driver/driver_default_init_script.sh
/module_driver/goodix_k1_480x400.bin
/proc/gt9xx_config
/proc/bus/input/devices
```

## 14. Configuração final recomendada

### `/module_driver/gt9xx_touch.sh`

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

### `/module_driver/driver_default_init_script.sh`

Deve manter:

```sh
sh gt9xx_touch.sh
```

E não deve conter:

```sh
sh tlsc6x.sh
```

## 15. Resumo final

Para repetir a adaptação em outra Ender 3 V3:

1. não mexa no display se a imagem já existe
2. remova `tlsc6x` do boot
3. mantenha só `gt9xx_touch`
4. use `gtp_irq_gpio=-1`
5. envie o blob Goodix em `/proc/gt9xx_config`
6. use o perfil final:

```text
x_max=480
y_max=800
exchange=0
```

Esse foi o conjunto que deixou o touch totalmente funcional.


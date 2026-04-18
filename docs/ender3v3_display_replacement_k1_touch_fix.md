# Ender 3 V3 Display Replacement (K1 Display) + Touch Fix

## Problema

O display original da Ender 3 V3 parou de funcionar.

- Não há reposição fácil no mercado
- Impressora fica inutilizável sem interface

---

## Solução alternativa

Foi utilizado um display da linha Creality K1 como substituto.

### Resultado inicial

- Display ligou normalmente
- Imagem funcionou corretamente
- Interface carregou sem erro

### Problema encontrado

O touch não funcionava.

- Nenhum comando respondia
- Interface travada

---

## Causa

- Conflito entre drivers (`gt9xx_touch` e `tlsc6x`)
- Configuração Goodix não aplicada no boot
- Inicialização incompleta do driver

---

## Solução

- Remover `tlsc6x` do boot
- Usar apenas `gt9xx_touch`
- Forçar polling (`gtp_irq_gpio=-1`)
- Aplicar config via `/proc/gt9xx_config`

---

## Resultado final

- Touch funcionando 100%
- Sistema estável após reboot

---

## Passo a passo completo

👉 Veja o tutorial completo:  
docs/tutorial_completo.md

👉 Veja o relatório técnico:  
docs/relatorio_tecnico.md

# Ender 3 V3 K1 Display Touch Fix (Goodix gt9xx)

> Fix definitivo para o problema de touch ao usar display da K1 na Ender 3 V3  
> Diagnóstico completo + solução persistente no boot (sem gambiarra manual)

---

## Contexto

Este projeto documenta a adaptação de um display da linha Creality K1
em uma Ender 3 V3, com foco na correção do touch Goodix (gt9xx).

A solução envolve análise de drivers, conflito de módulos e
injeção manual de configuração via /proc.

---

## Problema

- Display funciona normalmente
- Touch não responde corretamente
- Após reboot, o touch para completamente

---

## Causa

- Conflito entre drivers (`gt9xx_touch` vs `tlsc6x`)
- Configuração Goodix não aplicada no boot
- Inicialização incompleta do driver

---

## Solução

- Remover `tlsc6x` do boot
- Usar apenas `gt9xx_touch`
- Forçar polling (`gtp_irq_gpio=-1`)
- Aplicar config via `/proc/gt9xx_config`

---

## Configuração Final

```sh
insmod gt9xx_touch.ko \
 gtp_i2c_bus_num=4 \
 gtp_x_coords_max=480 \
 gtp_y_coords_max=800 \
 gtp_x_y_coords_exchange=0 \
 gtp_irq_gpio=-1

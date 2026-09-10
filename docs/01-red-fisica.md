# 01 — Red física y cableado estructurado

Documenta la capa física del lab: cableado, patch panel, switch y el troubleshooting del enlace que negociaba a 10 Mbps en lugar de 1 Gbps.

---

## Objetivo

Montar el cableado estructurado del rack con conexiones Gigabit estables y confiables, base de todo lo que corre encima (Proxmox, OPNsense, VLANs).

---

## Componentes

- **Patch panel:** Cat6 keystone, montado en rack Tecmojo
- **Switch:** TP-Link TL-SG108E (V6.0) — 8 puertos Gigabit, administrable
- **Cableado:** ⚠️ COMPLETAR (categoría y longitud aproximada de los cables usados, ej. "patch cords Cat6 de X metros")

---

## 🔧 Troubleshooting: enlace negociando a 10 Mbps

### Síntoma

Uno de los enlaces del lab negociaba a **10 Mbps** en lugar de **1000 Mbps (1 Gbps)**, pese a usar cableado y equipo Gigabit.

> Contexto técnico: un enlace Ethernet Gigabit (1000BASE-T) requiere los **4 pares** del cable correctamente conectados. Si un par queda abierto, invertido o mal terminado, el enlace hace *fallback* automático a una velocidad inferior (100 o 10 Mbps), que usa menos pares. Por eso un cable "que da internet" puede estar a 10 Mbps sin avisar.

### Diagnóstico

⚠️ COMPLETAR con lo que hiciste exactamente. Estructura sugerida:

1. **Aislamiento del problema:** se hizo un *bypass* del patch panel (conexión directa equipo–switch) para determinar si el problema estaba en el patch panel o en el cable/equipo.
2. **Resultado del bypass:** ⚠️ COMPLETAR (¿al saltarte el patch panel el enlace subió a 1 Gbps? Eso confirmaría que el problema estaba en la terminación del keystone).

### Causa raíz

⚠️ COMPLETAR — ¿cuál resultó ser el problema exacto?
Ejemplos comunes: un keystone mal ponchado, un par abierto/cruzado en la terminación, un conector defectuoso.

### Solución aplicada

⚠️ COMPLETAR — ¿qué hiciste para corregirlo?
Ejemplos: reponchar el keystone siguiendo el código de colores T568B, reemplazar el módulo, recrimpar el conector.

### Verificación (evidencia)

El enlace quedó negociando correctamente a **1000M Full**, con **cero errores** de paquetes:

```
Port    Status   Link Status   TxGoodPkt   TxBadPkt   RxGoodPkt   RxBadPkt
port 1  Enable   1000M Full     730391       0          1578023      0
port 2  Enable   1000M Full     1988405      0          136111       0
```

> Evidencia obtenida desde **Monitoring → Port Statistics** en la interfaz del switch TL-SG108E. Los contadores `TxBadPkt` y `RxBadPkt` en `0` confirman que la capa física quedó limpia, no solo funcional.

---

## Lecciones aprendidas

- Un enlace que "funciona" no necesariamente está a la velocidad correcta: **siempre verificar la velocidad negociada**, no solo la conectividad.
- El *fallback* a 10/100 Mbps es señal casi segura de un problema físico en el cableado (par abierto o mala terminación), no de configuración.
- Técnica de diagnóstico clave: **aislar por bypass** — quitar componentes de la cadena uno a uno (patch panel → cable → equipo) para localizar el punto de falla.
- La herramienta **Cable Test** del switch y las estadísticas de puerto (`Bad Packets`) son aliadas para validar la capa física.

---

## Estado

✅ Capa física operativa a 1 Gbps, estable y verificada.

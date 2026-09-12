# 01 — Red física y cableado estructurado

Documenta la capa física del lab: cableado, patch panel, switch y el troubleshooting del enlace que negociaba a 10 Mbps en lugar de 1 Gbps.

---

## Objetivo

Montar el cableado estructurado del rack con conexiones Gigabit estables y confiables, base de todo lo que corre encima (Proxmox, OPNsense, VLANs).

---

## Componentes

- **Patch panel:** Cat6 keystone, montado en rack Tecmojo
- **Switch:** TP-Link TL-SG108E (V6.0) — 8 puertos Gigabit, administrable
- **Cableado:** Patch cords UTP Cat6 prefabricados de fábrica, 30 cm, marca Enson (100% cobre)

---

## 🔧 Troubleshooting: enlace negociando a 10 Mbps

### Síntoma

Uno de los enlaces del lab negociaba a **10 Mbps** en lugar de **1000 Mbps (1 Gbps)**, pese a usar cableado y equipo Gigabit.

> Contexto técnico: un enlace Ethernet Gigabit (1000BASE-T) requiere los **4 pares** del cable correctamente conectados. Si un par queda abierto, invertido o mal terminado, el enlace hace *fallback* automático a una velocidad inferior (100 o 10 Mbps), que usa menos pares. Por eso un cable "que da internet" puede estar a 10 Mbps sin avisar.

### Diagnóstico

1. **Aislamiento del problema:** se sospechó del cable patch cord en uso y se reemplazó por uno nuevo (prefabricado) conectado directamente equipo–switch, para descartar el patch panel, el switch y el equipo como causa.
2. **Resultado del cambio:** al sustituir el cable, el enlace subió de inmediato a **1000M Full**. Esto aisló el problema al cable en sí — no al patch panel, al switch ni al equipo.

### Causa raíz

Uno de los patch cords en uso tenía una **terminación defectuosa** (mal ponchado en los conectores RJ45), lo que provocaba que el enlace hiciera *fallback* a 10 Mbps.

### Solución aplicada

Se reemplazaron los cables sospechosos por **patch cords Cat6 prefabricados de fábrica** (Enson, 30 cm, 100% cobre), eliminando el riesgo de una mala terminación manual.

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
- Técnica de diagnóstico clave: **aislar por sustitución** — cambiar un componente sospechoso (en este caso, el cable) por uno nuevo para confirmar o descartar la causa.
- Preferir patch cords **prefabricados de fábrica** sobre cables ponchados a mano reduce el riesgo de terminaciones defectuosas.
- La herramienta **Cable Test** del switch y las estadísticas de puerto (`Bad Packets`) son aliadas para validar la capa física.

---

## Estado

✅ Capa física operativa a 1 Gbps, estable y verificada.

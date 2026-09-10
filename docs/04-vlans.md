# 04 — Segmentación por VLANs (diseño)

> **Estado: diseñado, pendiente de ejecutar.** Se documenta el plan *antes* de aplicarlo, como en cualquier cambio de red en producción: diseño → análisis de riesgo → plan de rollback → ejecución.

---

## Problema que resuelve

Actualmente el TL-SG108E es una **red plana**: la WAN de OPNsense (hacia el ISP) y la LAN del lab comparten el mismo dominio de broadcast. Consecuencias:

- Dos servidores DHCP compiten (router del ISP vs. OPNsense) — cualquier equipo conectado al switch puede caer en la red equivocada.
- WAN y LAN **no están realmente aisladas**; el firewall existe pero se puede "rodear" por el switch.

Las VLANs crean dos switches lógicos independientes dentro del mismo switch físico.

---

## Diseño

### Mapa de puertos actual

| Puerto | Conectado a |
|---|---|
| 1 | Router / ONT del ISP |
| 2 | ThinkCentre — NIC onboard (`vmbr0`, LAN de OPNsense) |
| 3 | Raspberry Pi 5 |
| 4 | ThinkCentre — adaptador USB (`vmbr1`, WAN de OPNsense) |
| 5–8 | Libres (laptop de administración, futuros equipos) |

### VLANs

| VLAN | Nombre | Puertos | Modo | PVID |
|---|---|---|---|---|
| **10** | WAN | 1, 4 | Untagged | 10 |
| **20** | LAN | 2, 3, 5, 6, 7, 8 | Untagged | 20 |

- **VLAN 10 (WAN):** el router del ISP y la WAN de OPNsense se ven entre sí. Nadie más.
- **VLAN 20 (LAN):** la LAN de OPNsense, la Pi y todo lo que se conecte al lab. Solo salen a internet **a través del firewall**.

### Concepto: untagged + PVID (802.1Q)

Para puertos con **equipos finales** (PC, Pi, NICs de OPNsense) —no otro switch— la regla es:

- El puerto es miembro **untagged** de su VLAN → lo que *sale* va sin etiqueta.
- El **PVID** del puerto = número de su VLAN → lo que *entra* sin etiqueta se asigna a esa VLAN.
- Ambos valores deben coincidir. Los puertos *tagged* (trunk) solo se usan para enlazar switches o para *router-on-a-stick*; no aplican aquí.

Ruta en el TL-SG108E: *VLAN → 802.1Q VLAN* (crear VLANs y membresías) → *VLAN → 802.1Q PVID Setting* (asignar PVID por puerto).

---

## ⚠️ Análisis de riesgo

### Riesgo principal: perder acceso al switch a media configuración

- La IP de gestión del switch (`192.168.100.250`) vive en la subred del **ISP**, es decir, del lado **WAN**.
- La PC desde la que se administra el switch (con la utilidad *Easy Smart*) llega a él por: *PC → router del ISP → **puerto 1***.
- El puerto 1 es justo el que pasa a la VLAN 10. Un error de orden o de PVID puede dejar el switch inalcanzable con la configuración a medias.

### Riesgo secundario: dejar a OPNsense sin WAN

Si los puertos 1 y 4 no quedan en la **misma** VLAN, el firewall pierde internet.

### Mitigaciones

1. **Ruta de acceso alterna verificada:** la laptop, conectada en un puerto de la LAN (5–8), **también alcanza la GUI del switch**. Es el plan B si se pierde la ruta por el puerto 1.
2. **Acceso a Proxmox garantizado:** la PC principal se conecta directo al router del ISP, sin pasar por el switch → las VLANs no la afectan → siempre se puede abrir la consola de OPNsense desde Proxmox.
3. **Orden de ejecución:** primero crear las VLANs y membresías, después los PVID, y verificar tras cada paso.
4. **Rollback:** *System → System Reset* del switch (vuelve a fábrica, red plana) o *Backup and Restore* si se guardó la configuración previa.
5. **Decisión deliberada:** no ejecutar al final de una sesión larga. Cambio de red = sesión dedicada, con tiempo.

---

## Efectos esperados tras aplicar

- La laptop y la Raspberry Pi pasan a la LAN de OPNsense: reciben IPs `10.10.10.x` por DHCP. Si la Pi tenía IP estática en `192.168.100.x`, hay que reconfigurarla.
- Desaparece el conflicto de DHCP: cualquier equipo en los puertos 2–8 recibe IP **solo** de OPNsense.
- La IP estática temporal de la laptop (`10.10.10.50`) deja de ser necesaria → volver a DHCP.

### Pendiente de decidir
Dónde debe vivir la IP de gestión del switch. Lo ideal es administrarlo desde la **LAN protegida**, no desde el lado WAN. Opciones: mover la IP de gestión a `10.10.10.x` (tras verificar cómo la enlaza el TL-SG108E a una VLAN concreta), o mantenerla en WAN y aceptar el trade-off documentado.

---

## Evolución futura

Cuando OPNsense esté estable con dos NICs, migrar a **router-on-a-stick**: una sola NIC con un puerto *tagged* (trunk) y subinterfaces VLAN en OPNsense. Permite agregar más VLANs (por ejemplo, una **VLAN aislada para la VM de Kali** / red team) sin más hardware.

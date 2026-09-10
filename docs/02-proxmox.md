# 02 — Proxmox VE: host de virtualización

Instalación y configuración del hipervisor sobre el Lenovo ThinkCentre M715q, incluyendo repositorios, red (bridges) y ampliación de RAM.

---

## Hardware del host

| Recurso | Detalle |
|---|---|
| Equipo | Lenovo ThinkCentre M715q (mini PC, AMD Ryzen) |
| RAM | 20 GB DDR4 SO-DIMM (4 GB + 16 GB) |
| Disco | Almacenamiento local (`local` para ISOs, `local-lvm` para discos de VM) |
| NIC onboard | `enx6c4b90bab2a2` (Gigabit) |
| NIC USB | Realtek RTL8153 (`enx00e04c680bc8`), USB 3.0 Gigabit |
| WiFi | `wlp2s0` — sin uso (respaldo de emergencia) |

---

## Instalación

- **Proxmox VE 9.2** (basado en Debian 13 *trixie*).
- Acceso a la interfaz web en `https://<ip-proxmox>:8006`.
- Se asignó al host una IP fija alta (`.240`) dentro de la subred del ISP para evitar choques con el pool DHCP del router.

### Post-instalación: repositorios

Proxmox trae por defecto los repositorios *enterprise* (de pago). Sin suscripción, `apt update` falla. Solución:

1. **Desactivar** los repos enterprise (`pve-enterprise` y `ceph`).
2. **Activar** el repo `pve-no-subscription`.

> **Lección (Proxmox 9):** los repos usan el formato *deb822* (`/etc/apt/sources.list.d/*.sources`) con `Suites: trixie`. Los archivos enterprise pueden **no traer** la línea `Enabled:`, así que un `sed` de sustitución no hace nada — hay que **agregar** `Enabled: false` en lugar de sustituirla.

---

## 🔧 Troubleshooting: conflicto de IP

### Síntoma
Comportamiento errático del acceso a Proxmox (respondía a ratos).

### Diagnóstico
Con el cable de Proxmox **desconectado**, la IP seguía respondiendo a `ping` → otro dispositivo tenía la misma dirección. Confirmado con `arp -a` (MAC distinta a la del ThinkCentre).

### Solución
Mover Proxmox a una IP alta fuera del pool DHCP del router.
*Pendiente:* crear una reservación DHCP en el router del ISP para blindarla.

### Lección
`ping` con el cable desconectado + `arp -a` es la prueba más rápida para detectar IPs duplicadas.

---

## Red: bridges de Proxmox

Cada NIC física se expone a las VMs a través de un **Linux Bridge**. Diseño de dos bridges para separar WAN y LAN del firewall:

| Bridge | Puerto físico | IP en el host | Rol |
|---|---|---|---|
| `vmbr0` | `enx6c4b90bab2a2` (onboard) | Sí — IP de gestión de Proxmox | **LAN** del lab |
| `vmbr1` | `enx00e04c680bc8` (USB RTL8153) | **Ninguna** | **WAN** hacia el ISP |

> `vmbr1` se creó **sin IP ni gateway** a propósito: es solo el "cable" hacia el ISP. Quien maneja esa conexión es OPNsense, no el host. Ruta en la GUI: *Node → System → Network → Create → Linux Bridge → Apply Configuration*.

Verificación del adaptador USB antes de usarlo:

```bash
lsusb      # → Realtek Semiconductor Corp. RTL8153 Gigabit Ethernet Adapter
ip link    # → enx00e04c680bc8 (soportado nativamente por el kernel, sin drivers extra)
```

> **Lección:** Proxmox 9 usa nombres `enx<MAC>` para **todas** las NICs, sean onboard o USB.

---

## Ampliación de RAM (8 GB → 20 GB)

- El equipo traía **2 × 4 GB**, no 1 × 8 GB como se asumió. Con solo **2 slots SO-DIMM**, meter el módulo de 16 GB implicó retirar uno de 4 GB.
- Resultado: **4 GB + 16 GB = 20 GB** en *dual-channel flex mode*.
- Verificación:

```bash
dmidecode -t memory | grep -E "Size|Speed|Configured|Locator" | grep -v "No Module"
```

```
Size: 16 GB   Bank Locator: P0 CHANNEL A   Speed: 3200 MT/s   Configured Memory Speed: 1333 MT/s
Size: 4 GB    Bank Locator: P0 CHANNEL B   Speed: 2666 MT/s   Configured Memory Speed: 1333 MT/s
```

> **Lecturas que engañan:** `dmidecode` reporta el reloj del bus (1333 MHz); al ser *Double Data Rate*, la velocidad efectiva es **2666 MT/s**. El módulo de 3200 baja a 2666 para emparejarse con el más lento y con el límite de la plataforma. Comportamiento esperado, no un error.
>
> **Lección:** DDR4 SO-DIMM es obligatorio en el M715q (DDR3 es incompatible física y eléctricamente). Máximo soportado: 32 GB (2 × 16).

---

## Almacenamiento de ISOs

Las imágenes de instalación se guardan en `/var/lib/vz/template/iso/` (storage `local`). Descarga directa desde la shell del host con `wget`, sin pasar por la PC.

> **Detalle de la shell web:** al pegar comandos aparecía `^[[200~` al inicio y bash fallaba con `command not found`. Es el *bracketed paste mode* de la terminal. Solución: `printf '\e[?2004l'` (o escribir los comandos a mano).

---

## Estado

✅ Host operativo: Proxmox VE 9.2, 20 GB RAM, dos bridges (`vmbr0` LAN / `vmbr1` WAN) listos para el firewall.

# 03 — OPNsense: firewall / router del lab

Despliegue de OPNsense como VM en Proxmox, con dos interfaces físicas dedicadas (WAN y LAN), configuración inicial y actualización.

---

## ¿Por qué un firewall virtualizado?

Se evaluó comprar un appliance físico y se descartó por ahora:

- La versión virtualizada **enseña más** (bridges, passthrough de NICs, VLANs sobre el hipervisor).
- El lab *es* Proxmox: si el host se apaga, el lab entero se apaga, así que un firewall externo no aporta resiliencia real en este escenario.
- El presupuesto rinde más en RAM y UPS.
- Migrar a hardware dedicado después es trivial: exportar/importar el backup XML de OPNsense.

---

## ¿Por qué dos NICs físicas?

Un firewall necesita **dos patas**: WAN (hacia el ISP) y LAN (hacia el lab). El ThinkCentre tiene una sola NIC onboard, así que se agregó un adaptador USB RTL8153 como segunda puerta.

Se eligió empezar con **dos NICs físicas** en lugar de *router-on-a-stick* (una NIC con VLANs) porque:
- Es conceptualmente más claro mientras se aprende (WAN y LAN son cables distintos).
- Si se rompe la configuración de la LAN, la gestión sigue viva en la otra interfaz.

---

## Imagen de instalación

```bash
cd /var/lib/vz/template/iso/
wget https://<mirror>/releases/26.7/OPNsense-26.7-dvd-amd64.iso.bz2
wget https://<mirror>/releases/26.7/OPNsense-26.7-checksums-amd64.sha256

# Verificación de integridad
sha256sum OPNsense-26.7-dvd-amd64.iso.bz2
grep dvd OPNsense-26.7-checksums-amd64.sha256   # → hashes idénticos ✅

bunzip2 OPNsense-26.7-dvd-amd64.iso.bz2          # → OPNsense-26.7-dvd-amd64.iso (2.0 GB)
```

> Se usó la imagen **`dvd`** (ISO), no la `vga` (`.img`, pensada para USB físico). El checksum se comparó contra el archivo oficial del proyecto antes de usar la imagen.

---

## Especificación de la VM (ID 100)

| Parámetro | Valor | Motivo |
|---|---|---|
| OS type | Other | OPNsense es FreeBSD, no Linux |
| BIOS / Machine | SeaBIOS / i440fx | Defaults, suficientes |
| CPU | 2 cores, tipo **host** | Expone instrucciones reales de la CPU (mejor cifrado → útil para VPN) |
| RAM | 2048 MB, **ballooning desactivado** | Un firewall necesita memoria fija y garantizada |
| Disco | 20 GB (ide0, local-lvm) | OPNsense usa poco disco |
| `net0` | VirtIO → `vmbr0` | **LAN** |
| `net1` | VirtIO → `vmbr1` | **WAN** |
| Firewall de Proxmox en las NICs | **Desactivado** | OPNsense *es* el firewall; no se apilan dos |

---

## Instalación

1. Arranque en modo live → login `installer` / `opnsense`.
2. Keymap por defecto (US).
3. **Install (UFS)** en lugar de ZFS: ZFS reserva RAM para caché y los snapshots ya los da Proxmox a nivel VM. Menos capas para el mismo resultado.
4. Disco destino: `ada0` (20 GB) — **no** `cd0` (el DVD).
5. El instalador advierte que con 2 GB de RAM el copiado del live image es más lento (recomienda 3 GB). Se procedió sin problema; la operación normal de OPNsense corre bien con 2 GB.
6. Cambio de contraseña de root antes de terminar.
7. **Halt** (no reboot) → retirar el ISO del CD/DVD de la VM en Proxmox (*"Do not use any media"*) → Start. Así se evita que vuelva a arrancar el instalador.

---

## Asignación de interfaces

Al primer arranque OPNsense asignó las interfaces correctamente por sí solo:

| Interfaz OPNsense | NIC de la VM | Bridge | Rol |
|---|---|---|---|
| `vtnet0` | `net0` | `vmbr0` | **LAN** |
| `vtnet1` | `net1` | `vmbr1` | **WAN** |

La WAN obtuvo IP por **DHCP del router del ISP** de inmediato — es decir, internet funcionando desde el primer boot. (También recibió IPv6 por DHCP6: el ISP entrega IPv6.)

---

## LAN: subred propia

La LAN se movió de la `192.168.1.0/24` por defecto a una **subred propia** para el lab, desde la consola (opción `2) Set interface IP address`):

| Parámetro | Valor |
|---|---|
| IP de OPNsense (gateway de la LAN) | `10.10.10.1/24` |
| Servidor DHCP | Activado |
| Pool DHCP | `10.10.10.100` – `10.10.10.200` |
| Web GUI | HTTPS (no se bajó a HTTP) |

Acceso a la GUI: `https://10.10.10.1`

---

## Wizard de configuración inicial

| Pantalla | Decisión | Motivo |
|---|---|---|
| General | Timezone `America/Mexico_City` | Logs y eventos en hora local |
| General | DNSSEC activado en Unbound | Valida que las respuestas DNS no fueron manipuladas |
| General | Idioma en inglés | Documentación y comunidad están en inglés |
| WAN | Type: DHCP | La IP la entrega el router del ISP |
| WAN | **Block RFC1918: DESACTIVADO** | Ver *doble NAT* abajo |
| WAN | Block bogon: activado | Bloquea rangos IP no asignados; no interfiere |
| Deployment | IPsec: desactivado | La VPN planeada es WireGuard |

### ⚠️ Doble NAT y `Block RFC1918`

El firewall está **detrás del router del ISP** (doble NAT): su WAN recibe una IP **privada** (`192.168.100.x`) y su gateway es privado (`192.168.100.1`).

La opción *Block RFC1918 Private Networks* bloquea tráfico de rangos privados que llegue por la WAN. Con ella activa, OPNsense **bloquearía su propio gateway** y se quedaría sin internet. La propia GUI lo aclara: *"should only be set for WAN interfaces that use the public IP address space"*.

→ Desactivarla es **obligatorio** en un lab detrás de un router doméstico.

---

## Actualización

Tras el wizard, *System → Firmware → Updates*: 69 paquetes, 267 MiB, con reinicio. Resultado: **OPNsense 26.7.3** (FreeBSD 15.1).

> **Lección:** se olvidó tomar un **snapshot de la VM en Proxmox antes de actualizar**. Salió bien, pero la práctica correcta es: *VM → Snapshots → Take Snapshot* antes de cualquier cambio mayor. Regla desde ahora.

---

## 🔧 Troubleshooting: la laptop no recibía IP de OPNsense

### Síntoma
Al conectar la laptop al switch para entrar a la GUI, recibía una IP del **ISP** (`192.168.100.x`) en lugar de la LAN de OPNsense (`10.10.10.x`).

### Causa
El switch aún es una **red plana** (sin VLANs). Por él circulan dos servidores DHCP —el del router del ISP y el de OPNsense— y ganó el que respondió primero.

### Solución temporal
IP estática en la laptop (`10.10.10.50/24`, gateway y DNS `10.10.10.1`) para forzar la comunicación con OPNsense.

### Solución definitiva
Segmentar el switch con **VLANs** (ver `04-vlans.md`). Este incidente es la demostración práctica de *por qué* hacen falta.

---

## Estado

✅ Firewall operativo: OPNsense 26.7.3 enrutando internet a la LAN `10.10.10.0/24`, con DHCP, Unbound DNS y DNSSEC.
🔲 Pendiente: VLANs en el switch.

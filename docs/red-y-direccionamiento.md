# Red y direccionamiento

Referencia rápida de subredes, direcciones e interfaces del lab.

> Solo se documentan rangos **privados (RFC 1918)**. No se incluyen IPs públicas, credenciales ni datos del contrato con el ISP.

---

## Subredes

| Subred | Rol | Gateway | DHCP |
|---|---|---|---|
| `192.168.100.0/24` | Red del ISP (router/ONT). Lado **WAN** del lab | `192.168.100.1` (router del ISP) | Router del ISP |
| `10.10.10.0/24` | **LAN del lab** (detrás de OPNsense) | `10.10.10.1` (OPNsense) | OPNsense, pool `.100`–`.200` |

---

## Direcciones fijas

| Equipo | IP | Subred | Notas |
|---|---|---|---|
| Router / ONT del ISP | `192.168.100.1` | ISP | Gateway hacia internet |
| Proxmox (host) | `192.168.100.240` | ISP | GUI `:8006`. Fuera del pool DHCP del router |
| Switch TL-SG108E (gestión) | `192.168.100.250` | ISP | Estática. Ver nota en `04-vlans.md` sobre moverla a la LAN |
| OPNsense — WAN | DHCP (`192.168.100.x`) | ISP | Doble NAT (IP privada del router del ISP) |
| OPNsense — LAN | `10.10.10.1` | LAN | Gateway, DHCP y DNS (Unbound) del lab |
| Laptop de administración | `10.10.10.50` *(temporal)* | LAN | Estática mientras el switch es red plana; volver a DHCP tras las VLANs |

---

## Interfaces del host Proxmox (ThinkCentre M715q)

| Interfaz | Hardware | Bridge | Rol |
|---|---|---|---|
| `enx6c4b90bab2a2` (`nic0`) | NIC onboard Gigabit | `vmbr0` | LAN — gestión de Proxmox + LAN de OPNsense |
| `enx00e04c680bc8` | USB Realtek RTL8153 | `vmbr1` | WAN de OPNsense (sin IP en el host) |
| `wlp2s0` | WiFi interna | — | Sin uso; respaldo de emergencia |

---

## Interfaces de la VM OPNsense (VM 100)

| Interfaz | NIC de la VM | Bridge | Rol | Dirección |
|---|---|---|---|---|
| `vtnet0` | `net0` (VirtIO) | `vmbr0` | LAN | `10.10.10.1/24` |
| `vtnet1` | `net1` (VirtIO) | `vmbr1` | WAN | DHCP del ISP |

---

## Puertos del switch TL-SG108E

| Puerto | Conectado a | VLAN (diseño) |
|---|---|---|
| 1 | Router / ONT del ISP | 10 — WAN |
| 2 | ThinkCentre, NIC onboard (`vmbr0`) | 20 — LAN |
| 3 | Raspberry Pi 5 | 20 — LAN |
| 4 | ThinkCentre, USB RTL8153 (`vmbr1`) | 10 — WAN |
| 5–8 | Libres / laptop | 20 — LAN |

> Las VLANs aún no están aplicadas; el switch opera como red plana.

---

## Servicios de red

| Servicio | Dónde corre | Detalle |
|---|---|---|
| DHCP (LAN) | OPNsense | Pool `10.10.10.100`–`10.10.10.200` |
| DNS resolver | OPNsense (Unbound) | DNSSEC activado; registro automático de clientes DHCP |
| NAT | OPNsense | Salida de la LAN hacia el ISP (doble NAT con el router) |

---

## Fuera del lab

| Equipo | Conexión |
|---|---|
| PC principal / gaming | Cable **directo al router del ISP**. No pasa por el switch ni por OPNsense — aislada a propósito de cualquier experimento del lab. Desde aquí se administra Proxmox y el switch |

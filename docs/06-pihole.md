# 06 — Pi-hole: filtrado de DNS a nivel de red

Documenta la instalación de Pi-hole en la Raspberry Pi 5 y la configuración de OPNsense para que toda la LAN pase sus consultas DNS a través de él.

---

## Objetivo

Añadir una capa de filtrado de anuncios y trackers a nivel de red completa, sin depender de extensiones por dispositivo. Es el primer servicio "de valor" que corre sobre la infraestructura de red ya construida (Proxmox + OPNsense + VLANs).

---

## Arquitectura de la cadena DNS

```
Cliente LAN (VLAN 20)
       │  consulta DNS
       ▼
Dnsmasq / OPNsense (10.10.10.1)
       │  reenvía la consulta
       ▼
Pi-hole (10.10.10.173)
       │
       ├── Dominio en lista de bloqueo → responde 0.0.0.0
       └── Dominio legítimo → resuelve y regresa la respuesta
```

> Nota importante: como Dnsmasq **reenvía** las consultas en nombre del cliente, en el Query Log de Pi-hole el "Client" de todas las consultas aparece como `10.10.10.1` (la IP de OPNsense), no la IP del dispositivo original. Es el comportamiento esperado de esta arquitectura, no un error.

---

## Instalación

Pi-hole se instaló en la Raspberry Pi 5 (`10.10.10.173`, hostname `raspberrypi`) usando el instalador oficial (`curl -sSL https://install.pi-hole.net | bash`).

Configuración elegida durante el instalador:
- **Query logging:** activado (necesario para ver estadísticas y el Query Log)
- **Privacy mode:** 0 — Show everything (estadísticas completas)

Al finalizar, el instalador entrega:
- IP del servicio: `10.10.10.173`
- Panel de administración: `http://10.10.10.173/admin` (también accesible como `http://pi.hole/admin` dentro de la LAN)
- Contraseña temporal autogenerada — se reemplazó de inmediato con `sudo pihole setpassword`

---

## Configuración en OPNsense (DNS forwarding)

El objetivo es que toda la LAN siga usando Dnsmasq/OPNsense como su DNS (vía DHCP), pero que Dnsmasq reenvíe cada consulta a Pi-hole en vez de resolverla directamente.

1. **System → Settings → General → DNS servers**
   Se dejó un único DNS server: `10.10.10.173` (Pi-hole).
   Se verificó que **"Allow DNS server list to be overridden by DHCP/PPP on WAN"** estuviera desmarcada, para que el WAN de Totalplay no inyecte su propio DNS por encima.

2. **Services → Dnsmasq DNS & DHCP → General → DNS Query Forwarding**
   Se confirmó que **"Do not forward to system defined DNS servers"** estuviera desmarcada. Con esa casilla desmarcada, Dnsmasq reenvía automáticamente las consultas al DNS configurado en System → Settings → General (es decir, a Pi-hole), sin necesidad de escribir un upstream manual.

Con estos dos puntos, no fue necesario tocar nada más: el forwarding queda resuelto por la configuración por defecto de Dnsmasq una vez que el DNS del sistema apunta a Pi-hole.

---

## Verificación

Prueba realizada desde un cliente real de la LAN (laptop Windows en VLAN 20, no la propia Raspberry Pi):

```powershell
PS C:\WINDOWS\system32> nslookup google.com
Servidor:  Firewall
Address:  10.10.10.1

Respuesta no autoritativa:
Nombre:   google.com
Addresses:  ...

PS C:\WINDOWS\system32> nslookup doubleclick.net
Servidor:  Firewall
Address:  10.10.10.1

Nombre:   doubleclick.net
Addresses:  ::
            0.0.0.0
```

`doubleclick.net` (dominio de tracking conocido) resolvió a `0.0.0.0`, confirmando el bloqueo. `google.com` resolvió con normalidad.

Verificación cruzada en el **Query Log** de Pi-hole (`http://10.10.10.173/admin/queries`): ambas consultas aparecen registradas, con `doubleclick.net` marcado con el ícono de bloqueo (🚫 *Deny*) y el resto con el ícono de reenvío normal (☁️).

---

## Lecciones aprendidas

- Cuando el DNS se reenvía desde un router/firewall intermedio, el Query Log del servidor final (Pi-hole) muestra la IP del reenviador como "Client", no la del dispositivo original — hay que tenerlo en cuenta al leer los logs.
- No fue necesario configurar manualmente un "upstream DNS" en Dnsmasq: basta con dejar vacío ese campo y apuntar el DNS del sistema (System → Settings → General) a Pi-hole.
- Usar un dominio de tracking real y conocido (`doubleclick.net`) como prueba de bloqueo es más confiable que inventar uno — un typo o dominio inexistente puede dar un falso negativo (parece que "no bloqueó" cuando en realidad nunca estuvo en las listas).

---

## Estado

✅ Pi-hole instalado y operativo. Todo el tráfico DNS de la VLAN 20 pasa por Pi-hole vía Dnsmasq/OPNsense, con bloqueo de anuncios/trackers confirmado con pruebas reales.

**Pendiente:** Ansible en la misma Raspberry Pi (ver roadmap general del repo).

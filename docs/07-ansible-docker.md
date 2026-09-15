# 07 — Ansible: nodo de control y despliegue de Docker en la Raspberry Pi

Documenta la creación de un nodo de control de Ansible dedicado en Proxmox y el primer playbook de infraestructura como código del lab: instalar Docker Engine + Compose en la Raspberry Pi 5.

---

## Objetivo

Automatizar la configuración de los nodos del lab de forma **repetible, versionable y documentada**, en lugar de configurar a mano por SSH. Primer caso de uso: convertir la Raspberry Pi en un nodo multi-servicio con Docker.

---

## Decisión de arquitectura: ¿dónde vive el nodo de control?

| Opción                          | Veredicto | Razón                                                                                      |
| ------------------------------- | --------- | ------------------------------------------------------------------------------------------ |
| En la propia Raspberry Pi       | ❌         | El nodo de control debe estar separado de los nodos que administra                          |
| En la laptop (WSL)              | ❌         | Depende de que la laptop esté encendida; no forma parte de la infraestructura del lab       |
| **VM dedicada en Proxmox**      | ✅         | Siempre encendida, IP fija en la LAN, patrón real de producción (bastion / control node)   |

```
[ ansible-control (VM 101) ]  ──SSH con llave ed25519──▶  [ raspberrypi (10.10.10.173) ]
   10.10.10.10 · Debian 13                                    Docker se instala vía playbook
```

---

## 1. VM `ansible-control` en Proxmox

| Parámetro | Valor                                   |
| --------- | --------------------------------------- |
| VM ID     | 101                                     |
| SO        | Debian 13 (Trixie) netinst              |
| CPU / RAM | 1 vCPU / 1 GB                           |
| Disco     | 10 GB (local-lvm, SCSI)                 |
| Red       | `vmbr0` (VLAN 20 / LAN protegida)       |
| Software  | Sin escritorio — solo SSH server + utilidades estándar |
| Usuario   | `luis` con `sudo` (cuenta root bloqueada) |

La ISO se descargó directamente en Proxmox con **Download from URL** (storage `local` → ISO Images).

### IP fija vía reservación DHCP

Se creó una reservación en **OPNsense → Services → Dnsmasq DNS & DHCP → Hosts**: MAC `bc:24:11:2a:41:f9` → `10.10.10.10`, hostname `ansible-control`.

**Troubleshooting:** tras crear la reservación, la VM seguía recibiendo su lease dinámico anterior (`.162`) incluso después de reiniciar y de bajar/subir la interfaz. Causa: Dnsmasq conservaba en memoria los leases dinámicos viejos de esa MAC. Solución: **reiniciar el servicio Dnsmasq** en OPNsense (limpia los leases activos) y reiniciar la VM → tomó la `.10` de inmediato.

> Nota: Debian 13 minimal no incluye `dhclient`; para renovar el lease se usa `systemctl restart networking` o `ip link set <iface> down/up`.

---

## 2. Instalación de Ansible

```bash
sudo apt update
sudo apt install -y ansible
ansible --version   # ansible [core 2.19.11] · Ansible 12 · Python 3.13
```

---

## 3. Acceso SSH por llave hacia la Raspberry Pi

Ansible se conecta por SSH sin contraseña usando un par de llaves:

```bash
ssh-keygen -t ed25519 -C "ansible-control"   # sin passphrase (uso automatizado)
ssh-copy-id luis@10.10.10.173                # copia la llave pública a la Pi
ssh luis@10.10.10.173                        # debe entrar sin pedir contraseña
```

---

## 4. Estructura del proyecto Ansible

```
~/ansible-homelab/
├── inventory.ini        # qué máquinas administra Ansible
└── install-docker.yml   # qué hacer en ellas
```

Los archivos viven en este repo en [`ansible/`](https://github.com/luisperezglz/homelab/tree/main/ansible).

**Inventario** (`inventory.ini`): grupo `raspberry` con la Pi (`ansible_host=10.10.10.173`, `ansible_user=luis`).

Prueba de conectividad:

```bash
ansible -i inventory.ini raspberry -m ping
# raspberrypi | SUCCESS => { "ping": "pong" }
```

---

## 5. Playbook `install-docker.yml`

Instala Docker en la Pi siguiendo el método oficial (repo de Docker + llave GPG), en 7 tareas:

1. Dependencias (`ca-certificates`, `curl`)
2. Directorio `/etc/apt/keyrings`
3. Descarga de la llave GPG oficial de Docker
4. Alta del repositorio Docker — **`arch=arm64`** (la Pi 5 es ARM) y release tomado de `{{ ansible_distribution_release }}`
5. Instalación de `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`
6. Usuario `luis` añadido al grupo `docker`
7. Servicio `docker` iniciado y habilitado en el arranque

### Ejecución

```bash
ansible-playbook -i inventory.ini install-docker.yml --ask-become-pass
```

**Troubleshooting:** la primera ejecución falló con `Missing sudo password`. El playbook usa `become: true` y el usuario `luis` en la Pi requiere contraseña para `sudo`. Solución: pasar `--ask-become-pass` (`-K`), que pide la contraseña sudo una sola vez al inicio. Alternativa para automatización total (pendiente de evaluar): `sudo` sin contraseña para ese usuario vía `sudoers`.

Resultado:

```
PLAY RECAP
raspberrypi : ok=8  changed=4  unreachable=0  failed=0  skipped=0
```

---

## 6. Verificación en la Raspberry Pi

```bash
docker --version          # Docker version 29.8.0
docker compose version    # Docker Compose version v5.5.1
sudo docker run hello-world
# Hello from Docker! ... (imagen arm64v8)
```

> El grupo `docker` aplica al usuario tras cerrar y reabrir la sesión SSH; a partir de ahí `docker` funciona sin `sudo`.

---

## Lecciones aprendidas

- Pegar bloques largos de YAML con `cat << EOF` por SSH puede corromper el archivo (líneas mezcladas). Para playbooks, usar `nano` y verificar con `cat` antes de ejecutar.
- Un lease DHCP dinámico activo puede "ganarle" a una reservación recién creada; reiniciar el servicio DHCP es la forma limpia de forzarla.
- `--check` (dry-run) es buena práctica, pero tareas que dependen de pasos anteriores (p. ej. instalar desde un repo que aún no existe) pueden fallar en simulación sin que el playbook esté mal.
- En hosts ARM hay que fijar explícitamente `arch=arm64` en el repositorio de Docker.

---

## Estado

✅ Nodo de control Ansible operativo (`10.10.10.10`).
✅ Docker Engine + Compose desplegados en la Raspberry Pi vía playbook, verificado con `hello-world`.

**Siguiente:** primeros servicios en contenedores sobre la Pi (Docker Compose) y ampliar el inventario de Ansible a más nodos del lab.

# 🏠 Home Lab — Redes, Virtualización y Ciberseguridad

Laboratorio personal montado en un rack, construido como **entorno de aprendizaje** y **proyecto de portafolio** rumbo a una carrera en **Cloud & DevSecOps**.

El objetivo no es solo "que funcione", sino documentar el **porqué** de cada decisión de arquitectura, la configuración aplicada y —sobre todo— el **troubleshooting real** que surgió en el camino. Cada fallo resuelto vale tanto como cada servicio levantado.

---

## 🎯 Objetivos del proyecto

- Construir una **red segmentada** de tipo empresarial en miniatura (firewall, VLANs, DNS, DHCP propios).
- Practicar de forma hands-on los temas de mi ruta de certificaciones: **CCNA → AWS SAA → Terraform Associate → CKA → AWS Security Specialty / CKS**.
- Usar el lab como plataforma para proyectos progresivos: automatización, servicios, observabilidad, seguridad ofensiva/defensiva y orquestación.
- Documentar todo como si fuera infraestructura de producción.

---

## 🧱 Arquitectura (estado objetivo)

```
                    Internet (Totalplay ISP)
                            │
                    ┌───────────────┐
                    │  ONT Huawei   │
                    │ HG8145X6-10   │
                    └───────┬───────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
        [ PC gaming ]              [ Switch TL-SG108E ]
       (fuera del lab)             VLAN 10 (WAN) │ VLAN 20 (LAN)
                                          │
                                  ┌───────┴────────┐
                                  │  OPNsense (VM) │  ← Firewall
                                  │   WAN ── LAN   │
                                  └───────┬────────┘
                                          │ LAN 10.10.10.0/24
                        ┌─────────────────┼─────────────────┐
                        │                 │                 │
                 [ Proxmox host ]   [ Raspberry Pi 5 ]  [ VMs futuras ]
                 (ThinkCentre)      (Pi-hole, Ansible)  (Docker, k8s…)
```

> La PC de gaming se conecta **directo al router**, fuera del lab, para que ningún experimento afecte la conexión de juego.

---

## 🖥️ Inventario de hardware

| Componente | Modelo | Rol |
|---|---|---|
| Host de virtualización | Lenovo ThinkCentre M715q | Proxmox VE 9.2 |
| RAM del host | 20 GB DDR4 (4 GB + 16 GB, dual-channel flex) | — |
| Nodo secundario | Raspberry Pi 5 | Pi-hole, Ansible (planeado) |
| Switch | TP-Link TL-SG108E (V6.0) | Switch administrable, VLANs |
| Patch panel | Cat6 keystone (rack Tecmojo) | Cableado estructurado |
| Adaptador de red | USB-Ethernet Realtek RTL8153 | Segunda NIC (WAN de OPNsense) |
| ONT / ISP | Huawei OptiXstar HG8145X6-10 (Totalplay) | Salida a internet |

---

## 🛠️ Stack tecnológico

- **Virtualización:** Proxmox VE 9.2
- **Firewall / Router:** OPNsense 26.7.3 (FreeBSD)
- **Servicios de red:** Unbound DNS (con DNSSEC), DHCP
- **Automatización (planeado):** Ansible, Terraform
- **Contenedores (planeado):** Docker, k3s (Kubernetes)
- **Observabilidad (planeado):** Grafana, Prometheus
- **Seguridad (planeado):** Suricata (IDS), WireGuard (VPN), Wazuh (SIEM)

---

## ✅ Estado actual

| Componente | Estado |
|---|---|
| Cableado físico a 1 Gbps | ✅ Operativo (troubleshooting documentado) |
| Proxmox VE 9.2 | ✅ Instalado y configurado |
| Switch TL-SG108E administrable | ✅ IP fija, firmware al día |
| OPNsense (firewall) | ✅ Instalado, actualizado y enrutando |
| Segmentación por VLANs | 🔲 Diseñada, pendiente de ejecutar |
| Raspberry Pi 5 (Pi-hole/Ansible) | 🔲 Planeado |

---

## 📚 Documentación

| Documento | Contenido |
|---|---|
| [docs/01-red-fisica.md](docs/01-red-fisica.md) | Cableado, patch panel y el troubleshooting del enlace a 10 Mbps |
| [docs/02-proxmox.md](docs/02-proxmox.md) | Instalación y configuración del host de virtualización |
| [docs/03-opnsense.md](docs/03-opnsense.md) | Despliegue del firewall: WAN/LAN, doble NAT, DNS |
| [docs/04-vlans.md](docs/04-vlans.md) | Diseño de segmentación por VLANs (pendiente de ejecutar) |
| [docs/red-y-direccionamiento.md](docs/red-y-direccionamiento.md) | Tabla de direccionamiento IP e interfaces |

---

## 🗺️ Roadmap por fases

1. **Automatización base** — Pi-hole + Ansible en la Raspberry Pi (`CCNA`)
2. **Servicios sobre Proxmox** — Docker, Portainer, reverse proxy
3. **Observabilidad** — Grafana + Prometheus
4. **Seguridad** — Suricata, WireGuard, Wazuh
5. **Kubernetes** — clúster k3s (`CKA`)
6. **Infra as Code** — Terraform + GitOps (`Terraform Associate`)

---

## 👤 Autor

**Luis Pérez** — Estudiante de Ingeniería en Redes Inteligentes y Ciberseguridad
Enfocado en Cloud & DevSecOps · CCNA en progreso

---

> 📌 Este repositorio se actualiza de forma incremental conforme el lab evoluciona. La documentación de troubleshooting se conserva a propósito: refleja el proceso real de diagnóstico, no solo el resultado final.

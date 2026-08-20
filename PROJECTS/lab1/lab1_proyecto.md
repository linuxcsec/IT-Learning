# 🧪 lab1 — Proyecto de Red y Sysadmin

> Proyecto de laboratorio personal para aprender despliegue de servicios de red,
> administración de sistemas y comprender la arquitectura interna de una red real.

---

## 📋 Descripción

El proyecto **lab1** consiste en un entorno virtualizado completo sobre VMware Workstation,
partiendo de un firewall OPNsense como núcleo central de la red, al que se conectan
servidores Linux y Windows, así como clientes de prueba.

El objetivo es simular una red corporativa real en local, progresando por fases desde
la infraestructura base hasta servicios avanzados de seguridad y monitorización.

---

## 🗺️ Topología de red

```
Internet
    │
    │ (NAT - VMware)
    │
┌───▼────────────────────┐
│        OPNsense        │  ◄── Firewall / Router central
│  WAN : DHCP (NAT)      │
│  LAN : 192.168.10.1/24 │
└───────────┬────────────┘
            │
     LAN: 192.168.10.0/24
            │
    ┌───────┼────────────┐
    │       │            │
┌───▼──┐ ┌──▼──────┐ ┌──▼──────┐
│Ubuntu│ │ WinSrv  │ │Clientes │
│Server│ │  2022   │ │Win/Linux│
└──────┘ └─────────┘ └─────────┘
```

---

## 🖥️ Máquinas Virtuales

| VM | SO | RAM | Disco | IP | Rol |
|---|---|---|---|---|---|
| OPNsense | FreeBSD (OPNsense) | 2 GB | 20 GB | 192.168.10.1 | Firewall / Router |
| Ubuntu Server | Ubuntu Server 24.04 | 2 GB | 30 GB | 192.168.10.2 | Servidor Linux |
| Windows Server | Windows Server 2022 | 4 GB | 60 GB | 192.168.10.3 | AD / DNS / GPO |
| Cliente Windows | Windows 10/11 | 2–4 GB | 40 GB | DHCP | Endpoint |
| Cliente Linux *(opcional)* | Ubuntu Desktop / Kali | 1–2 GB | 15 GB | DHCP | Endpoint / pruebas |

---

## 🌐 Plan de direccionamiento

| Dirección / Rango | Uso |
|---|---|
| `192.168.10.0/24` | Red LAN principal |
| `192.168.10.1` | OPNsense — Gateway |
| `192.168.10.2` | Ubuntu Server — Estática |
| `192.168.10.3` | Windows Server — Estática |
| `192.168.10.100 – 192.168.10.200` | Pool DHCP — Clientes |

---

## ⚙️ Servicios planificados

### Red y seguridad perimetral (OPNsense)
- Firewall con reglas por zona
- NAT hacia Internet (WAN)
- DHCP Server para clientes LAN
- DNS Resolver (Unbound)
- IDS/IPS — Suricata
- VPN — WireGuard

### Servicios Linux (Ubuntu Server)
- Servidor web — Nginx
- Compartición de archivos — Samba
- Servidor NTP
- CA interna *(fase avanzada)*
- Syslog / métricas *(fase avanzada)*

### Directorio y políticas (Windows Server)
- Active Directory Domain Services (ADDS)
- DNS integrado con AD
- Group Policy Objects (GPO)
- Unión de clientes al dominio

---

## 🗓️ Fases del proyecto

| Fase | Descripción | Estado |
|---|---|---|
| **Fase 0** | Infraestructura VMware — redes virtuales | ⬜ Pendiente |
| **Fase 1** | OPNsense — instalación y configuración base | ⬜ Pendiente |
| **Fase 2** | Servicios de red core — DHCP y DNS | ⬜ Pendiente |
| **Fase 3** | Ubuntu Server — servicios Linux | ⬜ Pendiente |
| **Fase 4** | Windows Server — AD, DNS y GPO | ⬜ Pendiente |
| **Fase 5** | Clientes — unión al dominio y pruebas | ⬜ Pendiente |
| **Fase 6** | Seguridad — IDS, VPN y CA interna | ⬜ Pendiente |
| **Fase 7** | Monitorización y logging centralizado | ⬜ Pendiente |

---

## 🔧 Entorno de trabajo

- **Hipervisor:** VMware Workstation Pro
- **Host OS:** Windows
- **Red interna:** VMware Host-Only (adaptador dedicado para LAN lab1)
- **Red WAN simulada:** VMware NAT

---

## 📁 Estructura de notas (Obsidian)

```
NOTAS/
└── lab1/
    ├── README.md               ← este archivo
    ├── fase0_infraestructura.md
    ├── fase1_opnsense.md
    ├── fase2_dhcp_dns.md
    ├── fase3_ubuntu_server.md
    ├── fase4_windows_server.md
    ├── fase5_clientes.md
    ├── fase6_seguridad.md
    └── fase7_monitorizacion.md
```

---

## 📌 Notas

- Este lab parte desde cero, incluyendo OPNsense como firewall central desde el inicio.
- Se reemplaza el esquema anterior (sin firewall dedicado) por una topología más realista.
- Cada fase tendrá su propia nota en Obsidian con comandos, capturas y observaciones.

---

*Inicio del proyecto: Mayo 2026*

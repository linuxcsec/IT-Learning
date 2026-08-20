# Fase 0 — Infraestructura VMware

> Estado: ✅ Completada  
> Fecha: Mayo 2026

---

## Redes virtuales (VMware Virtual Network Editor)

| VMnet | Tipo | Subred | Uso |
|---|---|---|---|
| VMnet0 | NAT | 192.168.32.0/24 | WAN de OPNsense |
| VMnet1 | Host-Only | 10.10.10.0/24 | LAN interna del lab |

> ⚠️ El adaptador del host en VMnet1 tiene IP `10.10.10.1` asignada por DHCP de VMware.  
> Por ello la IP LAN de OPNsense se configuró como `10.10.10.1` y el host accede desde esa misma IP.  
> Las interfaces de OPNsense estaban invertidas inicialmente — se corrigió verificando MACs.

---

## VM OPNsense

| Parámetro | Valor |
|---|---|
| Nombre | `lab1-opnsense` |
| SO | OPNsense 26.1.6_2 (FreeBSD, amd64) |
| RAM | 2 GB |
| Disco | 20 GB |
| CPU | 1 vCPU |
| Network Adapter 1 | NAT (VMnet0) → WAN |
| Network Adapter 2 | Custom VMnet1 (Host-Only) → LAN |

---

## Interfaces OPNsense

| Interfaz | Adaptador | VMnet | IP | Rol |
|---|---|---|---|---|
| WAN | em0 | VMnet0 (NAT) | 192.168.126.128/24 (DHCP) | Salida a Internet |
| LAN | em1 | VMnet1 (Host-Only) | 10.10.10.1/24 | Red interna lab |

---

## Configuración wizard inicial

| Parámetro | Valor |
|---|---|
| Hostname | `opnsense` |
| Domain | `lab1.local` |
| DNS primario | `9.9.9.9` (Quad9) |
| DNS secundario | `149.112.112.112` (Quad9) |
| Timezone | `Atlantic/Canary` |
| WAN type | DHCP |
| Block RFC1918 | ✅ |
| Block bogon networks | ✅ |
| LAN IP | `10.10.10.1/24` |

---

## Acceso a la GUI

```
URL:      https://10.10.10.1
Usuario:  root
```

> Acceso desde el host Windows a través de VMnet1.

---

## Siguientes pasos → Fase 1

- Configurar DHCP server en OPNsense para clientes LAN
- Configurar DNS resolver (Unbound)
- Revisar reglas de firewall base

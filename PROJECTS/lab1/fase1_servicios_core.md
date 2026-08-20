# Fase 1 — Servicios de red core (OPNsense)

> Estado: ✅ Completada
> Fecha: Mayo 2026

---

## Objetivo

Configurar los servicios de red fundamentales en OPNsense:
DHCP (Kea), DNS Resolver (Unbound) y verificar conectividad end-to-end
con un cliente Linux (Ubuntu Server).

---

## 1. DHCP — Kea DHCPv4

> En OPNsense 26.x el DHCP moderno es **Kea**, sucesor del antiguo ISC DHCP.
> Solo gestiona DHCP (no DNS), es modular y escalable — estándar empresarial.

### Settings
| Campo | Valor |
|---|---|
| Enabled | ✅ |
| Interfaces | LAN |
| Valid lifetime | 4000 |
| Firewall rules | ✅ |

### Subnets
| Campo | Valor |
|---|---|
| Subnet | `10.10.10.0/24` |
| Description | LAN |
| Pools | `10.10.10.100 - 10.10.10.200` |

### Options (creadas en pestaña Options y enlazadas a la subnet)
| Description | Set Code | Set Data | Force |
|---|---|---|---|
| gateway-lan | routers [3] | `10.10.10.1` | ✅ |
| dns-lan | domain-name-servers [6] | `10.10.10.1` | ✅ |

---

## 2. DNS Resolver — Unbound

```
Services → Unbound DNS → General
```

| Campo | Valor | Motivo |
|---|---|---|
| Enable Unbound | ✅ | Activar servicio |
| Listen Port | 53 | Puerto DNS estándar |
| Network Interfaces | **LAN únicamente** | No exponer DNS a WAN (seguridad) |
| Enable DNSSEC | ✅ | Validación de respuestas |
| Outgoing Interfaces | WAN | Consultas al exterior salen por WAN |

> ⚠️ Nunca dejar Unbound escuchando en WAN → riesgo de DNS abierto a internet.

---

## 3. Verificación funcional (Ubuntu Server cliente)

```bash
ip a            # IP dentro del pool .100-.200
ip route        # default via 10.10.10.1
ping 9.9.9.9    # conectividad internet
ping google.com # resolución DNS
```

Resultado final correcto:
```
IP        → 10.10.10.129/24
Gateway   → default via 10.10.10.1
Internet  → OK (9.9.9.9)
DNS       → OK (google.com resuelve)
```

---

## 🔥 Troubleshooting (lecciones clave)

### Problema 1 — Cliente sin ruta por defecto
**Síntoma:** Ubuntu recibía IP y DNS pero no salida a internet. El gateway
llegaba como ruta de host (`10.10.10.1 dev ens33`) en vez de `default via`.

**Causa:** tras cambiar la IP LAN de OPNsense durante el setup inicial,
el servicio Kea quedó en estado inconsistente.

**Solución:** reinicio limpio del servicio.
```bash
/usr/local/etc/rc.d/kea restart
```

> 🔑 Lección: cambiar la IP de una interfaz NO basta — hay que reiniciar
> los servicios vinculados a esa IP (DHCP, DNS...).

---

### Problema 2 — IP fuera del pool (10.10.10.86)
**Síntoma:** el cliente recibía `.86`, fuera del pool `.100-.200`, sin gateway.

**Diagnóstico (con datos, no a ciegas):**
```bash
# Ver config real que ejecuta Kea (no la GUI)
cat /usr/local/etc/kea/kea-dhcp4.conf   # config CORRECTA → Kea no era el culpable

# Ver QUIÉN escucha en el puerto DHCP (67)
sockstat -4 -l | grep ':67'             # ¡Dnsmasq estaba activo!
```

**Causa raíz:** **Dnsmasq quedó habilitado** (al explorarlo antes) y competía
con Kea como segundo servidor DHCP en la misma LAN. Dnsmasq respondía más
rápido, entregaba el `.86` y sin la opción gateway de Kea.

**Solución:**
```
Services → Dnsmasq DNS & DHCP → General → Enable ☐ → Save → Apply
```
Después reiniciar Kea y renovar el lease en el cliente.

> 🔑 Lección: dos servidores DHCP en la misma red = conflicto.
> El más rápido gana. Verificar siempre el puerto 67 con sockstat.

---

## Comandos útiles aprendidos

| Comando | Uso |
|---|---|
| `sockstat -4 -l \| grep ':67'` | Ver qué proceso sirve DHCP |
| `cat /usr/local/etc/kea/kea-dhcp4.conf` | Config real de Kea |
| `/usr/local/etc/rc.d/kea restart` | Reiniciar Kea (FreeBSD) |
| `ls /usr/local/etc/rc.d/ \| grep <svc>` | Localizar nombre de servicio |
| `sudo systemctl restart systemd-networkd` | Renovar red en Ubuntu |
| `sudo ip addr flush dev ens33` | Limpiar IP de interfaz |

> Nota: OPNsense usa **tcsh**, no bash. `> archivo` falla con
> "invalid null command" — usar `echo -n "" > archivo` o `rm archivo`.

---

## Estado de servicios al cierre

| Servicio | Estado |
|---|---|
| Kea DHCPv4 | ✅ Running (único DHCP) |
| Unbound DNS | ✅ Running (LAN only) |
| Dnsmasq | ❌ Disabled (a propósito) |
| Firewall LAN→WAN | ✅ Regla por defecto activa |

---

## Siguientes pasos → Fase 2/3

- Completar instalación de Ubuntu Server
- Configurar IP estática o reserva DHCP para el servidor
- Servicios Linux: Nginx, Samba, NTP

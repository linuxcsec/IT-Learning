# Fase 5 — Clientes y verificación de dominio

> Estado: ✅ Completada
> Fecha: Junio 2026

---

## VM Cliente Windows 10

| Parámetro | Valor |
|---|---|
| Nombre VM | `lab1-client01` |
| SO | Windows 10 x64 |
| RAM | 2 GB |
| Disco | 40 GB |
| Network Adapter | VMnet1 (Host-Only) → LAN |
| Hostname | `lab1-client01` |
| IP | DHCP (pool 10.10.10.100-200) |

---

## Unión al dominio

### Prerrequisitos
```powershell
# Configurar DNS apuntando al DC (como administrador local)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses "10.10.10.3"

# Verificar resolución del dominio
Resolve-DnsName lab1.local
```

### Unión al dominio
```powershell
Add-Computer -DomainName "lab1.local" -Credential "LAB1\Administrator" -Restart
```

---

## Verificación de GPOs

### Comandos de diagnóstico
```powershell
gpupdate /force              # fuerza aplicación de GPOs
gpresult /r /user aura.lab1  # verifica GPOs aplicadas
```

### Resultado verificado con usuario aura.lab1

| GPO | Configuración | Resultado |
|---|---|---|
| GPO-Empleados-Restricciones | Sin Panel de Control | ✅ Bloqueado |
| GPO-Empleados-Restricciones | Sin CMD | ✅ Bloqueado |
| GPO-Empleados-RedDrive | Unidad Z: → \\10.10.10.2\compartido | ✅ Mapeada |

---

## Mapeo de unidad de red Samba

La GPO mapea la unidad Z: automáticamente al iniciar sesión. Al ser Samba
independiente de AD, solicita credenciales de Samba la primera vez:

```
Usuario:    aura-smb
Contraseña: (contraseña Samba)
☑ Remember my credentials
```

> 📌 Pendiente: integrar Samba con AD mediante **winbind** para SSO
> (Single Sign-On) — las credenciales del dominio se pasarían automáticamente
> sin pedir contraseña adicional.

### Mapeo manual (diagnóstico)
```powershell
net use Z: \\10.10.10.2\compartido /user:aura-smb "contraseña"
net use   # verificar unidades mapeadas
```

---

## IP estática en Ubuntu Server (fix definitivo)

Durante las fases 3-5 se detectó que Kea DHCP no aplicaba correctamente
la reserva de Ubuntu Server tras reinicios, entregando IPs del pool dinámico.

### Solución: IP estática via Netplan

`/etc/netplan/50-cloud-init.yaml`:
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 10.10.10.2/24
      routes:
        - to: default
          via: 10.10.10.1
      nameservers:
        addresses:
          - 10.10.10.1
          - 9.9.9.9
```

```bash
sudo netplan apply
```

> 🔑 Lección: los servidores con servicios críticos (Samba, NTP, Web)
> deben tener IP estática configurada en el propio sistema, no depender
> de reservas DHCP. El DHCP es para clientes, no para servidores.

---

## Estado final del lab (Fases 0-5)

```
10.10.10.1  →  OPNsense      →  Firewall, DHCP, DNS
10.10.10.2  →  lab1-srd      →  Nginx, Chrony, Samba (IP estática Netplan)
10.10.10.3  →  lab1-winsrv   →  AD DS, DNS, GPOs (IP estática PowerShell)
10.10.10.100-200  →  Pool DHCP clientes
10.10.10.133  →  lab1-client01  →  Windows 10, unido a lab1.local
```

---

## Siguientes pasos → Fase 6

- IDS/IPS con Suricata en OPNsense
- VPN WireGuard (acceso desde EndeavourOS)
- CA interna (firmar certificado Nginx)
- Reglas de firewall granulares

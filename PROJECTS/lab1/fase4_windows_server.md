# Fase 4 — Windows Server (lab1-winsrv)

> Estado: ✅ Completada
> Fecha: Junio 2026
> Servidor: lab1-winsrv (10.10.10.3)

---

## Especificaciones de la VM

| Parámetro | Valor |
|---|---|
| Nombre VM | `lab1-winsrv` |
| SO | Windows Server 2022 Standard (Desktop Experience) |
| RAM | 4 GB |
| Disco | 60 GB |
| CPU | 2 vCPU |
| Network Adapter | VMnet1 (Host-Only) → LAN |
| Hostname | `lab1-winsrv` |

---

## Direccionamiento (IP estática)

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.10.3 -PrefixLength 24 -DefaultGateway 10.10.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses "127.0.0.1","10.10.10.1"
```

| Parámetro | Valor |
|---|---|
| IP | `10.10.10.3/24` |
| Gateway | `10.10.10.1` |
| DNS primario | `127.0.0.1` (AD DNS local) |
| DNS secundario | `10.10.10.1` (Unbound/OPNsense) |

> ⚠️ Cuando se instala AD DS, Windows cambia el DNS a `127.0.0.1` automáticamente
> porque el propio DC es el servidor DNS del dominio. Se añade `10.10.10.1` como
> secundario para resolución hacia internet.

---

## Active Directory Domain Services (AD DS)

### Dominio
| Parámetro | Valor |
|---|---|
| Forest/Domain | `lab1.local` |
| NetBIOS | `LAB1` |
| Nivel funcional | Windows Server 2022 |

### Estructura de OUs

```
lab1.local
└── LAB1
    ├── Usuarios
    │   ├── Administradores
    │   └── Empleados
    ├── Equipos
    │   ├── Servidores
    │   └── Clientes
    └── Grupos
```

### Creación de OUs (PowerShell)

```powershell
New-ADOrganizationalUnit -Name "LAB1" -Path "DC=lab1,DC=local"
New-ADOrganizationalUnit -Name "Usuarios" -Path "OU=LAB1,DC=lab1,DC=local"
New-ADOrganizationalUnit -Name "Equipos" -Path "OU=LAB1,DC=lab1,DC=local"
New-ADOrganizationalUnit -Name "Grupos" -Path "OU=LAB1,DC=lab1,DC=local"
New-ADOrganizationalUnit -Name "Administradores" -Path "OU=Usuarios,OU=LAB1,DC=lab1,DC=local"
New-ADOrganizationalUnit -Name "Empleados" -Path "OU=Usuarios,OU=LAB1,DC=lab1,DC=local"
New-ADOrganizationalUnit -Name "Servidores" -Path "OU=Equipos,OU=LAB1,DC=lab1,DC=local"
New-ADOrganizationalUnit -Name "Clientes" -Path "OU=Equipos,OU=LAB1,DC=lab1,DC=local"
```

---

## Usuarios y Grupos

### Usuarios del dominio

| Usuario | UPN | OU | Estado |
|---|---|---|---|
| admin.lab1 | admin.lab1@lab1.local | Administradores | Enabled |
| aura.lab1 | aura.lab1@lab1.local | Empleados | Enabled |
| guest.lab1 | guest.lab1@lab1.local | Empleados | Enabled |

### Creación de usuarios (PowerShell)

```powershell
New-ADUser -Name "aura.lab1" `
  -SamAccountName "aura.lab1" `
  -UserPrincipalName "aura.lab1@lab1.local" `
  -Path "OU=Empleados,OU=Usuarios,OU=LAB1,DC=lab1,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Aura.Lab1#2026" -AsPlainText -Force) `
  -Enabled $true
```

> ⚠️ Si el usuario se crea con `Enabled: False` por complejidad de contraseña:
> ```powershell
> Set-ADAccountPassword -Identity "aura.lab1" -NewPassword (ConvertTo-SecureString "Aura.Lab1#2026" -AsPlainText -Force) -Reset
> Enable-ADAccount -Identity "aura.lab1"
> ```

### Grupos

```powershell
New-ADGroup -Name "GRP-Administradores" -GroupScope Global -GroupCategory Security -Path "OU=Grupos,OU=LAB1,DC=lab1,DC=local"
New-ADGroup -Name "GRP-Empleados" -GroupScope Global -GroupCategory Security -Path "OU=Grupos,OU=LAB1,DC=lab1,DC=local"

Add-ADGroupMember -Identity "GRP-Administradores" -Members "admin.lab1"
Add-ADGroupMember -Identity "GRP-Empleados" -Members "aura.lab1","guest.lab1"
```

---

## NTP — Sincronización contra lab1-srd

```powershell
w32tm /config /manualpeerlist:"10.10.10.2,0x8" /syncfromflags:manual /reliable:YES /update
Stop-Service "Windows Time"
Start-Service "Windows Time"
w32tm /resync /force
w32tm /query /status
```

Resultado esperado:
```
Source:       10.10.10.2
ReferenceId:  0x0A0A0A02
```

> ⚠️ Si Ubuntu Server (lab1-srd) no está arrancado, w32tm no puede sincronizar
> y devuelve "no time data was available". Arrancar Ubuntu primero.
> Lección: en producción, un servicio caído puede romper la autenticación AD
> silenciosamente (Kerberos tolera máx 5 minutos de desfase).

---

## DNS del dominio

### Configuración de registros A

```powershell
Add-DnsServerResourceRecordA -Name "lab1-winsrv" -ZoneName "lab1.local" -IPv4Address "10.10.10.3"
Add-DnsServerResourceRecordA -Name "lab1-srd" -ZoneName "lab1.local" -IPv4Address "10.10.10.2"
```

### Verificación

```powershell
Get-DnsServerResourceRecord -ZoneName "lab1.local" -RRType A
Resolve-DnsName lab1-winsrv.lab1.local
Resolve-DnsName lab1-srd.lab1.local
```

> ⚠️ Si aparecen registros duplicados con IPs incorrectas:
> ```powershell
> Remove-DnsServerResourceRecord -ZoneName "lab1.local" -RRType A -Name "lab1-winsrv" -RecordData "IP_incorrecta" -Force
> ```

### DNS Forwarding en OPNsense (Unbound)

```
Services → Unbound DNS → Overrides → Domain Overrides
Domain: lab1.local
IP:     10.10.10.3
```

Arquitectura DNS final:
```
Cliente
  │
  ▼
Unbound (10.10.10.1)
  ├── *.lab1.local  →  reenvía a AD DNS (10.10.10.3)
  └── resto         →  resuelve directamente (Quad9)
```

---

## GPOs

### Política de contraseñas (dominio completo)

```powershell
Set-ADDefaultDomainPasswordPolicy -Identity "lab1.local" `
  -MinPasswordLength 10 `
  -PasswordHistoryCount 5 `
  -MaxPasswordAge "90.00:00:00" `
  -MinPasswordAge "1.00:00:00" `
  -ComplexityEnabled $true `
  -LockoutThreshold 5 `
  -LockoutDuration "00:30:00" `
  -LockoutObservationWindow "00:30:00"
```

### GPO-Empleados-Restricciones (OU Empleados)

```powershell
New-GPO -Name "GPO-Empleados-Restricciones"
New-GPLink -Name "GPO-Empleados-Restricciones" -Target "OU=Empleados,OU=Usuarios,OU=LAB1,DC=lab1,DC=local"

# Sin Panel de Control
Set-GPRegistryValue -Name "GPO-Empleados-Restricciones" `
  -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
  -ValueName "NoControlPanel" -Type DWord -Value 1

# Sin CMD
Set-GPRegistryValue -Name "GPO-Empleados-Restricciones" `
  -Key "HKCU\Software\Policies\Microsoft\Windows\System" `
  -ValueName "DisableCMD" -Type DWord -Value 1
```

### GPO-Empleados-RedDrive (OU Empleados)

```powershell
New-GPO -Name "GPO-Empleados-RedDrive"
New-GPLink -Name "GPO-Empleados-RedDrive" -Target "OU=Empleados,OU=Usuarios,OU=LAB1,DC=lab1,DC=local"
```

Configurado via GPMC (`gpmc.msc`):
```
User Configuration → Preferences → Windows Settings → Drive Maps → New → Mapped Drive
  Action:   Create
  Location: \\10.10.10.2\compartido
  Label:    Compartido Lab1
  Drive:    Z:
  Reconnect: ✅
```

### Comandos útiles GPO

```powershell
gpupdate /force          # fuerza aplicación inmediata de GPOs
gpresult /r              # muestra GPOs aplicadas al usuario/equipo actual
gpresult /r /user aura.lab1  # GPOs aplicadas a un usuario específico
```

---

## Siguientes pasos → Fase 5

- Unión de cliente Windows 10 al dominio
- Verificación de GPOs en cliente
- Integración Samba con AD (winbind) → SSO

# Fase 3 — Servicios Linux (lab1-srd)

> Estado: ✅ Completada
> Fecha: Mayo 2026
> Servidor: lab1-srd (10.10.10.2)

---

## Servicios desplegados

| Servicio | Software | Función |
|---|---|---|
| Web | Nginx | Servidor web HTTPS |
| Tiempo | Chrony | Servidor NTP para la LAN |
| Archivos | Samba | Compartición SMB con usuarios/permisos |

---

## 1. Nginx — Servidor Web

### Instalación
```bash
sudo apt install nginx -y
sudo systemctl status nginx
```

### Página personalizada
```bash
sudo mkdir -p /var/www/lab1/html
sudo nano /var/www/lab1/html/index.html
sudo chown -R www-data:www-data /var/www/lab1
sudo chmod -R 755 /var/www/lab1
```

> Permisos 755: propietario (www-data) rwx, grupo y otros solo r-x.
> Directorios web → 755, archivos web → 644.

### Certificado autofirmado (TLS)
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/lab1.key \
  -out /etc/ssl/certs/lab1.crt
```
CN usado: `lab1-srd.lab1.local`

### Virtual Host con HTTPS + HSTS
`/etc/nginx/sites-available/lab1`:
```nginx
# Redirigir HTTP → HTTPS
server {
    listen 80;
    server_name lab1-srd lab1-srd.lab1.local 10.10.10.2;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl;
    server_name lab1-srd lab1-srd.lab1.local 10.10.10.2;

    ssl_certificate     /etc/ssl/certs/lab1.crt;
    ssl_certificate_key /etc/ssl/private/lab1.key;

    add_header Strict-Transport-Security "max-age=31536000" always;

    root /var/www/lab1/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Activación
```bash
sudo ln -s /etc/nginx/sites-available/lab1 /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

### Conceptos clave
- **301**: redirección permanente HTTP → HTTPS.
- **Puerto 80 abierto**: solo para redirigir, nunca sirve contenido. Mejora UX (el navegador prueba HTTP primero).
- **HSTS**: cabecera que fuerza al navegador a usar siempre HTTPS durante max-age (1 año). Protege contra SSL Stripping / MitM.
- **Certificado autofirmado**: cifra igual que uno real, pero el navegador avisa porque no hay CA de confianza detrás. Se resolverá con CA interna en Fase 6.

---

## 2. Chrony — Servidor NTP

### Instalación
```bash
sudo apt install chrony -y
```

### Configuración `/etc/chrony/chrony.conf`
```
pool ntp.ubuntu.com        iburst maxsources 4
pool 0.es.pool.ntp.org     iburst maxsources 2
allow 10.10.10.0/24
local stratum 10
```

### Verificación
```bash
sudo systemctl restart chrony
chronyc tracking    # Stratum 3, offset ~0.0002s, Leap Normal
chronyc clients     # clientes que sincronizan (vacío hasta añadir VMs)
```

### Conceptos clave — Jerarquía Stratum
```
Stratum 0 → fuente atómica/GPS
Stratum 1 → conectados directos a Stratum 0
Stratum 2 → pool.ntp.org
Stratum 3 → lab1-srd (chrony) ← aquí
Stratum 4 → resto de VMs del lab
```

- **Por qué es crítico NTP**: logs correlacionables, Kerberos (AD) tolera máx 5 min de desfase, validez de certificados TLS.
- **Por qué chrony**: cliente + servidor, converge rápido, ideal para VMs que se pausan/reanudan.

---

## 3. Samba — Servidor de archivos SMB

### Instalación
```bash
sudo apt install samba -y
sudo systemctl status smbd    # (NO samba-ad-dc, que es modo Active Directory)
```

### Usuarios y directorios
```bash
sudo useradd -M -s /sbin/nologin aura-smb
sudo useradd -M -s /sbin/nologin guest-smb

sudo mkdir -p /srv/samba/compartido
sudo mkdir -p /srv/samba/aura

sudo groupadd samba-users
sudo usermod -aG samba-users aura-smb
sudo usermod -aG samba-users guest-smb

# Contraseñas Samba (base de datos separada del sistema)
sudo smbpasswd -a aura-smb
sudo smbpasswd -a guest-smb
```

> Usuarios creados con `nologin`: pueden usar Samba pero NO iniciar sesión
> en el sistema. Buena práctica de seguridad (reduce superficie de ataque).

### Configuración `/etc/samba/smb.conf`
```ini
[global]
   workgroup = LAB1
   netbios name = SRD          # debe diferir del workgroup
   server string = lab1-srd Samba Server
   security = user
   map to guest = never
   log file = /var/log/samba/log.%m
   max log size = 50

[compartido]
   path = /srv/samba/compartido
   browseable = yes
   read only = no
   valid users = aura-smb, guest-smb
   write list = aura-smb
   create mask = 0664
   directory mask = 0775

[aura]
   path = /srv/samba/aura
   browseable = yes
   read only = no
   valid users = aura-smb
   create mask = 0660
   directory mask = 0770
```

### Verificación y arranque
```bash
testparm                       # Loaded services file OK
sudo systemctl restart smbd
```

### Acceso desde Windows
```
\\10.10.10.2
```
```powershell
# Limpiar credenciales cacheadas al cambiar de usuario
net use * /delete
```

### Matriz de acceso esperada
| Usuario | compartido | aura |
|---|---|---|
| aura-smb | lectura/escritura | lectura/escritura |
| guest-smb | solo lectura | sin acceso |

> ⚠️ PENDIENTE de ajuste fino: alinear permisos Linux (chmod/chown) con
> las reglas de Samba para que guest tenga solo lectura en "compartido"
> sin romper la escritura de aura. Se retomará más adelante.

---

## 🔑 SMB y Samba — conceptos en profundidad

### SMB (Server Message Block)
- Protocolo de Microsoft para compartir archivos/impresoras. Puerto **445/TCP** (el 139 era SMB sobre NetBIOS, obsoleto).
- Versiones: **SMB 1.0 = inseguro** (vector de WannaCry vía EternalBlue/MS17-010). Samba moderno usa SMB 3.x.

### Samba
- Implementación libre de SMB para Linux/Unix.
- Demonios:
  - `smbd` → sirve archivos e impresoras, autenticación.
  - `nmbd` → resolución de nombres NetBIOS (visibilidad por nombre).
  - `winbindd` → integración con dominios AD (Fase 4).
  - `samba-ad-dc` → modo controlador de dominio.

### Doble capa de permisos (clave)
```
Capa 1: Permisos Samba (smb.conf) → valid users, write list
Capa 2: Permisos Linux (chmod/chown) → rwx, propietario, grupo
→ El acceso requiere que AMBAS capas lo permitan.
→ Deben contar la MISMA historia o el comportamiento es inconsistente.
```

### Relevancia en ciberseguridad (enumeración)
```bash
enum4linux 10.10.10.2          # usuarios, recursos, políticas
smbclient -L //10.10.10.2      # listar recursos
smbmap -H 10.10.10.2           # mapear permisos
```
- Puerto 445 abierto en nmap → objetivo clásico de enumeración.
- Montar Samba desde el lado defensor da perspectiva para el lado ofensivo.

---

## Estado de servicios al cierre

| Servicio | Estado |
|---|---|
| Nginx (HTTPS + HSTS) | ✅ Running |
| Chrony (NTP server) | ✅ Running, Stratum 3 |
| smbd / nmbd | ✅ Running |

---

## Siguientes pasos → Fase 4

- Windows Server 2022
- Active Directory Domain Services (ADDS)
- DNS integrado con AD
- Sincronización NTP contra lab1-srd (crítico para Kerberos)

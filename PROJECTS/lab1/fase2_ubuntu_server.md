# Fase 2 — Ubuntu Server (lab1-srd)

> Estado: ✅ Completada
> Fecha: Mayo 2026

---

## Especificaciones de la VM

| Parámetro | Valor |
|---|---|
| Nombre VM | `lab1-srd` |
| SO | Ubuntu Server 24.04 LTS |
| RAM | 2 GB |
| Disco | 30 GB |
| CPU | 2 vCPU |
| Network Adapter | VMnet1 (Host-Only) → LAN |
| Hostname | `lab1-srd` |
| Usuario | `aura` |

---

## Direccionamiento

| Parámetro | Valor |
|---|---|
| IP | `10.10.10.2/24` |
| Gateway | `10.10.10.1` |
| DNS | `10.10.10.1` |
| Método | Reserva DHCP en Kea (por MAC) |

### Reserva Kea DHCPv4

| Campo | Valor |
|---|---|
| Subnet | `10.10.10.0/24` |
| IP address | `10.10.10.2` |
| MAC address | `00:0c:29:07:9a:6b` |
| Hostname | `lab1-srd` |
| Routers | `10.10.10.1` |
| DNS servers | `10.10.10.1` |
| Domain name | `lab1.local` |

---

## SSH

| Parámetro | Valor |
|---|---|
| Puerto | `2222` |
| Autenticación | Clave ed25519 (sin password) |
| Acceso desde host | `ssh -p 2222 aura@10.10.10.2` |

### Configuración del puerto (Ubuntu 24.04 usa socket activation)

> ⚠️ En Ubuntu 24.04, SSH usa **socket activation** de systemd.
> El puerto no se controla desde `sshd_config` sino desde el socket.

```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d
sudo nano /etc/systemd/system/ssh.socket.d/override.conf
```

Contenido del override:

```ini
[Socket]
ListenStream=
ListenStream=0.0.0.0:2222
ListenStream=[::]:2222
```

Aplicar cambios:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo ss -tlnp | grep 2222   # verificar que escucha en IPv4 + IPv6
```

> ⚠️ Si solo escucha en `[::]` (IPv6) y no en `0.0.0.0` (IPv4),
> la conexión desde el host será rechazada. Hay que especificar
> explícitamente `0.0.0.0:2222` en el override.

### Generación de claves en host Windows

```powershell
# Generar par de claves ed25519
ssh-keygen -t ed25519 -C "lab1-host"

# Copiar clave pública al servidor
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 aura@10.10.10.2 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### ssh-agent (opcional — evita escribir passphrase en cada conexión)

```powershell
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

---

## Verificación final

```bash
ip a | grep inet        # 10.10.10.2/24
ip route                # default via 10.10.10.1
ping -c 3 9.9.9.9       # internet OK
ping -c 3 google.com    # DNS OK
sudo ss -tlnp | grep 2222  # SSH escuchando en 0.0.0.0 y [::]
```

---

## Siguientes pasos → Fase 3

- Nginx — servidor web
- Samba — compartición de archivos
- NTP — servidor de tiempo

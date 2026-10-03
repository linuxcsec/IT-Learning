# Guía rápida de tshark

Guía de comandos de `tshark` (la versión de línea de comandos de Wireshark) para capturar y analizar tráfico en un Ubuntu Server. Pensada para uso académico en un laboratorio propio.

> **Aviso:** captura tráfico solo en tus propias máquinas o con autorización expresa.

---

## Índice

1. [Instalación y permisos](#1-instalación-y-permisos)
2. [Filtros de captura vs. filtros de visualización](#2-filtros-de-captura-vs-filtros-de-visualización)
3. [Identificar la interfaz](#3-identificar-la-interfaz)
4. [Capturar tráfico](#4-capturar-tráfico)
5. [Leer y filtrar una captura](#5-leer-y-filtrar-una-captura)
6. [Extraer campos concretos](#6-extraer-campos-concretos)
7. [Estadísticas](#7-estadísticas)
8. [Ejemplo práctico: DHCP (DORA)](#8-ejemplo-práctico-dhcp-dora)
9. [Otros ejemplos útiles](#9-otros-ejemplos-útiles)
10. [Ver la captura en Wireshark (GUI)](#10-ver-la-captura-en-wireshark-gui)
11. [Problemas habituales](#11-problemas-habituales)

---

## 1. Instalación y permisos

```bash
sudo apt update && sudo apt install tshark
```

Durante la instalación pregunta si los usuarios sin privilegios pueden capturar. Responde **Sí** y añade tu usuario al grupo `wireshark`:

```bash
sudo usermod -aG wireshark $USER
# Cierra sesión y vuelve a entrar para que se aplique
```

Si no apareció la pregunta o quieres cambiar la respuesta:

```bash
sudo dpkg-reconfigure wireshark-common
```

Si prefieres no tocar permisos, ejecuta `tshark` con `sudo`.

---

## 2. Filtros de captura vs. filtros de visualización

Son dos sintaxis **distintas**, y mezclarlas es el error más común.

| | Filtro de captura | Filtro de visualización |
|---|---|---|
| Opción | `-f` | `-Y` |
| Sintaxis | BPF | Wireshark |
| Cuándo actúa | Mientras captura (descarta el resto) | Al mostrar o leer (la captura guarda todo) |
| Ejemplo | `udp port 67 or udp port 68` | `dhcp` |
| Operadores | `and`, `or`, `not` | `&&`, `\|\|`, `!` |
| Ejemplo con IP | `host 192.168.1.10` | `ip.addr == 192.168.1.10` |

Si usas `-f dhcp` obtienes `Invalid capture filter "dhcp"`, porque `dhcp` es un filtro de visualización.

---

## 3. Identificar la interfaz

```bash
tshark -D          # Lista las interfaces que ve tshark
ip -br a           # Resumen de interfaces con sus IPs
```

Usa la interfaz que esté en la **misma red** que la máquina cuyo tráfico quieres ver. Los nombres (`enp0s3`, `ens33`, `eth0`...) dependen del sistema y del hipervisor: comprueba siempre el tuyo.

---

## 4. Capturar tráfico

```bash
# Capturar en pantalla (Ctrl+C para parar)
sudo tshark -i enp0s3

# Sin resolver nombres (más rápido y salida más limpia)
sudo tshark -i enp0s3 -n

# Guardar en un archivo .pcap
sudo tshark -i enp0s3 -w /tmp/captura.pcap

# Limitar por número de paquetes
sudo tshark -i enp0s3 -c 100 -w /tmp/captura.pcap

# Limitar por tiempo (segundos)
sudo tshark -i enp0s3 -a duration:60 -w /tmp/captura.pcap

# Buffer circular: ficheros de 10 MB, máximo 5
sudo tshark -i enp0s3 -b filesize:10240 -b files:5 -w /tmp/cap.pcap
```

### Filtros de captura (BPF) habituales

```bash
sudo tshark -i enp0s3 -f "host 192.168.1.20"                  # Un equipo
sudo tshark -i enp0s3 -f "src host 192.168.1.20"              # Solo origen
sudo tshark -i enp0s3 -f "net 192.168.1.0/24"                 # Una red
sudo tshark -i enp0s3 -f "tcp port 80"                        # Un puerto TCP
sudo tshark -i enp0s3 -f "udp port 53"                        # DNS
sudo tshark -i enp0s3 -f "icmp"                               # Ping
sudo tshark -i enp0s3 -f "arp"                                # ARP
sudo tshark -i enp0s3 -f "host 192.168.1.20 and tcp port 22"  # Combinado
sudo tshark -i enp0s3 -f "not port 22"                        # Excluir SSH
```

> `udp port 67 or 68` es incorrecto: el `68` queda sin protocolo. Escribe `udp port 67 or udp port 68`.

---

## 5. Leer y filtrar una captura

```bash
# Leer un archivo
tshark -r /tmp/captura.pcap

# Leer con filtro de visualización
tshark -r /tmp/captura.pcap -Y "http.request"
tshark -r /tmp/captura.pcap -Y "ip.addr == 192.168.1.20"
tshark -r /tmp/captura.pcap -Y "tcp.port == 22"
tshark -r /tmp/captura.pcap -Y "dns"
tshark -r /tmp/captura.pcap -Y "icmp"
tshark -r /tmp/captura.pcap -Y "tcp.flags.syn == 1 && tcp.flags.ack == 0"   # Solo SYN

# Detalle completo de los paquetes que coinciden
tshark -r /tmp/captura.pcap -Y "dns" -V

# Detalle solo de un protocolo concreto
tshark -r /tmp/captura.pcap -Y "dhcp" -O dhcp
```

Si el archivo lo creó `root` y no puedes leerlo, usa `sudo tshark -r ...` o cambia el propietario:

```bash
sudo chown $USER: /tmp/captura.pcap
```

---

## 6. Extraer campos concretos

Con `-T fields` y `-e` eliges qué columnas mostrar:

```bash
# Consultas DNS
tshark -r captura.pcap -Y "dns.flags.response == 0" -T fields \
  -e frame.time -e ip.src -e dns.qry.name

# Peticiones HTTP
tshark -r captura.pcap -Y "http.request" -T fields \
  -e ip.src -e http.host -e http.request.uri

# Puertos de cada paquete UDP
tshark -r captura.pcap -Y "udp" -T fields \
  -e frame.number -e ip.src -e udp.srcport -e ip.dst -e udp.dstport

# Cabecera y separador personalizados
tshark -r captura.pcap -T fields -E header=y -E separator=, \
  -e frame.number -e ip.src -e ip.dst > salida.csv
```

Para ver qué campos existen: `tshark -G fields | grep -i dhcp`.

---

## 7. Estadísticas

```bash
tshark -r captura.pcap -q -z io,phs          # Jerarquía de protocolos
tshark -r captura.pcap -q -z conv,tcp        # Conversaciones TCP
tshark -r captura.pcap -q -z conv,udp        # Conversaciones UDP
tshark -r captura.pcap -q -z conv,ip         # Conversaciones IP
tshark -r captura.pcap -q -z endpoints,ip    # Equipos que participan
tshark -r captura.pcap -q -z io,stat,1       # Paquetes por segundo
```

`-q` silencia la salida de paquetes y deja solo la estadística.

---

## 8. Ejemplo práctico: DHCP (DORA)

DHCP usa **UDP 67** (servidor) y **UDP 68** (cliente). El proceso completo son cuatro mensajes: **D**iscover, **O**ffer, **R**equest, **A**CK.

### Captura (en el servidor DHCP)

```bash
sudo tshark -i enp0s3 -f "udp port 67 or udp port 68" -w /tmp/dhcp.pcap
```

> Si el servidor es el propio servidor DHCP (Kea, dnsmasq, isc-dhcp-server), ve los cuatro mensajes sin depender del switch.

### Forzar el proceso desde el cliente

```
# Windows (CMD como administrador)
ipconfig /release && ipconfig /renew

# Linux
sudo dhclient -r enp0s3 && sudo dhclient -v enp0s3
```

Sin `release` previo, un `renew` con lease vigente puede enviar solo Request/ACK y no verás Discover ni Offer.

### Análisis

```bash
# Vista resumida
sudo tshark -r /tmp/dhcp.pcap

# Campos clave: tipo de mensaje, IP asignada y Transaction ID
tshark -r /tmp/dhcp.pcap -Y "dhcp" -T fields \
  -e frame.number -e ip.src -e ip.dst \
  -e dhcp.option.dhcp -e dhcp.ip.your -e dhcp.id

# Detalle de un tipo de mensaje concreto
tshark -r /tmp/dhcp.pcap -Y "dhcp.option.dhcp == 1" -V    # Discover
tshark -r /tmp/dhcp.pcap -Y "dhcp.option.dhcp == 2" -V    # Offer
tshark -r /tmp/dhcp.pcap -Y "dhcp.option.dhcp == 3" -V    # Request
tshark -r /tmp/dhcp.pcap -Y "dhcp.option.dhcp == 5" -V    # ACK
```

### Tipos de mensaje (`dhcp.option.dhcp`)

| Valor | Mensaje | Origen → Destino | Puertos |
|---|---|---|---|
| 1 | Discover | cliente (`0.0.0.0`) → broadcast | 68 → 67 |
| 2 | Offer | servidor → cliente | 67 → 68 |
| 3 | Request | cliente (`0.0.0.0`) → broadcast | 68 → 67 |
| 4 | Decline | cliente → servidor | 68 → 67 |
| 5 | ACK | servidor → cliente | 67 → 68 |
| 6 | NAK | servidor → cliente | 67 → 68 |
| 7 | Release | cliente → servidor | 68 → 67 |
| 8 | Inform | cliente → servidor | 68 → 67 |

### Qué comprobar

- Los cuatro mensajes del DORA comparten el mismo **Transaction ID** (`dhcp.id`). El Release lleva otro distinto, porque es un intercambio independiente.
- Request incluye la opción 50 (IP solicitada) y la 54 (identificador del servidor elegido).
- Offer y ACK llevan la configuración: máscara (1), router (3), DNS (6), tiempo de lease (51) y sufijo de dominio (15).

### Contrastar con los logs de Kea

```bash
sudo journalctl -u isc-kea-dhcp4-server -f
sudo cat /var/lib/kea/kea-leases4.csv
```

---

## 9. Otros ejemplos útiles

### ARP: quién pregunta por quién

```bash
sudo tshark -i enp0s3 -f "arp" -n
```

### Ping (ICMP echo request/reply)

```bash
sudo tshark -i enp0s3 -f "icmp" -n
```

### Handshake TCP (SYN, SYN/ACK, ACK)

```bash
sudo tshark -i enp0s3 -f "host 192.168.1.20 and tcp port 80" -w /tmp/tcp.pcap
tshark -r /tmp/tcp.pcap -T fields -e frame.number -e ip.src -e ip.dst -e tcp.flags.str
```

### DNS: consultas y respuestas

```bash
tshark -r captura.pcap -Y "dns" -T fields \
  -e frame.number -e ip.src -e dns.qry.name -e dns.a
```

### Seguir una conversación TCP

```bash
tshark -r captura.pcap -q -z follow,tcp,ascii,0
```

Con protocolos sin cifrar (HTTP, FTP, Telnet) se ven datos y credenciales en claro. Es un buen ejercicio para entender por qué se usan TLS y SSH.

### Extraer ficheros transferidos por HTTP

```bash
mkdir objetos
tshark -r captura.pcap --export-objects http,objetos
```

---

## 10. Ver la captura en Wireshark (GUI)

Si el servidor no tiene entorno gráfico, copia el `.pcap` a tu equipo:

```bash
scp usuario@ip-servidor:/tmp/captura.pcap .
```

O haz streaming en tiempo real por SSH hacia Wireshark:

```bash
ssh usuario@ip-servidor "sudo tshark -i enp0s3 -f 'host 192.168.1.20' -w -" | wireshark -k -i -
```

---

## 11. Problemas habituales

| Síntoma | Causa y solución |
|---|---|
| `Invalid capture filter "dhcp"` | Has usado sintaxis de visualización en `-f`. Usa `-f "udp port 67 or udp port 68"` o `-Y "dhcp"`. |
| `The file ... could not be opened: Permission denied` (al escribir) | Con `sudo`, `dumpcap` baja privilegios y no puede escribir en tu carpeta. Guarda en `/tmp` o añádete al grupo `wireshark` y captura sin `sudo`. |
| `You don't have permission to read the file` (al leer) | El archivo lo creó `root`. Usa `sudo tshark -r ...` o `sudo chown $USER: archivo.pcap`. |
| `Running as user "root"... This could be dangerous` | Aviso normal al usar `sudo`. |
| No aparece tráfico de otras máquinas | Interfaz equivocada, o el switch no reenvía tramas ajenas. El servidor solo ve broadcast, multicast y lo dirigido a él, salvo que actúe como gateway o uses port mirroring. |
| Solo veo Request y ACK en DHCP | El cliente renovó un lease vigente. Haz `ipconfig /release` (o `dhclient -r`) antes de renovar. |

---

## Referencias

- Manual de tshark: `man tshark`
- Filtros de visualización: https://wiki.wireshark.org/DisplayFilters
- Filtros de captura (BPF): https://wiki.wireshark.org/CaptureFilters

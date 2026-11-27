# VM Docker

Máquina virtual dedicada a ejecutar contenedores. Aloja los servicios de aplicación del laboratorio (Pi-hole, Uptime Kuma, Nextcloud, Homepage) y Portainer.

| | |
|---|---|
| **VM** | 100 — `VM-Docker` |
| **Sistema** | Debian 13 (*trixie*) |
| **IP** | `192.168.100.4/24` (estática) |
| **Recursos** | 2 vCPU · 8 GB RAM · disco de 50 GB |
| **Docker** | Docker Engine 29.3 · Docker Compose v5.1 |
| **Stacks** | `/opt/docker/<servicio>/docker-compose.yml` |

## Por qué una VM y no Docker en el host o en un LXC

- **No en el host:** el hipervisor queda limpio. Docker modifica reglas de red y requiere paquetes que no tienen que ver con Proxmox; separarlo evita interferencias y permite reinstalar Docker sin tocar el host.
- **No en un LXC:** dentro de una VM, Docker tiene su propio kernel y funciona sin ajustes. En un LXC comparte el kernel del host y puede chocar con sus restricciones, como pasó con Nginx Proxy Manager ([caso de overlay](troubleshooting/docker-overlay-en-lxc.md)).
- **Una sola unidad:** la VM se respalda, se restaura o se reemplaza completa, con todos sus contenedores y datos.

## Creación de la VM

Desde **Crear VM** en Proxmox, con una ISO de Debian guardada en `backup-hdd`:

| Pestaña | Parámetro | Valor | Motivo |
|---|---|---|---|
| SO | Tipo | Linux 6.x – 2.6 kernel (`l26`) | Le indica a Proxmox cómo optimizar la virtualización para un invitado Linux moderno |
| Sistema | BIOS / Máquina | SeaBIOS / i440fx | Valores simples y compatibles; Debian no necesita más |
| Sistema | Controlador SCSI | VirtIO SCSI single | Controlador paravirtualizado; permite un hilo de I/O por disco |
| Sistema | QEMU Agent | Activado | Proxmox puede consultar la IP de la VM y apagarla limpiamente |
| Disco | Almacenamiento / bus | `local-zfs` / SCSI | NVMe para que el sistema y los contenedores respondan rápido |
| Disco | Opciones | `discard`, `iothread`, `ssd` | `discard` permite que ZFS recupere el espacio que se borra dentro de la VM; `iothread` separa el I/O del disco; `ssd` le indica al invitado que es un disco sólido |
| CPU | Tipo / núcleos | `host` / 2 | `host` expone todas las funciones reales del procesador a la VM |
| Memoria | RAM | 8 GB | Margen para Nextcloud, MariaDB y el resto de los contenedores |
| Red | Puente | `vmbr0` (VirtIO) | La VM queda en la LAN como un equipo más |
| Opciones | Iniciar con el host | Sí | La VM arranca sola si se reinicia Proxmox |

Configuración resultante (`qm config 100`, extracto):

```
agent: 1
cores: 2
cpu: host
memory: 8192
net0: virtio=...,bridge=vmbr0,firewall=1
onboot: 1
ostype: l26
scsi0: local-zfs:vm-100-disk-0,discard=on,iothread=1,size=50G,ssd=1
scsihw: virtio-scsi-single
```

El disco se creó con 32 GB y luego se amplió a 50 GB. El instalador de Debian creó una partición de sistema de 47 GB y 2,6 GB de swap.

## Instalación del sistema

Debian 13 con la ISO *netinst*, instalación mínima con servidor SSH. Después de instalar se habilita el agente de QEMU:

```bash
apt install -y qemu-guest-agent
systemctl enable --now qemu-guest-agent
```

### Red

`/etc/network/interfaces`:

```
allow-hotplug ens18
iface ens18 inet static
    address 192.168.100.4/24
    gateway 192.168.100.1
```

La IP es fija porque esta VM aloja el DNS de la red (Pi-hole). Si su dirección cambiara, los clientes se quedarían sin resolución de nombres.

## Instalación de Docker

Docker se instala desde el **repositorio oficial**, no desde los paquetes de Debian, para tener versiones actuales y las correcciones de seguridad apenas se publican.

### 1. Requisitos y clave del repositorio

```bash
apt update
apt install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```

La clave permite que `apt` verifique que los paquetes vienen de Docker y no fueron alterados.

### 2. Repositorio

`/etc/apt/sources.list.d/docker.list`:

```
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable
```

### 3. Docker Engine y Compose

```bash
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

El servicio queda habilitado al instalarse, así que Docker y los contenedores con política de reinicio arrancan junto con la VM. No se usa ningún `/etc/docker/daemon.json`: Docker corre con su configuración por defecto.

### 4. Verificación

```bash
docker version --format 'Docker {{.Server.Version}}'   # Docker 29.3.0
docker compose version                                  # v5.1.0
docker run --rm hello-world
```

## Organización de los servicios

Cada servicio es un stack de Docker Compose con su propio directorio:

```
/opt/docker/
├── homepage/       docker-compose.yml + config/
├── nextcloud/      docker-compose.yml
├── pihole/         docker-compose.yml + etc-pihole/ + etc-dnsmasq.d/
└── uptime-kuma/    docker-compose.yml + uptime-kuma/
```

Junto al `docker-compose.yml` de cada servicio están sus datos persistentes, como volúmenes con rutas relativas. Así cada servicio se puede respaldar, mover o recrear sin afectar a los demás. Los datos de Nextcloud son la excepción: usan rutas absolutas en `/mnt/storage/appdata/`, que es una carpeta local de la VM, no el HDD del host.

Comandos de uso habitual, desde el directorio del servicio:

```bash
docker compose up -d        # crear o actualizar el stack
docker compose ps           # estado
docker compose logs -f      # registros
docker compose pull && docker compose up -d   # actualizar imágenes
```

### Redes

Cada stack tiene su propia red bridge (`<servicio>_default`), así que los contenedores de servicios distintos no se ven entre sí. Los puertos que necesitan acceso desde la LAN se publican en la IP de la VM:

| Puerto | Servicio |
|---|---|
| `53` TCP/UDP | Pi-hole (DNS) |
| `3000` | Homepage |
| `3001` | Uptime Kuma |
| `8081` | Pi-hole (web) |
| `8082` | Nextcloud |
| `9000` · `9443` | Portainer |

MariaDB (`3306`) no se publica: solo es accesible desde la red interna del stack de Nextcloud.

## Portainer

Interfaz web para administrar Docker: contenedores, imágenes, redes y volúmenes. Se instaló como apoyo visual al empezar con Docker. Hoy **no se usa** para administrar los servicios, que se gestionan con Compose desde la terminal.

A diferencia de los demás servicios, no tiene compose. Se lanzó con `docker run`. El comando equivalente, reconstruido a partir del contenedor (`docker inspect`), es:

```bash
docker run -d --name portainer --restart=always \
  -p 9000:9000 -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

- **`docker.sock`:** el socket de la API de Docker. Es lo que le permite a Portainer administrar los contenedores.
- **`portainer_data`:** volumen con nombre, gestionado por Docker. Es el único del laboratorio; los demás servicios usan carpetas junto a su compose.
- **Puertos:** `9443` sirve la interfaz por HTTPS, con un certificado autofirmado propio, y `9000`, por HTTP sin cifrar.

En la LAN se accede como `portainer.yocolab` a través de Nginx Proxy Manager, que se conecta al puerto `9443`.

En el primer acceso, Portainer pide crear el usuario administrador. Si no se completa dentro de unos minutos, la pantalla se bloquea por seguridad y hay que reiniciar el contenedor (`docker restart portainer`).

## Respaldos

En la instalación se configuró un trabajo de respaldo diario de la VM en Proxmox (21:00, modo *snapshot*, compresión ZSTD, hacia `backup-hdd`). **Ese trabajo ya no existe.** El único respaldo de la VM es uno manual del 17 de abril de 2026.

Como los datos de Nextcloud y de todos los stacks están dentro del disco de la VM, un respaldo de la VM los incluye a todos. Hoy no hay ninguno reciente.

## Verificación general

```bash
ip -br addr show ens18                 # 192.168.100.4/24
systemctl is-active docker qemu-guest-agent
docker compose ls                      # los 4 stacks en running
docker ps --format 'table {{.Names}}\t{{.Status}}'
df -h /                                # espacio del disco de la VM
```

## Limitaciones

- **Sin respaldos:** no hay trabajo de respaldo programado, y el único respaldo es de abril de 2026.
- **SSH permisivo:** acepta inicio de sesión como `root` con contraseña (`PermitRootLogin yes`, `PasswordAuthentication yes`). Es alcanzable desde la LAN y, por el subnet router de Tailscale, desde la tailnet.
- **Sin actualizaciones automáticas:** no está instalado `unattended-upgrades`, y el inventario mostró 14 paquetes pendientes.
- **DNS de la propia VM:** `/etc/resolv.conf` no tiene ningún `nameserver` (lo genera `dhcpcd`, aunque la interfaz es estática). Sin servidores definidos, el sistema consulta a `127.0.0.1`, donde responde el Pi-hole publicado por la propia VM. Si Pi-hole se detiene, la VM pierde la resolución de nombres, incluida la necesaria para descargar imágenes de Docker.
- **Portainer:** expuesto en HTTP sin cifrar (`9000`), con control total de Docker a través del socket, y sin uso.
- **Imágenes `latest`:** todos los servicios usan la etiqueta `latest`. Una actualización puede traer una versión mayor incompatible sin aviso; en Nextcloud, eso puede impedir que arranque.
- **Firewall de Proxmox:** la interfaz tiene `firewall=1`, pero el firewall está desactivado a nivel de Datacenter ([guía de Proxmox](proxmox-base.md#limitaciones)).

## Pendiente

- [ ] Trabajo de respaldo programado de la VM, con retención.
- [ ] Endurecer SSH: acceso con clave y `PermitRootLogin prohibit-password` o `no`.
- [ ] Instalar `unattended-upgrades` para las actualizaciones de seguridad.
- [ ] Definir `dns-nameservers` en `/etc/network/interfaces`.
- [ ] Decidir sobre Portainer: quitarlo, o pasarlo a compose y publicar solo el puerto `9443`.
- [ ] Fijar las versiones de las imágenes, empezando por Nextcloud.

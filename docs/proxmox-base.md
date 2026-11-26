# Proxmox VE — nodo base

Instalación y configuración del hipervisor sobre el que corre todo el laboratorio: red, almacenamiento ZFS, el HDD secundario y los almacenamientos de Proxmox.

| | |
|---|---|
| **Equipo** | Lenovo ThinkCentre M75q-1 |
| **Versión** | Proxmox VE 9.1 (Debian 13 *trixie*) · kernel 6.17 |
| **IP** | `192.168.100.2/24` en `vmbr0` |
| **Interfaz web** | `https://192.168.100.2:8006` |
| **Arranque** | UEFI, gestionado con `proxmox-boot-tool` |

## Hardware y discos

| Componente | Detalle |
|---|---|
| CPU | AMD Ryzen 3 3300GE, 4 núcleos / 8 hilos, AMD-V |
| RAM | 24 GB DDR4-2666 |
| NVMe 1 TB | Sistema, VM y LXC — pool ZFS `rpool` |
| HDD 2 TB | ext4 en `/mnt/storage` — ISOs, plantillas, respaldos y carpetas compartidas de Samba |

**Criterio de reparto:** el NVMe guarda lo que necesita baja latencia (el hipervisor y los discos de las VM y los LXC). El HDD guarda lo que ocupa mucho y se lee poco: imágenes de instalación, respaldos y archivos compartidos. Así el SSD no se llena con contenido secundario.

> Los datos de Nextcloud están en el NVMe, dentro del disco virtual de la VM Docker, no en el HDD.

## Red

### Etapa inicial: conexión compartida de Windows (ICS)

El laboratorio empezó sin acceso directo al router. Proxmox se conectó a un PC con Windows que compartía su conexión a Internet mediante **Internet Connection Sharing (ICS)**. ICS crea una red propia, `192.168.137.0/24`, con el PC Windows como puerta de enlace en `192.168.137.1`.

En esa etapa, la IP se asignó directamente a la interfaz física:

```
auto nic0
iface nic0 inet static
    address 192.168.137.69/24
    gateway 192.168.137.1
```

Se usó una IP fija porque la interfaz web (puerto 8006) y SSH dependen de que la dirección no cambie entre reinicios. Era una configuración **provisional**:

- Todo el laboratorio dependía de que el PC Windows estuviera encendido y compartiendo la conexión.
- La red `192.168.137.0/24` quedaba detrás de una NAT adicional, la de ICS, y los demás equipos de la casa no la alcanzaban directamente.
- Las VM y los LXC usan el puente `vmbr0`, y en esa etapa la IP estaba en `nic0`.

Esa etapa dejó rastros en la documentación de algunos servicios. Por ejemplo, el caso de Pi-hole "non-local network" muestra consultas desde `192.168.137.1`.

### Configuración actual

El laboratorio se trasladó a la red del router doméstico, `192.168.100.0/24`. La IP pasó al puente `vmbr0`, que usa `nic0` como puerto físico. `/etc/network/interfaces`:

```
auto lo
iface lo inet loopback

auto nic0
iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.100.2/24
    gateway 192.168.100.1
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0

source /etc/network/interfaces.d/*
```

- **`nic0` en `manual`:** la interfaz física no tiene IP propia; solo transporta el tráfico del puente.
- **`vmbr0`:** puente de Linux al que se conectan el host, las VM y los LXC. Todos quedan en la misma red que el router, como si fueran equipos físicos.
- **`bridge-stp off`, `bridge-fd 0`:** sin Spanning Tree ni retardo de reenvío. Con un solo puerto físico no hay bucles posibles, y la interfaz queda operativa de inmediato al arrancar.

#### Direcciones del laboratorio

| IP | Equipo |
|---|---|
| `.1` | Router |
| `.2` | Host Proxmox |
| `.3` | CT 101 — cloudflared |
| `.4` | VM 100 — Docker |
| `.5` | CT 102 — Nginx Proxy Manager |
| `.6` | CT 103 — Samba |
| `.7` | CT 104 — Tailscale |

Todas son estáticas, configuradas en cada equipo.

## Configuración inicial

### Repositorios

La primera ejecución de `apt update` devuelve **`401 Unauthorized`** en `enterprise.proxmox.com`. No es un problema de red: los repositorios *enterprise* solo funcionan con una suscripción activa. Los repositorios de Debian sí se descargan.

La corrección es deshabilitar los repositorios *enterprise* y habilitar el **no-subscription**, pensado para uso sin suscripción. Se hizo con el script post-instalación de la comunidad ([community-scripts: post-pve-install](https://community-scripts.org/scripts?id=post-pve-install)), que además aplica otros ajustes iniciales comunes.

Estado de los repositorios (`/etc/apt/sources.list.d/`):

| Repositorio | Componente | Estado |
|---|---|---|
| Debian `trixie`, `trixie-updates` | `main contrib non-free-firmware` | Habilitado |
| Debian `trixie-security` | `main contrib non-free-firmware` | Habilitado |
| Proxmox `pve` | `pve-no-subscription` | Habilitado |
| Proxmox `pve` | `pve-enterprise` | Deshabilitado |
| Proxmox `ceph-squid` | `enterprise` | Deshabilitado |
| Proxmox `pve` | `pve-test` | Deshabilitado |

Proxmox 9 usa el formato *deb822* (archivos `.sources`), donde cada repositorio se habilita o deshabilita con `Enabled:`.

### Actualización

```bash
apt update
apt full-upgrade
```

Se usa `full-upgrade` y no `upgrade` porque puede instalar o quitar dependencias cuando una actualización lo requiere. Es el método que recomienda Proxmox para actualizar el hipervisor.

## ZFS

El NVMe tiene un pool ZFS de un solo disco, `rpool`, creado por el instalador. ZFS combina gestor de volúmenes y sistema de archivos, y aporta:

- **Checksums en cada bloque:** detecta corrupción silenciosa. Con un solo disco puede detectarla, pero no repararla.
- **Copy-on-Write:** nunca sobrescribe un bloque en su lugar. Eso evita la corrupción por cortes durante una escritura y permite snapshots instantáneos.
- **Compresión transparente** y caché en RAM (**ARC**).

Propiedades aplicadas sobre `rpool` (los datasets hijos las heredan):

| Propiedad | Comando | Motivo |
|---|---|---|
| `compression=lz4` | `zfs set compression=lz4 rpool` | Compresión muy rápida; al escribir menos datos, muchas veces mejora el rendimiento. Solo afecta a lo escrito después del cambio. |
| `atime=off` | `zfs set atime=off rpool` | Evita escribir la fecha de acceso en cada lectura. Menos I/O y menos desgaste del SSD. |
| `xattr=sa` | `zfs set xattr=sa rpool` | Guarda los atributos extendidos (ACL, metadatos de contenedores) junto al archivo, sin lecturas adicionales. |
| `acltype=posix` | `zfs set acltype=posix rpool` | ACL POSIX, el modelo que esperan Linux y los contenedores LXC. |
| `autotrim=on` | `zpool set autotrim=on rpool` | Avisa al SSD qué bloques quedaron libres, para mantener el rendimiento de escritura en el tiempo. |

Verificación:

```bash
zfs get -H -o property,value compression,atime,xattr,acltype rpool
zpool get -H -o property,value autotrim rpool
zpool status rpool     # estado del pool y resultado del último scrub
```

## HDD como almacenamiento secundario

### 1. Identificar el disco

```bash
lsblk
```

El HDD aparece como `sda` (1,8 TB) y el NVMe como `nvme0n1`. Hay que confirmarlo antes de seguir: los pasos siguientes borran el disco elegido. El HDD traía particiones de un uso anterior.

### 2. Particionar y formatear

```bash
fdisk /dev/sda
#   g  → nueva tabla GPT
#   n  → nueva partición (valores por defecto: todo el disco)
#   w  → guardar y salir
mkfs.ext4 /dev/sda1
```

Una sola partición, porque el disco tiene un uso general. Se usa **ext4** porque es estable y simple, y este disco no necesita las funciones de ZFS.

### 3. Montar y hacerlo permanente

```bash
mkdir -p /mnt/storage
mount /dev/sda1 /mnt/storage      # prueba manual
blkid /dev/sda1                   # obtener el UUID
```

Línea en `/etc/fstab`:

```
UUID=<uuid-de-sda1> /mnt/storage ext4 defaults 0 2
```

Se usa el **UUID** y no `/dev/sda1` porque el nombre del dispositivo puede cambiar entre arranques si cambia el orden de detección de los discos.

Antes de reiniciar, se valida `fstab`. Un error en ese archivo puede impedir el arranque:

```bash
umount /mnt/storage
mount -a                          # sin errores = fstab correcto
findmnt /mnt/storage
```

### 4. Agregarlo a Proxmox

**Centro de datos → Almacenamiento → Agregar → Directorio**:

| Campo | Valor |
|---|---|
| ID | `backup-hdd` |
| Directorio | `/mnt/storage` |
| Contenido | Imagen ISO, Respaldo, Plantilla de contenedor |

No se habilita para discos de VM ni de contenedores: la latencia de un disco mecánico no sirve para cargas activas.

## Almacenamientos en Proxmox

`/etc/pve/storage.cfg`:

| ID | Tipo | Ubicación | Contenido |
|---|---|---|---|
| `local` | Directorio | `/var/lib/vz` (NVMe) | Configurado para ISO, respaldos y plantillas; sin uso en la práctica |
| `local-zfs` | ZFS | `rpool/data` (NVMe) | Discos de VM y LXC, con aprovisionamiento delgado (`sparse`) |
| `backup-hdd` | Directorio | `/mnt/storage` (HDD) | ISO, respaldos y plantillas de contenedor |

Con `sparse`, el tamaño asignado a cada disco virtual es un límite, no espacio reservado: del pool solo se consume lo que se usa.

## Máquinas virtuales y contenedores

| ID | Nombre | Tipo | Recursos | Disco | Privilegios |
|---|---|---|---|---|---|
| 100 | VM-Docker | VM | 2 vCPU · 8 GB | 50 GB | — |
| 101 | CF-Tunnel | LXC | 1 vCPU · 512 MB | 8 GB | Sin privilegios |
| 102 | LXC-Proxy | LXC | 1 vCPU · 1 GB | 25 GB | **Privilegiado** |
| 103 | LXC-Samba | LXC | 1 vCPU · 1 GB | 12 GB + `/mnt/storage` montado | **Privilegiado** |
| 104 | LXC-Tailscale | LXC | 1 vCPU · 512 MB | 8 GB | Sin privilegios |

## Verificación general

```bash
pveversion                     # versión de Proxmox
ip -br addr show vmbr0         # 192.168.100.2/24
zpool status rpool             # ONLINE, sin errores
pvesm status                   # los tres almacenamientos "active"
findmnt /mnt/storage           # HDD montado
systemctl --failed             # sin servicios caídos
```

## Limitaciones

- **Sin redundancia de discos:** `rpool` es un pool de un solo disco. ZFS detecta corrupción, pero no puede repararla, y una falla del NVMe se lleva el sistema y todas las VM y LXC.
- **Sin respaldos programados:** no hay ningún trabajo de respaldo configurado. El único respaldo existente es uno manual de la VM 100, de abril de 2026.
- **Respaldos y datos en el mismo disco:** `backup-hdd` y las carpetas de Samba comparten el HDD. Si el disco falla, se pierden ambos. No hay ninguna copia fuera del equipo.
- **Retención ilimitada:** `backup-hdd` tiene `prune-backups keep-all=1`; los respaldos nunca se borran solos.
- **Firewall desactivado:** las interfaces de las VM y los LXC tienen `firewall=1`, pero el firewall de Proxmox está deshabilitado a nivel de *Datacenter* (`pve-firewall status`: `disabled`). Esas marcas no filtran nada; solo crean los puentes `fwbr*`.
- **`fstab` sin `nofail`:** si el HDD falla o no se detecta, el host puede quedar detenido en modo de emergencia al arrancar.
- **Contenedores privilegiados:** el CT 102 (expuesto a Internet a través del túnel) y el CT 103 (con todo `/mnt/storage` montado).

## Pendiente

- [ ] Trabajo de respaldo programado para la VM y los LXC, con retención definida (reemplazar `keep-all=1`).
- [ ] Copia de los datos fuera del equipo.
- [ ] Decidir sobre el firewall de Proxmox: activarlo con reglas reales, o quitar `firewall=1` y documentar por qué.
- [ ] Agregar `nofail` al montaje del HDD en `fstab`, y marcar `backup-hdd` con `is_mountpoint` para que Proxmox no escriba en el NVMe si el HDD no está montado.
- [ ] Evaluar `zpool upgrade`: `zpool status` avisa que hay funciones de ZFS sin habilitar. Antes de hacerlo, revisar la compatibilidad con el arranque.

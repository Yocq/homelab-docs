# Samba

Servidor de archivos de la red. Comparte carpetas del HDD por **SMB**, el protocolo que usa Windows para compartir carpetas, de modo que se pueden abrir desde el Explorador de archivos de Windows y desde Nextcloud.

| | |
|---|---|
| **Plataforma** | CT 103 — `LXC-Samba`, LXC Debian 12 **privilegiado** |
| **IP** | `192.168.100.6/24` |
| **Recursos** | 1 vCPU · 1 GB RAM · 12 GB de disco |
| **Datos** | HDD del host, `/mnt/storage`, montado en el contenedor |
| **Recursos compartidos** | `Shared` · `Media` |
| **Usuarios** | `jonathan` y un segundo usuario (`usuario2` en esta guía) |
| **Puertos** | `445`, `139` (SMB) · `137`, `138` UDP (NetBIOS) |

## Por qué un LXC separado

El servicio de archivos no depende de la VM Docker: si esa VM se reinicia o falla, las carpetas compartidas siguen disponibles desde Windows. Además, Samba no necesita Docker, así que un LXC liviano basta.

## Almacenamiento

Los archivos viven en el HDD de 2 TB del host, montado en Proxmox en `/mnt/storage` ([guía de Proxmox](../../docs/proxmox-base.md#hdd-como-almacenamiento-secundario)):

```
/mnt/storage/
├── shared/     recurso [Shared]: carpetas privadas por usuario + carpeta común
└── media/      recurso [Media]: contenido multimedia
```

En el mismo disco están también los respaldos y las ISOs de Proxmox.

### Montaje en el contenedor

El directorio del host se monta en el LXC con un punto de montaje (*bind mount*):

```bash
pct set 103 -mp0 /mnt/storage,mp=/mnt/storage
```

En `/etc/pve/lxc/103.conf` queda como:

```
mp0: /mnt/storage,mp=/mnt/storage
```

### Propietarios y permisos

Linux controla el acceso por identificadores numéricos (UID/GID), no por nombres. En el host:

```bash
chown 1000:1000 /mnt/storage/shared /mnt/storage/media
chmod 775 /mnt/storage/shared /mnt/storage/media
chmod 755 /mnt/storage
```

- **`1000:1000`** es el primer usuario que se crea en un sistema Debian; aquí, `jonathan` dentro del contenedor de Samba.
- **`775`**: lectura, escritura y ejecución para el propietario y su grupo; solo lectura para el resto.
- **`755` en `/mnt/storage`**: solo root modifica la raíz del disco, pero los demás pueden recorrerla para llegar a sus carpetas.

**Por qué el contenedor es privilegiado:** en un LXC privilegiado, el UID 1000 dentro del contenedor es el mismo UID 1000 del host, así que los permisos coinciden directamente. En uno sin privilegios, los UID internos se desplazan (el 1000 interno equivale al 101000 en el host) y haría falta un mapeo de IDs para que el contenedor pudiera escribir en las carpetas.

## Instalación

Dentro del contenedor:

```bash
apt update
apt install -y samba
```

### Usuarios

Cada persona necesita un usuario del sistema, que es el propietario de sus archivos, y un usuario de Samba, con la contraseña que se usa desde Windows. Son dos bases distintas: Samba guarda sus contraseñas aparte de `/etc/shadow`.

```bash
adduser jonathan          # primer usuario del contenedor → UID 1000
smbpasswd -a jonathan     # registra el usuario en Samba y define su contraseña
id jonathan               # uid=1000 gid=1000
```

El segundo usuario se crea igual.

## Configuración

Recursos compartidos en `/etc/samba/smb.conf` (salida de `testparm -s`):

```ini
[Shared]
    path = /mnt/storage/shared
    read only = No
    valid users = jonathan usuario2

[Media]
    path = /mnt/storage/media
    read only = No
    valid users = jonathan usuario2
    hide unreadable = Yes
```

- **`read only = No`:** los recursos permiten escritura.
- **`valid users`:** solo esos usuarios pueden conectarse al recurso.
- **`hide unreadable = Yes`:** Samba oculta los archivos y carpetas que el usuario conectado no puede leer.

Después de cada cambio:

```bash
testparm -s               # valida la sintaxis y muestra la configuración efectiva
systemctl restart smbd
```

### Recurso `[homes]` deshabilitado

Debian incluye por defecto una sección `[homes]`, que comparte automáticamente el directorio personal (`/home/<usuario>`) de quien se conecta. En este laboratorio aparecía como una tercera carpeta, `jonathan`, que estaba en el disco del contenedor y no en el HDD, y además era de solo lectura: al intentar escribir, daba error.

Se deshabilitó comentando sus líneas con `;` en `smb.conf`. Las carpetas personales viven dentro de `Shared`.

### Carpetas privadas

Dentro de `Shared`, cada usuario tiene una carpeta privada, además de una carpeta común:

```bash
mkdir /mnt/storage/shared/jonathan
chown jonathan:jonathan /mnt/storage/shared/jonathan
chmod 700 /mnt/storage/shared/jonathan
```

Con `700`, solo el propietario puede leer, escribir o entrar. Así cada usuario tiene un espacio privado dentro del recurso común, sin crear un recurso de Samba por persona.

## Acceso desde Windows

1. En el Explorador de archivos, escribir `\\192.168.100.6` en la barra de direcciones.
2. Ingresar el usuario y la contraseña de Samba.
3. Aparecen los recursos `Shared` y `Media`.

Windows muestra un aviso sobre el *acceso de invitados no autenticado*: las versiones actuales bloquean por política las conexiones SMB sin autenticación. Es el comportamiento esperado, y por eso cada usuario tiene contraseña de Samba.

Para un acceso más cómodo, la carpeta privada se asigna como unidad de red: **Z:** → `\\192.168.100.6\Shared\jonathan`, con reconexión al iniciar sesión.

## Integración con Nextcloud

Nextcloud muestra estas mismas carpetas mediante su aplicación **External storage**, conectándose por SMB a `192.168.100.6`. Así los archivos son accesibles tanto desde el Explorador de Windows como desde la web y los clientes de Nextcloud. La configuración se documenta en la guía de Nextcloud.

## Verificación

En el contenedor:

```bash
systemctl is-active smbd nmbd       # active
testparm -s                         # sin errores
smbclient -L //192.168.100.6 -U jonathan     # lista Shared y Media
```

Desde Windows: `\\192.168.100.6` muestra `Shared` y `Media`, y se puede crear un archivo en la carpeta privada.

## Problemas conocidos

### Windows no vuelve a pedir credenciales

Después de un cambio en el servidor, Windows puede rechazar la conexión aunque el servidor funcione. La causa es que reutiliza la sesión SMB abierta (la conexión `IPC$`) y las credenciales en caché, y mientras esa sesión exista no vuelve a pedir usuario y contraseña. En PowerShell o CMD:

```powershell
net use                  # sesiones abiertas
net use * /delete        # cierra todas para forzar una autenticación nueva
```

### Contraseña de Samba desincronizada

Si el acceso falla con el usuario y la contraseña correctos, se puede volver a definir la contraseña en el contenedor:

```bash
smbpasswd -a jonathan    # vuelve a definir la contraseña en la base de Samba
smbpasswd -e jonathan    # asegura que la cuenta esté habilitada
systemctl restart smbd
```

## Limitaciones

- **Contenedor privilegiado con todo el HDD montado:** el root del contenedor equivale al root del host, y tiene acceso a todo `/mnt/storage`, incluidos los respaldos de Proxmox, aunque Samba solo comparta `shared` y `media`.
- **Carpetas privadas visibles en `Shared`:** `hide unreadable` está solo en `[Media]`. En `[Shared]`, cada usuario ve las carpetas privadas de los demás, aunque no pueda abrirlas.
- **Sin respaldo:** los archivos compartidos y los respaldos de Proxmox están en el mismo HDD, y no hay ninguna copia fuera de él.
- **Recursos de impresoras sin uso:** siguen activos `[printers]` y `[print$]`, que Debian trae por defecto.
- **NetBIOS activo:** `nmbd` responde en los puertos 137 y 138. Es un servicio de descubrimiento heredado; el acceso por IP o por nombre DNS no lo necesita.
- **Mantenimiento:** 24 paquetes pendientes y sin `unattended-upgrades`. SSH acepta contraseña para usuarios normales; root solo entra con clave.
- **Rastro de la red ICS:** `search mshome.net` en `/etc/resolv.conf` ([guía de cloudflared](../cloudflared/README.md#limitaciones)).

## Pendiente

- [ ] Agregar `hide unreadable = Yes` a `[Shared]`.
- [ ] Montar en el contenedor solo `shared` y `media`, no todo el HDD.
- [ ] Evaluar pasar a un contenedor sin privilegios, con mapeo de IDs.
- [ ] Quitar `[printers]` y `[print$]`, y evaluar deshabilitar `nmbd`.
- [ ] Respaldo de los archivos compartidos fuera del HDD.
- [ ] Monitor en Uptime Kuma para el puerto `445`.
- [ ] Instalar `unattended-upgrades`.

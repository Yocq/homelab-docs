# Tailscale

VPN para administrar el laboratorio a distancia. Un LXC funciona como **subnet router**: anuncia la red `192.168.100.0/24` a la tailnet, y los dispositivos autorizados alcanzan cualquier IP del laboratorio como si estuvieran en casa.

| | |
|---|---|
| **Plataforma** | LXC Debian **sin privilegios** — CT 104 |
| **IP LAN** | `192.168.100.7/24` · puerta de enlace `192.168.100.1` |
| **Recursos** | 1 núcleo · 512 MB RAM · 8 GB de disco |
| **Sistema** | Debian 12 (bookworm) · Tailscale 1.96 |
| **Rol** | Subnet router de `192.168.100.0/24` (sin exit node) |
| **DNS de la tailnet** | Pi-hole (`192.168.100.4`) |

## Por qué Tailscale y no el túnel de Cloudflare

Los dos resuelven el acceso remoto bajo CGNAT sin abrir puertos, pero con propósitos distintos:

| | Cloudflare Tunnel | Tailscale |
|---|---|---|
| Para qué | Publicar aplicaciones web para uso diario | Administrar el laboratorio |
| Quién llega | Cualquiera en Internet (con controles: Access, WAF, login) | Solo los dispositivos autorizados en la tailnet |
| Qué expone | Nombres concretos (`nextcloud.yocolab.dev`) | Toda la LAN, por la VPN |

Proxmox, SSH, Samba y los paneles de administración no deberían quedar publicados en Internet. Con Tailscale se alcanzan sin exponerlos.

Tailscale usa WireGuard. Cada dispositivo abre conexiones salientes y, cuando puede, se comunica directo con los demás (NAT traversal). Si no puede, el tráfico pasa por los servidores de retransmisión de Tailscale (DERP).

## Por qué un subnet router

Instalar Tailscale en cada VM y LXC sería poco práctico, y algunos equipos ni siquiera lo admiten. Un solo nodo anuncia la red completa y reenvía el tráfico hacia ella.

## Despliegue

### 1. Crear el LXC

LXC con plantilla Debian 12, **sin privilegios**, 1 núcleo, 512 MB de RAM, 8 GB de disco e IP fija `192.168.100.7/24` (puerta de enlace `192.168.100.1`).

Es sin privilegios porque es un punto de entrada a la red: un contenedor sin privilegios mapea su root a un usuario sin permisos en el host.

### 2. Dar acceso al dispositivo TUN

Tailscale necesita `/dev/net/tun` para crear su interfaz de red, y un LXC sin privilegios no tiene acceso a ese dispositivo por defecto. Con el contenedor detenido, en el host:

```bash
pct stop 104
cat >> /etc/pve/lxc/104.conf <<'EOF'
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF
pct start 104
```

Sin este paso, `tailscaled` queda en estado `failed`. El diagnóstico completo está en el [caso de troubleshooting](../../docs/troubleshooting/tailscale-dev-net-tun.md).

### 3. Instalar Tailscale

```bash
pct enter 104
apt update && apt install -y curl
curl -fsSL https://tailscale.com/install.sh | sh
systemctl status tailscaled     # active (running), "Needs login"
```

El script oficial detecta la distribución, agrega el repositorio de Tailscale e instala el paquete.

### 4. Habilitar el reenvío de paquetes

Un subnet router reenvía paquetes entre la interfaz `tailscale0` y la LAN. Linux descarta por defecto los paquetes que no van dirigidos a él mismo, así que hay que habilitar el *IP forwarding*:

```bash
cat > /etc/sysctl.d/99-tailscale.conf <<'EOF'
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sysctl -p /etc/sysctl.d/99-tailscale.conf
```

> En el despliegue original se descomentó `net.ipv4.ip_forward=1` en `/etc/sysctl.conf`. El archivo en `/etc/sysctl.d/` es la forma que recomienda Tailscale: no depende de que `/etc/sysctl.conf` exista en la versión de Debian, y deja la configuración separada e identificable.

### 5. Autenticar el nodo y anunciar la red

```bash
tailscale up --advertise-routes=192.168.100.0/24
```

El comando entrega una URL de autenticación. Al abrirla e iniciar sesión, el nodo queda asociado a la cuenta y se une a la tailnet con una IP del rango `100.64.0.0/10`. Tailscale usa ese rango, reservado para CGNAT, porque no se usa en redes domésticas y evita conflictos con la LAN.

### 6. Aprobar la ruta en la consola

En la consola de Tailscale, el nodo aparece con la etiqueta **Subnets** y la ruta en **Awaiting Approval**. Se aprueba en **Edit route settings**.

Tailscale exige aprobar cada subred anunciada para que un nodo no pueda desviar tráfico de la tailnet hacia una red sin autorización del administrador.

**Exit node** queda deshabilitado: este nodo solo da acceso a la LAN, no enruta todo el tráfico de Internet de los clientes.

### 7. Deshabilitar la expiración de la clave

Cada nodo de Tailscale se autentica con una clave que **expira periódicamente** (por defecto, cada 180 días). Al expirar, el nodo sale de la tailnet hasta que alguien vuelve a autenticarlo. En un equipo personal es una buena medida de seguridad; en un subnet router significa perder **todo** el acceso remoto sin aviso, justo cuando no estás en casa para arreglarlo.

En la consola: **Machines → `lxc-tailscale` → `…` → Disable key expiry**. El nodo queda marcado como *Expiry disabled*.

Esto ya ocurrió una vez en este laboratorio ([caso](../../docs/troubleshooting/tailscale-clave-expirada.md)).

### 8. Agregar los clientes

Se instala el cliente oficial en cada dispositivo y se autoriza en la tailnet:

| Dispositivo | Nombre en la tailnet |
|---|---|
| Notebook Windows | `jonathan-laptop` |
| Teléfono Android | `jonathan-phone` |

Los nombres se cambian en la consola para identificar cada equipo; también son los nombres que usa MagicDNS.

## Tailscale SSH

El nodo tiene **Tailscale SSH** activo (`RunSSH: true`): los dispositivos de la tailnet pueden abrir una sesión SSH en el LXC autenticándose con la identidad de Tailscale, sin claves ni contraseñas del sistema. Quién puede entrar lo definen las reglas `ssh` de la política de la tailnet.

## DNS: Pi-hole en la tailnet

Con la VPN activa, los clientes remotos siguen usando el DNS de la red en la que estén. Por eso no reciben el filtrado de Pi-hole ni resuelven los nombres internos.

En la consola, sección **DNS**:

1. Se agrega `192.168.100.4` (Pi-hole) como **nameserver global**.
2. Se activa **Override DNS servers**: los clientes ignoran el DNS de la red local en la que estén y envían todas sus consultas a Pi-hole por la VPN.

Pi-hole es alcanzable porque está en la red que anuncia el subnet router. El router además traduce las direcciones de origen (SNAT), así que Pi-hole ve las consultas como si vinieran de `192.168.100.7`, una IP de la LAN que ya acepta.

MagicDNS (`100.100.100.100`) sigue siendo el resolvedor local de cada cliente: responde los nombres de la tailnet y reenvía el resto a Pi-hole.

## Verificación

**Acceso remoto:** desde un notebook en otra red Wi-Fi y desde el teléfono con datos móviles, con Tailscale activo, la interfaz de Proxmox abre en su IP local:

```
https://192.168.100.2:8006
```

**Ruta anunciada** (en el LXC):

```bash
tailscale debug prefs | grep -A2 AdvertiseRoutes     # → "192.168.100.0/24"
```

**Estado de la tailnet** (en el LXC):

```bash
tailscale status
```

El notebook aparece con conexión **direct**: el tráfico viaja punto a punto, sin pasar por los servidores de retransmisión.

**DNS:** desde un cliente remoto, el panel de Pi-hole abre por nombre (`http://pihole.yocolab`, resuelto por el comodín `*.yocolab` y servido por Nginx Proxy Manager) y sus consultas aparecen en el Query Log.

## Actualización

Tailscale se instala desde su repositorio oficial, así que se actualiza con el sistema:

```bash
apt update && apt upgrade -y
```

## Problemas conocidos

- [Tailscale no inicia en un LXC sin privilegios (`/dev/net/tun`)](../../docs/troubleshooting/tailscale-dev-net-tun.md)
- [Se pierde el acceso remoto: clave del subnet router expirada](../../docs/troubleshooting/tailscale-clave-expirada.md)

## Limitaciones

- **Dependencia total de Pi-hole:** con *Override DNS servers*, si Pi-hole o la VM Docker se detienen, los clientes remotos se quedan **sin ninguna resolución DNS** mientras la VPN esté activa, no solo sin filtrado.
- **Sin control de acceso por dispositivo:** con la política por defecto de la tailnet, todo dispositivo autorizado alcanza toda la LAN. Para dos dispositivos propios es aceptable; con más usuarios no lo sería.
- **Un solo subnet router:** si el LXC se detiene, se pierde el acceso remoto a toda la LAN.
- **SSH alcanzable desde la tailnet:** el subnet router expone la LAN completa, incluido el puerto 22 de cada máquina. Hoy la tailnet tiene solo dos dispositivos propios, así que el riesgo es bajo, pero el endurecimiento de SSH en cada equipo depende de su propia configuración.

## Pendiente

- [x] Deshabilitar la expiración de la clave del nodo `lxc-tailscale` (ver [paso 7](#7-deshabilitar-la-expiración-de-la-clave)).
- [ ] Definir reglas de acceso (ACL) en la tailnet si se agregan usuarios.
- [ ] Monitor en Uptime Kuma para el LXC de Tailscale.

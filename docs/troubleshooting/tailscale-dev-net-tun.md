# Tailscale no inicia en un LXC sin privilegios (`/dev/net/tun`)

| | |
|---|---|
| **Componente** | LXC-Tailscale (CT 104), Debian, **sin privilegios** |
| **Síntoma** | `tailscale up` no logra conectarse con `tailscaled`, que queda en estado `failed` |
| **Solución aplicada** | Dar acceso al dispositivo TUN del host en `/etc/pve/lxc/104.conf` |
| **Estado** | Resuelto |

## Contexto

Tailscale se instaló en un LXC propio para que funcione como **subnet router**: un solo nodo anuncia la red `192.168.100.0/24` a la tailnet, y los dispositivos remotos alcanzan todo el laboratorio sin tener Tailscale instalado en cada VM.

El LXC se creó **sin privilegios** a propósito: es un punto de entrada a la red, y un contenedor sin privilegios mapea su root a un usuario sin permisos en el host.

La instalación se hizo con el script oficial:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

## Síntoma

`tailscale up` no logra comunicarse con el servicio `tailscaled`:

```bash
tailscale up
# failed to connect to local tailscaled ...
```

<!-- TODO: reemplazar por el mensaje exacto si se vuelve a reproducir -->

```bash
systemctl status tailscaled
# Active: failed (Result: exit-code)
```

El servicio se reiniciaba solo varias veces y volvía a fallar.

## Diagnóstico

El estado `failed` dice que el proceso termina, pero no por qué. El motivo está en los registros del servicio:

```bash
journalctl -u tailscaled -n 50 --no-pager
```

```
/dev/net/tun does not exist
```

## Causa

Tailscale usa WireGuard y crea una interfaz de red virtual (`tailscale0`) a través del dispositivo **TUN** del kernel, `/dev/net/tun`.

Un LXC no tiene kernel propio: usa el del host Proxmox. Por eso:

- No puede **cargar el módulo** TUN, porque el kernel pertenece al host.
- Un LXC **sin privilegios** tampoco tiene acceso a `/dev/net/tun` por defecto, ya que el acceso a dispositivos del host está restringido.

El servicio funcionaba, pero no tenía con qué crear su interfaz de red.

## Solución

Con el contenedor detenido, en el host Proxmox se agregan dos líneas a su configuración:

```bash
pct stop 104
nano /etc/pve/lxc/104.conf
```

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

- **`lxc.cgroup2.devices.allow: c 10:200 rwm`** autoriza al contenedor a usar el dispositivo de caracteres con número mayor 10 y menor 200, que es `/dev/net/tun`. `rwm` permite leerlo, escribirlo y crear el nodo.
- **`lxc.mount.entry: ... bind,create=file`** monta el `/dev/net/tun` del host dentro del contenedor y crea el archivo si no existe.

```bash
pct start 104
pct enter 104
ls -l /dev/net/tun             # debe existir: crw-rw-rw- ... 10, 200
systemctl status tailscaled    # active (running), estado "Needs login"
```

Es la configuración que indica la documentación de Tailscale para contenedores LXC sin privilegios.

## Resultado

`tailscaled` queda **active (running)**. Después, `tailscale up` entrega la URL de autenticación, el nodo se une a la tailnet y se configura como subnet router.

## Por qué no convertir el contenedor en privilegiado

Volver a crear el LXC como privilegiado también habría resuelto el error, pero le habría dado al contenedor mucho más acceso al host del que necesita. La solución aplicada concede acceso **a un solo dispositivo** y mantiene el aislamiento del contenedor sin privilegios. Es la diferencia con el LXC de Nginx Proxy Manager, que sí quedó privilegiado ([caso](docker-overlay-en-lxc.md)).

## Aprendizajes

- `systemctl status` dice **qué** falló; `journalctl -u <servicio>` dice **por qué**. Ante un servicio en `failed`, el journal es el primer lugar donde mirar.
- Todo lo que depende del kernel (módulos, dispositivos, montajes) se resuelve en el host, no dentro del LXC.
- Ante un problema de permisos, conviene conceder lo mínimo necesario, un dispositivo, antes que todo el acceso, un contenedor privilegiado.

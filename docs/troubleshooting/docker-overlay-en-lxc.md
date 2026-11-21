# Docker no puede montar overlay dentro de un LXC

| | |
|---|---|
| **Componente** | LXC de Nginx Proxy Manager (Debian, privilegiado, nesting activado) |
| **Síntoma** | `docker compose up -d` descarga la imagen, pero el contenedor no inicia |
| **Solución aplicada** | Cambiar el storage driver de Docker a `vfs` y ampliar el disco del LXC |
| **Estado** | Funciona, como solución provisional (ver [Pendiente](#pendiente)) |

## Contexto

Nginx Proxy Manager (NPM) se instaló en un LXC propio, separado de la VM Docker, para que el reverse proxy siga funcionando aunque la VM de aplicaciones se reinicie. NPM se distribuye como imagen Docker, así que hubo que ejecutar Docker **dentro** del LXC. Para eso el contenedor necesita la opción *nesting*.

Configuración inicial del LXC: 1 vCPU, 1 GB de RAM, **8 GB de disco**, disco raíz en `local-zfs`.

## Síntoma

Al levantar el stack, la descarga de la imagen termina bien pero la creación del contenedor falla con un error de montaje:

```
failed to mount ... overlay ... permission denied
```

## Diagnóstico

Docker usa por defecto el storage driver **overlay2**. Este driver arma el sistema de archivos de cada contenedor superponiendo las capas de la imagen mediante el sistema de archivos `overlay` del kernel.

En un LXC hay tres factores que pueden impedir ese montaje:

1. **El kernel es del host.** El LXC no tiene kernel propio; cualquier montaje overlay lo ejecuta el kernel de Proxmox, sujeto a las restricciones que el host impone al contenedor.
2. **Perfil de AppArmor y permisos de montaje.** Proxmox limita qué tipos de montaje puede hacer un contenedor.
3. **El disco raíz es un dataset ZFS.** overlay2 necesita un sistema de archivos de base compatible. El soporte de overlay sobre ZFS llegó recién con OpenZFS 2.2.

Para verificar qué driver intenta usar Docker:

```bash
docker info | grep "Storage Driver"
```

## Causa

Docker no pudo crear el montaje overlay dentro del LXC por las restricciones de montaje del contenedor, sobre un disco raíz ZFS.

> **Lo que no se verificó:** cuál de los factores anteriores fue el determinante. Para aislarlo habría que revisar `journalctl -u docker` dentro del LXC, `dmesg | grep -i apparmor` en el host y `zfs version` para conocer la versión de OpenZFS.

## Solución

### 1. Cambiar el storage driver a `vfs`

`vfs` no usa montajes overlay: guarda cada capa como una copia completa en un directorio normal. Por eso funciona en casi cualquier entorno.

```bash
cat > /etc/docker/daemon.json <<'EOF'
{ "storage-driver": "vfs" }
EOF

systemctl restart docker
docker info | grep "Storage Driver"   # → Storage Driver: vfs
```

### 2. Segundo error: falta de espacio

El siguiente intento de despliegue falla con:

```
disk quota exceeded
```

Es una consecuencia directa del cambio anterior. Como `vfs` copia cada capa completa en lugar de superponerlas, una imagen que con overlay2 ocupa unos cientos de MB puede multiplicar su tamaño en disco. Los 8 GB del LXC no alcanzaron.

### 3. Ampliar el disco del LXC

Desde la interfaz: **Recursos → Disco Root → Acción del volumen → Redimensionar**. El equivalente por consola en el host:

```bash
pct resize <CTID> rootfs 25G
```

Como el disco raíz es un dataset ZFS, el cambio se hace en caliente: Proxmox solo modifica la cuota del dataset y no hay que reformatear nada.

```bash
zfs get refquota rpool/data/subvol-<CTID>-disk-0
```

El disco pasó de **8 GB a 25 GB**, y el contenedor `nginx-proxy-manager` inició con los puertos 80, 81 y 443 publicados.

## Resultado

NPM funciona y es el punto de entrada de todos los proxy hosts. El costo de la solución fue:

- **Espacio:** el LXC necesita unas 3 veces el disco previsto.
- **Rendimiento:** `vfs` es más lento al crear contenedores y al actualizar imágenes.
- **Seguridad:** el LXC es privilegiado, y es el destino del tráfico que entra por el túnel de Cloudflare.

## Pendiente

`vfs` resolvió el problema, pero no es una configuración que conviene mantener. Alternativas por evaluar:

| Opción | Ventaja | Desventaja |
|---|---|---|
| LXC **sin privilegios** con overlay2 | Menos riesgo y uso de disco normal | Depende del soporte overlay en la versión de ZFS del host. Es una hipótesis por probar. |
| `fuse-overlayfs` dentro del LXC | Evita el montaje overlay del kernel | Una capa más que mantener, y rendimiento intermedio |
| Mover NPM a la VM Docker | Lo más simple; overlay2 funciona sin cambios | Pierde la independencia del proxy respecto de la VM de aplicaciones |

## Aprendizajes

- Un LXC no es una VM liviana: comparte el kernel del host, y todo lo que dependa del kernel (montajes, módulos, dispositivos) queda limitado por el host.
- Cambiar un componente para salir de un error puede crear el siguiente. Aquí, `vfs` resolvió el montaje y causó el problema de espacio.
- Una solución que funciona no es necesariamente la definitiva: hay que dejar registrado su costo.

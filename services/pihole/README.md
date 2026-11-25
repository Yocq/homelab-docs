# Pi-hole

Servidor DNS interno con filtrado de dominios. Resuelve los nombres internos del laboratorio y es el DNS de los clientes de la LAN y de la tailnet de Tailscale.

| | |
|---|---|
| **Plataforma** | VM Docker — `192.168.100.4` |
| **Versión** | Pi-hole v6 (`pihole/pihole:latest`) |
| **Puertos** | `53/tcp` y `53/udp` (DNS) · `8081` (web) |
| **Persistencia** | `/opt/docker/pihole/etc-pihole` · `/opt/docker/pihole/etc-dnsmasq.d` |
| **Upstream** | Cloudflare: `1.1.1.1` · `1.0.0.1` |
| **Panel** | `http://192.168.100.4:8081/admin` |

## Por qué en Docker y no en un LXC

Pi-hole es un servicio liviano. Correrlo en la VM Docker junto a los demás servicios livianos permite administrar, actualizar y respaldar todo con las mismas herramientas (Compose y el respaldo de la VM). Los LXC se reservan para servicios de borde que deben seguir funcionando aunque la VM se reinicie.

## Requisitos

- VM Docker con Docker Engine y el plugin Compose.
- Puerto `53` libre en la VM. Se verifica con `ss -tulpn | grep ':53 '`, que no debe devolver nada.

## Despliegue

### 1. Crear el directorio del servicio

```bash
sudo mkdir -p /opt/docker/pihole/{etc-pihole,etc-dnsmasq.d}
cd /opt/docker/pihole
```

- **`etc-pihole`** guarda la configuración (`pihole.toml`), las listas de bloqueo y la base de datos de consultas. Como queda fuera del contenedor, este se puede recrear o actualizar sin perder nada.
- **`etc-dnsmasq.d`** era el directorio de configuración adicional de dnsmasq en Pi-hole v5. En v6 la opción `misc.etc_dnsmasq_d` viene deshabilitada y este volumen queda sin uso; la configuración adicional se escribe en `misc.dnsmasq_lines`.

### 2. Crear el compose

Copia [`compose.yaml`](compose.yaml) al directorio como `docker-compose.yml` y reemplaza el marcador de la contraseña. Es el archivo tal como está desplegado.

Qué define:

- **`8081:80`**: la interfaz web se publica en 8081 para dejar el 80 libre a otros servicios de la VM.
- **`dns: 8.8.8.8, 1.1.1.1`**: el DNS que usa **el propio contenedor** para sus procesos internos, como la descarga de las listas de bloqueo. Así Pi-hole no depende de sí mismo para arrancar o actualizarse. No son los servidores upstream (ver paso 5).
- **`WEBPASSWORD`**: es la variable de Pi-hole v5. **La versión 6 la ignora**: la contraseña del panel se definió a mano con `pihole setpassword` y quedó guardada en el volumen ([caso](../../docs/troubleshooting/pihole-contrasena-web.md)).
- **`restart: unless-stopped`**: el servicio vuelve a iniciar después de reiniciar la VM o Docker.

> **Desfase conocido:** el compose desplegado conserva variables y volúmenes de v5. Funciona porque la configuración real vive en `etc-pihole/pihole.toml`, pero el archivo no describe cómo está configurado Pi-hole. Actualizarlo a las variables de v6 está en [Pendiente](#otros-pendientes).

### 3. Levantar el contenedor

```bash
docker compose up -d
docker compose ps        # estado esperado: running (healthy)
```

### 4. Configuración desde la interfaz

Dos ajustes se hicieron en el panel y quedan guardados en `pihole.toml`, no en el compose:

- **Settings → DNS → Interface settings → Permit all origins**: acepta consultas de cualquier origen. Es necesario con la red bridge de Docker, porque desde el contenedor la LAN no se ve como red local ([caso](../../docs/troubleshooting/pihole-non-local-network.md)).
- **Contraseña del panel**, con `docker exec -it pihole pihole setpassword`.

### 5. Servidores upstream

Los upstream son los servidores a los que Pi-hole reenvía las consultas que no puede responder él mismo, es decir, las que no están bloqueadas ni son registros locales. Se configuran en **Settings → DNS → Upstream DNS Servers**.

| Upstream | Proveedor |
|---|---|
| `1.1.1.1` | Cloudflare (primario) |
| `1.0.0.1` | Cloudflare (secundario) |

Para verlos por terminal:

```bash
docker exec pihole pihole-FTL --config dns.upstreams     # → [ 1.0.0.1, 1.1.1.1 ]
```

> Al revisar la configuración apareció un tercer upstream, `192.168.100.5`. Esa IP es el LXC de Nginx Proxy Manager, que no responde DNS (no escucha en el puerto 53). Las consultas que Pi-hole le enviaba fallaban y terminaban resueltas por Cloudflare. Se quitó de la lista.

### 6. Registros DNS locales

Todos los nombres internos apuntan al reverse proxy (Nginx Proxy Manager, `192.168.100.5`). Por eso se usa un registro comodín en lugar de un registro por servicio:

**Settings → All settings** (modo *Expert*) **→ Miscellaneous → `misc.dnsmasq_lines`:**

```
address=/*.yocolab/192.168.100.5
```

La directiva `address=/dominio/IP` de dnsmasq responde esa IP para el dominio y todos sus subdominios.

## Verificación

Desde un equipo Windows de la LAN (PowerShell):

```powershell
# Resolución externa a través de Pi-hole
Resolve-DnsName google.com -Server 192.168.100.4

# El comodín: un nombre que no existe en ningún registro debe devolver la IP del proxy
Resolve-DnsName prueba.yocolab -Server 192.168.100.4     # → 192.168.100.5
```

En el panel, **Query Log** debe mostrar esas consultas con la IP del cliente.

> Se usa `Resolve-DnsName` y no `nslookup` porque `nslookup` primero intenta la resolución inversa del propio servidor DNS. Si no hay registro PTR, muestra `Server: UnKnown` y timeouts aunque la resolución funcione.

## Monitoreo

Uptime Kuma tiene dos monitores sobre este servicio:

- **HTTP** sobre `http://192.168.100.4:8081/admin`: confirma que el panel responde.
- **DNS** sobre `google.com` contra `192.168.100.4`: confirma la función principal. El panel puede responder aunque la resolución DNS esté fallando.

## Actualización

```bash
cd /opt/docker/pihole
docker compose pull
docker compose up -d
```

## Problemas conocidos

- [La contraseña definida en el compose no funciona](../../docs/troubleshooting/pihole-contrasena-web.md)
- [Pi-hole ignora las consultas de la LAN ("non-local network")](../../docs/troubleshooting/pihole-non-local-network.md)

## Limitaciones

- **Punto único de falla:** si la VM Docker se detiene, los clientes que usan Pi-hole como único DNS se quedan sin resolución. Esto incluye a los dispositivos de Tailscale, que tienen *Override DNS* activo.
- *Permit all origins* es aceptable porque el puerto 53 no está expuesto a Internet: la conexión está detrás de CGNAT y no hay puertos redirigidos. No debe usarse en un equipo con el puerto 53 publicado.
- La imagen usa `latest`: una actualización puede traer cambios incompatibles, como pasó con las variables de v5 a v6.

## Pendiente

### Migrar a un LXC propio (planificado)

**Motivo.** El DNS es el servicio del que más depende el laboratorio: la navegación de la LAN, los clientes de Tailscale, el comodín `*.yocolab` y el propio Nextcloud, que usa Pi-hole como DNS. Hoy un reinicio de la VM Docker deja sin resolución a todos esos clientes. Moverlo a un LXC aplica el mismo criterio que ya se usa con el proxy, el túnel y la VPN: los servicios de infraestructura no dependen de la VM de aplicaciones.

**Diseño previsto:**

| | Actual | Previsto |
|---|---|---|
| Plataforma | Contenedor Docker en la VM Docker | LXC Debian **sin privilegios** |
| Instalación | Imagen `pihole/pihole` | Instalador oficial nativo, **sin Docker dentro del LXC** |
| IP | `192.168.100.4` (la de la VM) | IP fija propia <!-- TODO: definir IP --> |
| Modo de escucha | *Permit all origins* (necesario por la red bridge) | `LOCAL`: el LXC está en la LAN y ve la red real |
| Recursos | Compartidos con la VM | 1 vCPU, 512 MB RAM |

No se usa Docker dentro del LXC para no repetir los problemas del LXC de Nginx Proxy Manager: contenedor privilegiado, montajes overlay bloqueados y storage driver `vfs` ([caso](../../docs/troubleshooting/docker-overlay-en-lxc.md)).

**Plan de migración (sin corte de servicio):**

1. Crear el LXC e instalar Pi-hole.
2. Exportar la configuración actual con **Settings → Teleporter** e importarla en la nueva instancia (listas, registros locales y `misc.dnsmasq_lines`).
3. Validar la nueva instancia desde un solo equipo con `Resolve-DnsName google.com -Server <IP_NUEVA>`.
4. Cambiar los clientes de a uno y verificar cada cambio:
   - [ ] Nameserver global de Tailscale
   - [ ] `dns:` del compose de Nextcloud
   - [ ] Monitores HTTP y DNS de Uptime Kuma
   - [ ] Enlace en Homepage
   - [ ] DNS entregado por el router (si aplica)
5. Decidir qué hacer con la instancia de Docker: apagarla, o dejarla como **DNS secundario** sincronizado con nebula-sync.
6. Mover esta guía a la instalación en LXC y conservar la versión Docker como referencia.

**Alcance de la redundancia:** dos instancias en el mismo equipo físico protegen ante la falla de la VM o del servicio, pero no ante la caída del host completo.

### Otros pendientes

- [x] Completar la lista de servidores DNS upstream (ver [paso 5](#5-servidores-upstream)).
- [ ] Actualizar el compose a Pi-hole v6: `FTLCONF_webserver_api_password` en lugar de `WEBPASSWORD` (con la contraseña en un `.env` con permisos `600`), `FTLCONF_dns_listeningMode: "all"` y quitar el volumen `etc-dnsmasq.d`.
- [ ] Fijar la versión de la imagen (o del paquete, tras migrar) en lugar de `latest`.
- [ ] Incluir la configuración de Pi-hole en la estrategia de respaldos (exportación periódica con Teleporter).

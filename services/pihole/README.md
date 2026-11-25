# Pi-hole

Servidor DNS interno con filtrado de dominios. Resuelve los nombres internos del laboratorio y es el DNS de los clientes de la LAN y de la tailnet de Tailscale.

| | |
|---|---|
| **Plataforma** | VM Docker — `192.168.100.4` |
| **Versión** | Pi-hole v6 (`pihole/pihole:latest`) |
| **Puertos** | `53/tcp` y `53/udp` (DNS) · `8081` (web) |
| **Persistencia** | `/opt/docker/pihole/etc-pihole` |
| **Panel** | `http://192.168.100.4:8081/admin` |

## Por qué en Docker y no en un LXC

Pi-hole es un servicio liviano. Correrlo en la VM Docker junto a los demás servicios livianos permite administrar, actualizar y respaldar todo con las mismas herramientas (Compose, Portainer y el respaldo de la VM). Los LXC se reservan para servicios de borde que deben seguir funcionando aunque la VM se reinicie.

## Requisitos

- VM Docker con Docker Engine y el plugin Compose.
- Puerto `53` libre en la VM. Se verifica con `ss -tulpn | grep ':53 '`, que no debe devolver nada.

## Despliegue

### 1. Crear el directorio del servicio

```bash
sudo mkdir -p /opt/docker/pihole/etc-pihole
cd /opt/docker/pihole
```

`etc-pihole` guarda la configuración (`pihole.toml`), las listas de bloqueo y la base de datos de consultas. Como queda fuera del contenedor, este se puede recrear o actualizar sin perder nada.

> En Pi-hole v6 ya no hace falta el volumen `etc-dnsmasq.d`: la opción `misc.etc_dnsmasq_d` viene deshabilitada y la configuración adicional de dnsmasq se escribe en `misc.dnsmasq_lines`.

### 2. Crear `compose.yaml` y `.env`

Copia [`compose.yaml`](compose.yaml) y [`.env.example`](.env.example) al directorio. Después:

```bash
cp .env.example .env
nano .env          # definir TZ y PIHOLE_WEB_PASSWORD
chmod 600 .env
```

Qué define el compose:

- **`8081:80`**: la interfaz web se publica en 8081 para dejar el 80 libre a otros servicios de la VM.
- **`FTLCONF_webserver_api_password`**: contraseña del panel. En v6 reemplaza a `WEBPASSWORD`, que ya no tiene efecto ([caso](../../docs/troubleshooting/pihole-contrasena-web.md)).
- **`FTLCONF_dns_listeningMode: "all"`**: acepta consultas de cualquier origen. Es necesario con la red bridge de Docker ([caso](../../docs/troubleshooting/pihole-non-local-network.md)).
- **`restart: unless-stopped`**: el servicio vuelve a iniciar después de reiniciar la VM o Docker.

> Los valores definidos con variables `FTLCONF_` quedan fijos: no se pueden cambiar desde la interfaz web mientras estén en el compose.

### 3. Levantar el contenedor

```bash
docker compose up -d
docker compose ps        # estado esperado: running (healthy)
```

### 4. Registros DNS locales

Todos los nombres internos apuntan al reverse proxy (Nginx Proxy Manager, `192.168.100.5`). Por eso se usa un registro comodín en lugar de un registro por servicio:

**Settings → All settings** (modo *Expert*) **→ Miscellaneous → `misc.dnsmasq_lines`:**

```
address=/*.yocolab/192.168.100.5
```

La directiva `address=/dominio/IP` de dnsmasq responde esa IP para el dominio y todos sus subdominios.

<!-- TODO: completar con la lista real de servidores DNS upstream configurados -->

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
- `listeningMode: all` es aceptable porque el puerto 53 no está expuesto a Internet: la conexión está detrás de CGNAT y no hay puertos redirigidos. No debe usarse en un equipo con el puerto 53 publicado.
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
| Modo de escucha | `all` (necesario por la red bridge) | `LOCAL`: el LXC está en la LAN y ve la red real |
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

- [ ] Fijar la versión de la imagen (o del paquete, tras migrar) en lugar de `latest`.
- [ ] Incluir la configuración de Pi-hole en la estrategia de respaldos (exportación periódica con Teleporter).

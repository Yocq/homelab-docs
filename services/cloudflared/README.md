# Cloudflare Tunnel (cloudflared)

Publica en Internet los servicios web de uso diario del laboratorio sin abrir puertos en el router. Un conector instalado en la LAN mantiene conexiones salientes hacia Cloudflare, y Cloudflare entrega por ellas las solicitudes que llegan a cada subdominio.

| | |
|---|---|
| **Plataforma** | CT 101 — `CF-Tunnel`, LXC Debian 12 **sin privilegios** |
| **IP** | `192.168.100.3/24` |
| **Recursos** | 1 vCPU · 512 MB RAM · 8 GB de disco |
| **Versión** | cloudflared 2026.3.0 |
| **Túnel** | `homelab-tunnel`, dominio `yocolab.dev` |
| **Puertos entrantes** | Ninguno |

## Por qué un túnel

El proveedor de Internet usa **CGNAT**: varios clientes comparten una misma IP pública, y las conexiones que llegan desde Internet nunca alcanzan el router doméstico. Por eso la forma tradicional de publicar un servicio (abrir un puerto y apuntar un dominio a la IP de la casa) no es posible.

Cloudflare Tunnel invierte la dirección de la conexión:

1. `cloudflared`, dentro de la LAN, **abre conexiones salientes** hacia Cloudflare y las mantiene activas.
2. Cuando alguien visita un subdominio publicado, Cloudflare envía la solicitud por esas conexiones ya establecidas.
3. `cloudflared` la entrega al servicio interno configurado para ese subdominio.

Además de funcionar bajo CGNAT:

- **No se abre ningún puerto** en el router.
- **La IP de la casa no se expone**: los subdominios resuelven a direcciones de Cloudflare.
- **El certificado HTTPS público lo gestiona Cloudflare**, sin intervención en el laboratorio.
- Se pueden poner **controles en el borde** (Access, WAF) antes de que una solicitud llegue a la red.

## Requisitos

- El dominio `yocolab.dev` administrado en Cloudflare: sus servidores DNS deben apuntar a Cloudflare.

## Despliegue

### 1. Crear el túnel en Cloudflare

En el panel de Cloudflare: **Networking → Tunnels → Create Tunnel**, con el nombre `homelab-tunnel`.

El túnel queda **Inactive** hasta que se registra un conector. Cloudflare muestra los comandos de instalación por sistema operativo (aquí, *Debian 64-bit*) e incluye un **token** que identifica y autoriza al conector. El token equivale a una contraseña: no se publica ni se guarda en el repositorio.

### 2. Crear el LXC

| Parámetro | Valor | Motivo |
|---|---|---|
| Plantilla | Debian 12 (en `backup-hdd`) | |
| Contenedor sin privilegios | **Sí** | `cloudflared` no necesita privilegios ni Docker. Es el punto de entrada desde Internet: si se comprometiera, su root no equivale al root del host |
| CPU / RAM / disco | 1 vCPU / 512 MB / 8 GB | `cloudflared` es un proceso liviano (usa unos 30 MB de RAM) |
| Red | `vmbr0`, `192.168.100.3/24`, puerta de enlace `192.168.100.1` | |

Va en un LXC propio, separado del proxy y de la VM Docker, para que el punto de entrada desde Internet quede aislado y se pueda reiniciar o reemplazar sin afectar a los servicios.

### 3. Instalar cloudflared

Desde la consola del host, `pct enter 101` abre una shell de root dentro del contenedor sin necesidad de SSH. Allí se agrega el repositorio de Cloudflare y se instala el paquete:

```bash
mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared bookworm main' | tee /etc/apt/sources.list.d/cloudflared.list
apt-get update && apt-get install -y cloudflared
cloudflared --version
```

- La clave GPG permite que `apt` verifique la firma de los paquetes.
- `bookworm` es la versión de Debian de la plantilla del contenedor.

> **Problema encontrado:** el primer intento consistió en pegar en una sola línea el bloque de comandos que muestra Cloudflare. El bloque trae comentarios que empiezan con `#`, y al quedar todo en la misma línea, la shell tomó el primer `#` como inicio de comentario y descartó el resto. No se ejecutó nada, y tampoco apareció un error. El intento siguiente falló con `Unable to locate package cloudflared`, porque el repositorio nunca se había agregado. La solución fue ejecutar cada comando por separado.

### 4. Registrar el conector como servicio

```bash
cloudflared service install <TOKEN>
```

El conector queda registrado como servicio de systemd: arranca con el contenedor y se reconecta solo si pierde la conexión. Unidad generada (`/etc/systemd/system/cloudflared.service`, con el token oculto):

```ini
[Unit]
Description=cloudflared
After=network-online.target
Wants=network-online.target

[Service]
TimeoutStartSec=15
Type=notify
ExecStart=/usr/bin/cloudflared --no-autoupdate tunnel run --token <TOKEN>
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

`--no-autoupdate` desactiva la actualización automática del binario: `cloudflared` se actualiza con `apt`, igual que el resto del sistema.

En el panel, el túnel pasa a **Healthy** con un conector activo. Cada conector abre cuatro conexiones repartidas entre más de un centro de datos de Cloudflare (aquí, Santiago y São Paulo), para tolerar la caída de uno.

## Rutas publicadas

Cada servicio se publica como **Public hostname** del túnel: subdominio, dominio y URL del servicio interno. Cloudflare crea automáticamente el registro DNS del subdominio apuntando al túnel.

| Subdominio | Servicio interno | Pasa por NPM |
|---|---|---|
| `nextcloud.yocolab.dev` | `http://192.168.100.5:80` (Nginx Proxy Manager → Nextcloud) | Sí |
| `home.yocolab.dev` | `http://192.168.100.4:3000` (Homepage) | No |

El tramo entre el navegador y Cloudflare viaja cifrado con HTTPS. El tramo entre `cloudflared` y el servicio viaja por HTTP dentro de la LAN.

Las dos rutas siguen criterios distintos: Nextcloud entra por Nginx Proxy Manager, que lo enruta según el nombre de host, y Homepage va directo al servicio ([guía de NPM](../nginx-proxy-manager/README.md#relación-con-el-túnel-de-cloudflare)).

> Al principio también estaban publicados los paneles de Nginx Proxy Manager, Uptime Kuma y Pi-hole. Se retiraron del túnel porque publicar paneles de administración en Internet amplía la superficie de ataque. Hoy solo se alcanzan desde la LAN o por Tailscale.

## Controles en el borde

El túnel permite **llegar** a los servicios, pero no decide **quién** puede hacerlo. Esos controles se aplican en Cloudflare, antes de que la solicitud entre al túnel.

### Homepage: Cloudflare Access

Homepage muestra la estructura interna del laboratorio (servicios, nombres e IP), así que no debe quedar abierto a cualquiera. Está protegido con **Cloudflare Access**, dentro de Cloudflare Zero Trust (plan *Free*, equipo `yocolab`):

- **Aplicación** de tipo *Self-hosted* para `home.yocolab.dev`.
- **Política** `Auth`: acción *Allow*, con una regla que incluye solo el correo autorizado. Access funciona con denegación por defecto: si ninguna política permite el acceso, se bloquea.
- **Proveedor de identidad:** Google, como único método. Se configuró con un cliente OAuth propio en Google Cloud (tipo *Aplicación web*, audiencia externa en modo de prueba), con la URI de redirección `https://yocolab.cloudflareaccess.com/cdn-cgi/access/callback`. Como hay un solo método, Access redirige directo al inicio de sesión de Google.

El ID y el secreto del cliente OAuth se guardan solo en Cloudflare y en Google Cloud.

Este enfoque corresponde al modelo **Zero Trust**: conocer la dirección no basta para entrar; cada acceso se valida según identidad y política.

### Nextcloud: login propio, 2FA y WAF

Nextcloud **no** está detrás de Access. Lo estuvo al principio, pero se retiró porque los clientes de escritorio y móvil dejaban de sincronizar: después de iniciar sesión, hablan directo con la API de Nextcloud sin la cookie de Access, y Access les responde con una redirección a Google que no pueden completar. Excluir la API de Access con una política *Bypass* se descartó, porque dejaría sin proteger justamente la parte más expuesta ([caso completo](../../docs/troubleshooting/nextcloud-clientes-cloudflare-access.md)).

En su lugar:

- **Autenticación de Nextcloud:** 2FA y su protección integrada contra fuerza bruta.
- **Regla WAF** en **Security → Security rules → Custom rules**, con acción **Block**:

  ```
  (http.host eq "nextcloud.yocolab.dev" and ip.src.country ne "CL")
  ```

  Bloquea en el borde de Cloudflare toda solicitud a Nextcloud que no venga de Chile, antes de que entre al túnel. El filtro por `http.host` la limita a Nextcloud. Se usa *Block* y no *Managed Challenge* porque el desafío es una página interactiva que los clientes tampoco pueden resolver.

## Verificación

En el LXC:

```bash
systemctl status cloudflared                  # active (running)
journalctl -u cloudflared -n 20 --no-pager    # conexiones registradas con Cloudflare
```

En el panel de Cloudflare: el túnel en **Healthy**, con su conector.

Desde fuera de la casa:

- `https://home.yocolab.dev` redirige al inicio de sesión de Google antes de mostrar Homepage.
- `https://nextcloud.yocolab.dev` muestra el inicio de sesión de Nextcloud, con certificado válido emitido para el dominio.
- En **Security → Events** aparecen las solicitudes a Nextcloud bloqueadas por la regla WAF, y la aplicación móvil sigue sincronizando desde datos móviles.

## Actualización

```bash
apt update && apt upgrade -y
systemctl restart cloudflared
```

## Problemas conocidos

- [Los clientes de Nextcloud no sincronizan con Cloudflare Access](../../docs/troubleshooting/nextcloud-clientes-cloudflare-access.md)
- Comandos pegados en una sola línea: los comentarios `#` anulan el resto (ver [paso 3](#3-instalar-cloudflared)).

## Limitaciones

- **Un solo conector:** si el LXC o su conexión se caen, Nextcloud y Homepage dejan de estar disponibles desde Internet.
- **Token en texto plano en el contenedor:** está en la unidad de systemd. Cualquiera con root en el CT 101 puede leerlo y suplantar el conector. Además, el token aparece en una captura del registro original de la instalación; si ese documento se comparte, hay que rotarlo desde Cloudflare.
- **Login de Nextcloud expuesto a Internet:** sin Access, la página de inicio de sesión y la API son alcanzables desde Chile, incluso desde una VPN con salida en Chile. La mitigan el 2FA, la protección contra fuerza bruta y la regla WAF, pero ninguna evita una vulnerabilidad de Nextcloud explotable antes del inicio de sesión: mantenerlo actualizado sigue siendo la medida principal.
- **Fuera de Chile no hay acceso a Nextcloud:** mientras la regla esté activa, la sincronización y los enlaces compartidos no funcionan desde el extranjero.
- **Nextcloud pasa por un contenedor privilegiado:** la ruta de Nextcloud entra a Nginx Proxy Manager, que corre en un LXC privilegiado.
- **Tramo interno sin cifrar:** entre `cloudflared` y los servicios, el tráfico viaja por HTTP dentro de la LAN.
- **Rastro de la red ICS:** el `/etc/resolv.conf` del contenedor tiene `search mshome.net`, el dominio que asigna Windows con Internet Connection Sharing, en la sección que gestiona Proxmox. Es un sobrante de la etapa inicial del laboratorio ([guía de Proxmox](../../docs/proxmox-base.md#etapa-inicial-conexión-compartida-de-windows-ics)); lo mismo ocurre en los CT 102 y 103.
- **Mantenimiento:** 24 paquetes pendientes y sin `unattended-upgrades`. SSH acepta contraseña para usuarios normales; root solo entra con clave.

## Pendiente

- [ ] Unificar el criterio de las rutas: todas por Nginx Proxy Manager o todas directas al servicio.
- [ ] Quitar el dominio de búsqueda `mshome.net` de la configuración DNS.
- [ ] Monitor en Uptime Kuma para el túnel o las URL públicas.
- [ ] Instalar `unattended-upgrades`.
- [ ] Evaluar un segundo conector, por ejemplo en otro LXC, para que el acceso desde Internet no dependa de un solo contenedor.

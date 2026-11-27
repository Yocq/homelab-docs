# Homelab

Laboratorio de infraestructura doméstico sobre **Proxmox VE**, operado como un entorno de servicios permanente. Lo uso para practicar virtualización, redes, contenedores y otras cosas. Además de documentar cómo se diseña, se despliega y se diagnostica cada componente.

> Es un entorno de laboratorio, no de producción. Las decisiones priorizan aprender y documentar con un presupuesto bajo, y cada una explica sus limitaciones.

---

## Arquitectura

```mermaid
%%{init: {"flowchart": {"curve": "step", "nodeSpacing": 30, "rankSpacing": 50}}}%%
flowchart LR
    internet((Internet))
    cf["Cloudflare<br/>Tunnel · Access · WAF"]
    ts["Tailscale<br/>dispositivos remotos"]

    clientes["Clientes LAN<br/>192.168.100.0/24"]
    subgraph pve["Proxmox VE · 192.168.100.2"]
        cfd["LXC cloudflared<br/>.3"]
        npm["LXC Nginx Proxy Manager<br/>.5"]
        tsr["LXC Tailscale<br/>subnet router · .7"]
        smb["LXC Samba<br/>.6"]
        subgraph vm["VM Docker · .4"]
            home[Homepage]
            nc["Nextcloud + MariaDB"]
            kuma[Uptime Kuma]
            port[Portainer]
            pihole[Pi-hole]
        end
    end

    internet --> cf --> cfd
    cfd -->|home.yocolab.dev| home
    cfd -->|nextcloud.yocolab.dev| npm
    npm --> nc
    nc -->|SMB| smb
    clientes -.->|"*.yocolab"| npm
    npm -.-> kuma & port & pihole
    clientes -.->|DNS| pihole
    ts -.-> tsr -.->|acceso a la LAN| clientes
```

Líneas continuas: tráfico que entra desde Internet por el túnel. Líneas punteadas: uso desde la LAN o por Tailscale.

> [!NOTE]
> Este diagrama se reemplazará por uno hecho en Visio.

**Dos caminos de entrada, con propósitos distintos:**

- **Cloudflare Tunnel**: publica en Internet solo las aplicaciones web de uso diario (Nextcloud y Homepage). No hay puertos abiertos en el router, algo necesario porque la conexión está detrás de CGNAT.
  - **Homepage** está detrás de **Cloudflare Access**: antes de llegar al panel se exige autenticación con Google.
  - **Nextcloud** usa su propio inicio de sesión con **2FA** y una **regla WAF de Cloudflare** que solo admite tráfico desde Chile. Access se retiró porque impedía iniciar sesión con los clientes móvil.
- **Tailscale (subnet router)**: administración remota. Proxmox, SSH, Samba y los paneles de administración **no** se publican en Internet; se alcanzan por VPN como si estuviera en la LAN.

## Hardware

Instalación y configuración: [Proxmox base](docs/proxmox-base.md) · [VM Docker](docs/vm-docker.md).

| Componente | Detalle |
|---|---|
| Equipo | Lenovo ThinkCentre M75q-1 (mini-PC) |
| CPU | AMD Ryzen 3 3300GE, 4 núcleos / 8 hilos, AMD-V |
| RAM | 24 GB DDR4-2666 |
| SSD 1 TB | Proxmox, VM y LXC — pool ZFS `rpool` (un solo disco) |
| HDD 2 TB | ext4 en `/mnt/storage`: ISOs, respaldos, plantillas y datos de usuario |

## Servicios

| Servicio | Plataforma | Función | Acceso |
|---|---|---|---|
| [Pi-hole](services/pihole/) | VM Docker | DNS interno, filtrado, wildcard `*.yocolab` | LAN / Tailscale |
| [Uptime Kuma](services/uptime-kuma/) | VM Docker | Monitoreo de disponibilidad | LAN / Tailscale |
| [Portainer](docs/vm-docker.md#portainer) | VM Docker | Administración de Docker | LAN / Tailscale |
| Nextcloud + MariaDB | VM Docker | Almacenamiento y colaboración | Internet (login + 2FA, WAF Chile) |
| Homepage | VM Docker | Panel de accesos | Internet (Access) |
| [Nginx Proxy Manager](services/nginx-proxy-manager/) | LXC | Reverse proxy por nombre de host | LAN / Tailscale |
| Samba | LXC | Recursos SMB para Windows y Nextcloud | LAN / Tailscale |
| cloudflared | LXC sin privilegios | Conector del túnel de Cloudflare | — |
| [Tailscale](services/tailscale/) | LXC sin privilegios | VPN y subnet router de la LAN | — |

## Decisiones de diseño

- **Docker dentro de una VM, no en el host.** El hipervisor queda limpio y la VM se puede respaldar, restaurar o reemplazar como una unidad.
- **Servicios de borde en LXC separados** (proxy, túnel, VPN). Siguen funcionando aunque la VM de aplicaciones se reinicie.
- **Cloudflare Tunnel en lugar de abrir puertos.** Funciona bajo CGNAT, no expone la IP del hogar y permite agregar controles en el borde (Access, WAF) sin tocar el servidor.
- **Tailscale para administrar, no el túnel.** Las interfaces de administración nunca quedan publicadas en Internet.
- **SSD para cargas activas, HDD para datos y respaldos.** El SSD queda reservado para lo que necesita baja latencia.

## Troubleshooting

Casos reales encontrados durante el montaje, documentados como problema → diagnóstico → causa → solución.

- [Docker no puede montar overlay dentro de un LXC](docs/troubleshooting/docker-overlay-en-lxc.md)
- [La contraseña definida en el compose no funciona en Pi-hole](docs/troubleshooting/pihole-contrasena-web.md)
- [Pi-hole ignora las consultas DNS de la LAN ("non-local network")](docs/troubleshooting/pihole-non-local-network.md)
- [Tailscale no inicia en un LXC sin privilegios (`/dev/net/tun`)](docs/troubleshooting/tailscale-dev-net-tun.md)
- [Se pierde el acceso remoto: clave del subnet router de Tailscale expirada](docs/troubleshooting/tailscale-clave-expirada.md)

## Roadmap

**Hecho**
- [x] Proxmox VE con ZFS (compresión LZ4, autotrim) e HDD integrado como almacenamiento
- [x] VM Docker con Pi-hole, Uptime Kuma, Portainer, Nextcloud y Homepage
- [x] Reverse proxy con resolución wildcard en Pi-hole
- [x] Acceso remoto: Cloudflare Tunnel (Access en Homepage, 2FA + WAF en Nextcloud) y Tailscale con subnet routing

**Siguiente**
- [ ] Respaldos de todos los LXC con política de retención
- [ ] Copia de los datos de usuario fuera del HDD y prueba de restauración
- [ ] Alertas en Uptime Kuma en Telegram
- [ ] Laboratorios de redes con Containerlab
- [ ] Diagrama de arquitectura en Visio

## Estructura del repositorio

```
.
├── README.md
├── docs/
│   ├── proxmox-base.md    # instalación y configuración del hipervisor
│   ├── vm-docker.md       # VM de contenedores, Docker y Portainer
│   └── troubleshooting/   # casos documentados + plantilla
└── services/              # un directorio por servicio: README, compose.yaml, .env.example
    ├── nginx-proxy-manager/
    ├── pihole/
    ├── tailscale/
    └── uptime-kuma/
```

> Credenciales, tokens y archivos `.env` no se versionan. Cada servicio incluirá un `.env.example` con las variables necesarias.

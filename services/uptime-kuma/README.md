# Uptime Kuma

Monitoreo de disponibilidad de los servicios del laboratorio. Comprueba cada servicio a intervalos regulares, registra su estado y el tiempo de respuesta, y concentra todo en un panel.

| | |
|---|---|
| **Plataforma** | VM Docker — `192.168.100.4` |
| **Puerto** | `3001` (web) |
| **Persistencia** | `/opt/docker/uptime-kuma/uptime-kuma` → `/app/data` |
| **Acceso** | `http://192.168.100.4:3001` · `http://kuma.yocolab` (vía Nginx Proxy Manager) |

## Por qué Uptime Kuma

Con varios servicios funcionando, revisar a mano cada interfaz deja de ser práctico. Uptime Kuma responde a una pregunta simple, si el servicio está arriba y responde, y se configura en minutos.

No reemplaza a una plataforma de métricas como Prometheus + Grafana, que guarda series históricas de CPU, memoria o disco. En esta etapa del laboratorio hace falta saber **si algo se cayó**, no analizar tendencias.

## Requisitos

- VM Docker con Docker Engine y el plugin Compose.
- Puerto `3001` libre en la VM.

## Despliegue

### 1. Crear el directorio del servicio

```bash
sudo mkdir -p /opt/docker/uptime-kuma/uptime-kuma
cd /opt/docker/uptime-kuma
```

`uptime-kuma/` contiene la base de datos interna: los monitores, los usuarios y el historial de disponibilidad. Sin este volumen, recrear el contenedor borra toda la configuración.

### 2. Crear `compose.yaml`

Copia [`compose.yaml`](compose.yaml) al directorio. El servicio no necesita variables de entorno ni secretos: el usuario administrador se crea desde la interfaz web en el primer acceso.

### 3. Levantar el contenedor

```bash
docker compose up -d
docker compose ps        # estado esperado: running (healthy)
```

### 4. Primer acceso

Abre `http://192.168.100.4:3001` y crea el usuario administrador.

### 5. Publicación por nombre (opcional)

En Nginx Proxy Manager se crea el proxy host `kuma.yocolab` → `http://192.168.100.4:3001` con **Websockets Support** activado. Uptime Kuma actualiza el estado de los monitores en tiempo real por WebSockets; sin esa opción el panel carga, pero no se actualiza.

## Monitores

| Monitor | Tipo | Objetivo | Qué valida |
|---|---|---|---|
| Pi-hole | HTTP | `http://192.168.100.4:8081/admin` | Que el panel web responde |
| Pi-hole_DNS | DNS | `google.com` resuelto por `192.168.100.4:53` | Que Pi-hole resuelve consultas, que es su función principal |

<!-- TODO: agregar el resto de los monitores configurados actualmente -->

Pi-hole tiene dos monitores porque el panel y el motor DNS pueden fallar por separado. Un monitor HTTP en verde no garantiza que la resolución DNS funcione.

## Verificación: prueba de caída controlada

Un monitor que nunca se probó en rojo no demuestra que sepa detectar una caída. Por eso se hizo una prueba deliberada:

1. Con el monitor de Pi-hole en estado **Funcional**, se detuvo el contenedor `pihole` desde Portainer.
2. Uptime Kuma marcó el monitor en **DOWN** en el siguiente ciclo de comprobación, con el error:

   ```
   [DOWN] connect ECONNREFUSED <IP_VM>:8081
   ```

3. Se volvió a iniciar el contenedor y el monitor regresó a **Funcional**.

`ECONNREFUSED` significa que la VM respondió, pero que ningún proceso escuchaba en el puerto 8081. Ese error distingue "el servicio se detuvo" de "el host no responde", que se vería como un timeout.

<!-- TODO: agregar capturas img/monitor-pihole-down.png y img/monitor-pihole-up.png -->

El monitor DNS se validó con la respuesta real: estado **Funcional**, latencia de alrededor de 1 ms en la LAN y varios registros A devueltos para `google.com`. Eso confirma el ciclo completo: Pi-hole recibe la consulta, la reenvía al servidor upstream y devuelve la respuesta.

## Actualización

```bash
cd /opt/docker/uptime-kuma
docker compose pull
docker compose up -d
```

## Limitaciones

- **No avisa:** no hay notificaciones configuradas. Una caída solo se ve si alguien abre el panel.
- **Monitorea desde adentro:** corre en la misma VM que la mayoría de los servicios que vigila. Si la VM Docker se cae, Uptime Kuma se cae con ella y no registra nada. Tampoco puede detectar la caída del host completo.
- **Cobertura parcial:** solo se monitorea Pi-hole. Nginx Proxy Manager, Nextcloud, Samba, el túnel y Proxmox todavía no tienen monitor.

## Pendiente

- [ ] Notificaciones por Telegram.
- [ ] Monitores para el resto de los servicios: Proxmox (`:8006`), Nginx Proxy Manager, Nextcloud, Samba (TCP `445`), Homepage y la URL pública de Nextcloud.
- [ ] Mover Uptime Kuma fuera de la VM Docker, a una VM de monitoreo junto con Prometheus + Grafana, para que la caída de la VM de aplicaciones sí genere una alerta.
- [ ] Chequeo externo (*heartbeat*) para detectar la caída del host completo.
- [ ] Fijar la versión de la imagen en lugar de `latest`: confirmar la versión instalada (en el panel, **Settings → About**) y usar esa etiqueta.

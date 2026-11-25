# Se pierde el acceso remoto: clave del subnet router expirada

| | |
|---|---|
| **Componente** | LXC-Tailscale (CT 104), subnet router de `192.168.100.0/24` — [guía](../../services/tailscale/README.md) |
| **Síntoma** | Desde fuera de casa no se alcanza ninguna IP del laboratorio |
| **Solución aplicada** | Volver a autenticar el nodo y deshabilitar la expiración de su clave |
| **Estado** | Resuelto |

## Contexto

Todo el acceso remoto al laboratorio (Proxmox, SSH, Samba, paneles de administración) pasa por un único nodo de Tailscale, el LXC 104, que anuncia la red `192.168.100.0/24` a la tailnet.

## Síntoma

El acceso remoto dejó de funcionar: desde los dispositivos de la tailnet ya no se alcanzaban las IP de la LAN.

<!-- TODO: completar con lo observado (mensaje en el cliente, estado del nodo en la consola) si se recuerda -->

## Diagnóstico

Al revisar el LXC, la sesión de Tailscale del nodo había **expirado**. En la consola de Tailscale, un nodo en este estado aparece marcado como expirado en **Machines**.

Dentro del LXC, el estado se comprueba con:

```bash
tailscale status
```

## Causa

Tailscale autentica cada nodo con una clave que **expira periódicamente** (por defecto, cada 180 días). Cuando vence, el nodo queda fuera de la tailnet hasta que se vuelve a autenticar.

En un equipo personal, la expiración obliga a revalidar de vez en cuando quién usa el dispositivo. En un subnet router el efecto es otro: **es un punto único de acceso**. Al expirar su clave se corta el acceso remoto a toda la LAN de una vez, y no se puede arreglar a distancia, porque el camino para llegar al LXC era justamente ese nodo.

## Solución

**1. Volver a autenticar el nodo**, desde la consola de Proxmox o en la LAN:

```bash
pct enter 104
tailscale up --force-reauth --advertise-routes=192.168.100.0/24
```

<!-- TODO: confirmar el comando exacto usado -->

`--force-reauth` fuerza un nuevo inicio de sesión aunque el nodo tenga una sesión guardada, y entrega una URL de autenticación.

Hay que repetir `--advertise-routes`: `tailscale up` aplica la configuración completa que recibe, y Tailscale rechaza el comando si se omite una opción que ya estaba configurada.

**2. Deshabilitar la expiración de la clave:** en la consola, **Machines → `lxc-tailscale` → `…` → Disable key expiry**.

## Verificación

```bash
tailscale status                                      # el nodo aparece conectado
tailscale debug prefs | grep -A2 AdvertiseRoutes      # → "192.168.100.0/24"
```

En la consola, el nodo muestra **Expiry disabled** y la ruta sigue aprobada. Desde un cliente fuera de casa, `https://192.168.100.2:8006` vuelve a abrir.

## Aprendizajes

- Una medida de seguridad pensada para equipos personales puede ser un punto de falla en infraestructura. Hay que revisar los valores por defecto según el rol del nodo.
- Un camino de acceso único hace que cualquier falla en él sea difícil de arreglar a distancia. Por eso la consola local de Proxmox, o un segundo camino de acceso, sigue siendo necesaria.
- Lo que se arregla una vez debería quedar en el procedimiento de instalación (paso 7 de la guía), no solo en la memoria.

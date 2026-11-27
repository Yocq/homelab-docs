# Los clientes de Nextcloud no sincronizan con Cloudflare Access

| | |
|---|---|
| **Componente** | Nextcloud publicado por Cloudflare Tunnel (`nextcloud.yocolab.dev`) — [guía de cloudflared](../../services/cloudflared/README.md) |
| **Síntoma** | Con Cloudflare Access activo, los clientes de escritorio y móvil de Nextcloud dejan de funcionar |
| **Solución aplicada** | Retirar Access de Nextcloud y restringir el acceso por país con una regla WAF |
| **Estado** | Resuelto |

## Contexto

Nextcloud se publicó en Internet con Cloudflare Tunnel y se protegió con **Cloudflare Access**, que exige autenticarse con Google antes de que la solicitud llegue al servicio. Desde el navegador funcionaba sin problemas.

## Síntoma

Al intentar iniciar sesión desde el equipo móvil, los clientes de Nextcloud (escritorio y móvil) no lograban sincronizar.

## Causa

La aplicación inicia sesión mediante **Login Flow v2**: abre el navegador para autenticar al usuario y, en ese paso, Access no interfiere porque la interacción es manual.

Pero después, la aplicación se comunica directamente con la API de Nextcloud (`/remote.php/dav`, `/ocs` y `/index.php/login/v2/poll`) usando una contraseña de aplicación, **sin la cookie de sesión de Access**. Access responde a esas solicitudes con una redirección al inicio de sesión de Google que el cliente no puede completar, por lo que la sincronización falla.

## Alternativa descartada

Se podían excluir esas rutas de Access con una política **Bypass**. Se descartó porque las rutas excluidas son justamente la API, la parte más expuesta del servicio: Access quedaría protegiendo solo la interfaz web, a cambio de una configuración más compleja.

## Solución

### 1. Retirar Access de Nextcloud

Se elimina la aplicación de Access para `nextcloud.yocolab.dev`. La autenticación queda a cargo de Nextcloud, con **autenticación de dos factores (2FA)** y su **protección integrada contra fuerza bruta**.

### 2. Restringir el acceso por país

Para reducir la exposición, se agrega una regla personalizada del WAF de Cloudflare en **Security → Security rules → Custom rules**, con acción **Block**:

```
(http.host eq "nextcloud.yocolab.dev" and ip.src.country ne "CL")
```

- La regla bloquea en el borde de Cloudflare todas las solicitudes a Nextcloud que no provienen de Chile, **antes de que entren al túnel**.
- El filtro por `http.host` limita la regla a Nextcloud y no afecta al resto de la zona `yocolab.dev`.
- Se usa **Block** en lugar de **Managed Challenge** porque el desafío es una página interactiva que los clientes de Nextcloud tampoco pueden resolver.

## Verificación

- En **Security → Events** aparecen las solicitudes bloqueadas por la regla.
- La aplicación móvil sigue sincronizando desde datos móviles.

## Limitaciones

- La regla reduce el tráfico automatizado, pero no protege frente a solicitudes desde IP chilenas ni desde VPN con salida en Chile.
- Tampoco evita una vulnerabilidad de Nextcloud explotable antes del inicio de sesión, por lo que **mantener el servicio actualizado sigue siendo la medida principal**.
- Fuera de Chile, la sincronización y los enlaces compartidos dejan de funcionar mientras la regla esté activa.

## Aprendizajes

- Un control de acceso pensado para personas (inicio de sesión en el navegador) no sirve para clientes que hablan directo con una API: hay que revisar **quién** se conecta a un servicio, no solo **cómo** se ve desde el navegador.
- Excluir rutas de una protección puede dejar sin proteger justamente lo más expuesto.
- Una restricción en el borde reduce la exposición, pero no reemplaza la seguridad del propio servicio.

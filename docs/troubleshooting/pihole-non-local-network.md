# Pi-hole ignora las consultas DNS de la LAN ("non-local network")

| | |
|---|---|
| **Componente** | Pi-hole v6 en Docker (red bridge), VM Docker |
| **Síntoma** | El panel funciona, pero las consultas DNS de los clientes terminan en timeout |
| **Solución aplicada** | Modo de escucha *Permit all origins* (`listeningMode: all`) |
| **Estado** | Resuelto |

## Contexto

Pi-hole se desplegó en la VM Docker con la red bridge por defecto, publicando el puerto 53 en la VM. Primero se probó como DNS desde un solo equipo, antes de configurarlo para toda la red.

## Síntoma

Desde el cliente Windows, todas las consultas a Pi-hole terminaban en timeout:

```powershell
nslookup google.com <IP_PIHOLE>
# DNS request timed out.
```

El Query Log del panel no mostraba ninguna consulta del cliente.

## Diagnóstico

**1. ¿El puerto está publicado?** Dentro de la VM, el puerto 53 escuchaba en todas las interfaces:

```bash
ss -tulpn | grep ':53 '
# udp  UNCONN  0.0.0.0:53 ...
# tcp  LISTEN  0.0.0.0:53 ...
```

Así se descartó un problema de Docker o de firewall: las consultas llegaban.

**2. ¿Qué hace Pi-hole con ellas?** El diagnóstico interno de Pi-hole (**Tools → Pi-hole diagnosis**) mostró:

```
ignoring query from non-local network 192.168.137.1
```

> La IP corresponde a la red en la que estaba el laboratorio en ese momento (la conexión compartida de Windows). Hoy el laboratorio está en 192.168.100.0/24.

## Causa

Pi-hole v6 trae por defecto el modo de escucha **LOCAL**: solo responde consultas que vienen de redes conectadas directamente a sus interfaces.

Con la red **bridge** de Docker, la única interfaz del contenedor está en la red interna de Docker (172.x). Desde su punto de vista, la LAN real no es una red local, así que descarta esas consultas como medida de seguridad. Lo hace sin enviar respuesta, y por eso el cliente ve un timeout y no un error.

## Solución

En el panel: **Settings → DNS → Interface settings → Permit all origins**.

Es el cambio que se aplicó, y queda guardado en `pihole.toml`. El equivalente en el compose, que permitiría reproducirlo sin pasar por la interfaz, todavía no está aplicado (ver pendientes de la [guía](../../services/pihole/README.md#otros-pendientes)):

```yaml
environment:
  FTLCONF_dns_listeningMode: "all"
```

Después del cambio, el Query Log empezó a registrar las consultas del cliente y a aplicar el bloqueo.

### Por qué es seguro en este caso

"Permit all origins" convierte a Pi-hole en un resolvedor abierto para cualquiera que alcance el puerto 53. Aquí no es un riesgo porque ese puerto no es accesible desde Internet: la conexión está detrás de CGNAT y no hay redirección de puertos. Si el puerto quedara publicado, el servidor podría usarse en ataques de amplificación DNS.

Una alternativa más restrictiva es `network_mode: host`. Así Pi-hole ve las interfaces reales de la VM y el modo LOCAL funciona, a cambio de perder el aislamiento de red del contenedor y el mapeo del puerto web a 8081.

## Resultado

Las consultas de los clientes se resuelven y aparecen en el Query Log con su IP de origen. Uptime Kuma mantiene un monitor DNS sobre Pi-hole que valida esto de forma continua.

### Nota sobre los timeouts parciales de `nslookup`

Aun después de la corrección, `nslookup` mostraba algunos timeouts. La causa más probable no es Pi-hole: `nslookup` intenta primero resolver el nombre del propio servidor DNS (consulta PTR de su IP), y si no hay registro inverso muestra `Server: UnKnown` y esperas. Para validar DNS en Windows es más confiable `Resolve-DnsName google.com -Server <IP_PIHOLE>`.

## Aprendizajes

- Un timeout no significa que el servicio esté caído. El puerto estaba abierto y el servicio activo; era una política la que descartaba los paquetes sin responder.
- En Docker con red bridge, "local" se define desde la red del contenedor, no desde la del host. Esto afecta a cualquier servicio que filtre por origen.
- Abrir una política restrictiva es aceptable solo si se entiende qué protegía y qué la reemplaza (aquí, que el puerto no sea alcanzable desde fuera).

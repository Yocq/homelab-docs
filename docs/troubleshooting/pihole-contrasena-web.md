# La contraseña definida en el compose no funciona en Pi-hole

| | |
|---|---|
| **Componente** | Pi-hole v6 en la VM Docker |
| **Síntoma** | La contraseña de `WEBPASSWORD` en el compose es rechazada en el panel web |
| **Solución aplicada** | `pihole setpassword` dentro del contenedor; luego, la variable correcta de v6 |
| **Estado** | Resuelto en la práctica; el compose sigue con la variable de v5 (pendiente) |

## Contexto

El primer despliegue de Pi-hole definía la contraseña del panel con la variable que aparece en la mayoría de las guías:

```yaml
environment:
  WEBPASSWORD: <contraseña>
```

## Síntoma

El contenedor inicia sin errores y el panel carga en `http://<IP>:8081/admin`, pero la contraseña es rechazada aunque coincide con la del compose.

## Diagnóstico

1. Se descartó un problema de red: el panel responde, así que el contenedor y el puerto funcionan.
2. Se descartó un error al escribir: la contraseña se copió directamente del archivo.
3. Al revisar la documentación de Pi-hole v6 apareció la causa: la variable cambió de nombre.

Los logs lo confirman. Si no hay una contraseña válida, Pi-hole genera una aleatoria y la registra:

```bash
docker logs pihole | grep -i random
```

## Causa

Pi-hole v6 cambió su sistema de configuración y renombró las variables de entorno. `WEBPASSWORD` pasó a ser `FTLCONF_webserver_api_password`. La imagen v6 ignora la variable antigua y, como no hay ninguna contraseña definida, asigna una aleatoria.

> **Corrección respecto a la primera documentación:** en un principio atribuí el problema a que `WEBPASSWORD` "solo se usaba en la creación inicial del volumen". Esa explicación era incorrecta: la variable no tiene efecto en v6.

## Solución

### Solución inmediata (la que se aplicó)

Cambiar la contraseña desde dentro del contenedor:

```bash
docker exec -it pihole pihole setpassword
```

La contraseña se guarda como hash en `/etc/pihole/pihole.toml`. Como ese directorio es un volumen persistente, el cambio sobrevive a reinicios y a la recreación del contenedor.

### Solución definitiva (pendiente)

Todavía no está aplicada. Consiste en usar la variable de v6, con la contraseña en un `.env` que no se versiona:

```yaml
environment:
  FTLCONF_webserver_api_password: ${PIHOLE_WEB_PASSWORD}
```

```bash
docker compose up -d     # recrea el contenedor con la nueva variable
```

## Resultado

El acceso al panel funciona con la contraseña definida con `pihole setpassword`. El compose desplegado todavía conserva `WEBPASSWORD`, que no tiene efecto; actualizarlo está en los pendientes de la [guía de Pi-hole](../../services/pihole/README.md#otros-pendientes).

## Aprendizajes

- Cuando una variable "no hace nada", lo primero es verificar que exista en la versión instalada. Las guías de Internet suelen ser de versiones anteriores.
- `latest` puede traer cambios incompatibles en cualquier actualización. Para un servicio crítico conviene fijar la versión de la imagen y leer las notas antes de actualizar.
- Una explicación que suena razonable no reemplaza a la documentación: la primera hipótesis era plausible, pero no era la causa.

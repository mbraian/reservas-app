# app-reservas-frontend

Interfaz web para reservar salas: muestra una grilla de salas por franja horaria para la fecha
elegida, y permite reservar, cancelar y administrar salas. Es una SPA escrita en
**React + Vite**.

No conoce la URL de la API: pide siempre rutas relativas a `/api/...` de su propio origen, y
quien reenvía esas requests al backend es el servidor web que sirve la SPA.

## Cómo correrlo

Necesita **la API corriendo en `http://localhost:3001`**. Con eso:

```bash
npm install
npm run dev
```

Abre en **http://localhost:5173**. El servidor de desarrollo de Vite ya reenvía `/api/...` a
`localhost:3001`; está configurado en [`vite.config.js`](vite.config.js).

Para generar los archivos estáticos de producción:

```bash
npm run build      # deja el resultado en dist/
npm run preview    # sirve dist/ para revisarlo
```

## Servido como sitio estático

`dist/` son archivos estáticos: los sirve cualquier servidor web. En [`nginx/templates/`](nginx/templates/)
está la configuración de nginx que hace falta, y que resuelve tres cosas:

- reenvía `/api/...` al backend;
- responde `/healthz` con `ok`;
- manda cualquier otra ruta a `index.html`, porque el ruteo lo hace la SPA.

Esa configuración es una **plantilla**: toma el destino del backend de dos variables de entorno,
y hay que reemplazarlas al arrancar el servidor (la imagen oficial de nginx lo hace sola con
`envsubst` sobre `/etc/nginx/templates/*.template`).

| Variable | Ejemplo | Para qué |
|---|---|---|
| `API_HOST` | `reservas-api` | Host del backend |
| `API_PORT` | `3000` | Puerto del backend |

> nginx resuelve el nombre de `API_HOST` **una sola vez, al arrancar**. Si en ese momento no
> resuelve, el proceso no arranca: falla con `host not found in upstream`. El backend tiene que
> estar levantado y ser alcanzable por ese nombre antes de arrancar el servidor web.

El sitio escucha en el puerto **8080**.

## Estructura

```
src/App.jsx           pantalla principal
src/api.js            cliente HTTP contra /api
src/components/       grilla de reservas, selector de fecha y modales
nginx/templates/      configuración de nginx para servir dist/
vite.config.js        puerto de desarrollo y proxy hacia la API
```

### Justificación técnica: Uso obligatorio de `NGINX_ENVSUBST_FILTER` en el Dockerfile

En la imagen `reservas-frontend`, la configuración de Nginx se parametriza en tiempo de ejecución utilizando una plantilla procesada por el script de inicio de la imagen base (`20-envsubst-on-templates.sh`).

La variable de entorno `NGINX_ENVSUBST_FILTER=^API_` es de carácter **estrictamente obligatorio** debido a que la herramienta `envsubst` reemplaza por defecto cualquier patrón que comience con el signo `$` en el archivo procesado.

En un archivo de configuración de Nginx conviven dos tipos de variables:
1. **Variables de entorno de la infraestructura:** `${API_HOST}` y `${API_PORT}`, destinadas a resolverse dinámicamente según el entorno.
2. **Variables internas del servidor web Nginx:** `$uri`, `$host` y `$proxy_add_x_forwarded_for`, necesarias para el enrutamiento de la SPA y el reenvío de encabezados HTTP hacia el backend.

Si no se define `NGINX_ENVSUBST_FILTER`:
* `envsubst` intenta resolver las variables internas de Nginx contra el entorno del sistema operativo.
* Al no existir variables del sistema llamadas `uri` o `host`, son sustituidas por cadenas vacías `""`.
* Esto genera una configuración sintácticamente inválida o mutilada de forma silenciosa, rompiendo directivas críticas como `try_files` y la cabecera `Host` del proxy inverso.

Al definir `NGINX_ENVSUBST_FILTER=^API_`, se instruye a \`envsubst\` a sustituir exclusivamente aquellas variables cuyo nombre inicie con el prefijo `API_`, preservando intactas las variables nativas de Nginx.
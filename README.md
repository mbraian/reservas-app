# Reservas App

Esta es una aplicación de reserva de salas y laboratorios por franja horaria. Está compuesta por tres capas: una interfaz web en React, una API REST en Node.js con Express, y una base de datos MySQL 8.4.

## Arquitectura

    navegador



    
    reservas-frontend
    React + Vite
    nginx:8080
    (host 3000)
    
    reservas-api
    Node 20 + Express
    :3000
    (host 3001, solo
    depuración)
    
    reservas-db
    MySQL 8.4
    :3306
    (sin puerto
    publicado)

| Capa | Imagen | Puerto interno | Puerto publicado |
|---|---|---|---|
| reservas-frontend | React + Vite servido por nginx | 8080 | 3000 |
| reservas-api | Node 20 + Express | 3000 | 3001 (solo depuración) |
| reservas-db | MySQL 8.4 | 3306 | - |

El frontend es el único punto de entrada: nadie le habla a la base directamente, y a la API le habla el frontend.

## Medición y Comparativa de Imágenes

A continuación se presenta la comparativa cuantitativa de pesos entre la versión inicial ingenua y las imágenes optimizadas de producción:
| Imagen / Componente | Etiqueta | Imagen Base | Tamaño en Disco (`DISK USAGE`) | Tamaño Comprimido (`CONTENT SIZE`) | % Reducción (vs. Naive) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`reservas-api` (Ingenua)** | `:naive` | `node:20` (Debian) | **1.59 GB** | 393 MB | *Línea de base (0%)* |
| **`reservas-api` (Definitiva)** | `:v1` / `:v2` | `node:20-alpine` | **205 MB** | 50.7 MB | **87.11 %** |
| **`reservas-frontend`** | `:v1` | `nginxinc/nginx-unprivileged:1.27-alpine` | **76.2 MB** | 21.9 MB | **95.21 %** |

---

### Justificación de Decisiones de Arquitectura
Cada reducción en el tamaño de las imágenes responde a una decisión técnica deliberada:
1. **Adopción de Alpine Linux (`node:20-alpine` en lugar de `node:20`):** Reemplazó una distribución Debian completa con herramientas de compilación (\`gcc\`, \`make\`, \`python3\`) y librerías de desarrollo innecesarias en producción por un sistema operativo base de \~5 MB, eliminando más de 1.3 GB de peso muerto.

2. **Instalación determinista con `npm ci --omit=dev`:** Garantizó una instalación limpia y reproducible a partir de \`package-lock.json\`, omitiendo todas las dependencias declaradas en \`devDependencies\` (linters, suites de test, utilidades de desarrollo) que no intervienen en el runtime de producción.

3. **Multi-Stage Build en el Backend (\`deps\` + Etapa Final):** Aisló el proceso de instalación en una etapa constructora efímera. La imagen final solo importó el directorio \`/app/node\_modules\` compilado, descartando las cachés internas de npm (\`\~/.npm\`) y archivos temporales generados durante la instalación.

4. **Configuración de \`.dockerignore\`:** Evitó transferir al demonio de Docker y a las capas de la imagen el directorio local \`node\_modules\` (compilado para la arquitectura del host en macOS), el historial de control de versiones (\`.git\`) y archivos de variables de entorno locales (\`.env\`).

5. **Multi-Stage Fuerte en el Frontend (Node Build + Nginx Unprivileged Runtime):** Separó radicalmente la etapa de compilación (React + Vite con Node.js) del entorno de ejecución final. La imagen final solo contiene los artefactos estáticos minificados (\`dist/\`) servidos por Nginx, eliminando por completo a Node, npm, \`node\_modules\` y el código fuente original del contenedor de producción.

6. **Principio de Menor Privilegio (\`USER node\` y \`USER nginx\`):** Aunque no altera el tamaño físico de la imagen (costo de 0 bytes), degradó los privilegios del proceso en tiempo de ejecución, impidiendo que el contenedor opere como \`root\` (UID 0) y reduciendo el radio de daño ante una eventual brecha de seguridad.

## Ejecucion del contenedor de Bases de Datos MYSQL

### COMANDO:

```docker run -d --name reservas-db --network reservas-net -v reservas-db-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=reservas_root_pass -e MYSQL_DATABASE=reservasdb -e MYSQL_USER=reservas_user -e MYSQL_PASSWORD=reservas_pass mysql:8.4```

### ID del contenedor
d336a439e10e54fad41abf6c18d4267fe11959e85bd396e18225d6925a5e1c28

### Criterio de ejecucion sin parametro `-p`:
La base de datos constituye la capa más interna y crítica de la arquitectura. Su único cliente es la API (`reservas-api`), la cual reside dentro de la red aislada `reservas-net`. Omitir el parámetro `-p` garantiza que no se exponga ningún puerto en las interfaces de red del host anfitrión (evitando reglas NAT innecesarias y riesgos de seguridad perimetral), forzando a que todo acceso ocurra exclusivamente de forma interna a través del switch virtual y la resolución DNS de Docker.

## Documentación del Orden de Arranque y Modos de Falla

### Secuencia Estricta de Inicialización
Para desplegar la aplicación multicapa de forma exitosa mediante Docker CLI, debe respetarse estrictamente el siguiente orden de aprovisionamiento:

1. **Red (`reservas-net`):** Provee el switch virtual bridge y el servidor DNS interno que permite la resolución de nombres entre contenedores.
2. **Volumen (`reservas-db-data`):** Garantiza el almacenamiento persistente con ciclo de vida independiente para el motor de base de datos.
3. **Base de Datos (`reservas-db`):** Inicializa el catálogo de MySQL 8.4 en `/var/lib/mysql` y abre el socket TCP definitivo en el puerto `3306`.
4. **Backend API (`reservas-api`):** Requiere que la base esté activa para resolver `DB_HOST=reservas-db`, conectarse al puerto `3306` y ejecutar la siembra del esquema de datos (`SEED_DEMO=true`).
5. **Frontend / Reverse Proxy (`reservas-frontend`):** Requiere que la API esté corriendo en la red para que Nginx pueda resolver el upstream `reservas-api:3000` durante su inicialización.

---

### Matriz de Impacto por Inversión de Pasos

| Inversión Realizada | Comportamiento Técnico y Mensaje de Error | Consecuencia Operativa |
| :--- | :--- | :--- |
| **Base antes que Red** | El demonio de Docker rechaza el flag `--network`.<br />`Error response from daemon: network reservas-net not found` | Falla inmediata: el contenedor de base de datos no llega a crearse. |
| **Base antes que Volumen** | Si no se precrea el volumen nombrado, Docker lo crea automáticamente, pero existe el riesgo crítico de generar un *bind mount* accidental si se añade una barra (`/`) o punto (`.`), o de omitir parámetros avanzados de drivers de almacenamiento. | Persistencia vulnerable o datos enlazados a rutas efímeras no deseadas. |
| **API antes que Base** | La API intenta resolver `reservas-db` o conectarse al puerto TCP `3306` mientras el motor aún no abrió el socket definitivo (`ready for connections`).<br />`Error: connect ECONNREFUSED 172.x.x.x:3306` | Falla no capturada en Express: el proceso de Node.js muere y el contenedor pasa a estado `Exited (1)`. |
| **Frontend antes que API** | Nginx parsea su configuración al iniciar e intenta resolver la IP del host definido en la directiva `proxy_pass http://reservas-api:3000/;`.<br />`nginx: [emerg] host not found in upstream "reservas-api"` | Nginx no reintenta resoluciones asíncronas: el proceso aborta inmediatamente con código de error `1` (`Exited (1)`). |

---

### Limitación Operativa del Despliegue Actual

Actualmente, todo este ordenamiento jerárquico, las configuraciones de puertos y las variables de entorno residen **exclusivamente en la memoria operativa de quien ejecuta los comandos en la terminal**, careciendo de un artefacto declarativo que estandarice la arquitectura. Asimismo, requiere que el operador vigile manualmente los logs del motor para detectar el momento exacto en que comienza a escuchar en el puerto `3306`.

Esta limitación se subsana en la etapa siguiente mediante **Docker Compose**, donde la totalidad de la infraestructura se declara en un archivo `compose.yaml` y la coordinación de arranque se automatiza mediante la directiva `depends_on` con la cláusula `condition: service_healthy`.

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

---

### Particularidades del Healthcheck de MySQL (`reservas-db`)

#### a) ¿Por qué se escribe `$$` y no `$` en la contraseña?

Si se utilizara un único signo `$`, Docker Compose interpretaría la variable como una interpolación a resolver en la máquina anfitriona al momento de leer el archivo YAML. Esto generaría dos problemas graves:

1. Si la variable no está exportada en el host ni en el archivo `.env`, Compose la sustituye por una cadena vacía, haciendo fallar el comando `mysqladmin` por contraseña inválida.
2. Si estuviera definida en el host, Compose resolvería el valor en texto plano dentro de la línea de comando del contenedor. Esto provocaría que la credencial secreta quedara expuesta en texto claro para cualquier usuario mediante `docker inspect reservas-db`.

Al escribir `$$`, Compose escapa el símbolo y escribe un único `$` literal en la definición interna del contenedor. De este modo, la interpolación no la realiza el anfitrión al leer el manifiesto, sino que la expande el intérprete de shell (`sh`) dentro del propio contenedor en tiempo de ejecución, leyendo la variable de entorno protegida en su propio espacio de nombres.

#### b) ¿Por qué hace falta `start_period: 40s`?

La primera inicialización del motor MySQL 8.4 sobre un directorio de datos vacío es un proceso pesado que involucra la creación de los espacios de tablas de InnoDB, diccionarios de sistema, certificados SSL y usuarios. Este arranque inicial puede demorar entre 20 y 35 segundos.

El parámetro `start_period: 40s` otorga una ventana de gracia inicial: durante esos primeros 40 segundos, las pruebas de salud fallidas no acumulan penalizaciones en el contador de reintentos (`retries: 20`). Sin este período de gracia, los intentos fallidos acumulados durante el arranque provocarían que Docker declarara el contenedor como `unhealthy` de forma prematura, aun cuando el motor se encontraba inicializando normalmente.

#### Particularidad adicional: `-h 127.0.0.1` vs `-h localhost`

La inicialización de MySQL ocurre en dos tiempos:

1. Primero arranca un servidor temporal sin soporte de red, que solo atiende peticiones locales a través de un archivo de socket UNIX para crear la base de datos y los usuarios.
2. Luego, detiene ese proceso y arranca el servidor definitivo que abre el puerto TCP `3306`.

En los clientes de MySQL de Linux, especificar `-h localhost` es una instrucción especial que desvía la conexión al socket UNIX local. Si el healthcheck usara `-h localhost`, el servidor temporal respondería positivamente y Docker marcaría la base como `healthy` prematuramente, provocando que la API intente conectarse por TCP y falle con `ECONNREFUSED`.

Al forzar `-h 127.0.0.1`, obligamos a que la conexión sea exclusivamente a través de la pila TCP/IP, garantizando que el ping solo devuelva éxito cuando el servidor definitivo esté listo para aceptar conexiones externas en el puerto 3306.

---

### Punto 3: Healthcheck de reservas-api sin dependencias adicionales

#### ¿Por qué no corresponde instalar `curl` en la imagen solo para el healthcheck?

Instalar paquetes adicionales como `curl` en una imagen de producción contradice las buenas prácticas de ingeniería y optimización aprendidas en la Semana 2:

1. **Minimización de la superficie de ataque:** Cada binario y librería de soporte instalada en el sistema operativo base amplía el número potencial de vulnerabilidades (CVEs) reportadas por escáneres de seguridad como Trivy o Docker Scout.
2. **Eficiencia y peso:** Aumenta innecesariamente el tamaño final de la imagen en disco y ralentiza las transferencias por red.
3. **Capacidades nativas del runtime:** Desde Node.js v18, la API estándar `fetch` se encuentra integrada de forma nativa en el motor. Por lo tanto, el propio runtime de la aplicación dispone de las herramientas necesarias para evaluar su salud mediante una petición HTTP local (`http://127.0.0.1:3000/health`) y emitir un código de salida (`process.exit(0)` si es exitoso, `process.exit(1)` ante fallas), haciendo redundante e indeseable la instalación de herramientas externas.

---

### Punto 5: depends_on con condición y asimetría de dependencias

La sincronización entre capas se implementó con `condition: service_healthy` en lugar de la forma corta `depends_on: [servicio]`. Los dos casos no son equivalentes y responden a problemas arquitectónicos distintos:

#### a) `reservas-api` respecto de `reservas-db`:

* **El problema:** La API abre un pool de conexiones TCP hacia MySQL en el puerto 3306 al instanciarse. La primera inicialización de MySQL es lenta y ocurre en dos tiempos.
* **Por qué no alcanza la forma corta:** La forma corta `depends_on: [reservas-db]` únicamente espera a que el contenedor de la base reciba la orden de inicio (`Started`), no a que el motor esté listo para aceptar conexiones (`ready for connections`). Si se usara la forma corta, la API intentaría conectarse a un puerto TCP aún no abierto y colapsaría con el error `ECONNREFUSED`.
* **El puente hacia Kubernetes:** La API cuenta con una lógica interna de reintento de conexión de aproximadamente 60 segundos. Este reintento de aplicación es fundamental porque en Kubernetes la directiva `depends_on` no existe; ante la falta de sincronización declarativa en el orquestador, los pods reintentan su conexión en tiempo de ejecución (evitando el estado `CrashLoopBackOff`).

#### b) `reservas-frontend` respecto de `reservas-api`:

* **El problema:** Nginx actúa como reverse proxy hacia la API (`proxy_pass http://reservas-api:3000`). En su arquitectura interna, Nginx resuelve las direcciones IP de sus *upstreams* por DNS **una única vez, en el instante exacto en que arranca su proceso principal**.
* **Por qué es obligatorio un `depends_on`:** Si el frontend arranca antes de que el contenedor `reservas-api` haya sido instanciado en la red virtual, el DNS de Docker no encuentra el host y Nginx colapsa de inmediato con el error fatal `host not found in upstream "reservas-api"`.
* **Aporte de `service_healthy`:** Para que Nginx no muera, alcanzaría con la forma corta (basta con que el contenedor de la API exista para que su nombre resuelva en el DNS). Sin embargo, utilizar `condition: service_healthy` garantiza además que la API esté plenamente operativa y respondiendo en el puerto 3000, evitando que los primeros usuarios que ingresen a la aplicación reciban respuestas de error `502 Bad Gateway`.

---

### Punto 7: Ciclo de Vida de los Datos — Diferencia entre `down` y `down -v`

En una arquitectura multicapa contenerizada existen dos planos con ciclos de vida desacoplados: el **plano efímero de cómputo** (contenedores y redes virtuales) y el **plano persistente de almacenamiento** (volúmenes de datos).

#### a) `docker compose down` (Apagado estándar y seguro)

* **Alcance:** Detiene y elimina únicamente los recursos efímeros: los contenedores de los tres servicios (`frontend`, `api` y `db`) y la red virtual `reservas-net`.
* **Comportamiento del dato:** **El volumen nombrado `reservas-db-data` no se elimina**. Los archivos del motor MySQL permanecen intactos en el host.
* **Uso operativo:** Es el comando habitual para finalizar una jornada de desarrollo o aplicar cambios de código. Al volver a ejecutar `docker compose up -d`, los contenedores se recrean, pero se reenganchan al volumen existente, manteniendo intactas todas las reservas creadas por los usuarios.

#### b) `docker compose down -v` (Destrucción total del entorno)

* **Alcance:** Elimina los recursos efímeros (contenedores y redes) y, mediante la bandera `-v` (`--volumes`), **destruye físicamente los volúmenes nombrados declarados en el archivo**.
* **Comportamiento del dato:** Toda la información transaccional de MySQL se destruye de forma irreversible.
* **Uso operativo:** Se utiliza únicamente cuando se necesita resetear el entorno a un estado virgen (por ejemplo, para probar migraciones de esquema desde cero o limpiar almacenamiento local). Al volver a levantar el stack con `docker compose up -d`, el volumen se crea vacío y la API se ve forzada a recrear las tablas y sembrar datos de prueba iniciales (`SEED_DEMO`).

---

### Punto 8: Gestión de Configuración y Secretos (.env vs Kubernetes)

#### El problema de los valores literales en producción

Mantener valores sensibles y parámetros de infraestructura directamente en el archivo `docker-compose.yml` presenta tres problemas críticos en entornos de producción:

1. **Fuga de secretos:** El archivo de composición se versiona en Git. Dejar contraseñas (`MYSQL_ROOT_PASSWORD`, `DB_PASSWORD`) en texto plano expone las credenciales a cualquier persona con acceso al repositorio o al historial de commits.
2. **Violación de inmutabilidad:** Impide utilizar el mismo artefacto en distintos entornos (desarrollo, testing, staging, producción). Cada entorno requiere configuraciones, credenciales y destinos de base de datos diferentes.
3. **Falta de auditoría:** No permite rotar contraseñas de forma independiente del ciclo de vida del código.

#### La solución intermedia con Compose: `.env` y `.env.example`

La práctica estándar en Docker Compose consiste en:

* Versionar un archivo `.env.example` que documenta el contrato de variables requerido y valores de referencia para el equipo.
* Crear localmente un archivo `.env` con los valores reales, excluido estrictamente del repositorio mediante `.gitignore`.
* En el `docker-compose.yml`, interpolar mediante la sintaxis `${VARIABLE}` o `${VARIABLE:?falta en .env}`.

#### El límite de `.env`: Por qué no es una solución completa

El archivo `.env` retira la contraseña de Git, pero **no la protege en tiempo de ejecución**:

* Al ejecutar `docker compose config` o `docker inspect <contenedor>`, las credenciales continúan mostrándose en texto plano en la máquina anfitriona.
* El archivo `.env` reside en el disco del host con los permisos estándar del usuario.

#### El Salto a Kubernetes (Semana 7): `ConfigMap` vs `Secret`

En la orquestación avanzada con Kubernetes, la gestión de configuración se desacopla en dos objetos nativos de primer nivel:

1. **`ConfigMap` (Configuración no sensible):**
   * Almacena datos ordinarios en pares clave-valor: nombres de host (`DB_HOST: reservas-db`), puertos (`PORT: 3000`), endpoints de upstream (`API_HOST`), flags de ejecución (`SEED_DEMO: "true"`) y versiones (`APP_VERSION`).
   * Permite modificar el comportamiento de la aplicación sin tener que recompilar las imágenes de contenedor.

2. **`Secret` (Información confidencial y credenciales):**
   * Almacena contraseñas, claves privadas, tokens y certificados (`MYSQL_ROOT_PASSWORD`, `DB_PASSWORD`).
   * **Seguridad reforzada:** Se almacena de forma aislada en la base distribuida `etcd` (pudiendo cifrarse en reposo) y su acceso se restringe mediante políticas de RBAC (Role-Based Access Control).
   * **Inyección segura:** Se monta dentro del Pod como un sistema de archivos temporal en memoria RAM (`tmpfs`), garantizando que la contraseña nunca se escriba en el disco del nodo ni quede expuesta en texto claro en los metadatos del Pod.

---

## Revisión de Seguridad (Punto 9)

| Componente | Usuario (no-root) | Puertos publicados | Exposición / Rol en la arquitectura | Vulnerabilidades Sistema Base (Trivy) | Vulnerabilidades Dependencias (App) | ¿Tienen versión corregida? (Fixed Version) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`reservas-db`** | `mysql` (UID 999/root en inicio) | **Ninguno** | Interno en `reservas-net`. Base aislada sin acceso desde el host. | — *(Imagen oficial)* | — | — |
| **`reservas-api`** | `node` (UID 1000) | `3001:3000` *(o 3030)* | Depuración local con `curl`. En semana 6 pasará a ser un `Service` interno (`ClusterIP`). | ~51 hallazgos (Alpine 3.23) | **0 hallazgos** (Node modules limpios) | Pocas con parche inmediato disponible. |
| **`reservas-frontend`** | `nginx` (UID 101) | `3000:8080` | **Único punto de entrada** para el navegador (Ingress / Reverse Proxy). | ~159 hallazgos (Alpine 3.21) | **0 hallazgos** (Multi-stage build eliminó Node/npm) | **Sí, la gran mayoría**. Indica que la imagen base de Alpine quedó desactualizada. |

---

\### Punto 10: Aplicación del Principio de Menor Privilegio (\`cap\_drop\` / \`cap\_add\`) Se aplicó la directiva \`cap\_drop: [ALL]\` a los tres servicios de la arquitectura para despojarlos de todas las capacidades por defecto del Kernel de Linux. #### 1\. Comportamiento de las imágenes propias (\`reservas-api\` y \`reservas-frontend\`) Ambos contenedores continuaron funcionando normalmente en estado \`healthy\` sin requerir ninguna capability. \* \*\*Causa:\*\* Fueron construidos en la Semana 2 bajo usuarios no privilegiados (\`USER node\` UID 1000 y \`USER nginx\` UID 101) y escuchan en puertos no privilegiados (&gt;1024). Al no ejecutar operaciones administrativas sobre el sistema de archivos ni cambiar de identidad en tiempo de ejecución, operan de forma segura con cero capabilities. #### 2\. Diagnóstico incremental de \`reservas-db\` (MySQL 8.4) Al aplicar \`cap\_drop: [ALL]\`, el contenedor de la base finalizó con código de error (\`exited (1)\`). Se diagnosticaron y agregaron de forma granular las capabilities requeridas leyendo el registro de error en cada paso: \* \*\*Paso 1:\*\* El comando inicial de verificación falló con: \`[ERROR] [MY-010126] [Server] setgid: Operation not permitted\` \* \*\*Solución:\*\* Se agregó \`cap\_add: [SETGID]\`, permitiendo al proceso invocar \`setgid()\` para cambiar al grupo de sistema \`mysql\`. \* \*\*Paso 2:\*\* Al reintentar, el proceso avanzó pero falló en la mutación de usuario: \`[ERROR] ... setuid: Operation not permitted\` \* \*\*Solución:\*\* Se agregó \`cap\_add: [SETGID, SETUID]\`, permitiendo invocar \`setuid()\` para adoptar la identidad del usuario sin privilegios \`mysql\` (UID 999). \* \*\*Resultado final:\*\* Con únicamente \`SETGID\` y \`SETUID\`, el motor MySQL 8.4 inició el servidor definitivo, abrió el puerto 3306 y alcanzó el estado \`healthy\`. #### 3\. ¿Por qué la base de datos requiere capabilities y las otras dos capas no? A diferencia de nuestras aplicaciones Node.js y Nginx, que nacen directamente con su usuario final asignado en el Dockerfile, la imagen oficial de MySQL está diseñada para gestionar la inicialización de volúmenes persistentes. Para ello, su script de entrada arranca inicialmente como \`root\` dentro del contenedor y luego traspasa la ejecución del proceso al usuario \`mysql\`. Esa transición de privilegios en tiempo de ejecución (\`root\` -&gt; \`mysql\`) exige llamadas protegidas al Kernel (\`setuid\`/\`setgid\`) que requieren explícitamente dichas capacidades.

---

\### Punto 11 - Comparativa Arquitectónica: docker-compose.yml vs docker run (Semana 3 vs Semana 4) El paso de la ejecución imperativa mediante comandos individuales \`docker run\` hacia la orquestación declarativa con \`docker compose\` redefine la gestión de la infraestructura de \`reservas-app\`. A continuación se detalla qué desapareció, qué apareció y la correspondencia directa entre ambas técnicas. ### 1\. ¿Qué desapareció? \* \*\*La memoria del operador y el orden manual:\*\* En la Semana 3, el orden de inicio (\`red ──► volumen ──► db ──► api ──► frontend\`) residía exclusivamente en la cabeza de quien operaba la terminal. Un error en la secuencia rompía la aplicación. En Compose, el orden lo resuelve el orquestador mediante un Grafo Acíclico Dirigido (DAG). \* \*\*La espera humana activa:\*\* Se eliminó la necesidad de monitorear manualmente \`docker logs -f reservas-db\` esperando el mensaje \`port: 3306\` para recién escribir el siguiente comando. \* \*\*Los comandos imperativos fragmentados:\*\* Desaparecieron los comandos independientes \`docker network create\`, \`docker volume create\`, \`docker build\` y los tres \`docker run\` cargados de banderas propensas a errores tipográficos. \* \*\*El flag \`--name\` manual:\*\* Compose asume el control del nombrado lógico a través de la clave del servicio, garantizando nombres consistentes para el DNS interno. \* \*\*El acoplamiento al ciclo de vida manual:\*\* Se terminó el problema de contenedores huérfanos o conexiones residuales rotas tras reiniciar la base de datos; el ciclo de vida de todo el stack se unifica en una sola entidad. --- ### 2\. ¿Qué apareció? \* \*\*Infraestructura como Código (IaC) versionable:\*\* El archivo \`docker-compose.yml\` se convierte en la única fuente de verdad (\*single source of truth\*), convive con el código fuente en el repositorio Git y se audita en revisiones de código (Pull Requests). \* \*\*Orquestación sincronizada por salud real (\`depends\_on\` + \`condition: service\_healthy\`):\*\* Las compuertas de bloqueo ya no esperan un estado de proceso trivial (\`Started\`), sino que verifican que la capa inferior esté realmente lista para recibir tráfico (\`healthy\`). \* \*\*Pruebas de salud continuas (\*Healthchecks\*):\*\* Monitorización automatizada en tiempo de ejecución (ping TCP en MySQL, \`fetch\` nativo en Node.js y \`wget\` en Alpine Nginx) sin instalar dependencias redundantes como \`curl\`. \* \*\*Construcción y etiquetado declarativo (\`build:\` + \`image:\`):\*\* Compose se encarga de compilar las imágenes locales faltantes y versionarlas antes de levantar los servicios mediante \`docker compose up --build\`. \* \*\*Principio de menor privilegio granular (\`cap\_drop\` / \`cap\_add\`):\*\* Se despojaron los contenedores de las 14 capacidades por defecto del Kernel de Linux, restituyendo en la base de datos únicamente las llamadas indispensables (\`SETGID\`, \`SETUID\`, \`DAC\_OVERRIDE\`). \* \*\*Contrato de configuración estandarizado:\*\* Separación formal entre variables de entorno y manifiestos a través de la tríada \`.env.example\`, \`.env\` y \`.gitignore\`. \* \*\*Ciclo de vida atómico y seguro (\`up\` / \`down\` / \`down -v\`):\*\* La capacidad de levantar el entorno completo con un comando y destruirlo de forma predecible protegiendo la persistencia del dato. --- ### 3\. Matriz de Traducción de Infraestructura | Concepto de Infraestructura | Semana 3: Enfoque Manual (\`docker run\` / CLI) | Semana 4: Enfoque Declarativo (\`docker-compose.yml\`) | Rol en la Arquitectura | | :--- | :--- | :--- | :--- | | \*\*Definición de servicio\*\* | \`docker run -d --name
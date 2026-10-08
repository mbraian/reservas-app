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
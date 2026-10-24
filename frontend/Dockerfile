# ========================================================
# Etapa 1: Dependencias (deps)
# ========================================================
FROM node:20-alpine AS build
# FROM --platform=$BUILDPLATFORM node:20-alpine AS build # Usado para compilar y pushear la imagen en MAC para que la utilice un nodo con otra arquitectura

WORKDIR /app

# Aprovechamiento de caché: manifiestos primero
COPY package.json package-lock.json ./

# Instalamos TODAS las dependencias (Vite se necesita para compilar)
RUN npm ci

# Copiamos el código fuente de React
COPY . .

# Compilamos el proyecto (genera la carpeta /app/dist)
RUN npm run build


# ========================================================
# Etapa 2: Servidor Web Estático (Nginx sin privilegios)
# ========================================================
FROM nginxinc/nginx-unprivileged:1.27-alpine

# Variables de entorno para resolución en tiempo de arranque
ENV API_HOST=reservas-api \
    API_PORT=3000 \
    NGINX_ENVSUBST_FILTER=^API_

# Copiamos ÚNICAMENTE el resultado estático compilado
COPY --from=build /app/dist /usr/share/nginx/html

# Copiamos la plantilla de configuración del proxy hacia el hook de templates
COPY nginx/templates/default.conf.template /etc/nginx/templates/default.conf.template

# NOTA: No declaramos "USER nginx" porque la imagen base
# nginxinc/nginx-unprivileged ya corre como UID 101 de fábrica.

EXPOSE 8080
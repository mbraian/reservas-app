# ========================================================
# Etapa 1: Dependencias (deps)
# ========================================================
FROM node:20-alpine AS deps

WORKDIR /app

# Copiamos únicamente los manifiestos de dependencias
COPY package.json package-lock.json ./

# Instalación limpia y reproducible solo para producción
RUN npm ci --omit=dev


# ========================================================
# Etapa 2: Imagen Final de Ejecución
# ========================================================
# Base mínima
FROM node:20-alpine

WORKDIR /app

# Variable de entorno para optimización de Express
ENV NODE_ENV=production

# (2-e) Versionado inmutable observable
ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION}

# Copia de dependencias y código con asignación al usuario no privilegiado
COPY --chown=node:node --from=deps /app/node_modules ./node_modules
COPY --chown=node:node . .

# Establecer el usuario sin privilegios para tiempo de ejecución
USER node

EXPOSE 3000
CMD ["npm", "start"]
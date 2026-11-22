# syntax=docker/dockerfile:1
# admin (Node.js / Express + TypeScript, panel de administración). Se compila con dependencias de desarrollo y a la imagen final pasan
# solo dist/, public/ y las dependencias de producción. Corre con el usuario sin privilegios «node».

# --- Etapa 1: compilación ---
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --no-audit --no-fund
COPY . .
RUN npm run build

# --- Etapa 2: dependencias de producción ---
FROM node:22-alpine AS prod-deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev --no-audit --no-fund

# --- Etapa 3: ejecución ---
FROM node:22-alpine
ENV NODE_ENV=production PORT=8080
WORKDIR /app
COPY --from=prod-deps --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --chown=node:node public ./public
COPY --chown=node:node package.json ./
USER node
EXPOSE 8080
CMD ["node", "dist/app.js"]

# syntax=docker/dockerfile:1
# checkout (NestJS / TypeScript, Yarn 4 versionado en .yarn/releases).
# Se compila con todas las dependencias y a la imagen final pasan solo dist/ y las dependencias de producción.

# --- Etapa 1: compilación ---
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn ./.yarn
RUN yarn install --immutable
COPY . .
RUN yarn build

# --- Etapa 2: dependencias de producción ---
FROM node:22-alpine AS prod-deps
WORKDIR /app
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn ./.yarn
RUN yarn workspaces focus --all --production

# --- Etapa 3: ejecución ---
FROM node:22-alpine
ENV NODE_ENV=production PORT=8080
WORKDIR /app
COPY --from=prod-deps --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --chown=node:node package.json ./
USER node
EXPOSE 8080
CMD ["node", "dist/main.js"]

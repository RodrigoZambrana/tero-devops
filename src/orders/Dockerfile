# syntax=docker/dockerfile:1
# orders (Go). Binario estático (sin CGO) sobre distroless/static: sin shell, sin gestor de paquetes, usuario no-root.

# --- Etapa 1: compilación ---
FROM golang:1.24-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/orders .

# --- Etapa 2: ejecución ---
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/orders /app/orders
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app/orders"]

# syntax=docker/dockerfile:1
# catalog (Go). Necesita CGO porque el modo en memoria usa SQLite (go-sqlite3), así que se compila
# contra glibc y corre sobre distroless/base (trae glibc pero no shell ni gestor de paquetes).

# --- Etapa 1: compilación ---
FROM golang:1.24-bookworm AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=1 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/catalog .

# --- Etapa 2: ejecución ---
FROM gcr.io/distroless/base-debian12:nonroot
COPY --from=build /out/catalog /app/catalog
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app/catalog"]

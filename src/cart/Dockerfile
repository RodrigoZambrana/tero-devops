# syntax=docker/dockerfile:1
# cart (Python / FastAPI). Las dependencias se instalan en un virtualenv en la etapa de build y solo
# el virtualenv y el código pasan a la imagen final, que corre con un usuario sin privilegios.

# --- Etapa 1: dependencias ---
FROM python:3.12-slim AS build
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install -r requirements.txt

# --- Etapa 2: ejecución ---
FROM python:3.12-slim
ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    CART_PERSISTENCE_PROVIDER=in-memory \
    PORT=8080
RUN groupadd --system --gid 10001 app && useradd --system --uid 10001 --gid app --no-create-home app
WORKDIR /app
COPY --from=build /opt/venv /opt/venv
COPY --chown=app:app app ./app
USER 10001:10001
EXPOSE 8080
CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT}"]

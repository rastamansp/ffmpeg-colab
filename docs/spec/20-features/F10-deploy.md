# F10 — Deploy (Portainer + Traefik + SSL)

## Objetivo

Deploy em produção via Portainer, expondo frontend e API com SSL automático via Traefik.

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F10-01 | `studio.gwan.cloud` serve o frontend (nginx:alpine + build React) |
| REQ-F10-02 | `api-studio.gwan.cloud` serve a API NestJS |
| REQ-F10-03 | SSL automático via Let's Encrypt (Traefik GWAN) |
| REQ-F10-04 | Ambos os serviços na `gwan-network` |
| REQ-F10-05 | Health check: `GET /api/health` (API) e `GET /` (web) |
| REQ-F10-06 | `ffmpeg` disponível no container da API (para o worker) |
| REQ-F10-07 | Variáveis de ambiente sensíveis configuradas no Portainer (não no compose) |

## docker-compose.yml (referência para Portainer)

```yaml
name: gwan-studio

services:
  web:
    build:
      context: ./gwan-studio
      dockerfile: frontend/Dockerfile
    image: gwan/studio-web:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.studio-web.rule=Host(`studio.gwan.cloud`)"
      - "traefik.http.routers.studio-web.entrypoints=websecure"
      - "traefik.http.routers.studio-web.tls.certresolver=letsencrypt"
      - "traefik.http.services.studio-web.loadbalancer.server.port=80"
    networks:
      - gwan-network
    restart: unless-stopped
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

  api:
    build:
      context: ./gwan-studio
      dockerfile: backend/Dockerfile
    image: gwan/studio-api:latest
    environment:
      - DJANGO_SETTINGS_MODULE=config.settings.production
      - PORT=3018
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.studio-api.rule=Host(`api-studio.gwan.cloud`)"
      - "traefik.http.routers.studio-api.entrypoints=websecure"
      - "traefik.http.routers.studio-api.tls.certresolver=letsencrypt"
      - "traefik.http.services.studio-api.loadbalancer.server.port=3018"
    networks:
      - gwan-network
    restart: unless-stopped
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

  celery-worker:
    build:
      context: ./gwan-studio
      dockerfile: backend/Dockerfile
    image: gwan/studio-api:latest
    command: celery -A config worker --loglevel=info --concurrency=2
    environment:
      - DJANGO_SETTINGS_MODULE=config.settings.production
    networks:
      - gwan-network
    restart: unless-stopped
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

networks:
  gwan-network:
    external: true
    name: gwan-network
```

## Dockerfile backend (referência)

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app/backend

COPY backend/requirements/production.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY backend/ .

EXPOSE 3018
HEALTHCHECK CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:3018/api/health/')" || exit 1

# Daphne serve HTTP + WebSocket (ASGI)
CMD ["daphne", "-b", "0.0.0.0", "-p", "3018", "config.asgi:application"]
```

## Variáveis de ambiente produção (Portainer)

```
DJANGO_SETTINGS_MODULE=config.settings.production
SECRET_KEY=<random-50-char>
DEBUG=False
ALLOWED_HOSTS=api-studio.gwan.cloud
DATABASE_URL=postgresql://studio_user:xxx@postgres.gwan.cloud:5432/studio
REDIS_URL=redis://cache.gwan.cloud:6379/0
CELERY_BROKER_URL=redis://cache.gwan.cloud:6379/0
CELERY_RESULT_BACKEND=redis://cache.gwan.cloud:6379/1
MINIO_ENDPOINT=s3.gwan.cloud
MINIO_ACCESS_KEY=xxx
MINIO_SECRET_KEY=xxx
MINIO_BUCKET=studio
MINIO_USE_SSL=True
ANTHROPIC_API_KEY=sk-ant-xxx
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxx
OAUTH_REDIRECT_URI=https://api-studio.gwan.cloud/api/oauth/youtube/callback/
OAUTH_ENCRYPTION_KEY=<32-byte-hex>
PORT=3018
```

## Regras de deploy

- Deploy exclusivo via **Portainer** (nunca SSH direto).
- OAuth callback URL registrada no GCP: `https://api-studio.gwan.cloud/api/oauth/youtube/callback/`.
- Migrations via `docker exec` após o primeiro deploy:
  ```bash
  docker exec <container-api> python manage.py migrate
  docker exec <container-api> python manage.py create_studio_bucket
  ```
- `celery-worker` usa a mesma imagem da API (mesmo `Dockerfile`) com `command` sobrescrito.

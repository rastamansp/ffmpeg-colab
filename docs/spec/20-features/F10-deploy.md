# F10 — Deploy (Portainer + Traefik + SSL)

## Objetivo

Deploy em produção via Portainer, expondo frontend e API com SSL automático via Traefik.

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F10-01 | `studio.gwan.cloud` serve HTML + API + WebSocket — container único `app` (Daphne) |
| REQ-F10-02 | WhiteNoise serve arquivos estáticos — sem nginx separado |
| REQ-F10-03 | SSL automático via Let's Encrypt (Traefik GWAN) |
| REQ-F10-04 | Serviços `app` e `celery-worker` na `gwan-network` |
| REQ-F10-05 | Health check: `GET /api/health/` |
| REQ-F10-06 | `ffmpeg` disponível no container do worker Celery |
| REQ-F10-07 | Variáveis de ambiente sensíveis configuradas no Portainer (não no compose) |

## docker-compose.yml (referência para Portainer)

```yaml
name: gwan-studio

services:
  app:
    build:
      context: ./gwan-studio/backend
      dockerfile: Dockerfile
    image: gwan/studio-app:latest
    environment:
      - DJANGO_SETTINGS_MODULE=config.settings.production
      - PORT=3018
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.studio.rule=Host(`studio.gwan.cloud`)"
      - "traefik.http.routers.studio.entrypoints=websecure"
      - "traefik.http.routers.studio.tls.certresolver=letsencrypt"
      - "traefik.http.services.studio.loadbalancer.server.port=3018"
    networks:
      - gwan-network
    restart: unless-stopped
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }

  celery-worker:
    build:
      context: ./gwan-studio/backend
      dockerfile: Dockerfile
    image: gwan/studio-app:latest
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

## Dockerfile (referência)

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements/production.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Coleta estáticos para WhiteNoise servir em produção
RUN python manage.py collectstatic --noinput

EXPOSE 3018
HEALTHCHECK CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:3018/api/health/')" || exit 1

# Daphne serve HTTP + WebSocket + arquivos estáticos via WhiteNoise (ASGI)
CMD ["daphne", "-b", "0.0.0.0", "-p", "3018", "config.asgi:application"]
```

## Variáveis de ambiente produção (Portainer)

```
DJANGO_SETTINGS_MODULE=config.settings.production
SECRET_KEY=<random-50-char>
DEBUG=False
ALLOWED_HOSTS=studio.gwan.cloud
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
OAUTH_REDIRECT_URI=https://studio.gwan.cloud/api/oauth/youtube/callback/
OAUTH_ENCRYPTION_KEY=<32-byte-hex>
PORT=3018
```

## Regras de deploy

- Deploy exclusivo via **Portainer** (nunca SSH direto).
- OAuth callback URL registrada no GCP: `https://studio.gwan.cloud/api/oauth/youtube/callback/`.
- Migrations e bucket via `docker exec` após o primeiro deploy:
  ```bash
  docker exec <container-app> python manage.py migrate
  docker exec <container-app> python manage.py create_studio_bucket
  ```
- `celery-worker` usa a mesma imagem (`gwan/studio-app:latest`) com `command` sobrescrito.
- `collectstatic` já roda no `docker build` (ver Dockerfile) — não precisa rodar manualmente.

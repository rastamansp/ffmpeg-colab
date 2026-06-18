# Dev environment — Gwan Studio

## Pré-requisitos

| Ferramenta | Versão mínima | Motivo |
|-----------|--------------|--------|
| Python | 3.12 | backend Django + templates |
| ffmpeg | 6+ | worker local |
| Docker | 24+ | banco + Redis + MinIO local |
| git | qualquer | clone |

> **Node.js não é necessário.** Frontend é servido pelo Django (templates + static). Tailwind em dev pode ser carregado via CDN no `base.html`.

> **Windows sem ffmpeg no PATH:** use o container docker para dev:
> `docker run --rm -v "$(pwd):/work" -w /work jrottenberg/ffmpeg:6-alpine <args>`

## Bootstrap (Windows PowerShell)

```powershell
cd apps/ffmpeg-colab

# 1. Clona o repo (quando existir)
# git clone https://github.com/rastamansp/gwan-studio gwan-studio

# 2. Cria .env
cp gwan-studio/.env.example gwan-studio/backend/.env

# 3. Sobe infra local (PG + Redis + MinIO)
docker compose -f gwan-studio/docker-compose.dev.yml up -d

# 4. Setup backend Python
cd gwan-studio/backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements/development.txt

# 5. Migrations + cria bucket MinIO
python manage.py migrate
python manage.py create_studio_bucket  # management command customizado

# 6. Inicia app Django (HTML + API + WebSocket em :3018)
python manage.py runserver 3018
# Acesse http://localhost:3018 — Django serve HTML, API e estáticos

# 7. Em outro terminal: Celery worker
cd gwan-studio/backend
.\.venv\Scripts\Activate.ps1
celery -A config worker --loglevel=info
```

> **Sem etapa de frontend separada.** Os templates Django são servidos pelo `runserver`. HTMX, Alpine.js e Tailwind são carregados via CDN em `base.html` no modo `DEBUG=True` (sem build step).

## Variáveis de ambiente (backend/.env)

```env
# Django
SECRET_KEY=django-insecure-dev-only
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=postgresql://studio:studio@localhost:5451/studio

# Redis (broker Celery + channel layer Channels)
REDIS_URL=redis://localhost:6379/0
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/1

# MinIO
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_BUCKET=studio
MINIO_USE_SSL=False

# IA
ANTHROPIC_API_KEY=sk-ant-xxx

# Google / YouTube OAuth 2.0
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxx
OAUTH_REDIRECT_URI=http://localhost:3018/api/oauth/youtube/callback/
STATICFILES_STORAGE=whitenoise.storage.CompressedManifestStaticFilesStorage
OAUTH_ENCRYPTION_KEY=<32-byte-hex-gerado-com: python -c "import secrets; print(secrets.token_hex(32))">

# Servidor
PORT=3018
```

## Infra local (docker-compose.dev.yml)

```yaml
# gwan-studio/docker-compose.dev.yml
services:
  postgres-studio:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: studio
      POSTGRES_USER: studio
      POSTGRES_PASSWORD: studio
    ports:
      - "5451:5432"   # slot 18 PG isolado

  redis-dev:
    image: redis:7-alpine
    ports:
      - "6379:6379"   # Redis compartilhado (slot 18 se Redis isolado: 6398)

  minio-dev:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
```

> Se a infra compartilhada (PostgreSQL `5432`, Redis `6379`, MinIO `9000`) estiver disponível localmente,
> aponte `DATABASE_URL` e `REDIS_URL` para ela e crie o banco `studio` e bucket `studio` manualmente.

## Portas dev (slot 18)

| Serviço | Porta |
|---------|-------|
| App Django (HTML + API + WS) | `3018` |
| PostgreSQL isolado | `5451` |
| Redis isolado | `6398` |

## Comandos Django úteis

```bash
# Criar migration
python manage.py makemigrations

# Aplicar migrations
python manage.py migrate

# Shell Django (debug)
python manage.py shell

# Celery worker (foreground)
celery -A config worker --loglevel=info

# Celery beat (tarefas agendadas — Fase B)
celery -A config beat --loglevel=info

# Flower (monitor Celery — opcional)
celery -A config flower --port=5555
```

## Google OAuth (configuração GCP)

1. Console GCP → APIs & Serviços → Credenciais → Criar ID do cliente OAuth.
2. Tipo: **Aplicativo Web**.
3. URIs de redirecionamento autorizados:
   - Dev: `http://localhost:3018/api/oauth/youtube/callback/`
   - Prod: `https://api-studio.gwan.cloud/api/oauth/youtube/callback/`
4. Ativar: **YouTube Data API v3**.
5. Copiar `client_id` e `client_secret` para o `.env`.

## Configuração de settings Django por ambiente

```python
# config/settings/base.py     — configurações comuns
# config/settings/development.py — DEBUG=True, sqlite opcionalmente
# config/settings/production.py  — DEBUG=False, SECURE_*, ALLOWED_HOSTS restritos

# Seleção via variável de ambiente:
# DJANGO_SETTINGS_MODULE=config.settings.production
```

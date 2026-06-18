# F00 — Setup do projeto

## Objetivo

Criar o repositório `gwan-studio` com backend Django 5.2 LTS e frontend React/Vite, seguindo os princípios de **SDD**, **Clean Architecture**, **SOLID** e uso de **Use Cases** no backend.

## Princípios obrigatórios

### Spec Driven Design
Toda feature tem spec (`REQ-FXX-NN` + `RN-FXX-NN`) antes de qualquer linha de código. O SDD em `docs/spec/` é a fonte de verdade.

### Clean Architecture (backend Django)

```
domain/ → application/ → infrastructure/ → presentation/
```

- Nenhum `import django` em `domain/` ou `application/`.
- Views DRF apenas serializam/deserializam e chamam use case.
- Celery tasks apenas resolvem dependências e delegam ao use case.

### SOLID na prática

| Princípio | Como verificar no PR |
|-----------|---------------------|
| S | cada arquivo de use case tem uma única função pública |
| O | novo adapter = novo arquivo; zero edição em `ports.py` |
| L | substituir `MinioStorageAdapter` por `LocalStorageAdapter` não quebra nenhum use case |
| I | se um use case injeta `IVideoWorkerPort`, ele não precisa saber de `IAiVisionPort` |
| D | use case recebe o port no `__init__`, nunca instancia o adapter diretamente |

### Use Cases

Arquivo por use case, nomeado como verbo + substantivo:

```python
# application/jobs/start_merge_job.py
class StartMergeJobUseCase:
    def __init__(self, job_repo: IJobRepository, event_bus: IEventBusPort): ...
    def execute(self, project_id: str, source_ids: list[str]) -> Job: ...
```

### daisyUI (frontend)

Todos os componentes UI base vêm do **daisyUI** (plugin Tailwind). Nenhuma biblioteca React de UI (MUI, Ant Design, Chakra, shadcn/ui). Ver [frontend-architecture.md](../40-architecture/frontend-architecture.md).

### design.md — fonte de verdade visual

Toda decisão de aparência (cor, espaçamento, ícone, comportamento de componente, dark mode) está em [`40-architecture/design.md`](../40-architecture/design.md). **Nenhum componente novo deve ser criado sem consultar esse arquivo.** Se uma situação não estiver coberta, o design.md deve ser atualizado antes de implementar.

## Estrutura de repositório

```
gwan-studio/
├── backend/                     # Django 5.2 LTS — studio.gwan.cloud (HTML + API + WS)
│   ├── config/                  # settings, urls, asgi
│   │   ├── settings/
│   │   │   ├── base.py
│   │   │   ├── development.py
│   │   │   └── production.py
│   │   ├── urls.py
│   │   └── asgi.py              # ASGI entry point (Channels)
│   ├── domain/                  # entidades (dataclasses) + ports (ABCs)
│   │   ├── entities.py
│   │   └── ports.py
│   ├── application/             # use cases (framework-agnostic)
│   │   ├── projects/
│   │   ├── sources/
│   │   ├── jobs/
│   │   ├── thumbnails/
│   │   ├── seo/
│   │   └── publish/
│   ├── infrastructure/          # adapters concretos
│   │   ├── storage/             # MinIO adapter (boto3)
│   │   ├── ai/                  # Claude Vision + Text (anthropic SDK)
│   │   ├── youtube/             # YouTube Data API v3 (google-api-python-client)
│   │   ├── ffmpeg/              # FFmpeg adapter (ffmpeg-python / subprocess)
│   │   └── orm/                 # Django ORM models + repositories
│   ├── presentation/            # views HTML + API DRF + Channels WS
│   │   ├── views/               # Django views → HTML (Django Templates)
│   │   ├── api/                 # DRF ViewSets → JSON (usados pelo HTMX e clientes externos)
│   │   └── ws/                  # Django Channels consumers (envia HTML parcial via WS)
│   ├── templates/               # Django Templates (server-side rendering)
│   │   ├── base.html
│   │   ├── components/          # partials reutilizáveis (_job_status, _job_log…)
│   │   ├── projects/
│   │   ├── sources/
│   │   ├── merge/
│   │   ├── export/
│   │   ├── thumbnail/
│   │   ├── seo/
│   │   └── publish/
│   ├── static/                  # arquivos estáticos
│   │   ├── js/                  # htmx.min.js, htmx-ext-ws.js, alpine.min.js
│   │   └── css/                 # app.css (Tailwind + daisyUI compilados)
│   ├── tasks/                   # Celery tasks (chamam use cases)
│   ├── requirements/
│   │   ├── base.txt
│   │   ├── development.txt
│   │   └── production.txt
│   ├── manage.py
│   └── Dockerfile
│
├── docker-compose.yml           # dev local (app + celery-worker)
└── .env.example
```

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F00-01 | Django 5.2 LTS com DRF 3.x e Django Channels 4.x (ASGI) |
| REQ-F00-02 | Celery 5 + Redis como broker de tasks |
| REQ-F00-03 | Templates Django em `backend/templates/`; arquivos estáticos em `backend/static/` |
| REQ-F00-04 | Dockerfile único: `python:3.12-slim` com ffmpeg + `collectstatic` na build |
| REQ-F00-05 | `whitenoise[brotli]` serve estáticos em produção (sem nginx separado) |
| REQ-F00-06 | Health check `GET /api/health/` retornando `{ "status": "ok" }` |
| REQ-F00-07 | Porta slot 18: Daphne em `3018` (HTML + API + WebSocket unificados) |
| REQ-F00-08 | Django migrations para todos os models |
| REQ-F00-09 | `docs/spec/40-architecture/design.md` existe e é consultado antes de qualquer trabalho de UI |

## Dependencies Python (requirements/base.txt)

```
Django==5.2.*
djangorestframework==3.*
django-channels[daphne]==4.*
channels-redis==4.*
celery[redis]==5.*
django-celery-results==2.*
psycopg[binary]==3.*
boto3==1.*
anthropic==0.*
google-api-python-client==2.*
google-auth-oauthlib==1.*
ffmpeg-python==0.2.*
Pillow==10.*
cryptography==42.*
python-decouple==3.*
django-htmx==1.*
whitenoise[brotli]==6.*
```

## Variáveis de ambiente necessárias (.env)

```env
# Django
SECRET_KEY=django-insecure-xxx
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=postgresql://studio:studio@localhost:5451/studio

# Redis (broker Celery + channel layer)
REDIS_URL=redis://localhost:6379/0

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
OAUTH_ENCRYPTION_KEY=<32-byte-hex>

# Dev server
PORT=3018
```

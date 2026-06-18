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

### shadcn/ui (frontend)

Todos os componentes UI base vêm do shadcn/ui (`npx shadcn-ui add`). Nenhuma biblioteca de UI concorrente (MUI, Ant Design, Chakra). Ver [frontend-architecture.md](../40-architecture/frontend-architecture.md#design-system--shadcnui).

### design.md — fonte de verdade visual

Toda decisão de aparência (cor, espaçamento, ícone, comportamento de componente, dark mode) está em [`40-architecture/design.md`](../40-architecture/design.md). **Nenhum componente novo deve ser criado sem consultar esse arquivo.** Se uma situação não estiver coberta, o design.md deve ser atualizado antes de implementar.

## Estrutura de repositório

```
gwan-studio/
├── backend/                     # Django 5.2 LTS — api-studio.gwan.cloud
│   ├── config/                  # settings, urls, asgi, wsgi
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
│   │   ├── ai/                  # Claude Vision + Text adapter (anthropic SDK)
│   │   ├── youtube/             # YouTube Data API v3 adapter (google-api-python-client)
│   │   ├── ffmpeg/              # FFmpeg adapter (ffmpeg-python / subprocess)
│   │   └── orm/                 # Django ORM models + repositories
│   ├── presentation/            # Django apps (views, serializers, consumers WS)
│   │   ├── projects/            # app Django: views DRF + urls
│   │   ├── sources/
│   │   ├── jobs/
│   │   ├── thumbnails/
│   │   ├── seo/
│   │   ├── publish/
│   │   └── ws/                  # Django Channels consumers
│   ├── tasks/                   # Celery tasks (chamam use cases)
│   │   ├── merge.py
│   │   ├── export.py
│   │   ├── thumbnail.py
│   │   └── publish.py
│   ├── requirements/
│   │   ├── base.txt
│   │   ├── development.txt
│   │   └── production.txt
│   ├── manage.py
│   └── Dockerfile
│
├── frontend/                    # React 18 + Vite — studio.gwan.cloud
│   ├── src/
│   │   ├── features/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── lib/
│   ├── Dockerfile
│   └── package.json
│
├── docker-compose.yml           # dev local (+ Celery worker)
└── .env.example
```

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F00-01 | Django 5.2 LTS com DRF 3.x e Django Channels 4.x (ASGI) |
| REQ-F00-02 | Celery 5 + Redis como broker de tasks |
| REQ-F00-03 | TypeScript strict no frontend (React 18 + Vite) |
| REQ-F00-04 | Dockerfile backend: `python:3.12-slim` com ffmpeg instalado |
| REQ-F00-05 | Dockerfile frontend: multi-stage `node:20-alpine` → `nginx:alpine` |
| REQ-F00-06 | Health check `GET /api/health/` retornando `{ "status": "ok" }` |
| REQ-F00-07 | Portas slot 18: API Gunicorn/Uvicorn `3018`, Web `5191` no dev local |
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

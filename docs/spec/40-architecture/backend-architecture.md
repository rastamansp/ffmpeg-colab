# Arquitetura de backend — Gwan Studio

## Stack

- **Django 5.2 LTS** (framework web)
- **Django REST Framework 3.x** (API REST)
- **Django Channels 4.x** (WebSocket via ASGI)
- **Celery 5** + **Redis** (jobs assíncronos)
- **Django ORM** + **PostgreSQL** (persistência)
- **Pillow** (render de thumbnails)
- **ffmpeg-python** / subprocess (operações FFmpeg)
- **anthropic** SDK (Claude Vision + Text)
- **google-api-python-client** (YouTube Data API v3)
- **boto3** (MinIO / S3)

## Clean Architecture com Django

```
backend/
├── domain/                   # Núcleo: sem dependência de framework
│   ├── entities.py           # dataclasses Python (Project, Source, Job, etc.)
│   └── ports.py              # ABCs (interfaces) dos adapters
│
├── application/              # Use cases: injectam ports, sem Django direto
│   ├── projects/
│   │   ├── create_project.py
│   │   ├── get_project.py
│   │   └── list_projects.py
│   ├── sources/
│   │   └── upload_source.py
│   ├── jobs/
│   │   ├── start_merge_job.py
│   │   ├── start_export_job.py
│   │   ├── start_thumbnail_job.py
│   │   ├── start_seo_job.py
│   │   └── start_publish_job.py
│   ├── thumbnails/
│   │   └── select_thumbnail.py
│   └── seo/
│       ├── update_seo.py
│       └── approve_seo.py
│
├── infrastructure/           # Adapters: implementam os ports
│   ├── storage/
│   │   └── minio_storage.py  # IObjectStoragePort via boto3
│   ├── ai/
│   │   ├── claude_vision.py  # IAiVisionPort via anthropic SDK
│   │   └── claude_text.py    # IAiTextPort via anthropic SDK
│   ├── youtube/
│   │   └── youtube_publish.py # IYouTubePublishPort via google-api-python-client
│   ├── ffmpeg/
│   │   └── ffmpeg_worker.py  # IVideoWorkerPort via ffmpeg-python
│   └── orm/
│       ├── models.py         # Django ORM models
│       ├── repositories.py   # implementações dos repository ports
│       └── migrations/
│
├── presentation/             # Delivery: Django views + Channels consumers
│   ├── projects/
│   │   ├── views.py          # DRF ViewSets
│   │   ├── serializers.py
│   │   └── urls.py
│   ├── sources/
│   ├── jobs/
│   ├── thumbnails/
│   ├── seo/
│   ├── publish/
│   └── ws/
│       └── consumers.py      # WebSocket consumers (Channels)
│
├── tasks/                    # Celery tasks — ponte entre Celery e use cases
│   ├── merge.py
│   ├── export.py
│   ├── thumbnail.py
│   └── publish.py
│
└── config/
    ├── settings/base.py
    ├── urls.py
    └── asgi.py               # Channels ASGI routing
```

## Ports (ABCs Python)

```python
# domain/ports.py

from abc import ABC, abstractmethod
from typing import BinaryIO, Iterator

class IObjectStoragePort(ABC):
    @abstractmethod
    def put_object(self, key: str, data: BinaryIO, content_type: str) -> None: ...
    @abstractmethod
    def get_object(self, key: str) -> BinaryIO: ...
    @abstractmethod
    def delete_object(self, key: str) -> None: ...
    @abstractmethod
    def get_signed_url(self, key: str, expires_in: int) -> str: ...
    @abstractmethod
    def object_exists(self, key: str) -> bool: ...

class IVideoWorkerPort(ABC):
    @abstractmethod
    def merge(self, source_keys: list[str], output_key: str) -> dict: ...
    @abstractmethod
    def export(self, input_key: str, output_key: str, settings: dict) -> dict: ...
    @abstractmethod
    def extract_frames(self, video_key: str, n_frames: int) -> list[str]: ...
    @abstractmethod
    def render_thumbnail(self, frame_key: str, plan: dict, output_key: str) -> str: ...

class IAiVisionPort(ABC):
    @abstractmethod
    def plan_thumbnails(self, frame_keys: list[str], project_name: str, seo_title: str | None) -> list[dict]: ...

class IAiTextPort(ABC):
    @abstractmethod
    def generate_seo(self, project_name: str, channel_name: str, context: str | None) -> dict: ...

class IYouTubePublishPort(ABC):
    @abstractmethod
    def upload_video(self, video_key: str, title: str, description: str,
                     tags: list[str], thumbnail_key: str, refresh_token: str) -> dict: ...
```

## Celery Tasks — ponte com use cases

```python
# tasks/merge.py
from celery import shared_task
from application.jobs.execute_merge_job import ExecuteMergeJobUseCase
from infrastructure.di import get_container   # injeção de dependência simples

@shared_task(bind=True, max_retries=3)
def merge_job(self, job_id: str):
    container = get_container()
    use_case = ExecuteMergeJobUseCase(
        job_repo=container.job_repository(),
        storage=container.object_storage(),
        video_worker=container.video_worker(),
        event_bus=container.job_event_bus(),
    )
    use_case.execute(job_id)
```

## WebSocket — Django Channels

```python
# presentation/ws/consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer

class JobConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.project_id = self.scope['url_route']['kwargs']['project_id']
        await self.channel_layer.group_add(f"project_{self.project_id}", self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(f"project_{self.project_id}", self.channel_name)

    async def job_update(self, event):
        await self.send(text_data=json.dumps(event['data']))
```

```python
# Celery task emite via channel layer (após update no banco):
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

channel_layer = get_channel_layer()
async_to_sync(channel_layer.group_send)(
    f"project_{project_id}",
    {"type": "job.update", "data": {"jobId": job_id, "status": "done", ...}}
)
```

## Django ORM Models

```python
# infrastructure/orm/models.py
from django.db import models
import uuid

class Project(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    name = models.CharField(max_length=100)
    channel_name = models.CharField(max_length=200)
    youtube_channel_id = models.CharField(max_length=100, blank=True)
    oauth_refresh_token_enc = models.TextField(blank=True)
    export_settings = models.JSONField(default=dict)
    status = models.CharField(max_length=50, default='draft')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

class Source(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    project = models.ForeignKey(Project, on_delete=models.CASCADE, related_name='sources')
    original_filename = models.CharField(max_length=500)
    storage_key = models.CharField(max_length=1000)
    camera = models.CharField(max_length=100, blank=True)
    take_number = models.IntegerField(null=True, blank=True)
    duration_sec = models.IntegerField(null=True)
    size_bytes = models.BigIntegerField(null=True)
    status = models.CharField(max_length=50, default='uploading')
    uploaded_at = models.DateTimeField(auto_now_add=True)

class Job(models.Model):
    JOB_TYPES = [('merge','merge'),('export','export'),('thumbnail','thumbnail'),
                 ('seo','seo'),('publish','publish')]
    JOB_STATUSES = [('pending','pending'),('running','running'),
                    ('done','done'),('failed','failed')]
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    project = models.ForeignKey(Project, on_delete=models.CASCADE, related_name='jobs')
    type = models.CharField(max_length=50, choices=JOB_TYPES)
    status = models.CharField(max_length=50, choices=JOB_STATUSES, default='pending')
    input_params = models.JSONField(default=dict)
    output_params = models.JSONField(default=dict)
    logs = models.TextField(blank=True)
    error = models.TextField(blank=True)
    started_at = models.DateTimeField(null=True)
    finished_at = models.DateTimeField(null=True)
```

## Configuração ASGI (Channels)

```python
# config/asgi.py
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from presentation.ws.routing import websocket_urlpatterns

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(websocket_urlpatterns)  # ws://host/ws/projects/<id>/
    ),
})
```

## Regras de Clean Architecture

- **Use cases são puros**: recebem ports por parâmetro, sem `import django` direto.
- **Views DRF são finas**: recebem DTO → chamam use case → retornam Response. Sem lógica de negócio.
- **Celery tasks chamam use cases**: não contêm lógica de domínio.
- **Models ORM não são entidades**: entities.py usa dataclasses; ORM é detalhe de infra.
- **Adapters são substituíveis**: trocar boto3 por MinIO SDK = novo adapter, zero mudança no domínio.
- **Testes de use case**: mock dos ports via `unittest.mock` — sem Django, banco ou FFmpeg.

# Contexto do sistema (C4 nível 1–2) — Gwan Studio

## Contexto (C4 Nível 1)

```mermaid
flowchart LR
    user([Criador\nde conteúdo])
    subgraph studio[Gwan Studio]
      web[Web SPA\nstudio.gwan.cloud]
      api[API NestJS\napi-studio.gwan.cloud]
      worker[FFmpeg Worker\n+ AI pipeline]
    end
    claude[(Anthropic\nClaude API)]
    youtube[(YouTube\nData API v3)]
    minio[(MinIO / S3\ns3.gwan.cloud)]
    pg[(PostgreSQL\ngwan-infra)]

    user -->|upload footage + gerencia projeto| web
    web -->|REST: projects/jobs/publish| api
    web <-->|WS: status/logs/preview| api
    api -->|spawn job| worker
    worker -->|Claude Vision: thumbnail plan| claude
    worker -->|Claude Text: SEO| claude
    worker -->|upload vídeo + metadados| youtube
    worker -->|put/get artefatos| minio
    api -->|persistência de projetos| pg
    web -->|download via URL assinada| minio
    user -->|link footer| site[gwan.cloud]
```

## Containers (C4 Nível 2)

| Container | Tecnologia | Responsabilidade | Domínio |
|-----------|-----------|------------------|---------|
| **Web** | React 18 + Vite + TS + shadcn/ui | Upload de footage, gerenciamento de projeto, acompanhamento de jobs, aprovação de thumbnails, SEO editor, publicação | `studio.gwan.cloud` |
| **API** | **Django 5.2 LTS** + DRF 3.x + Django Channels 4.x (ASGI) | Projetos, sources, jobs, thumbnails, SEO, publicação YouTube; despacha tasks Celery; repassa eventos via WebSocket | `api-studio.gwan.cloud` |
| **Celery Worker** | Celery 5 + ffmpeg-python / subprocess | Executa jobs pesados (merge, export, frames, thumbnails, upload YouTube) — processo separado, sem bloquear a API | container `celery-worker` |
| **Redis** | Redis 7 (infra GWAN compartilhada) | Broker Celery + channel layer Django Channels (pub/sub WebSocket) | `cache.gwan.cloud` |
| **Claude** | Anthropic API (externo) | Vision: analisa frames e planeja layout de thumbnail. Text: gera título, descrição e tags SEO | `ANTHROPIC_API_KEY` — só no backend |
| **YouTube Data API** | Google API v3 (externo) | Upload de vídeo, gestão de metadados | OAuth 2.0 — refresh token só no backend |
| **MinIO** | S3-compatible (infra GWAN) | Workspace do projeto, sources, merged video, exported video, thumbnails; URLs assinadas | `s3.gwan.cloud` (compartilhado, bucket `studio`) |
| **PostgreSQL** | PostgreSQL (infra GWAN) | Projetos, sources, jobs, thumbnails, SEO metadata, OAuth tokens (Django ORM) | `postgres.gwan.cloud` (compartilhado) |

## Limites e integrações

- **MinIO e PostgreSQL são infra compartilhada** (P0 do gwan-infra) — Studio usa bucket `studio` e schema/db dedicado.
- **Chaves de IA e OAuth no backend apenas** — nunca expostas ao browser.
- **Celery Worker executa fora do processo Django** — FS efêmero por job, não bloqueia a API.
- **Download** ocorre direto do MinIO via URL assinada — a API não faz proxy de binários.
- **OAuth tokens do YouTube** são persistidos criptografados no banco — o criador autentica uma vez por projeto.
- **Celery + Redis na Fase A** (fila durável por design — jobs sobrevivem a restarts desde o início).

## Fluxo de sequência (produção de vídeo ponta a ponta)

```mermaid
sequenceDiagram
    participant W as Web SPA
    participant A as API (Django/DRF)
    participant Wk as Celery Worker
    participant C as Claude
    participant YT as YouTube API
    participant M as MinIO

    W->>A: POST /api/projects (cria projeto)
    W->>A: POST /api/projects/:id/sources (multipart, footage)
    A->>M: put studio/<project>/sources/*
    A-->>W: 201 { sources[] }

    W->>A: POST /api/projects/:id/jobs/merge { source_order[] }
    A->>A: cria Job(type=merge, status=pending)
    A->>Wk: merge_job.delay(job_id)
    A-->>W: 202 { job_id }
    Wk->>M: get sources
    Wk->>Wk: ffmpeg concat (stream copy)
    Wk->>M: put merged.mp4
    Wk->>A: atualiza Job(status=done) via ORM
    A-->>W: WS job.update { status: "done", preview_url }

    W->>A: POST /api/projects/:id/jobs/thumbnail
    A->>Wk: thumbnail_job.delay(job_id)
    Wk->>M: get merged.mp4
    Wk->>Wk: ffmpeg extrai frames (-vf fps=...)
    Wk->>C: Vision: analisa frames + planeja 3 layouts
    Wk->>Wk: Pillow renderiza 3 JPEGs 1280x720
    Wk->>M: put thumbnails/A.jpg, B.jpg, C.jpg
    A-->>W: WS job.update { thumbnails: [urlA, urlB, urlC] }

    W->>A: POST /api/projects/:id/jobs/seo { context }
    A->>C: Text: gera titulo + descricao + tags
    A->>A: salva SeoMetadata no banco
    A-->>W: 200 { title, description, tags }
    Note over W: Criador revisa SEO + escolhe thumbnail

    W->>A: POST /api/projects/:id/jobs/export { settings }
    A->>Wk: export_job.delay(job_id)
    Wk->>M: get merged.mp4
    Wk->>Wk: ffmpeg encode (codec/bitrate)
    Wk->>M: put final.mp4
    A-->>W: WS job.update { status: "done", download_url }

    W->>A: POST /api/projects/:id/jobs/publish { thumbnail_variant }
    A->>Wk: publish_job.delay(job_id)
    Wk->>M: get final.mp4 + thumbnail
    Wk->>YT: upload resumable + set thumbnail
    Wk->>A: salva PublishRecord { video_id }
    A-->>W: WS job.update { status: "published", youtube_url }
```

## Ambientes

| Ambiente | Web | API | Worker | MinIO | Claude/YouTube |
|----------|-----|-----|--------|-------|----------------|
| **Dev local** (slot 18) | `localhost:5191` | `localhost:3018/api` | processo local (ffmpeg no PATH) | infra compartilhada ou MinIO local | chaves dev |
| **Produção** | `studio.gwan.cloud` | `api-studio.gwan.cloud/api` | worker dedicado | `s3.gwan.cloud` (bucket `studio`) | chaves prod |

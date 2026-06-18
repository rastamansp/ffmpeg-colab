# Contratos REST + WebSocket — Gwan Studio

Base URL produção: `https://api-studio.gwan.cloud/api`  
Base URL dev local: `http://localhost:3018/api`

---

## Projetos

### `POST /api/projects`
Cria novo projeto.

**Body:**
```json
{ "name": "EP 42 - Tour Cabreuva", "channel_name": "Canal do Pedro" }
```

**Response 201:**
```json
{
  "id": "proj_uuid",
  "name": "EP 42 - Tour Cabreuva",
  "channel_name": "Canal do Pedro",
  "status": "draft",
  "created_at": "2026-06-17T10:00:00Z"
}
```

---

### `GET /api/projects`
Lista projetos paginados.

**Query:** `?page=1&limit=20&status=exported`

**Response 200:**
```json
{
  "data": [ { "id", "name", "status", "created_at", "youtube_url"? } ],
  "total": 42,
  "page": 1,
  "limit": 20
}
```

---

### `GET /api/projects/:id`
Detalhe completo do projeto.

**Response 200:**
```json
{
  "id": "proj_uuid",
  "name": "...",
  "status": "exported",
  "channel_name": "...",
  "youtube_channel_id": "UC...",
  "export_settings": { "codec": "copy" },
  "sources": [ { "id", "original_filename", "camera", "take_number", "duration_sec", "status" } ],
  "jobs": {
    "merge": { "id", "status", "started_at", "finished_at" },
    "export": { "id", "status" },
    "thumbnail": { "id", "status" },
    "seo": { "id", "status" },
    "publish": { "id", "status" }
  },
  "thumbnails": [ { "id", "variant", "output_url", "selected" } ],
  "seo": { "title", "description", "tags", "approved" },
  "publish_record": { "youtube_video_id", "youtube_url", "status" }
}
```

---

### `PATCH /api/projects/:id`
Atualiza nome, canal ou export settings.

---

### `DELETE /api/projects/:id`
Soft delete (arquiva projeto).

---

## Sources (footage)

### `POST /api/projects/:id/sources`
Upload de arquivos de vídeo.

**Content-Type:** `multipart/form-data`  
**Fields:** `files` (array), `camera[]` (opcional), `take_number[]` (opcional)

**Response 201:**
```json
[
  { "id": "src_uuid", "original_filename": "clip01.mp4", "status": "uploading", "size_bytes": 500000000 }
]
```

---

### `GET /api/projects/:id/sources`

**Response 200:** array de `Source` com `status`, `duration_sec`, `storage_key`

---

### `GET /api/projects/:id/sources/:sourceId/url`
URL assinada para preview (5 min).

**Response 200:** `{ "url": "https://s3.gwan.cloud/..." }`

---

### `DELETE /api/projects/:id/sources/:sourceId`
Remove source (banco + MinIO). Erro se source incluído em merge `done`.

---

## Jobs

### `POST /api/projects/:id/jobs/merge`
Inicia merge de clipes.

**Body:**
```json
{ "source_order": ["src_uuid_1", "src_uuid_2", "src_uuid_3"] }
```

**Response 202:** `{ "job_id": "job_uuid", "status": "pending" }`

---

### `POST /api/projects/:id/jobs/export`
Inicia exportação do vídeo final.

**Body (opcional):**
```json
{ "settings": { "codec": "h264", "bitrate": "8M", "resolution": "1080p" } }
```

**Response 202:** `{ "job_id": "job_uuid", "status": "pending" }`

---

### `GET /api/projects/:id/jobs/export/url`
URL assinada do `final.mp4` (24h).

**Response 200:** `{ "url": "https://s3.gwan.cloud/studio/.../final.mp4?..." }`

---

### `POST /api/projects/:id/jobs/thumbnail`
Inicia geração de thumbnails com IA.

**Response 202:** `{ "job_id": "job_uuid", "status": "pending" }`

---

### `POST /api/projects/:id/jobs/seo`
Gera SEO com Claude.

**Body (opcional):** `{ "context": "Vídeo de cicloturismo na Serra da Mantiqueira..." }`

**Response 202:** `{ "job_id": "job_uuid", "status": "pending" }`

---

### `GET /api/projects/:id/jobs`
Lista jobs do projeto (paginados, desc).

---

### `GET /api/projects/:id/jobs/:jobId`
Detalhe do job com logs completos.

---

## Thumbnails

### `PATCH /api/projects/:id/thumbnails/:thumbId/select`
Seleciona variante para publicação (limpa `selected` das outras).

**Response 200:** `{ "id", "variant", "selected": true }`

---

## SEO

### `PATCH /api/projects/:id/seo`
Edita título, descrição e/ou tags.

**Body:** `{ "title"?, "description"?, "tags"? }`  
**Efeito colateral:** `approved` volta para `false`

---

### `POST /api/projects/:id/seo/approve`
Aprova o SEO atual.

**Response 200:** `{ "approved": true }`

---

## Publicação YouTube

### `GET /api/projects/:id/oauth/youtube`
Inicia fluxo OAuth. Retorna URL de redirect para o frontend navegar.

**Response 200:** `{ "redirect_url": "https://accounts.google.com/o/oauth2/auth?..." }`

---

### `GET /api/oauth/youtube/callback`
Callback OAuth (Google redirect aqui). Troca code por tokens, salva refresh token criptografado.

**Query:** `?code=xxx&state=<project_id>`  
**Response:** redirect para `/projects/:id` (frontend)

---

### `DELETE /api/projects/:id/oauth/youtube`
Revoga e remove OAuth do projeto.

---

### `POST /api/projects/:id/jobs/publish`
Inicia upload no YouTube.

**Body:** `{ "thumbnail_variant": "A" }`  
**Pré-condição:** SEO aprovado + thumbnail selecionada + final.mp4 existe

**Response 202:** `{ "job_id": "job_uuid", "status": "pending" }`

---

### `GET /api/projects/:id/publish`
Status da publicação.

**Response 200:** `{ "video_id"?, "youtube_url"?, "status": "published" }`

---

## Saúde

### `GET /api/health`

**Response 200:** `{ "status": "ok", "timestamp": "2026-06-17T10:00:00Z" }`

---

## WebSocket

**Endpoint:** `wss://api-studio.gwan.cloud` (prod) / `ws://localhost:3018` (dev)  
**Protocolo:** Socket.io ou ws nativo

### Eventos cliente → servidor

```typescript
// Subscrever projeto
socket.emit('subscribe', { projectId: 'proj_uuid' })

// Cancelar subscrição
socket.emit('unsubscribe', { projectId: 'proj_uuid' })
```

### Eventos servidor → cliente

```typescript
// Estado inicial ao subscrever
socket.on('project.snapshot', (data: {
  projectId: string,
  status: string,
  active_job?: { jobId: string, type: string, status: string, recent_logs: string[] }
}) => { ... })

// Atualização incremental de job
socket.on('job.update', (data: {
  jobId: string,
  type: 'merge' | 'export' | 'thumbnail' | 'seo' | 'publish',
  status: 'running' | 'done' | 'failed',
  log_line?: string,
  output_params?: object,
  error?: string
}) => { ... })
```

---

## Códigos de erro

| HTTP | Código | Descrição |
|------|--------|-----------|
| 400 | `INVALID_SOURCE_ORDER` | source_order contém IDs inválidos ou sources não-ready |
| 400 | `CODEC_MISMATCH` | Sources com codecs diferentes (stream copy impossível) |
| 400 | `SEO_NOT_APPROVED` | Publicação tentada sem SEO aprovado |
| 400 | `NO_THUMBNAIL_SELECTED` | Publicação tentada sem thumbnail selecionada |
| 400 | `OAUTH_NOT_CONFIGURED` | Publicação tentada sem OAuth YouTube |
| 409 | `JOB_ALREADY_RUNNING` | Tentativa de iniciar job enquanto outro do mesmo tipo está `running` |
| 404 | `PROJECT_NOT_FOUND` | Projeto não existe ou foi arquivado |
| 413 | `FILE_TOO_LARGE` | Source maior que `MAX_SOURCE_MB` |

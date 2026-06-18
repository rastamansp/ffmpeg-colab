# F02 — Upload de footage (sources)

## Objetivo

Upload de arquivos de vídeo bruto para o MinIO. Suporta múltiplos arquivos, com identificação de câmera e take.

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F02-01 | `POST /api/projects/:id/sources` aceita `multipart/form-data` com 1+ arquivos |
| REQ-F02-02 | Cada arquivo pode ter campos opcionais `camera` e `take_number` |
| REQ-F02-03 | Durante o upload, o source fica em `status = 'uploading'` |
| REQ-F02-04 | Ao concluir, a API extrai duração e tamanho do arquivo e persiste em `Source` |
| REQ-F02-05 | `GET /api/projects/:id/sources` lista todos os sources do projeto |
| REQ-F02-06 | `DELETE /api/projects/:id/sources/:sourceId` remove source (MinIO + banco) |
| REQ-F02-07 | `GET /api/projects/:id/sources/:sourceId/url` retorna URL assinada de preview (5 min) |

## Regras de negócio

| ID | Regra |
|----|-------|
| RN-F02-01 | Formatos aceitos: `mp4`, `mov`, `avi`, `mkv` |
| RN-F02-02 | Tamanho máximo por arquivo: `MAX_SOURCE_MB` (default 2 GB) |
| RN-F02-03 | Chave MinIO: `studio/<project_id>/sources/<uuid>-<filename>` |
| RN-F02-04 | Extração de duração via `ffprobe` (ou `fluent-ffmpeg.ffprobe`) no worker |
| RN-F02-05 | Sources com `status = 'error'` exibem mensagem de erro e permitem re-upload |
| RN-F02-06 | Não é possível deletar source se já foi incluído num merge `done` |

## Upload multipart direto para MinIO (fluxo alternativo — Fase B)

Na Fase A, o upload passa pela API (NestJS → stream para MinIO via `IObjectStoragePort`).
Na Fase B, pode-se implementar upload direto ao MinIO via URL pré-assinada para reduzir latência em arquivos grandes.

## UI (Web)

- Dropzone com drag-and-drop, múltiplos arquivos.
- Para cada arquivo: campo `câmera` (select ou texto livre) + `take` (número).
- Barra de progresso por arquivo (progresso do multipart).
- Lista de sources após upload: miniatura (thumbnail do vídeo), duração, câmera/take, ações (preview, deletar).

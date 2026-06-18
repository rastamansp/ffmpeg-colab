# F04 — Exportação do vídeo final

## Objetivo

Render do vídeo final com as configurações de codec, bitrate e resolução do projeto. Quando `codec = 'copy'`, o export é idêntico ao merge (cópia).

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F04-01 | `POST /api/projects/:id/jobs/export` inicia job de export com `settings` opcionais |
| REQ-F04-02 | O worker usa `merged.mp4` como input |
| REQ-F04-03 | Output salvo em `studio/<project_id>/final.mp4` |
| REQ-F04-04 | `GET /api/projects/:id/jobs/export/url` retorna URL assinada para download do final (válida 24h) |
| REQ-F04-05 | Status do projeto avança para `exported` após job `done` |

## Regras de negócio

| ID | Regra |
|----|-------|
| RN-F04-01 | `merged.mp4` deve existir no MinIO antes de iniciar export |
| RN-F04-02 | Se `codec = 'copy'`, o export é uma cópia simples (sem re-encode) — pode ser feito sem spawn FFmpeg pesado |
| RN-F04-03 | Se `codec = 'h264'` ou `'h265'`, usa `libx264`/`libx265` com `bitrate` e `resolution` das settings |
| RN-F04-04 | Settings do export podem ser sobrescritas por chamada (priority: request body > project settings > defaults) |
| RN-F04-05 | Apenas 1 job de export ativo por projeto por vez |

## Configurações de export padrão

```typescript
const defaults: ExportSettings = {
  codec: 'copy',
  resolution: 'source',
  fps: undefined,        // mantém fps original
  bitrate: undefined,    // mantém bitrate original (apenas para h264/h265)
};
```

## UI (Web)

- Seção "Export" no detalhe do projeto.
- Form opcional: codec (copy/h264/h265), resolução, bitrate.
- Botão "Exportar" → log stream durante render.
- Concluído: botão de download + tamanho do arquivo + link para avançar ao Thumbnail.

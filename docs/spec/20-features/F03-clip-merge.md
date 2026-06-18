# F03 — Merge de clipes

## Objetivo

Concatenar os sources selecionados na ordem desejada em um único arquivo de vídeo, usando FFmpeg stream copy (sem re-encode).

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F03-01 | `POST /api/projects/:id/jobs/merge` inicia job de merge com `{ source_order: [sourceId, ...] }` |
| REQ-F03-02 | O worker baixa os sources do MinIO, gera concat list e executa `ffmpeg -f concat -c copy` |
| REQ-F03-03 | Output salvo em `studio/<project_id>/merged.mp4` |
| REQ-F03-04 | Duração e tamanho do merged são salvos em `job.output_params` |
| REQ-F03-05 | Logs do FFmpeg são streamados via WebSocket em tempo real |
| REQ-F03-06 | Em caso de falha, `job.status = 'failed'` com `error` populado |
| REQ-F03-07 | `GET /api/projects/:id/jobs/merge/url` retorna URL assinada para preview do merged (válida 1h) |

## Regras de negócio

| ID | Regra |
|----|-------|
| RN-F03-01 | `source_order` deve referenciar apenas sources com `status = 'ready'` do projeto |
| RN-F03-02 | Mínimo 1 source na ordem |
| RN-F03-03 | Streams copy exige que todos os sources tenham o mesmo codec de vídeo e áudio — API valida antes de iniciar o job |
| RN-F03-04 | Se codecs divergem, o frontend exibe aviso e sugere re-encode (requereria alterar `export_settings.codec`) |
| RN-F03-05 | Apenas 1 job de merge ativo por projeto por vez |
| RN-F03-06 | Re-executar merge cria novo job e sobrescreve `merged.mp4` no MinIO |

## Pipeline FFmpeg

```bash
# Worker gera concat list temporária
echo "file 'source-a.mp4'" >> /tmp/concat.txt
echo "file 'source-b.mp4'" >> /tmp/concat.txt

# Executa merge stream copy
ffmpeg -f concat -safe 0 -i /tmp/concat.txt -c copy merged.mp4
```

## UI (Web)

- Lista de sources com drag-and-drop para ordenação.
- Botão "Merge" → confirma ordem → inicia job.
- Durante o merge: barra de progresso + log stream (terminal minimizável).
- Concluído: player de preview do `merged.mp4` + botão para avançar ao Export.

# Requisitos não-funcionais — Gwan Studio

## Performance

| Requisito | Meta | Observação |
|-----------|------|------------|
| Merge stream copy | < 2× duração do vídeo | FFmpeg `-c copy` é muito rápido; bottleneck é I/O MinIO |
| Export h264 | < 5× duração do vídeo | Depende do hardware; aceitável para uso esporádico |
| Geração de SEO (Claude) | < 30 s | Chamada única; timeout configurável |
| Thumbnail AI (extração + Claude + render) | < 2 min | 12 frames + 1 chamada Vision + 3 renders |
| Upload YouTube | depende do tamanho | Progresso via WS; sem timeout fixo no MVP |
| Resposta da API (endpoints síncronos) | < 500 ms p95 | Excluindo jobs assíncronos |

## Limites de upload

| Variável | Default | Configurável |
|---------|---------|-------------|
| `MAX_SOURCE_MB` | 2.048 (2 GB) | variável de ambiente |
| Formatos aceitos | mp4, mov, avi, mkv | hardcoded |
| Máximo de sources por projeto | 50 | hardcoded |
| Thumbnail máxima (YouTube) | 2 MB JPEG | regra YouTube |

## Segurança

| Área | Controle |
|------|---------|
| `ANTHROPIC_API_KEY` | apenas no backend — nunca retornada pela API ou exposta ao frontend |
| `GOOGLE_CLIENT_SECRET` | apenas no backend |
| OAuth refresh token | criptografado com AES-256-GCM (`OAUTH_ENCRYPTION_KEY`) em repouso no banco |
| Access token YouTube | obtido em runtime, nunca persistido |
| URLs assinadas MinIO | TTL curto (preview: 5 min, download: 24 h); sem acesso público ao bucket |
| SQL injection | TypeORM com queries parametrizadas |
| Path traversal | storage keys são UUIDs controlados pelo backend (nunca input do usuário) |
| CORS | configurado para `studio.gwan.cloud` apenas em produção |

## Observabilidade

| Item | Implementação |
|------|--------------|
| Logs de jobs | armazenados em `job.logs` (banco); streamados via WS durante execução |
| Logs de container | `json-file` driver, max 10 MB, 3 rotações (padrão GWAN) |
| Health check | `GET /api/health` — verificado pelo Portainer |
| Erros críticos | `job.error` no banco + log de container (sem alerting no MVP) |

## Disponibilidade

- **MVP:** sem SLA formal — uso interno do criador.
- **Dados:** PostgreSQL compartilhado (P0 GWAN) tem backup automático.
- **Artefatos:** MinIO sem lifecycle policy no MVP — limpeza manual.
- **Jobs em memória:** perdidos em restart — aceitável na Fase A (criador re-executa).

## Custos de IA (estimativa por projeto)

| Operação | Modelo | Tokens estimados | Custo ~USD |
|---------|--------|-----------------|-----------|
| Thumbnail plan (6 frames) | claude-sonnet-4-6 Vision | ~2.000 input + 500 output | ~$0.02 |
| SEO (texto) | claude-haiku-4-5 | ~500 input + 1.000 output | < $0.01 |
| **Total por vídeo** | — | — | **~$0.03** |

> Atualizar com preços atuais antes do deploy. Ver skill `claude-api` para tabela de preços.

## Stack backend — escolhas Django

| Componente | Escolha | Motivo |
|-----------|---------|--------|
| Framework web | Django 5.2 LTS | LTS até abril 2028; equipe já conhece Python do pipeline Colab |
| REST API | DRF 3.x | padrão Django para APIs |
| WebSocket | Django Channels 4.x + channels-redis | ASGI nativo; sem dependência Node |
| Jobs assíncronos | Celery 5 + Redis | durável desde o início (sem "em memória" no MVP) |
| ORM | Django ORM | integração nativa; migrations automáticas |
| Render thumbnail | Pillow | mesma lib usada no pipeline Colab original |
| FFmpeg | ffmpeg-python + subprocess | mesma linguagem do pipeline |
| Servidor ASGI | Daphne (incluso no Channels) | suporte a HTTP + WebSocket no mesmo processo |

## Fase B — extensões NFR

- **Lifecycle policy MinIO**: expirar sources após 30 dias, final.mp4 após 90 dias.
- **Rate limiting**: django-ratelimit para endpoints de jobs.
- **Auth multi-usuário**: Django auth + JWT (djangorestframework-simplejwt) + roles (criador vs. editor).
- **Celery beat**: tarefas agendadas (limpeza de jobs antigos, relatórios).
- **Flower**: monitor de workers Celery (porta 5555, acesso restrito).

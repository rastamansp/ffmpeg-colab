# SDD — Gwan Studio

**Spec Driven Design** do produto **Gwan Studio** — pipeline web de produção de vídeo para YouTube. O usuário faz upload de footage bruto, orquestra o merge de clipes, gera thumbnails com IA, obtém metadados SEO e publica no YouTube — tudo em uma interface web.

Esta documentação descreve **comportamento**, **contratos** e **arquitetura** do sistema. O código nos repositórios `gwan-studio` (frontend) e `gwan-studio-api` (backend) é instância desta especificação.

**Origem:** extraído do projeto [`ffmpeg-colab`](https://github.com/acentauric/ffmpeg-colab) (pipeline Google Colab) — a web app substitui a execução manual de células por um pipeline assíncrono gerenciado.

## O que muda do Colab para a web app

| Item | Colab original | Gwan Studio (este SDD) |
|------|----------------|------------------------|
| Execução | Células Python sequenciais, manual | Pipeline assíncrono por job, com status em tempo real |
| UI | Output de célula, texto puro | Interface React com visualização de progresso e preview |
| Storage | Google Drive | **MinIO compartilhado** (`s3.gwan.cloud`) |
| Auth YouTube | OAuth manual na sessão Colab | OAuth 2.0 persistido por projeto |
| Thumbnails | Script Python + PIL | Worker Django + Claude Vision (planejamento AI) |
| SEO | Gemini/OpenAI via API direta | Claude via API Django — chave só no backend |
| Deploy | Colab cloud (efêmero) | **Portainer + Traefik + SSL** (GWAN self-hosted) |
| Estado | Variáveis de sessão + JSON local | PostgreSQL (projetos) + MinIO (artefatos) |
| Runtime backend | Python (Colab) | **Python** (Django 5.2 LTS — continuidade natural) |

## Como ler este SDD

| Você é... | Comece por |
|-----------|------------|
| Stakeholder / PO | [00-overview/product-vision.md](00-overview/product-vision.md) |
| Engenheiro novo | [00-overview/system-context.md](00-overview/system-context.md) → [10-domain/domain-model.md](10-domain/domain-model.md) |
| Implementando a API | [40-architecture/backend-architecture.md](40-architecture/backend-architecture.md) → [20-features/](20-features/) F01–F09 |
| Implementando a Web | [40-architecture/frontend-architecture.md](40-architecture/frontend-architecture.md) → Django Templates + HTMX, F01, F05, F06, F07 |
| Design visual / UX | [40-architecture/design.md](40-architecture/design.md) — tokens, componentes, wizard, a11y |
| Contratos REST/WS | [30-contracts/api-contracts.md](30-contracts/api-contracts.md) |
| Deploy / infra | [40-architecture/deployment-architecture.md](40-architecture/deployment-architecture.md) |
| Bootstrap local | [40-architecture/dev-environment.md](40-architecture/dev-environment.md) |
| Segurança / custo / limites | [50-nfr/non-functional.md](50-nfr/non-functional.md) |

## Estrutura

```
spec/
├── 00-overview/          Visão de produto, contexto C4, glossário
├── 10-domain/            Modelo de domínio (Project, Source, Job, Thumbnail…)
├── 20-features/          Specs de feature (REQ/RN)
├── 30-contracts/         API REST, WebSocket, contrato do worker, MinIO
├── 40-architecture/      Backend (Clean Arch), frontend, deploy GWAN, dev bootstrap
└── 50-nfr/               Segurança, limites de upload/processamento, custo IA
```

## Índice de features

### Fase 0 — Validação de navegação (telas dummy, sem backend)

> **Obrigatório antes da Fase A.** Nenhum serviço real (banco, Celery, FFmpeg, IA, YouTube) é necessário. O objetivo é navegar pelo wizard completo com dados hardcoded e validar a UX com o time.

| ID | Feature | Spec |
|----|---------|------|
| **P01** | Telas dummy — wizard completo navegável com dados mock | [P01-telas-dummy.md](20-features/P01-telas-dummy.md) |

### Fase A — pipeline core (MVP)

| ID | Feature | Spec |
|----|---------|------|
| **F00** | Setup monorepo (Django 5.2 LTS + React/Vite + Clean Architecture) | [F00-monorepo-setup.md](20-features/F00-monorepo-setup.md) |
| **F01** | Gestão de projetos (criar, listar, arquivar) | [F01-project-management.md](20-features/F01-project-management.md) |
| **F02** | Upload de footage (multi-arquivo, câmeras, takes) | [F02-source-upload.md](20-features/F02-source-upload.md) |
| **F03** | Merge de clipes (FFmpeg stream copy + ordenação manual) | [F03-clip-merge.md](20-features/F03-clip-merge.md) |
| **F04** | Exportação do vídeo final (configurações de codec/bitrate) | [F04-export.md](20-features/F04-export.md) |
| **F05** | Thumbnail com IA (extração de frames → plan → render, 3 variantes) | [F05-thumbnail-ai.md](20-features/F05-thumbnail-ai.md) |
| **F06** | SEO com IA (título, descrição, tags via Claude) | [F06-seo-ai.md](20-features/F06-seo-ai.md) |
| **F07** | Publicação no YouTube (upload + metadados OAuth 2.0) | [F07-youtube-publish.md](20-features/F07-youtube-publish.md) |
| **F08** | Storage MinIO (workspace, artefatos, URLs assinadas) | [F08-storage-minio.md](20-features/F08-storage-minio.md) |
| **F09** | Jobs API + WebSocket (status/logs/preview em tempo real) | [F09-jobs-websocket.md](20-features/F09-jobs-websocket.md) |
| **F10** | Deploy (Portainer + Traefik + SSL) | [F10-deploy.md](20-features/F10-deploy.md) |

### Fase B — extensões

| ID | Feature | Spec |
|----|---------|------|
| F11 | YouTube Shorts (recorte 9:16 + reframe automático) | [F11-youtube-shorts.md](20-features/F11-youtube-shorts.md) |
| F12 | Extração de frames → Google Photos | [F12-google-photos.md](20-features/F12-google-photos.md) |
| F13 | Trilha sonora (BGM — ducking automático na fala) | [F13-bgm.md](20-features/F13-bgm.md) |
| F14 | Gerador de teaser (corte automático do momento-chave) | [F14-teaser.md](20-features/F14-teaser.md) |
| F15 | Telemetria GPX (Garmin/Wahoo → chapter markers no vídeo) | [F15-gpx-telemetry.md](20-features/F15-gpx-telemetry.md) |
| F16 | Ingestão Google Drive (download automático de fontes) | [F16-google-drive.md](20-features/F16-google-drive.md) |

Legenda: **Fase A** entrega o pipeline core ponta a ponta · **Fase B** adiciona extensões de conteúdo e integrações externas.

## Princípios de engenharia

### Fase 0 antes de qualquer serviço

**Sempre criar telas dummy antes de implementar o backend.** O desenvolvimento segue esta ordem obrigatória:

```
Fase 0 (telas dummy) → aprovação do time → Fase A (serviços reais)
```

Na Fase 0, todas as views retornam dados Python hardcoded — zero banco, zero Celery, zero IA. O objetivo é validar a navegação e o UX antes de qualquer investimento em infraestrutura. Ver [P01-telas-dummy.md](20-features/P01-telas-dummy.md).

---

### Spec Driven Design (SDD)

O desenvolvimento é guiado por esta especificação. **Nenhuma feature é implementada sem REQ/RN documentados.** O código é instância do SDD — dúvidas de comportamento são resolvidas na spec, não no código.

### Clean Architecture

Camadas independentes de framework, testáveis em isolamento:

```
domain → application → infrastructure → presentation
```

| Camada | Conteúdo | Regra |
|--------|----------|-------|
| **domain** | entidades (dataclasses) + ports (ABCs) | zero dependências externas |
| **application** | use cases | recebem ports por injeção; sem `import django` |
| **infrastructure** | adapters (MinIO, Claude, YouTube, FFmpeg, ORM) | implementam os ports |
| **presentation** | DRF views + Channels consumers | finas, sem lógica de domínio |

### SOLID

| Princípio | Aplicação neste projeto |
|-----------|------------------------|
| **S** — Single Responsibility | cada use case tem uma única responsabilidade; um arquivo por use case |
| **O** — Open/Closed | novos adapters adicionam código sem modificar ports ou use cases |
| **L** — Liskov Substitution | qualquer `IObjectStoragePort` (MinIO, S3, local) é intercambiável |
| **I** — Interface Segregation | ports separados por capacidade: `IVideoWorkerPort`, `IAiVisionPort`, `IAiTextPort` |
| **D** — Dependency Inversion | use cases dependem de ABCs (domain), nunca de implementações concretas (infra) |

### Use Cases

Cada ação do usuário mapeada em um use case dedicado em `application/`:

```
CreateProject · GetProject · ListProjects
UploadSource
StartMergeJob · StartExportJob
StartThumbnailJob · SelectThumbnail
StartSeoJob · ApproveSeo
StartPublishJob
```

Use cases são **framework-agnostic**: testados com `unittest.mock`, sem banco, sem HTTP, sem FFmpeg real.

### Django Templates + HTMX + daisyUI + design.md (frontend)

O frontend é **server-side rendering nativo Django**. Sem React, sem Vite, sem Node.js em produção.

- **Django Templates** — HTML gerado pelo servidor, sem SPA.
- **HTMX 2.x** — interatividade via atributos HTML (`hx-post`, `hx-get`, `hx-swap`); extensão `ws` para WebSocket em tempo real.
- **Alpine.js 3.x** — estado local mínimo no browser (drag-and-drop, toggles).
- **daisyUI 4.x** — componentes Tailwind sem JS (`btn`, `card`, `badge`, `modal`).

**Todas as regras de design estão em [`40-architecture/design.md`](40-architecture/design.md)**: paleta, tipografia, espaçamento, padrões de componentes daisyUI, responsividade, acessibilidade e dark mode. Nenhuma decisão visual deve ser tomada sem consultar esse arquivo.

---

## Convenções

- Requisitos: `REQ-FXX-NN` · Regras de negócio: `RN-FXX-NN` · Conformidade GWAN: `CONF-NN`
- Diagramas: **Mermaid**
- Backend em **Clean Architecture** com Django: domain (dataclasses/ABCs) → application (use cases) → infrastructure (adapters DRF/ORM/MinIO/FFmpeg) → presentation (views/consumers)
- IA: **Claude** (thumbnails + SEO); chaves **nunca** no client (só backend)
- **Jobs assíncronos** via **Celery + Redis** + **WebSocket** via **Django Channels** (FFmpeg é pesado; nunca request síncrono)
- Artefatos em **MinIO** (S3), nunca no filesystem persistente do container
- Google OAuth 2.0: tokens **só no backend** (refresh token persistido em banco, nunca exposto ao client)
- Deploy produção: **exclusivo via Portainer** (GWAN self-hosted)
- Portas dev: **slot 18** — API `3018`, Web `5191` (ver [alocação de portas](../../../../docs/spec/40-padroes/alocacao-de-portas.md))

## Versão

| | |
|---|---|
| Versão do SDD | 0.2.0 |
| Data | 2026-06-17 |
| Status | Draft — Fase A (pipeline core). Repositório de código a criar. |
| Stack Fase A | **Django 5.2 LTS** (Clean Architecture) · DRF 3.x · Django Channels 4.x · Celery 5 + Redis · **Django Templates + HTMX 2.x + Alpine.js + daisyUI** · Claude API (Vision + Text) · FFmpeg (subprocess/ffmpeg-python) · Google OAuth 2.0 / YouTube Data API v3 · MinIO (boto3) · PostgreSQL · WhiteNoise |

### Changelog

**0.2.0 (2026-06-17)** — migração de backend: NestJS → **Django 5.2 LTS**. Continuidade natural com o pipeline Python original (`ffmpeg-colab`). Jobs via Celery + Redis (substitui worker Node.js). WebSocket via Django Channels.

**0.1.0 (2026-06-17)** — versão inicial extraída do `ffmpeg-colab` (Colab pipeline → web app, stack NestJS).

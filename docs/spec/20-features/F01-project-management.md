# F01 — Gestão de projetos

## Objetivo

Criar, listar e arquivar projetos. Cada projeto representa a produção de um vídeo.

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F01-01 | `POST /api/projects` cria projeto com `name`, `channel_name` |
| REQ-F01-02 | `GET /api/projects` lista projetos paginados, ordenados por `created_at desc` |
| REQ-F01-03 | `GET /api/projects/:id` retorna projeto com sources, último job de cada tipo e status |
| REQ-F01-04 | `PATCH /api/projects/:id` atualiza `name`, `channel_name`, `export_settings` |
| REQ-F01-05 | `DELETE /api/projects/:id` arquiva (soft delete) o projeto |
| REQ-F01-06 | Status do projeto é calculado automaticamente a partir dos jobs concluídos |

## Regras de negócio

| ID | Regra |
|----|-------|
| RN-F01-01 | `name` é obrigatório, máximo 100 caracteres |
| RN-F01-02 | Projeto recém-criado tem status `draft` |
| RN-F01-03 | Status `draft` → `sources_ready` quando ao menos 1 source está em `ready` |
| RN-F01-04 | Não é possível deletar projeto com `status = 'published'` — apenas arquivar |
| RN-F01-05 | `export_settings.codec = 'copy'` é o padrão (stream copy, sem re-encode) |

## UI (Web)

**Tela: Lista de projetos**
- Cards com nome, status badge, data de criação, thumbnail (se existir), link para YouTube (se publicado).
- Botão "Novo Projeto" → modal de criação.
- Filtro por status.

**Tela: Detalhe do projeto**
- Header: nome, status, canal YouTube.
- Seções: Sources → Merge → Export → Thumbnails → SEO → Publicar.
- Cada seção mostra o status do job correspondente e botão de ação.
- Log stream ao vivo (WebSocket) durante jobs em execução.

## Fluxo Clean Architecture

```
CreateProjectUseCase
  → ProjectRepository (IProjectRepository)
  → retorna Project criado

GetProjectUseCase
  → ProjectRepository
  → retorna Project com jobs summary
```

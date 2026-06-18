# P01 — Telas dummy (Fase 0 — validação de navegação)

## Objetivo

Criar todas as telas da plataforma com **dados hardcoded** antes de implementar qualquer serviço real (banco, Celery, FFmpeg, IA, YouTube). O objetivo é validar a navegação, o fluxo do wizard e o visual com o time antes de começar o backend.

> **Regra de ouro da Fase 0**: nenhuma view acessa banco, S3, Celery ou API externa. Tudo é mock. O servidor Django sobe com `python manage.py runserver` sem Redis, sem MinIO, sem PostgreSQL.

---

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-P01-01 | Todas as telas do wizard navegáveis (6 steps: Sources → Merge → Export → Thumbnails → SEO → Publicar) |
| REQ-P01-02 | Tabs habilitadas/desabilitadas conforme estado mock do projeto (simulado em variável Python) |
| REQ-P01-03 | Cada step exibe o estado "concluído" com dados fictícios (nomes de arquivo fake, preview placeholder, SEO de exemplo) |
| REQ-P01-04 | `JobLogStream` visível com logs hardcoded (simula progresso de FFmpeg) |
| REQ-P01-05 | Todos os estados de job visíveis como HTML estático: `pending`, `running`, `done`, `failed` |
| REQ-P01-06 | daisyUI aplicado em todos os componentes — visual final (paleta, tipografia, espaçamento conforme `design.md`) |
| REQ-P01-07 | HTMX funcional na navegação entre tabs (hx-get carrega partial correto) |
| REQ-P01-08 | Formulários presentes e com campos corretos, mas submit retorna mock response (sem efeito real) |
| REQ-P01-09 | Dark mode toggle funcionando (troca `data-theme` no `<html>`) |
| REQ-P01-10 | Responsivo em `md` e `lg` (verificar layout conforme `design.md`) |

---

## Regras de negócio

| ID | Regra |
|----|-------|
| RN-P01-01 | Views dummy retornam contexto Python puro (dicts com dados fake) — zero ORM |
| RN-P01-02 | Nenhuma dependência de serviços externos: sem `DATABASE_URL`, sem `REDIS_URL`, sem `MINIO_*`, sem `ANTHROPIC_API_KEY` |
| RN-P01-03 | Estado do wizard mockado via variável de contexto `project_phase` (ex.: `"merge_done"`, `"seo_approved"`) — permite testar todas as combinações de tabs habilitadas/desabilitadas |
| RN-P01-04 | Upload de arquivo presente na tela mas interceptado por view dummy (retorna lista fake de sources) |
| RN-P01-05 | Aprovação da Fase 0 pelo time é pré-requisito para iniciar qualquer feature da Fase A |

---

## Telas a implementar

### Base (`templates/base.html`)
- Nav com logo, nome do projeto, toggle dark mode
- Container principal com `max-w-5xl mx-auto`
- Bloco de mensagens Django (`{% for message in messages %}`)
- Scripts: `htmx.min.js`, `htmx-ext-ws.js`, `alpine.min.js`
- Tailwind + daisyUI via CDN (sem build step na Fase 0)

### Lista de projetos (`templates/projects/list.html`)
- Tabela/grid com 2–3 projetos fake
- Botão "Novo projeto" (abre modal daisyUI com form de criação — submit mock)
- Badge de status por projeto

### Detalhe do projeto (`templates/projects/detail.html`)
- Header: nome do projeto, canal, badge de status
- Tabs daisyUI com 6 steps (ver estados abaixo)
- Conteúdo do step carregado via `hx-get` (partial HTML)

### Step 1 — Sources (`templates/sources/_dropzone.html`)
- Dropzone com ícone de upload + texto
- Lista de 3 arquivos fake (nome, tamanho, câmera, status `ready`)
- Botão "Remover" (mock)

### Step 2 — Merge (`templates/merge/_editor.html` + `_preview.html`)
- Lista ordenável com drag handle (Alpine.js x-sort — apenas visual, sem persistência)
- Botão "Iniciar Merge"
- Preview: `<video>` com poster placeholder + texto "merged.mp4 (2 min 34 s)"
- `_job_log.html` com 5–6 linhas de log fake

### Step 3 — Export (`templates/export/_settings.html` + `_preview.html`)
- Select de codec (h264/h265/copy), input de bitrate
- Botão "Exportar"
- Preview: `<video>` placeholder + botão download (href="#")

### Step 4 — Thumbnails (`templates/thumbnail/_generator.html` + `_picker.html`)
- Botão "Gerar thumbnails com IA"
- Grid 3 cards (A / B / C) com imagem placeholder 1280×720
- Botão "Selecionar" em cada card (mock — marca o card com badge "Selecionado")

### Step 5 — SEO (`templates/seo/_generator.html` + `_editor.html`)
- Textarea de contexto + botão "Gerar SEO"
- Campos editáveis: título (ex.: "Como eu pedalei 200 km em 3 dias"), descrição, tags
- Botão "Aprovar SEO" (muda badge para "Aprovado")

### Step 6 — Publicar (`templates/publish/_oauth.html` + `_panel.html`)
- Botão "Conectar YouTube" (mock — exibe badge "Conectado")
- Checklist de pré-condições (thumbnail selecionada ✓, SEO aprovado ✓, export concluído ✓)
- Botão "Publicar no YouTube" (mock — exibe toast de sucesso)

---

## Estrutura de views dummy

```python
# presentation/views/dummy.py
# Todas as views retornam contexto hardcoded — sem ORM, sem serviços

MOCK_PROJECT = {
    "id": "mock-001",
    "name": "Pedalada 200km — Serra Gaúcha",
    "channel_name": "@ramortinho",
    "phase": "seo_approved",   # controla quais tabs ficam habilitadas
}

MOCK_SOURCES = [
    {"id": "s1", "original_filename": "GoPro_001.mp4", "camera": "GoPro", "duration_sec": 1234, "status": "ready"},
    {"id": "s2", "original_filename": "GoPro_002.mp4", "camera": "GoPro", "duration_sec": 987,  "status": "ready"},
    {"id": "s3", "original_filename": "iPhone_001.mp4", "camera": "iPhone", "duration_sec": 456, "status": "ready"},
]

MOCK_JOB_LOGS = [
    "[00:00] Iniciando merge de 3 fontes...",
    "[00:02] ffmpeg: concat filter aplicado",
    "[00:15] ffmpeg: processando 00:01:00 / 00:02:34",
    "[00:28] ffmpeg: processando 00:02:00 / 00:02:34",
    "[00:31] ffmpeg: concluído — merged.mp4 (287 MB)",
    "[00:32] Upload para MinIO: studio/mock-001/merged.mp4 ✓",
]

def project_detail(request, project_id):
    return render(request, "projects/detail.html", {
        "project": MOCK_PROJECT,
        "steps": _build_steps(MOCK_PROJECT["phase"]),
    })

def sources_step(request, project_id):
    template = "sources/_dropzone.html" if request.htmx else "projects/detail.html"
    return render(request, template, {"project": MOCK_PROJECT, "sources": MOCK_SOURCES})

# ... demais views seguem o mesmo padrão
```

---

## Critérios de aceite da Fase 0

Antes de iniciar qualquer feature da Fase A, o time deve validar:

- [ ] Navegar pelos 6 steps sem erro
- [ ] Visualizar todos os estados de job (`pending`, `running`, `done`, `failed`) como HTML
- [ ] Confirmar que o visual segue o `design.md` (paleta, tipografia, daisyUI)
- [ ] Confirmar que o wizard faz sentido para o fluxo real de produção de vídeo
- [ ] Identificar ajustes de UX necessários (reorganização de steps, campos faltantes, etc.)
- [ ] Aprovação explícita do Ramon antes de prosseguir para a Fase A

---

## O que NÃO fazer na Fase 0

- ❌ Conectar banco de dados (nem SQLite)
- ❌ Subir Redis, MinIO ou qualquer serviço Docker
- ❌ Chamar Claude API, YouTube API ou FFmpeg
- ❌ Implementar lógica de negócio nas views (validações, cálculos)
- ❌ Avançar para Fase A sem aprovação explícita do time

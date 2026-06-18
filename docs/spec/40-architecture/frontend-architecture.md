# Arquitetura de frontend — Gwan Studio

## Stack

- **Django Templates** — motor nativo Django (server-side rendering)
- **HTMX 2.x** — interatividade sem SPA (`hx-post/hx-get`, `hx-swap`, extensão WebSocket)
- **Alpine.js 3.x** — estado local no browser (drag-and-drop, toggles, timers)
- **Tailwind CSS 3.x** — utilitários de estilo (CDN em dev, Tailwind CLI standalone em prod)
- **daisyUI 4.x** — componentes Tailwind sem JS (`btn`, `card`, `badge`, `progress`, `modal`)

> **Motivação**: Django Templates + HTMX é a stack nativa Django. Zero framework JS separado, zero processo Node em produção. O mesmo container Daphne serve HTML, API REST e WebSocket. Bundle total: `htmx.min.js` (~14 kb) + `alpine.min.js` (~15 kb).

## Estrutura de pastas (dentro de `backend/`)

```
backend/
├── templates/
│   ├── base.html                  # layout base (nav, toast, scripts HTMX/Alpine)
│   ├── components/                # partials reutilizáveis
│   │   ├── _job_status.html       # badge colorido por status
│   │   ├── _job_log.html          # terminal de logs (HTMX WS swap)
│   │   ├── _progress_bar.html     # barra de progresso genérica
│   │   └── _video_player.html     # player nativo com URL assinada MinIO
│   ├── projects/
│   │   ├── list.html              # GET /projects/
│   │   └── detail.html            # GET /projects/<id>/ (hub com tabs)
│   ├── sources/
│   │   └── _dropzone.html         # partial HTMX: upload drag-and-drop
│   ├── merge/
│   │   ├── _editor.html           # partial: lista ordenável (Alpine.js x-sort)
│   │   └── _preview.html          # partial: player merged.mp4
│   ├── export/
│   │   ├── _settings.html         # partial: form codec/bitrate
│   │   └── _preview.html          # partial: player final.mp4 + download
│   ├── thumbnail/
│   │   ├── _generator.html        # partial: botão gerar + log stream
│   │   └── _picker.html           # partial: grid A/B/C + selecionar
│   ├── seo/
│   │   ├── _generator.html        # partial: campo contexto + botão gerar
│   │   └── _editor.html           # partial: título/descrição/tags + aprovar
│   └── publish/
│       ├── _oauth.html            # partial: botão conectar YouTube OAuth
│       └── _panel.html            # partial: checklist + botão publicar
│
└── static/
    ├── js/
    │   ├── htmx.min.js            # HTMX 2.x
    │   ├── htmx-ext-ws.js         # extensão WebSocket do HTMX
    │   └── alpine.min.js          # Alpine.js 3.x
    └── css/
        └── app.css                # Tailwind compilado (inclui daisyUI)
```

## Django Views (`presentation/views/`)

Cada view:
1. Renderiza template completo na navegação direta.
2. Retorna partial HTML quando a requisição vem do HTMX (`HX-Request: true`).

```python
# presentation/views/projects.py
from django.shortcuts import render
from django_htmx.http import HttpResponseClientRefresh
from application.projects.get_project import GetProjectUseCase

def project_detail(request, project_id):
    project = GetProjectUseCase(...).execute(project_id)
    template = "projects/_detail_partial.html" if request.htmx else "projects/detail.html"
    return render(request, template, {"project": project})
```

> `request.htmx` é fornecido pelo pacote `django-htmx` (adicionado em `requirements/base.txt`).

## HTMX — padrões de uso

### Iniciar job (POST → substitui bloco de status)

```html
<button class="btn btn-primary"
        hx-post="{% url 'jobs:start_merge' project.id %}"
        hx-target="#job-status"
        hx-swap="outerHTML"
        hx-indicator="#spinner">
  Iniciar Merge
</button>
<div id="job-status">{% include "components/_job_status.html" %}</div>
```

### Upload de footage (multipart)

```html
<form hx-post="{% url 'sources:upload' project.id %}"
      hx-encoding="multipart/form-data"
      hx-target="#source-list"
      hx-swap="innerHTML">
  {% csrf_token %}
  <input type="file" name="files" multiple accept=".mp4,.mov,.avi,.mkv">
  <button class="btn btn-outline" type="submit">Enviar</button>
</form>
```

### Logs em tempo real (WebSocket via HTMX ext)

```html
<div hx-ext="ws"
     ws-connect="/ws/projects/{{ project.id }}/"
     id="job-log"
     class="font-mono text-xs bg-base-300 p-4 h-48 overflow-y-auto"
     role="log"
     aria-live="polite">
  {% for line in job.log_lines %}
    <p>{{ line }}</p>
  {% endfor %}
</div>
```

> O Channels consumer envia fragmentos HTML via WebSocket. O HTMX ext `ws` faz `innerHTML` swap automático no `#job-log` ao receber cada mensagem.

### Ordenação drag-and-drop (Alpine.js)

```html
<ul x-data="{ sources: {{ sources_json|safe }} }" x-sort="sources">
  <template x-for="src in sources" :key="src.id">
    <li x-sort:item="src.id" class="flex items-center gap-2 p-2 bg-base-200 rounded cursor-grab">
      <span class="drag-handle">⠿</span>
      <span x-text="src.original_filename"></span>
    </li>
  </template>
</ul>
<input type="hidden" name="source_order"
       :value="JSON.stringify(sources.map(s => s.id))">
```

## Wizard de projeto (tabs HTMX)

O detalhe do projeto é um wizard linear de 6 steps via tabs daisyUI com conteúdo carregado por `hx-get`:

```html
<!-- templates/projects/detail.html -->
<div role="tablist" class="tabs tabs-bordered">
  {% for step in steps %}
    <button role="tab"
            class="tab {% if step.active %}tab-active{% endif %}"
            hx-get="{% url step.view_name project.id %}"
            hx-target="#step-content"
            hx-swap="innerHTML"
            hx-push-url="false"
            {% if not step.enabled %}disabled aria-disabled="true"{% endif %}>
      {{ step.label }}
      <span class="badge badge-sm {{ step.badge_class }} ml-1">{{ step.status_label }}</span>
    </button>
  {% endfor %}
</div>
<div id="step-content" class="py-6">
  {% include current_step_template %}
</div>
```

Steps e pré-condições:

| # | Label | Habilitado quando |
|---|-------|-------------------|
| 1 | Sources | sempre |
| 2 | Merge | ≥ 2 sources com status `ready` |
| 3 | Export | job merge `done` |
| 4 | Thumbnails | job merge `done` |
| 5 | SEO | sempre (independente do export) |
| 6 | Publicar | thumbnail selecionada + SEO aprovado |

## Django Channels — Channels consumer envia HTML

```python
# presentation/ws/consumers.py
class JobConsumer(AsyncWebsocketConsumer):
    async def job_update(self, event):
        job = event["data"]
        html = render_to_string("components/_job_status.html", {"job": job})
        await self.send(text_data=html)   # HTMX faz swap direto no DOM
```

## Dependências adicionadas

```
# requirements/base.txt (acrescentar)
django-htmx==1.*       # request.htmx + middleware
whitenoise[brotli]==6.*  # serve static files sem nginx
```

## Convenções

- Templates em `snake_case`; partials prefixados com `_` (ex: `_job_status.html`).
- Views detectam HTMX via `request.htmx` (nunca via `request.headers` direto).
- `{% url %}` em todos os links e forms — nunca hardcode de URL.
- `{% csrf_token %}` obrigatório em todo `<form>` com método POST.
- Datas via `{{ date|date:"d/m/Y H:i" }}` (pt-BR).
- Dados dinâmicos locais (drag-and-drop, timer, toggle) → Alpine.js.
- Tudo que envolve o servidor → HTMX.
- Sem `fetch()` ou `axios` no código JS manual.

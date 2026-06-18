# Design — Gwan Studio

Regras de design visual e de experiência do usuário. Todo trabalho de frontend **deve seguir este documento**. Dúvidas de aparência são resolvidas aqui, não no código.

---

## Identidade visual

| Token | Valor | Uso |
|-------|-------|-----|
| Cor primária | `#18181B` (zinc-900) | backgrounds escuros, texto principal dark |
| Cor de ação | `#3B82F6` (blue-500) | botões de ação primária, links, foco |
| Cor de sucesso | `#22C55E` (green-500) | job concluído, status "done" |
| Cor de erro | `#EF4444` (red-500) | job falhou, alertas críticos |
| Cor de aviso | `#F59E0B` (amber-500) | job em progresso, alertas informativos |
| Cor de texto muted | `#71717A` (zinc-500) | labels secundários, metadados |
| Background base | `#FAFAFA` (zinc-50) | fundo da app (modo claro) |

> Todos os tokens são configurados no `tailwind.config.js` via daisyUI theme `gwan`. Dark mode automático via `data-theme="gwan-dark"` (daisyUI) ou `class="dark"` (Tailwind).

---

## Tipografia

| Uso | Fonte | Classe Tailwind |
|-----|-------|-----------------|
| Corpo padrão | Inter (system-ui fallback) | `text-sm text-zinc-700` |
| Título de página | Inter 600 | `text-2xl font-semibold tracking-tight` |
| Título de seção | Inter 500 | `text-lg font-medium` |
| Label de campo | Inter 400 | `text-sm font-medium text-zinc-700` |
| Metadado / caption | Inter 400 | `text-xs text-zinc-500` |
| Código / log | JetBrains Mono (monospace) | `font-mono text-sm` |

---

## Espaçamento e layout

- **Grid**: 12 colunas, gap `gap-4` (16 px). Conteúdo máximo `max-w-5xl mx-auto px-4`.
- **Padding de card**: `p-6` interno.
- **Separação entre seções**: `space-y-6`.
- **Formulários**: campos com `space-y-4`; label acima do input (nunca placeholder como substituto de label).
- **Sidebar** (se houver): `w-64` fixa, `border-r`.

---

## Componentes — regras de uso (daisyUI)

> daisyUI é um plugin Tailwind — os componentes são **classes CSS**, sem JS de terceiro. Funcionam em qualquer template Django.

### Button

```html
<!-- ação primária -->
<button class="btn btn-primary">Iniciar Merge</button>

<!-- ação secundária -->
<button class="btn btn-outline">Ver detalhes</button>

<!-- ação em lista (ghost) -->
<button class="btn btn-ghost btn-sm">Editar</button>

<!-- ação irreversível -->
<button class="btn btn-error">Excluir projeto</button>
```

- Botão de ação principal: **um por seção**, alinhado à direita com `flex justify-end`.
- Estado de loading: `<button class="btn btn-primary loading">` — nunca duplo clique (usar `hx-disabled-elt="this"` no HTMX).

### Card

```html
<div class="card bg-base-100 shadow-sm border border-base-200">
  <div class="card-body">
    <h2 class="card-title">Step 2 — Merge</h2>
    <p class="text-sm text-base-content/70">Ordene os clipes e inicie o merge.</p>
    <!-- conteúdo -->
  </div>
</div>
```

### Badge (status de Job)

| Status | Classe daisyUI | Ícone |
|--------|---------------|-------|
| `pending` | `badge badge-outline` | ⏳ |
| `running` | `badge badge-info` | ⟳ (classe `animate-spin`) |
| `done` | `badge badge-success` | ✓ |
| `failed` | `badge badge-error` | ✗ |

```html
<span class="badge badge-success">✓ Concluído</span>
```

### Progress

```html
<progress class="progress progress-primary w-full" value="60" max="100"></progress>
<p class="text-xs text-base-content/70 mt-1">Enviando 3 de 5 arquivos...</p>
```

### Toast (alerts)

```html
<!-- via Django messages framework + include no base.html -->
<div class="alert alert-success">
  <span>Merge concluído com sucesso.</span>
</div>

<div class="alert alert-error">
  <span>{{ message }}</span>
</div>
```

Duração controlada por Alpine.js (`x-data="{ show: true }" x-show="show" x-init="setTimeout(() => show = false, 4000)"`). Nunca usar `alert()` nativo.

### Modal (Dialog)

```html
<!-- Confirmação de exclusão -->
<dialog id="modal-delete" class="modal">
  <div class="modal-box">
    <h3 class="font-bold text-lg">Excluir projeto?</h3>
    <p class="py-4">Esta ação não pode ser desfeita.</p>
    <div class="modal-action">
      <form method="dialog"><button class="btn">Cancelar</button></form>
      <button class="btn btn-error"
              hx-delete="{% url 'projects:delete' project.id %}"
              hx-target="body">Confirmar</button>
    </div>
  </div>
</dialog>
<button class="btn btn-ghost btn-sm"
        onclick="document.getElementById('modal-delete').showModal()">
  Excluir
</button>
```

---

## Wizard de projeto (UX)

O detalhe de um projeto é dividido em **6 steps lineares** via tabs daisyUI. Regras:

1. Cada tab tem um **badge de status** ao lado do título.
2. Tab bloqueada: atributo `disabled`, cursor `not-allowed`, badge `badge-outline`.
3. Tab ativa: sem `disabled`, badge condizente com status do job.
4. Navegação: o usuário pode **revisitar** steps concluídos.
5. Progresso do job exibido em terminal scrollável: `h-48 overflow-y-auto bg-base-300 font-mono text-xs p-4`.

---

## Responsividade

| Breakpoint | Comportamento |
|-----------|---------------|
| `sm` (< 640 px) | layout coluna única; sidebar colapsada em menu hambúrguer |
| `md` (640–1024 px) | duas colunas onde aplicável (lista + detalhe) |
| `lg` (> 1024 px) | layout completo três colunas / sidebar fixa |

> MVP: priorizar `md` e `lg`. Mobile (`sm`) é nice-to-have na Fase A.

---

## Acessibilidade

- Todos os componentes interativos têm foco visível (`ring-2 ring-blue-500`).
- Imagens de thumbnail com `alt` descritivo.
- `aria-label` em botões que usam apenas ícone.
- Contraste mínimo: 4.5:1 para texto corpo (WCAG AA).
- `JobLogStream` com `role="log"` e `aria-live="polite"`.

---

## Dark mode

A app suporta dark mode via classe `dark` no `<html>`. O shadcn/ui usa variáveis CSS HSL — o toggle alterna o tema sem recarregar.

- Implementação: `next-themes` (ou `ThemeProvider` customizado com `localStorage`).
- Ícone: sol/lua no header, canto superior direito.
- Padrão inicial: `system` (respeita preferência do SO).

---

## Ícones

Usar **Lucide React** (incluso pelo shadcn/ui):

```tsx
import { Upload, Merge, Download, Wand2, Youtube, CheckCircle2 } from 'lucide-react'
```

- Tamanho padrão: `size={16}` inline com texto, `size={20}` standalone.
- Nunca misturar com outras bibliotecas de ícones no mesmo projeto.

---

## O que NÃO fazer

- ❌ Cores hardcoded (`#fff`, `rgb(...)`) — usar tokens Tailwind.
- ❌ Estilos inline (`style={{ color: 'red' }}`) — usar classes Tailwind.
- ❌ Componentes MUI, Ant Design, Chakra, shadcn/ui — **apenas daisyUI** (sem React).
- ❌ `alert()`, `confirm()`, `prompt()` nativos — usar Dialog/Toast.
- ❌ Imagens sem `alt`.
- ❌ Botão primário desabilitado sem feedback visual (tooltip ou mensagem explicando por que).

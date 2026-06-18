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

> Todos os tokens são mapeados em variáveis CSS HSL via `tailwind.config.ts` para compatibilidade com shadcn/ui (dark mode automático via `class="dark"`).

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

## Componentes — regras de uso (shadcn/ui)

### Button

| Variante | Quando usar |
|----------|-------------|
| `default` | ação primária de um step (Merge, Exportar, Publicar) |
| `outline` | ação secundária ou opcional |
| `ghost` | ações em listas/tabelas (editar, remover) |
| `destructive` | ações irreversíveis (excluir projeto, revogar OAuth) |

- Botão de ação principal: **um por seção**, alinhado à direita.
- Estado de loading: sempre `disabled` + spinner (nunca duplo clique).

### Card

Wrapper padrão de cada step do wizard:

```tsx
<Card>
  <CardHeader>
    <CardTitle>Step 2 — Merge</CardTitle>
    <CardDescription>Ordene os clipes e inicie o merge.</CardDescription>
  </CardHeader>
  <CardContent>
    {/* conteúdo */}
  </CardContent>
</Card>
```

### Badge (status de Job)

| Status | Variante | Ícone |
|--------|----------|-------|
| `pending` | `outline` | ⏳ |
| `running` | `secondary` (azul suave) | ⟳ (spin) |
| `done` | `default` (verde) | ✓ |
| `failed` | `destructive` | ✗ |

### Progress

- Usado para upload de footage e progresso de FFmpeg.
- Sempre acompanhado de texto descritivo (`"Enviando 3 de 5 arquivos..."`).

### Toast

- Sucesso: `toast({ title: "Merge concluído", variant: "default" })` — dura 4 s, dismissível.
- Erro: `toast({ title: "Falhou", description: err.message, variant: "destructive" })` — dura 8 s.
- Nunca usar `alert()` nativo.

### Dialog

- Confirmações de ação irreversível (exclusão, revogação OAuth): Dialog com dois botões (`Cancelar` / `Confirmar`).
- OAuth flow YouTube: Dialog modal com iframe/redirect.

---

## Wizard de projeto (UX)

O detalhe de um projeto é dividido em **6 steps lineares** via `Tabs` do shadcn/ui. Regras:

1. Cada tab tem um **indicador de status** (badge ao lado do título).
2. Tab bloqueada: `disabled`, cursor `not-allowed`, badge `outline`.
3. Tab ativa: sem `disabled`, badge condizente com status do job.
4. Navegação: o usuário pode **revisitar** steps concluídos (tabs não bloqueadas retroativamente).
5. Progresso do job exibido em `JobLogStream` — terminal scrollável fixo em `h-48`, fundo `bg-zinc-950 text-zinc-100 font-mono text-xs`.

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
- ❌ Componentes MUI, Ant Design, Chakra — **apenas shadcn/ui**.
- ❌ `alert()`, `confirm()`, `prompt()` nativos — usar Dialog/Toast.
- ❌ Imagens sem `alt`.
- ❌ Botão primário desabilitado sem feedback visual (tooltip ou mensagem explicando por que).

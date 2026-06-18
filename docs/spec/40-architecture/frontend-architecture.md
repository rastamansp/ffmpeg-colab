# Arquitetura de frontend — Gwan Studio

## Stack

- **React 18** + **Vite** + **TypeScript strict**
- **Tailwind CSS** + **shadcn/ui** (componentes GWAN padrão)
- **TanStack Query** (data fetching + cache)
- **Zustand** (estado global leve — ex.: job ativo, WS connection)
- **React Router v6** (SPA routing)
- **Socket.io-client** (WebSocket)

## Estrutura de pastas

```
apps/web/src/
├── features/
│   ├── projects/
│   │   ├── ProjectList.tsx        # lista de projetos
│   │   ├── ProjectDetail.tsx      # hub do projeto (seções em steps)
│   │   └── ProjectForm.tsx        # criar/editar projeto
│   ├── sources/
│   │   ├── SourceDropzone.tsx     # upload com drag-and-drop
│   │   └── SourceList.tsx         # lista de sources + preview
│   ├── merge/
│   │   ├── MergeEditor.tsx        # drag-and-drop de ordenação + botão merge
│   │   └── MergePreview.tsx       # player do merged.mp4
│   ├── export/
│   │   ├── ExportSettings.tsx     # form de configurações
│   │   └── ExportPreview.tsx      # player + botão download
│   ├── thumbnail/
│   │   ├── ThumbnailGenerator.tsx # botão gerar + log stream
│   │   └── ThumbnailPicker.tsx    # grid A/B/C + selecionar
│   ├── seo/
│   │   ├── SeoGenerator.tsx       # campo contexto + botão gerar
│   │   └── SeoEditor.tsx          # título/descrição/tags editáveis + aprovar
│   └── publish/
│       ├── YoutubeConnect.tsx      # OAuth flow
│       └── PublishPanel.tsx        # checklist + botão publicar + progresso
│
├── components/
│   ├── ui/                        # shadcn/ui re-exports
│   ├── JobLogStream.tsx           # terminal live de logs (WebSocket)
│   ├── JobStatusBadge.tsx         # badge colorido por status
│   ├── VideoPlayer.tsx            # player nativo com URL assinada
│   └── ProgressBar.tsx            # barra de progresso genérica
│
├── hooks/
│   ├── useProjectWebSocket.ts     # subscrição WS + estado do job ativo
│   ├── useProjects.ts             # TanStack Query para projetos
│   ├── useSources.ts              # upload + listagem de sources
│   └── useJob.ts                  # polling/WS de status de job
│
├── lib/
│   ├── api.ts                     # cliente axios com base URL
│   ├── ws.ts                      # socket.io-client singleton
│   └── utils.ts                   # formatters, helpers
│
├── pages/
│   ├── Home.tsx                   # redirect para /projects
│   ├── Projects.tsx               # lista
│   └── ProjectPage.tsx            # detalhe (usa ProjectDetail)
│
└── main.tsx                       # entry point + providers
```

## Fluxo de UX (detalhe do projeto)

O detalhe do projeto é um **wizard linear** com steps habilitados conforme o status avança:

```
[1] Sources    → [2] Merge    → [3] Export   → [4] Thumbnails → [5] SEO → [6] Publicar
  Upload         Ordena e        Codec/bitrate   Gerar 3         Gerar e     YouTube
  footage        faz merge       (opcional)      variantes       aprovar     OAuth + upload
```

Cada step:
- **Bloqueado** (cinza) se pré-condição não atendida.
- **Pronto para ação** (azul) — botão de iniciar disponível.
- **Em progresso** — `JobLogStream` visível, botão desabilitado.
- **Concluído** (verde) — resultado exibido, botão para avançar ao próximo step.
- **Falhou** (vermelho) — mensagem de erro + botão re-tentar.

## `useProjectWebSocket`

```typescript
function useProjectWebSocket(projectId: string) {
  // Conecta ao WS e subscreve ao projeto
  // Atualiza estado local do job ativo via Zustand
  // Ao receber 'job.update' com status 'done', invalida query do projeto (TanStack)
  // Reconexão automática com backoff exponencial
}
```

## Design system — shadcn/ui

O frontend usa **shadcn/ui** como biblioteca de componentes. Diferenciais frente a bibliotecas tradicionais (MUI, Ant Design):

| Característica | shadcn/ui |
|----------------|-----------|
| Instalação | `npx shadcn-ui add button` — código copiado para `src/components/ui/`, não dependência opaca |
| Customização | editar diretamente o arquivo copiado; sem override de CSS de terceiro |
| Primitivos | Radix UI — ARIA e navegação por teclado corretos por padrão |
| Estilização | Tailwind CSS + variáveis CSS (tokens de cor, radius) |
| Bundlesize | apenas componentes usados entram no bundle |

### Componentes disponíveis no projeto

```
components/ui/
├── button.tsx          # Button (variant: default/outline/ghost/destructive)
├── card.tsx            # Card + CardHeader + CardContent + CardFooter
├── badge.tsx           # Badge (status dos jobs: pending/running/done/failed)
├── dialog.tsx          # Dialog (confirmações, OAuth flow)
├── progress.tsx        # Progress (upload + processamento FFmpeg)
├── tabs.tsx            # Tabs (steps do wizard do projeto)
├── input.tsx           # Input + Label
├── textarea.tsx        # Textarea (SEO description)
├── separator.tsx       # Separator
├── toast.tsx           # Toast (feedback de ações)
└── skeleton.tsx        # Skeleton (loading states)
```

### Tokens de cor (tailwind.config.ts)

```typescript
// Mapeados nas variáveis CSS do shadcn/ui (HSL)
colors: {
  primary: 'hsl(var(--primary))',        // ação principal
  secondary: 'hsl(var(--secondary))',    // ação secundária
  destructive: 'hsl(var(--destructive))', // erro/exclusão
  muted: 'hsl(var(--muted))',            // texto secundário
  accent: 'hsl(var(--accent))',          // hover/highlight
}
```

## Convenções

- Componentes em PascalCase; hooks em camelCase com `use` prefix.
- Cada feature tem seu próprio diretório — sem componentes "globais" para lógica de feature.
- Chamadas de API apenas via `lib/api.ts` (nunca fetch direto em componentes).
- Datas formatadas em pt-BR com `Intl.DateTimeFormat`.
- Sem dependência de estado global para dados de servidor — TanStack Query é a fonte de verdade.
- Novos componentes UI via `npx shadcn-ui add <nome>` — nunca criar do zero o que o shadcn/ui já oferece.

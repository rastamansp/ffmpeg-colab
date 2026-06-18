# F05 — Thumbnail com IA

## Objetivo

Gerar 3 variantes de thumbnail (1280×720 px) usando Claude Vision para planejar o layout e o worker para renderizar.

## Pipeline

```
merged.mp4
    │
    ▼
[1] Extração de frames candidatos (ffmpeg, N=12 frames distribuídos)
    │
    ▼
[2] Claude Vision analisa frames + contexto do vídeo
    → retorna 3 ThumbnailPlan (A, B, C) — frame escolhido + layout + texto + paleta
    │
    ▼
[3] Worker renderiza cada variante (sharp + canvas ou Jimp)
    → aplica texto sobreposto, ajusta cores/contraste, gera 1280×720 JPEG
    │
    ▼
[4] Upload para MinIO: studio/<proj>/thumbnails/A.jpg, B.jpg, C.jpg
```

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F05-01 | `POST /api/projects/:id/jobs/thumbnail` inicia job de geração de thumbnails |
| REQ-F05-02 | Worker extrai 12 frames do `merged.mp4` em intervalos uniformes |
| REQ-F05-03 | Claude recebe os frames (base64) + `project.name` + `seo.title` (se existir) e retorna 3 planos |
| REQ-F05-04 | Worker renderiza cada variante como JPEG 1280×720, qualidade 90 |
| REQ-F05-05 | 3 objetos `Thumbnail` são criados no banco com `plan`, `frame_key`, `output_key` |
| REQ-F05-06 | URLs assinadas das 3 variantes são retornadas ao frontend via WS |
| REQ-F05-07 | `PATCH /api/projects/:id/thumbnails/:thumbId/select` marca 1 variante como `selected = true` |

## Regras de negócio

| ID | Regra |
|----|-------|
| RN-F05-01 | `merged.mp4` (ou `final.mp4`) deve existir antes do job de thumbnail |
| RN-F05-02 | Claude recebe no máximo 6 frames por chamada (para respeitar limite de tokens/contexto) — usa os 6 mais relevantes (início, clímax, final) |
| RN-F05-03 | `ANTHROPIC_API_KEY` nunca exposta ao frontend |
| RN-F05-04 | Se o render falhar em 1 variante, as outras 2 ainda são entregues |
| RN-F05-05 | Re-executar thumbnail gera novos planos e sobrescreve as 3 variantes anteriores |
| RN-F05-06 | Antes da publicação, exatamente 1 variante deve ter `selected = true` |

## Prompt Claude Vision (estrutura)

```
Você é um especialista em thumbnails de YouTube com alto CTR.

Analise os frames abaixo do vídeo intitulado "${project.name}".
${seoTitle ? `Título SEO: ${seoTitle}` : ''}

Retorne exatamente 3 planos de thumbnail (variantes A, B e C) no formato JSON:
{
  "plans": [
    {
      "variant": "A",
      "frame_index": <0-11>,
      "description": "<descrição do layout>",
      "text_overlay": "<texto para sobrepor>",
      "color_palette": ["#hex1", "#hex2"],
      "focus_area": "face|action|landscape|product",
      "font_style": "bold|clean|dramatic"
    },
    ...
  ]
}

Cada variante deve ter estilo distinto (ex.: uma com rosto em destaque, uma com texto grande, uma com paisagem).
```

## UI (Web)

- Seção "Thumbnails" no detalhe do projeto.
- Botão "Gerar Thumbnails" → log stream enquanto extrai frames e chama IA.
- Concluído: grid com 3 variantes lado a lado (A, B, C).
- Cada card: imagem, plano resumido (texto do overlay, estilo), botão "Selecionar".
- Variante selecionada exibe badge "Escolhida".
- Botão "Regenerar" disponível a qualquer momento.

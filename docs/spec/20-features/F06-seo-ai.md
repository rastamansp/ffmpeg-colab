# F06 — SEO com IA

## Objetivo

Gerar título, descrição e tags para o vídeo usando Claude, com base no nome do projeto, contexto fornecido pelo criador e (opcionalmente) transcrição do vídeo.

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F06-01 | `POST /api/projects/:id/jobs/seo` inicia geração de SEO com `{ context?: string }` |
| REQ-F06-02 | Claude retorna `{ title, description, tags[] }` em uma chamada |
| REQ-F06-03 | Resultado salvo em `SeoMetadata` do projeto |
| REQ-F06-04 | `PATCH /api/projects/:id/seo` permite editar título, descrição e tags antes da publicação |
| REQ-F06-05 | `POST /api/projects/:id/seo/approve` define `approved = true` |
| REQ-F06-06 | SEO pode ser regenerado a qualquer momento (cria novo registro, substitui o anterior) |

## Regras de negócio

| ID | Regra |
|----|-------|
| RN-F06-01 | `title` máximo 100 caracteres (limite YouTube) |
| RN-F06-02 | `description` máximo 5000 caracteres |
| RN-F06-03 | `tags` total máximo 500 caracteres somados |
| RN-F06-04 | `approved = true` é pré-condição para publicação (ver F07) |
| RN-F06-05 | Ao editar o SEO, `approved` volta para `false` automaticamente |
| RN-F06-06 | `ANTHROPIC_API_KEY` nunca exposta ao frontend |

## Prompt Claude (estrutura)

```
Você é especialista em SEO para YouTube em português brasileiro.

Vídeo: "${project.name}"
Canal: "${project.channel_name}"
${context ? `Contexto adicional: ${context}` : ''}

Gere metadados otimizados para YouTube no formato JSON:
{
  "title": "<título atraente, max 100 chars>",
  "description": "<descrição completa, max 5000 chars, inclua timestamps se relevante>",
  "tags": ["tag1", "tag2", ...]
}

Regras:
- Título: deve gerar curiosidade e incluir palavras-chave principais.
- Descrição: primeiro parágrafo resume o vídeo (aparece na busca), segundo parágrafo expande, terceiro tem CTAs (inscrição, outros vídeos).
- Tags: 10-20 tags relevantes, da mais ao menos específica.
- Linguagem: português brasileiro, tom do canal "${project.channel_name}".
```

## UI (Web)

- Seção "SEO" no detalhe do projeto.
- Campo "Contexto" (opcional): o criador descreve o vídeo em texto livre para dar contexto ao Claude.
- Botão "Gerar SEO" → spinner enquanto chama a API.
- Resultado: campos editáveis (título, textarea de descrição, chips de tags).
- Contador de caracteres para título e tags (limites YouTube).
- Botão "Aprovar SEO" → habilita publicação.
- Badge de status: `não aprovado` / `aprovado`.

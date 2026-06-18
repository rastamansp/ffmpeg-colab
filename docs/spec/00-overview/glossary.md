# Glossário — Gwan Studio

## Domínio de negócio

| Termo | Definição |
|-------|-----------|
| **Projeto** | Unidade de produção de um vídeo. Agrupa fontes, jobs, thumbnails e metadados. |
| **Source** | Arquivo de vídeo bruto (footage) enviado pelo criador. Pode pertencer a uma câmera e take específicos. |
| **Câmera** | Identificador da câmera que gravou o source (ex.: `cam-a`, `cam-b`, `gopro`). Usado para organizar o merge. |
| **Take** | Número sequencial de gravação de uma cena. Múltiplos takes da mesma câmera = criador escolhe o melhor. |
| **Merge** | Concatenação dos sources selecionados em um único arquivo de vídeo (operação FFmpeg `concat`). |
| **Stream copy** | Modo FFmpeg `-c copy` — os streams são copiados sem re-encode. Muito mais rápido, mas os sources devem ter o mesmo codec. |
| **Export** | Render do vídeo final com as configurações de codec, bitrate e resolução desejadas. |
| **Thumbnail** | Imagem de capa do vídeo no YouTube (1280×720 px, JPEG). O Studio gera 3 variantes (A/B/C) para A/B test. |
| **Plan de thumbnail** | Texto gerado pelo Claude Vision descrevendo o layout de cada variante (posição de texto, frame selecionado, paleta). |
| **SEO metadata** | Conjunto de título, descrição e tags gerado pelo Claude para otimização de busca no YouTube. |
| **Publicação** | Upload do vídeo exportado no YouTube com metadados e thumbnail. |
| **Canal YouTube** | Canal alvo associado ao projeto via OAuth 2.0. |
| **BGM** | Background Music — trilha sonora de fundo com ducking automático durante falas (Fase B). |
| **Shorts** | Formato vertical 9:16 do YouTube, duração ≤ 60 s (Fase B). |
| **Teaser** | Trecho curto (~15–30 s) extraído automaticamente do ponto mais relevante do vídeo (Fase B). |
| **GPX** | Arquivo de telemetria GPS (Garmin, Wahoo) usado para gerar chapter markers no vídeo (Fase B). |

## Técnico

| Termo | Definição |
|-------|-----------|
| **Job** | Unidade de processamento assíncrono (merge, export, thumbnail, seo, publish). Tem `id`, `type`, `status`, `logs`. |
| **Job status** | `pending` → `running` → `done` \| `failed` |
| **Worker** | Processo separado que executa jobs pesados (FFmpeg, upload YouTube). Não bloqueia a API NestJS. |
| **MinIO bucket `studio`** | Bucket S3 no MinIO compartilhado (`s3.gwan.cloud`) onde ficam todos os artefatos de vídeo. |
| **Workspace** | Pasta no bucket `studio` dedicada a um projeto: `studio/<project-id>/` |
| **URL assinada** | URL temporária de acesso a um objeto MinIO, sem exposição de credenciais. |
| **fluent-ffmpeg** | Biblioteca Node.js que encapsula a CLI do FFmpeg em uma API fluent. Usada pelo worker. |
| **OAuth 2.0 PKCE** | Fluxo de autenticação Google sem client_secret exposto — usado no frontend para iniciar o consent do YouTube. |
| **Refresh token** | Token de longa duração do OAuth Google — persistido criptografado no banco, nunca enviado ao browser. |
| **Claude Vision** | Capacidade multimodal do Claude de analisar imagens — usada para selecionar frames e planejar thumbnails. |
| **Clean Architecture** | Padrão de arquitetura do backend GWAN: `domain` → `application` (use cases) → `infrastructure` (adapters) → `presentation` (controllers). |
| **Port** | Interface (TypeScript) que define o contrato de uma integração sem acoplar ao framework. Ex.: `IObjectStoragePort`, `IVideoWorkerPort`. |
| **BullMQ** | Biblioteca de fila Redis para Node.js — usada na Fase B para jobs duráveis com retomada. |

## Padrões de nomenclatura

| Padrão | Exemplo |
|--------|---------|
| Identificador de projeto | `proj_<uuid>` |
| Chave MinIO de source | `studio/<proj_id>/sources/<filename>` |
| Chave MinIO de merged | `studio/<proj_id>/merged.mp4` |
| Chave MinIO de exported | `studio/<proj_id>/final.mp4` |
| Chave MinIO de thumbnail | `studio/<proj_id>/thumbnails/<variant>.png` |
| Status de publicação | `unpublished` \| `publishing` \| `published` \| `failed` |

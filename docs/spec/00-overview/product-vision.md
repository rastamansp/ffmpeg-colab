# Visão de produto — Gwan Studio

## Pitch

**Gwan Studio** é um estúdio web de produção de vídeo para YouTubers. O criador faz upload do footage bruto gravado (multi-câmera, múltiplos takes), escolhe a ordem dos clipes, e o sistema cuida do resto: merge com FFmpeg, geração de 3 variantes de thumbnail por IA, textos SEO (título, descrição, tags) e upload direto no YouTube — tudo acompanhado em tempo real numa interface limpa.

**Desenvolvido por [gwan.cloud](https://gwan.cloud)** — a aplicação exibe link de volta para o site (footer).

**Origem técnica:** evolução do pipeline [`ffmpeg-colab`](https://github.com/acentauric/ffmpeg-colab), que rodava como células sequenciais no Google Colab. O Gwan Studio extrai as mesmas funcionalidades e as entrega como produto web, sem dependência do Colab.

## Problema

O workflow atual de produção de vídeo para YouTube exige:

1. Abrir o Google Colab, conectar runtime, autenticar manualmente.
2. Executar ~10 estágios de células em sequência — qualquer erro na sessão recomeça tudo.
3. Configurar credenciais OAuth a cada sessão nova.
4. Esperar o processamento FFmpeg num ambiente de CPU compartilhada.
5. Copiar/colar metadados SEO de ferramentas externas.
6. Montar thumbnails num script Python separado.

**Resultado:** processo frágil, lento e que só o criador técnico consegue operar.

## Proposta de valor

- **Interface web:** qualquer colaborador faz upload e acompanha o progresso — sem Python, sem Colab.
- **Pipeline robusto:** jobs assíncronos com estado persistido — reiniciar o browser não perde o progresso.
- **Thumbnails por IA:** Claude Vision analisa os frames, planeja o layout e o worker renderiza 3 variantes prontas para A/B test.
- **SEO automático:** Claude gera título, descrição e tags a partir do contexto do vídeo — editável antes do upload.
- **Um clique para o YouTube:** OAuth 2.0 persistido por projeto — autenticação única, upload direto.

## Personas

| Persona | Necessidade | Como Gwan Studio atende |
|---------|-------------|--------------------------|
| **YouTuber solo** | Produzir vídeos sem depender do Colab | Interface web + pipeline gerenciado |
| **Editor / colaborador** | Participar da produção sem acesso ao Colab | Upload e acompanhamento via browser |
| **Canal com volume** | Padronizar o processo entre vídeos | Projetos reutilizáveis + templates de thumbnail |

## Escopo da Fase A (pipeline core)

**Entrega ponta a ponta:**

1. Criar projeto → define canal YouTube alvo e configurações de exportação.
2. Upload de footage → multi-arquivo, organizado por câmera/take.
3. Merge de clipes → ordena takes, FFmpeg concatena em stream copy (sem re-encode).
4. Exportação → render final (opções de bitrate/codec se necessário).
5. Thumbnail → extração automática de frames candidatos → Claude Vision planeja → worker renderiza 3 variantes.
6. SEO → Claude gera título + descrição + tags; criador revisa e salva.
7. Publicação → upload no YouTube com título/descrição/tags/thumbnail escolhida.
8. Preview + download via MinIO (URL assinada).

Estado dos jobs: **PostgreSQL** (dados do projeto) + **MinIO** (artefatos de vídeo).

## Fora de escopo (Fase A)

- YouTube Shorts / recorte 9:16 → **Fase B** (F11).
- Google Photos frame extraction → **Fase B** (F12).
- BGM (trilha sonora com ducking) → **Fase B** (F13).
- Gerador de teaser → **Fase B** (F14).
- Telemetria GPX (chapter markers) → **Fase B** (F15).
- Ingestão direta do Google Drive → **Fase B** (F16).
- Fila durável / retomada de jobs em crash → **Fase B** (BullMQ + Redis).
- Multi-workspace / múltiplos usuários por conta → **Fase futura**.

## Métricas de sucesso (Fase A)

- Projeto completo (`published`) para vídeo de até `MAX_UPLOAD_MB` (configurável, default 2 GB).
- Merge concluído em menos de 2× a duração do vídeo final (stream copy).
- 3 variantes de thumbnail geradas e disponíveis para seleção.
- SEO gerado em < 30 s (chamada Claude).
- Upload YouTube concluído com `video_id` retornado.
- Logs do pipeline visíveis em tempo real (latência WebSocket < 1 s).

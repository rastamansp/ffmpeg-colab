# F08 — Storage MinIO

## Objetivo

Gerenciar artefatos de vídeo no MinIO (S3 compartilhado `s3.gwan.cloud`), incluindo upload de sources, armazenamento de outputs e geração de URLs assinadas.

## Estrutura de chaves no bucket `studio`

```
studio/
└── <project_id>/
    ├── sources/
    │   ├── <uuid>-<filename.mp4>   # footage bruto (Source)
    │   └── ...
    ├── frames/
    │   ├── frame-00.jpg            # frames candidatos para thumbnail
    │   └── ...
    ├── thumbnails/
    │   ├── A.jpg                   # variante A (1280×720)
    │   ├── B.jpg                   # variante B
    │   └── C.jpg                   # variante C
    ├── merged.mp4                  # output do merge job
    └── final.mp4                   # output do export job
```

## Port (interface)

```typescript
interface IObjectStoragePort {
  putObject(key: string, data: Buffer | Readable, contentType: string): Promise<void>;
  getObject(key: string): Promise<Readable>;
  deleteObject(key: string): Promise<void>;
  deletePrefix(prefix: string): Promise<void>;  // limpa workspace do projeto
  getSignedUrl(key: string, expiresInSeconds: number): Promise<string>;
  objectExists(key: string): Promise<boolean>;
}
```

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F08-01 | Upload de sources faz stream direto para MinIO (não salva em disco na API) |
| REQ-F08-02 | Workers baixam e enviam para MinIO via stream (sem acumular em memória) |
| REQ-F08-03 | URL assinada de preview de source válida por 5 minutos |
| REQ-F08-04 | URL assinada de download de `final.mp4` válida por 24 horas |
| REQ-F08-05 | Ao deletar um projeto (soft delete), os objetos MinIO **não** são removidos imediatamente |
| REQ-F08-06 | TTL automático no bucket é configurado manualmente via MinIO Console (Fase B introduz lifecycle policy) |

## Regras de negócio

| ID | Regra |
|----|-------|
| RN-F08-01 | Credenciais MinIO (`MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`) **apenas no backend** |
| RN-F08-02 | O bucket `studio` deve existir antes do primeiro deploy — criado via MinIO Console ou script de init |
| RN-F08-03 | Política do bucket: privado (sem acesso público) — acesso apenas via URLs assinadas |
| RN-F08-04 | `deletePrefix` não é chamado automaticamente — somente por operação explícita do criador (Fase B) |

## Adapter de implementação

```typescript
// infrastructure/adapters/minio-storage.adapter.ts
import { Client as MinioClient } from 'minio';

class MinioStorageAdapter implements IObjectStoragePort {
  // ...implementação com @aws-sdk/client-s3 ou minio npm package
}
```

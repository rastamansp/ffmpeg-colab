# F09 — Jobs API + WebSocket

## Objetivo

API de consulta de jobs e gateway WebSocket para streaming em tempo real de status, logs e previews.

## Requisitos

| ID | Requisito |
|----|-----------|
| REQ-F09-01 | `GET /api/projects/:id/jobs` lista todos os jobs do projeto (paginados, ordenados por `started_at desc`) |
| REQ-F09-02 | `GET /api/projects/:id/jobs/:jobId` retorna job completo com `logs` |
| REQ-F09-03 | WebSocket (`ws://api-studio.gwan.cloud`) — cliente subscreve ao projeto via `subscribe { projectId }` |
| REQ-F09-04 | Servidor emite `job.update` com `{ jobId, type, status, log_line?, output_params? }` |
| REQ-F09-05 | Logs do worker são incrementais — cada linha é emitida assim que chega (tail-like) |
| REQ-F09-06 | Ao reconectar, o cliente recebe o estado atual do projeto + últimas 100 linhas de log do job ativo |

## Eventos WebSocket

### Cliente → Servidor

```typescript
// Subscreve a um projeto
{ event: 'subscribe', data: { projectId: string } }

// Cancela subscrição
{ event: 'unsubscribe', data: { projectId: string } }
```

### Servidor → Cliente

```typescript
// Atualização de job
{
  event: 'job.update',
  data: {
    jobId: string,
    type: JobType,
    status: JobStatus,
    log_line?: string,        // linha incremental de log
    output_params?: object,   // preenchido quando status = 'done'
    error?: string            // preenchido quando status = 'failed'
  }
}

// Estado inicial ao subscrever
{
  event: 'project.snapshot',
  data: {
    projectId: string,
    status: ProjectStatus,
    active_job?: {
      jobId: string,
      type: JobType,
      status: JobStatus,
      recent_logs: string[]   // últimas 100 linhas
    }
  }
}
```

## Arquitetura interna

```
Worker
  │ emite via EventEmitter
  ▼
JobEventBus (IJobEventBus)     ← port no domain
  │ implementado por
  ▼
InMemoryJobEventBus            ← adapter (Fase A)
BullMQJobEventBus              ← adapter (Fase B)
  │ repassa para
  ▼
WS Gateway (NestJS @WebSocketGateway)
  │ emite para salas por projectId
  ▼
Frontend (useProjectWebSocket hook)
```

## Regras de negócio

| ID | Regra |
|----|-------|
| RN-F09-01 | Um único WebSocket gateway serve todos os projetos (rooms por `projectId`) |
| RN-F09-02 | Job em memória é perdido em restart do processo — aceitável na Fase A |
| RN-F09-03 | Logs são truncados a 10.000 linhas por job para evitar crescimento ilimitado |
| RN-F09-04 | O frontend re-conecta automaticamente com backoff exponencial (max 30s) |

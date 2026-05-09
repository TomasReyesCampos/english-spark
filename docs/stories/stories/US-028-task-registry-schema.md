# US-028: Task Registry schema + persistence

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 0-1
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

El Task Registry es el **catálogo central** que describe cada
task AI: qué hace, qué proveedor preferido, fallbacks, prompt
template versionado, schema de input/output, budget.

Sin Registry, las tasks viven hardcoded en código y cambiar
proveedor = refactor. Con Registry, cambiar proveedor = update de
una row.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4 (Task Registry).

## Scope

### In

- Schema Postgres `task_registry`:
  ```sql
  CREATE TABLE task_registry (
    task_id           TEXT PRIMARY KEY,           -- 'score_pronunciation'
    version           TEXT NOT NULL,              -- '1.0.0'
    description       TEXT NOT NULL,
    input_schema      JSONB NOT NULL,             -- Zod schema as JSON
    output_schema     JSONB NOT NULL,
    provider_chain    JSONB NOT NULL,             -- ordered list of providers + models
    prompt_template_path TEXT,                    -- path to .txt file in repo
    budget_usd_per_call NUMERIC(10,6) NOT NULL,
    timeout_ms        INT NOT NULL DEFAULT 30000,
    is_active         BOOLEAN NOT NULL DEFAULT true,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
  );

  CREATE INDEX idx_task_registry_active ON task_registry(task_id) WHERE is_active = true;
  ```
- Loader: cargar registry de Postgres a KV cache al startup del
  Worker (TTL 5 min).
- Helper `getTask(task_id)` que lee de KV cache.
- Migration script para seed inicial con las 10+ tasks MVP del
  catálogo (ver `ai-gateway-strategy.md` §4.2). Cada task con
  `is_active = false` inicialmente; se activa cuando US específica
  termina.
- Endpoint `GET /admin/tasks` (auth interna) para listar registry.
- Validation que cada task registrado tiene Zod schemas válidos
  para input + output.
- Telemetry: `ai_gateway.registry_loaded`,
  `ai_gateway.task_lookup` con cache hit rate.

### Out

- Provider adapters concretos (US-029).
- Prompt templates physical files (cubrir en stories de tasks
  individuales; aquí solo path).
- UI admin para editar registry (post-MVP).

## Acceptance criteria

- **Given** la migration aplicada, **When** se inserta una task con
  `task_id = 'score_pronunciation'`, **Then** queda persistida con
  todos los campos required.
- **Given** registry cargado en KV, **When** Worker llama
  `getTask('score_pronunciation')`, **Then** retorna objeto Task
  desde cache (no hit a Postgres).
- **Given** TTL del cache expira, **When** próxima query, **Then**
  re-fetcha de Postgres y actualiza KV.
- **Given** seed inicial aplicado, **When** se hace
  `SELECT * FROM task_registry`, **Then** existen filas para las
  10+ tasks MVP del catálogo (todas con `is_active = false`).
- **Given** GET `/admin/tasks` con `X-Internal-Auth`, **When**
  responde, **Then** retorna array con todos los tasks + sus
  versions + status.
- **Given** una task con `input_schema` no parseable como Zod,
  **When** validation runs, **Then** queda flagged en logs y NO
  se acepta `is_active = true` para esa task.
- **Given** se intenta `getTask('inexistent')`, **When** lookup,
  **Then** retorna `null` y `task_not_found` count++ en metrics.
- **Given** cache miss + Postgres timeout, **When** loader falla,
  **Then** se usa snapshot anterior del cache (stale-while-error
  pattern) y se loguea SEV-2.

## Developer details

### Owning service

`apps/workers/ai-gateway` + Postgres migration.

### Dependencies

- US-027: Worker skeleton.
- US-002: schema Postgres (extender con esta tabla nueva).

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.1 — Task schema autoritativo.
- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.2 — catálogo de 10+ tasks MVP.
- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.3 — persistencia.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/registry/task-registry.ts
export interface Task {
  task_id: string;
  version: string;
  description: string;
  input_schema: any;
  output_schema: any;
  provider_chain: ProviderConfig[];
  prompt_template_path?: string;
  budget_usd_per_call: number;
  timeout_ms: number;
  is_active: boolean;
}

interface ProviderConfig {
  provider: 'anthropic' | 'openai' | 'google' | 'azure' | 'elevenlabs' | 'dalle3' | 'heygen';
  model: string;                  // 'claude-haiku-4-5', 'gpt-4o-mini'
  weight: number;                 // 100 = primary, 5 = canary, 0 = fallback only
}

export async function loadRegistry(env: Env): Promise<void> {
  const tasks = await db.query<Task>(`
    SELECT * FROM task_registry WHERE is_active = true
  `);
  for (const task of tasks) {
    await env.REGISTRY_KV.put(`task:${task.task_id}`, JSON.stringify(task), {
      expirationTtl: 300, // 5 min
    });
  }
  console.log({ event: 'ai_gateway.registry_loaded', count: tasks.length });
}

export async function getTask(taskId: string, env: Env): Promise<Task | null> {
  const cached = await env.REGISTRY_KV.get(`task:${taskId}`);
  if (cached) {
    return JSON.parse(cached);
  }

  // Cache miss: load + cache
  const result = await db.query<Task>(`
    SELECT * FROM task_registry WHERE task_id = $1 AND is_active = true
  `, [taskId]);

  if (result.length === 0) return null;

  await env.REGISTRY_KV.put(`task:${taskId}`, JSON.stringify(result[0]), {
    expirationTtl: 300,
  });
  return result[0];
}
```

### Seed migration

```sql
-- migrations/0XXX_task_registry_seed.sql
INSERT INTO task_registry (task_id, version, description, input_schema, output_schema, provider_chain, prompt_template_path, budget_usd_per_call, timeout_ms, is_active) VALUES
('score_pronunciation', '1.0.0', 'Score pronunciation against target text using Azure',
  '{...}'::jsonb, '{...}'::jsonb,
  '[{"provider": "azure", "model": "pronunciation_assessment_v1", "weight": 100}]'::jsonb,
  null, 0.005, 15000, false),
('transcribe_user_audio', '1.0.0', 'Transcribe user audio with Whisper',
  '{...}'::jsonb, '{...}'::jsonb,
  '[{"provider": "openai", "model": "whisper-1", "weight": 100}, {"provider": "google", "model": "speech-to-text-v2", "weight": 0}]'::jsonb,
  null, 0.006, 30000, false),
-- ... resto de las 10+ tasks
;
```

(Schemas reales del catálogo en
`ai-gateway-strategy.md` §4.2.)

### Integration points

- Postgres (Supabase).
- Cloudflare KV (cache).
- US-029 (consumer del registry).
- US-043 (admin endpoint reuse).

### Notas técnicas

- TTL del cache es 5 min — balance entre freshness y latencia.
- Stale-while-error: si Postgres falla, usar último valor cacheado
  hasta que se recupere.
- Provider chain con weights: ver
  `ai-gateway-strategy.md` §8 (rollout progresivo) para A/B
  routing.

## Definition of Done

- [ ] Migration `task_registry` aplicada.
- [ ] Seed inicial con 10+ tasks (todas inactivas).
- [ ] Loader + getTask functions implementadas.
- [ ] Cache hit rate > 95% (verificable en metrics).
- [ ] Endpoint admin lista tasks.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec
  `ai-gateway-strategy.md` §4.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-029 a US-043.*

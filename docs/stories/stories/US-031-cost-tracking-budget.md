# US-031: Cost tracking + budget enforcement

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 0-1
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Los LLMs cuestan dinero por uso. Sin cost tracking + enforcement,
una task con bug puede gastar miles de dólares en horas. Esta
story implementa:

1. Persistir cost de cada invocation (per task, per user, per
   provider).
2. Aggregate budgets diarios:
   - Per task (cap del task).
   - Per user (anti-abuse).
   - Global (kill switch).
3. Enforcement: si budget excedido, rechazar invocation con error
   claro.
4. Alerts cuando budget se acerca al cap.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§10 (cost tracking + observability).

## Scope

### In

- Tabla `ai_invocations`:
  ```sql
  CREATE TABLE ai_invocations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id      TEXT NOT NULL,
    task_id         TEXT NOT NULL,
    user_id         UUID,                       -- nullable para internal calls
    provider_used   TEXT NOT NULL,
    model_used      TEXT NOT NULL,
    cost_usd        NUMERIC(12,6) NOT NULL,
    latency_ms      INT NOT NULL,
    status          TEXT NOT NULL,              -- 'success' | 'error'
    error_code      TEXT,
    invoked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
  );

  CREATE INDEX idx_ai_invocations_user_date
    ON ai_invocations(user_id, invoked_at DESC) WHERE user_id IS NOT NULL;
  CREATE INDEX idx_ai_invocations_task_date
    ON ai_invocations(task_id, invoked_at DESC);
  ```
- Cada invocation exitosa persiste row con `cost_usd` calculado por
  el adapter.
- Budgets configurables vía env vars o tabla `budgets`:
  ```sql
  CREATE TABLE budgets (
    scope_type      TEXT NOT NULL,    -- 'task' | 'user' | 'global'
    scope_id        TEXT,             -- task_id, user_id, NULL for global
    daily_limit_usd NUMERIC(10,2) NOT NULL,
    soft_alert_pct  INT NOT NULL DEFAULT 80,
    hard_block_pct  INT NOT NULL DEFAULT 100,
    PRIMARY KEY (scope_type, scope_id)
  );
  ```
- Default budgets seeded:
  - Global: $50/day.
  - Per user (default): $1/day.
  - Per task: variable según task (config en seed).
- Pre-invocation check: query suma de cost_usd de last 24h vs
  budget. Si excede `hard_block_pct`: rechazar con
  `budget_exceeded`.
- Alert cuando se cruza `soft_alert_pct`: emit event
  `ai_gateway.budget_warning` (consumido por monitoring).
- Telemetry: cada invocation emite
  `ai_gateway.invocation_completed` con metrics.

### Out

- UI dashboard de costos (post-MVP, US-041 cubre dashboard dev).
- Per-user budget tier (premium users con limit más alto) —
  post-MVP.
- Auto-disable de tasks con high cost — post-MVP.

## Acceptance criteria

- **Given** una invocation exitosa de Anthropic Haiku con tokens
  1000 input + 500 output, **When** completa, **Then** se persiste
  row en `ai_invocations` con `cost_usd ≈ 0.0028`.
- **Given** una invocation que falla, **When** error, **Then** se
  persiste row con `status = 'error'` y `error_code`, `cost_usd = 0`
  (no charge by failed attempt unless provider does).
- **Given** un user gastó $0.95 hoy y task budget per-user es $1,
  **When** intenta invocar nueva task, **Then** se permite (95% <
  100%).
- **Given** mismo user intenta otra invocation que costaría $0.10,
  **When** pre-check, **Then** rechaza con
  `{ error: 'budget_exceeded', scope: 'user', current: 0.95, limit: 1.0 }`.
- **Given** total global gastado hoy llega a $40 (de $50 budget),
  **When** se cruza 80%, **Then** `ai_gateway.budget_warning` se
  emite con `scope: 'global', usage_pct: 80`.
- **Given** una task con per-task budget $5/day, **When** se gasta
  $5.01 acumulado en esa task, **Then** próxima invocation rechaza
  con scope=`task`.
- **Given** internal call sin user_id, **When** se persiste, **Then**
  `user_id IS NULL` y solo cuenta para budget global + per-task.
- **Given** day rollover (medianoche UTC), **When** próxima query,
  **Then** budgets están reset (query usa `> date_trunc('day', now())`).

## Developer details

### Owning service

`apps/workers/ai-gateway/src/cost-tracking.ts` + Postgres.

### Dependencies

- US-027, US-028, US-029, US-030.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §10 — métricas + observabilidad + alertas.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/cost-tracking.ts
export async function preInvocationBudgetCheck(
  taskId: string,
  userId: string | null,
  estimatedCostUsd: number,
  env: Env
): Promise<{ allowed: boolean; reason?: string }> {
  // Check global budget
  const globalSpent = await getDailySpent('global', null, env);
  const globalBudget = await getBudget('global', null, env);
  if (globalSpent + estimatedCostUsd > globalBudget.daily_limit_usd) {
    return { allowed: false, reason: 'global_budget_exceeded' };
  }

  // Check per-task budget
  const taskSpent = await getDailySpent('task', taskId, env);
  const taskBudget = await getBudget('task', taskId, env);
  if (taskBudget && taskSpent + estimatedCostUsd > taskBudget.daily_limit_usd) {
    return { allowed: false, reason: 'task_budget_exceeded' };
  }

  // Check per-user budget (if applicable)
  if (userId) {
    const userSpent = await getDailySpent('user', userId, env);
    const userBudget = await getBudget('user', userId, env);
    if (userSpent + estimatedCostUsd > userBudget.daily_limit_usd) {
      return { allowed: false, reason: 'user_budget_exceeded' };
    }
  }

  // Soft-alert: emit event if crossing 80% any scope
  if (globalSpent / globalBudget.daily_limit_usd >= 0.8) {
    track('ai_gateway.budget_warning', {
      scope: 'global', usage_pct: Math.floor((globalSpent / globalBudget.daily_limit_usd) * 100),
    });
  }

  return { allowed: true };
}

export async function persistInvocation(record: InvocationRecord, env: Env) {
  await env.DB.execute(`
    INSERT INTO ai_invocations
      (request_id, task_id, user_id, provider_used, model_used, cost_usd, latency_ms, status, error_code)
    VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
  `, [
    record.request_id, record.task_id, record.user_id,
    record.provider_used, record.model_used, record.cost_usd,
    record.latency_ms, record.status, record.error_code,
  ]);

  track('ai_gateway.invocation_completed', record);
}

async function getDailySpent(
  scopeType: 'global' | 'task' | 'user',
  scopeId: string | null,
  env: Env
): Promise<number> {
  let query = `
    SELECT COALESCE(SUM(cost_usd), 0) AS total
    FROM ai_invocations
    WHERE invoked_at > date_trunc('day', now())
      AND status = 'success'
  `;
  const params = [];
  if (scopeType === 'task') {
    query += ' AND task_id = $1';
    params.push(scopeId);
  } else if (scopeType === 'user') {
    query += ' AND user_id = $1';
    params.push(scopeId);
  }
  const result = await env.DB.query(query, params);
  return Number(result[0].total);
}
```

### Default budgets seeded

```sql
INSERT INTO budgets (scope_type, scope_id, daily_limit_usd) VALUES
  ('global', NULL, 50.00),
  ('user', NULL, 1.00),                -- default per user; overridable
  ('task', 'generate_initial_roadmap', 5.00),
  ('task', 'score_pronunciation', 10.00),
  ('task', 'transcribe_user_audio', 8.00),
  ('task', 'generate_audio_tts', 5.00),
  ('task', 'generate_image', 5.00);
```

### Integration points

- US-030 (orchestrator llama pre-check antes y persist después).
- Postgres.
- Telemetry / monitoring.

### Notas técnicas

- Pre-check usa `estimatedCostUsd` antes de invocar; post-call
  persiste el `actualCostUsd` del adapter.
- Budget enforcement es **soft** en MVP (rechaza pero no bloquea
  user). En producción puede degradar UX si hits frecuentes.
- Day rollover en UTC: simple, no per-user TZ.

## Definition of Done

- [ ] Tablas `ai_invocations` + `budgets` migrations aplicadas.
- [ ] Default budgets seeded.
- [ ] `preInvocationBudgetCheck` implementado.
- [ ] `persistInvocation` implementado.
- [ ] Soft-alert events emitidos a 80%.
- [ ] Hard-block en 100%.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec
  `ai-gateway-strategy.md` §10.
- [ ] PR aprobada y mergeada.

---

*Depende de US-027/028/029/030. Bloqueante para tasks de Slice 2-3.*

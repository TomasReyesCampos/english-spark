# US-043: Admin endpoint para inspeccionar tasks

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 2
**Story points:** 2
**Persona:** Admin (dev + ops debugging)
**Owner:** —

---

## Contexto

Devs y ops necesitan inspeccionar el estado del Gateway sin tocar
DB directamente:
- Listar tasks registradas + sus configs.
- Ver invocations recientes de una task.
- Probar una task con input arbitrario (sandbox).
- Health summary.

Esta story consolida los endpoints `/admin/*` ya esbozados en
stories previas (US-028, US-041, US-042) y agrega:
- `GET /admin/tasks/:task_id/recent` (últimas invocations).
- `POST /admin/tasks/:task_id/test` (sandbox invocation).

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4.3 + §10.

## Scope

### In

- Endpoints (todos behind `X-Internal-Auth`):
  - `GET /admin/tasks` — lista todas las tasks con metadata
    resumida (task_id, version, is_active, daily_cost_today).
  - `GET /admin/tasks/:task_id` — detalle completo de una task
    (config + provider chain + recent metrics).
  - `GET /admin/tasks/:task_id/recent?limit=50` — últimas N
    invocations (request_id, latency, cost, status, error_code).
  - `POST /admin/tasks/:task_id/test` — invocación sandbox que
    NO persiste cost ni cuenta para budget.
  - `GET /admin/prompts/:task_id` — versions disponibles (de
    US-042).
  - `GET /admin/prompts/:task_id/:version` — content (de US-042).
  - `GET /admin/health` — snapshot (de US-041).
- Sandbox `/test` endpoint:
  - Invoca task igual que producción.
  - Marca log con `is_sandbox = true`.
  - NO cuenta hacia budget enforcement.
  - Retry / fallback se ejecutan normal.
- Auth dual: `X-Internal-Auth` para endpoints destructivos /
  mutating; lectura puede aceptar también token de admin user
  (post-MVP).
- CORS configurado para permitir admin UI futura (post-MVP).
- Rate limiting separado del production: 10 req/min para sandbox.

### Out

- UI admin gráfica (post-MVP — MVP es solo curl-friendly).
- Endpoints de modificación de registry (UPDATE / DELETE) —
  post-MVP, MVP usa migrations directas.

## Acceptance criteria

- **Given** GET `/admin/tasks` con auth válido, **When** responde,
  **Then** retorna array con summary de cada task del registry.
- **Given** GET `/admin/tasks/score_pronunciation`, **When**
  responde, **Then** retorna detalle completo con
  provider_chain, current_version, total_invocations_24h,
  avg_latency_24h, total_cost_24h.
- **Given** GET `/admin/tasks/score_pronunciation/recent?limit=10`,
  **When** responde, **Then** retorna array de hasta 10
  invocations recientes ordenadas DESC.
- **Given** POST `/admin/tasks/score_pronunciation/test` con
  body de input válido, **When** se ejecuta, **Then** retorna
  output normal pero `ai_invocations.is_sandbox = true`.
- **Given** sandbox call cuesta $0.005, **When** se ejecuta y se
  persiste, **Then** budget tracking NO incluye este cost (filtros
  `is_sandbox = false`).
- **Given** sandbox limit 10/min excedido, **When** 11vo request,
  **Then** retorna 429.
- **Given** request sin `X-Internal-Auth`, **When** llega a
  cualquier `/admin/*`, **Then** retorna 401.
- **Given** task_id inexistente en GET `/admin/tasks/foo`,
  **When** responde, **Then** retorna 404
  `{ error: 'task_not_found' }`.

## Developer details

### Owning service

`apps/workers/ai-gateway/src/handlers/admin/`.

### Dependencies

- US-027/028/030/031.
- US-041 (logging) + US-042 (prompts) — comparten endpoints.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.3 + §10.

### Schema extension

```sql
ALTER TABLE ai_invocations
  ADD COLUMN is_sandbox BOOLEAN NOT NULL DEFAULT false;

-- Asegurar budget queries excluyen sandbox
-- (Update queries de US-031 a incluir AND is_sandbox = false)
```

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/handlers/admin/tasks.ts
export async function handleAdminListTasks(request: Request, env: Env) {
  if (!isInternalAuth(request, env)) return unauthorized();

  const tasks = await env.DB.query(`
    SELECT
      tr.task_id, tr.version, tr.is_active,
      COALESCE(stats.invocations_today, 0) AS invocations_today,
      COALESCE(stats.cost_today, 0) AS cost_today
    FROM task_registry tr
    LEFT JOIN (
      SELECT task_id, COUNT(*) AS invocations_today, SUM(cost_usd) AS cost_today
      FROM ai_invocations
      WHERE invoked_at > date_trunc('day', now()) AND is_sandbox = false
      GROUP BY task_id
    ) stats ON stats.task_id = tr.task_id
    ORDER BY tr.task_id
  `);

  return jsonResponse({ tasks });
}

export async function handleAdminTaskDetail(
  request: Request, env: Env, taskId: string
) {
  if (!isInternalAuth(request, env)) return unauthorized();

  const task = await getTask(taskId, env);
  if (!task) return jsonResponse({ error: 'task_not_found' }, 404);

  const stats = await env.DB.query(`
    SELECT
      COUNT(*) AS total_invocations_24h,
      AVG(latency_ms) AS avg_latency_24h,
      SUM(cost_usd) AS total_cost_24h,
      AVG(CASE WHEN status = 'error' THEN 1 ELSE 0 END) AS error_rate_24h
    FROM ai_invocations
    WHERE task_id = $1 AND invoked_at > now() - interval '24 hours'
      AND is_sandbox = false
  `, [taskId]);

  return jsonResponse({ task, stats: stats[0] });
}

export async function handleAdminTaskRecent(
  request: Request, env: Env, taskId: string
) {
  if (!isInternalAuth(request, env)) return unauthorized();

  const url = new URL(request.url);
  const limit = Math.min(parseInt(url.searchParams.get('limit') ?? '50'), 200);

  const recent = await env.DB.query(`
    SELECT request_id, provider_used, latency_ms, cost_usd,
           status, error_code, invoked_at, prompt_version
    FROM ai_invocations
    WHERE task_id = $1 AND is_sandbox = false
    ORDER BY invoked_at DESC
    LIMIT $2
  `, [taskId, limit]);

  return jsonResponse({ recent });
}

export async function handleAdminTaskTest(
  request: Request, env: Env, taskId: string
) {
  if (!isInternalAuth(request, env)) return unauthorized();

  const body = await request.json();

  // Sandbox rate limit
  const allowed = await checkSandboxRateLimit(env);
  if (!allowed) return jsonResponse({ error: 'rate_limited' }, 429);

  // Invoke con flag
  try {
    const result = await invokeWithFallback(taskId, body.input, {
      ...body.context, request_id: crypto.randomUUID(), env, is_sandbox: true,
    });
    return jsonResponse({ output: result.output, metadata: result.metadata });
  } catch (error: any) {
    return jsonResponse({ error: error.code, message: error.message }, 500);
  }
}
```

### Routing

```typescript
// apps/workers/ai-gateway/src/index.ts (extension)
if (url.pathname === '/admin/tasks' && method === 'GET') {
  return handleAdminListTasks(request, env);
}
const taskMatch = url.pathname.match(/^\/admin\/tasks\/([^\/]+)(\/recent|\/test)?$/);
if (taskMatch) {
  const [, taskId, suffix] = taskMatch;
  if (suffix === '/recent' && method === 'GET') return handleAdminTaskRecent(request, env, taskId);
  if (suffix === '/test' && method === 'POST') return handleAdminTaskTest(request, env, taskId);
  if (!suffix && method === 'GET') return handleAdminTaskDetail(request, env, taskId);
}
```

### Integration points

- Postgres (queries).
- US-027 routing.
- US-028 registry.
- US-030 orchestrator (sandbox invocations).
- US-031 cost tracking (excluye sandbox).

### Notas técnicas

- Sandbox calls SÍ persisten en `ai_invocations` con flag `is_sandbox`
  para audit trail, pero excluidos de budget.
- Doc `docs/ops/ai-gateway-admin-api.md` (a crear) con curl
  examples de cada endpoint.
- CORS NO se habilita en MVP (admin via curl/internal tools).

## Definition of Done

- [ ] 4 endpoints admin (list, detail, recent, test) implementados.
- [ ] Sandbox flag funcional + excluído de budget.
- [ ] Sandbox rate limit 10/min.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit de cada handler.
- [ ] Test integration end-to-end con curl.
- [ ] Doc `docs/ops/ai-gateway-admin-api.md` con examples.
- [ ] Validation contra spec `ai-gateway-strategy.md` §4.3 + §10.
- [ ] PR aprobada y mergeada.

---

*Cierre EPIC-05.*

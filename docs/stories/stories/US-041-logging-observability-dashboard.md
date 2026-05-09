# US-041: Logging + observability dashboard

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 2
**Story points:** 3
**Persona:** Admin (ops + dev visibility)
**Owner:** —

---

## Contexto

Sin observability, debugging del Gateway es ciego. Esta story
implementa:
- Logging estructurado de cada invocation (request_id, task,
  provider, latency, cost, status).
- Dashboard interno (Grafana o equivalente) con métricas clave.
- Alertas críticas.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§10 (cost tracking + observability).

## Scope

### In

- Structured logging en formato JSON con campos:
  - `request_id`, `timestamp`, `task_id`, `user_id`,
    `provider_used`, `model_used`, `latency_ms`, `cost_usd`,
    `status` (success/error), `error_code`.
- Log destination: Cloudflare Logs + opcional sink externo
  (Datadog/Loki) post-MVP.
- Dashboard con paneles:
  1. **Health overview:** error rate por task last 1h, last 24h.
  2. **Latency:** P50, P95, P99 por task.
  3. **Cost:** spending por día (últimos 7 días), por task, por
     provider.
  4. **Fallback usage:** % de calls que usaron fallback (debe ser
     <5% si primary funciona bien).
  5. **Budget gauges:** % spent del daily limit (global, top tasks,
     top users).
  6. **Provider health:** isHealthy() check status para los 6
     providers.
- Alertas Slack / PagerDuty:
  - SEV-1: error rate > 20% por 5 min consecutivos.
  - SEV-2: budget global > 90% antes de las 18hs UTC.
  - SEV-2: latency P95 > 2x baseline durante 10 min.
  - SEV-3: provider isHealthy=false durante 5 min.
- `GET /admin/health` endpoint que retorna snapshot del estado
  actual.

### Out

- Per-user analytics (post-MVP).
- A/B testing analytics (post-MVP).
- Self-healing (auto-disable provider con high error rate) —
  post-MVP.

## Acceptance criteria

- **Given** una invocation, **When** completa, **Then** un log
  JSON estructurado con todos los campos requeridos aparece en
  Cloudflare Logs.
- **Given** dashboard configurado, **When** dev abre, **Then** ve
  los 6 paneles con datos last 24h.
- **Given** error rate de task `score_pronunciation` sube a 25%,
  **When** monitor evalúa por 5 min consecutivos, **Then** alerta
  SEV-1 dispara a Slack.
- **Given** spending global supera 90% del budget ($45 de $50),
  **When** monitor checa a 17:00 UTC, **Then** alerta SEV-2.
- **Given** ElevenLabs API endpoint cae, **When** isHealthy retorna
  false durante 5 min, **Then** alerta SEV-3.
- **Given** GET `/admin/health`, **When** se llama con
  X-Internal-Auth, **Then** retorna JSON con
  `{ providers: { anthropic: 'ok', openai: 'ok', ... },
     last_24h: { invocations, error_rate, total_cost } }`.
- **Given** logs de las últimas 24h, **When** dev consulta,
  **Then** puede filtrar por `task_id`, `user_id`, `status`,
  `provider`.
- **Given** invocation falla, **When** error logged, **Then**
  `error_code` es uno del enum estándar (rate_limited,
  invalid_request, etc.) — no genérico.

## Developer details

### Owning service

`apps/workers/ai-gateway/src/observability/` + Grafana / monitoring
externo.

### Dependencies

- US-027/028/029/030/031 (foundation completa).
- Cuenta Grafana Cloud o equivalente (Cloudflare Dashboards básicos
  como fallback MVP).
- Slack webhook configurado.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §10.1, §10.2, §10.3.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/observability/logger.ts
export interface InvocationLog {
  timestamp: string;
  request_id: string;
  task_id: string;
  user_id: string | null;
  provider_used: string;
  model_used: string;
  latency_ms: number;
  cost_usd: number;
  status: 'success' | 'error';
  error_code?: string;
  task_version: string;
}

export function logInvocation(log: InvocationLog) {
  console.log(JSON.stringify({ ...log, level: 'info', source: 'ai_gateway' }));
}

export function logError(error: Error, context: any) {
  console.error(JSON.stringify({
    timestamp: new Date().toISOString(),
    level: 'error',
    source: 'ai_gateway',
    error_message: error.message,
    error_stack: error.stack,
    ...context,
  }));
}

// apps/workers/ai-gateway/src/handlers/admin-health.ts
export async function handleAdminHealth(request: AuthedRequest, env: Env) {
  if (request.headers.get('X-Internal-Auth') !== env.INTERNAL_AUTH_TOKEN) {
    return jsonResponse({ error: 'unauthorized' }, 401);
  }

  const providers = ['anthropic', 'openai', 'google', 'azure', 'elevenlabs', 'dalle3'];
  const healthChecks = await Promise.all(
    providers.map(async (p) => {
      try {
        const adapter = getAdapter(p, env);
        const ok = await Promise.race([
          adapter.isHealthy(),
          new Promise<boolean>(r => setTimeout(() => r(false), 3000)),
        ]);
        return [p, ok ? 'ok' : 'degraded'];
      } catch {
        return [p, 'down'];
      }
    })
  );

  const last24h = await env.DB.query(`
    SELECT
      COUNT(*) AS invocations,
      AVG(CASE WHEN status = 'error' THEN 1 ELSE 0 END) AS error_rate,
      SUM(cost_usd) AS total_cost
    FROM ai_invocations
    WHERE invoked_at > now() - interval '24 hours'
  `);

  return jsonResponse({
    providers: Object.fromEntries(healthChecks),
    last_24h: last24h[0],
    timestamp: new Date().toISOString(),
  });
}
```

### Dashboard config (Grafana JSON)

`docs/ops/grafana-ai-gateway-dashboard.json` con queries SQL contra
`ai_invocations` table:

```sql
-- Panel 1: Error rate por task last 24h
SELECT task_id, AVG(CASE WHEN status='error' THEN 1 ELSE 0 END) * 100 AS error_pct
FROM ai_invocations
WHERE invoked_at > now() - interval '24 hours'
GROUP BY task_id;

-- Panel 3: Cost por día last 7 días
SELECT date_trunc('day', invoked_at) AS day, SUM(cost_usd) AS spent
FROM ai_invocations
WHERE invoked_at > now() - interval '7 days'
GROUP BY day ORDER BY day;
```

### Alertas (configurable per env)

```yaml
# docs/ops/ai-gateway-alerts.yaml
alerts:
  - name: error_rate_critical
    severity: SEV-1
    query: error rate > 0.20 for 5 min
    notify: slack#alerts-prod, pagerduty
  - name: budget_warning
    severity: SEV-2
    query: total_cost_today > 0.90 * daily_global_limit
    notify: slack#alerts-cost
  - name: provider_down
    severity: SEV-3
    query: isHealthy = false for any provider for 5 min
    notify: slack#alerts-providers
```

### Integration points

- Postgres (queries para dashboard).
- Cloudflare Logs (stream de logs).
- Slack webhooks.
- PagerDuty (post-MVP).

### Notas técnicas

- Grafana Cloud free tier suficiente para MVP (10k métricas/mes).
- Cloudflare Logs Push gratis a Logpush destinations (R2, S3).
- Alertas vía cron Worker ejecuta queries cada 5 min y dispara
  webhooks.

## Definition of Done

- [ ] Structured logging implementado.
- [ ] 6 paneles del dashboard funcionales.
- [ ] 3 alertas operacionales (SEV-1, SEV-2, SEV-3).
- [ ] Endpoint `/admin/health` funcional.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit del logger.
- [ ] Test manual: trigger error scenario, verificar alerta llega.
- [ ] Doc `docs/ops/ai-gateway-runbook.md` (a crear).
- [ ] Validation contra spec `ai-gateway-strategy.md` §10.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para go-live de production (sin observability no
deployar).*

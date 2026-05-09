# US-030: Multi-provider fallback chain

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 0-1
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Cada task del Registry define un `provider_chain` (lista ordenada
con weights). La lógica de fallback debe:

1. Elegir provider primary según weights (ej: Anthropic 95%,
   OpenAI 5% canary).
2. Si primary falla con error transient: probar next en chain.
3. Si todos fallan: throw error final con detalle del intento.
4. Logging de cada attempt para debugging.

Esta story implementa el **orchestrator** que envuelve los
adapters del US-029.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§8 (rollout progresivo) + §3 (arquitectura).

## Scope

### In

- Función `invokeWithFallback(taskId, input, env)`:
  - Lee `provider_chain` del task.
  - Selecciona primary según weights (deterministic per request_id
    para reproducibility).
  - Invoca adapter primary.
  - Si error es transient (rate_limited, provider_unavailable,
    unknown_provider_error): falla, intenta next provider en chain.
  - Si error es permanent (invalid_request, validation_failed):
    fail-fast, NO try fallbacks.
  - Trackea qué provider terminó respondiendo para metrics.
- Weight selection: hash del request_id mod 100 → si dentro del
  weight del primary, usa primary; sino próximo según weights.
  - Esto permite A/B determinístico (mismo request_id → mismo
    routing).
- Total timeout enforced por task (cada attempt cuenta hacia el
  total).
- Logging de cada attempt con `attempt_index`, `provider`,
  `latency_ms`, `status`.
- Telemetry: `ai_gateway.fallback_used` cuando primary falla y
  secondary se usa exitosamente.

### Out

- Health-based routing (skip provider si reciente
  `isHealthy=false`) — post-MVP.
- Circuit breaker (auto-disable provider con >X% errors) —
  post-MVP.
- Provider warmup / pre-fetching — post-MVP.

## Acceptance criteria

- **Given** task con
  `provider_chain: [{anthropic, weight: 100}, {openai, weight: 0}]`,
  **When** se invoca, **Then** Anthropic se llama primero el 100%
  de las veces.
- **Given** task con `[{anthropic, 95}, {openai, 5}]`, **When** se
  invocan 1000 requests, **Then** ~950 van a Anthropic, ~50 a
  OpenAI (variance ±20).
- **Given** Anthropic responde 429 (rate_limited), **When** fallback
  ejecuta, **Then** se llama OpenAI como secondary, retorna response
  exitosa.
- **Given** Anthropic responde 400 (invalid_request), **When**
  procesa, **Then** NO se intenta fallback, throws error inmediato.
- **Given** primary y secondary ambos fallan transient, **When**
  ejecuta, **Then** intenta tercer provider en chain. Si no hay:
  throws con `attempts: 2, last_error: ...`.
- **Given** request_id estable, **When** mismo request invocado 2x,
  **Then** mismo provider primario es elegido (determinismo).
- **Given** task tiene `timeout_ms: 10000`, **When** primary toma 8s
  + secondary toma 5s, **Then** total > timeout_ms y throws con
  `timeout_exceeded`.
- **Given** `ai_gateway.fallback_used` event, **When** se emite,
  **Then** payload incluye `task_id`, `primary_provider`,
  `fallback_provider_used`, `primary_error`.

## Developer details

### Owning service

`apps/workers/ai-gateway/src/orchestrator.ts`.

### Dependencies

- US-027, US-028, US-029.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §3 — fallback chain.
- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §8 — rollout progresivo con weights.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/orchestrator.ts
const TRANSIENT_ERRORS = new Set([
  'rate_limited', 'provider_unavailable', 'unknown_provider_error',
]);
const PERMANENT_ERRORS = new Set([
  'invalid_request', 'unsupported_model', 'auth_failed',
]);

export async function invokeWithFallback(
  taskId: string,
  input: any,
  context: { request_id: string; env: Env }
): Promise<{ output: any; metadata: InvokeMetadata }> {
  const task = await getTask(taskId, context.env);
  if (!task) throw new Error('task_not_found');

  // Validate input against task.input_schema
  const validated = validateInput(input, task.input_schema);

  // Select provider order based on weights + request_id hash
  const orderedProviders = selectProviderOrder(task.provider_chain, context.request_id);

  const taskStartTime = Date.now();
  const attempts: Array<{ provider: string; latency: number; status: string; error?: string }> = [];
  let lastError: ProviderError | null = null;

  for (let i = 0; i < orderedProviders.length; i++) {
    const config = orderedProviders[i];

    if (Date.now() - taskStartTime > task.timeout_ms) {
      throw new Error('timeout_exceeded');
    }

    const adapter = getAdapter(config.provider, context.env);
    const attemptStart = Date.now();

    try {
      const output = await invokeAdapterByType(adapter, task, validated, config);
      attempts.push({
        provider: config.provider,
        latency: Date.now() - attemptStart,
        status: 'ok',
      });

      // Telemetry: fallback_used if not first
      if (i > 0) {
        track('ai_gateway.fallback_used', {
          task_id: taskId,
          primary_provider: orderedProviders[0].provider,
          fallback_provider_used: config.provider,
          primary_error: lastError?.code,
        });
      }

      return {
        output,
        metadata: {
          provider_used: config.provider,
          model: config.model,
          attempts: attempts.length,
          total_latency_ms: Date.now() - taskStartTime,
          task_version: task.version,
        },
      };
    } catch (error: any) {
      attempts.push({
        provider: config.provider,
        latency: Date.now() - attemptStart,
        status: 'error',
        error: error.code,
      });
      lastError = error;

      if (PERMANENT_ERRORS.has(error.code)) {
        throw error; // No fallback on permanent errors
      }

      if (!TRANSIENT_ERRORS.has(error.code)) {
        throw error; // Unknown error: fail-fast
      }
      // Else: try next provider
    }
  }

  throw new Error(JSON.stringify({
    error: 'all_providers_failed',
    attempts,
    last_error: lastError?.message,
  }));
}

function selectProviderOrder(chain: ProviderConfig[], requestId: string): ProviderConfig[] {
  // Hash request_id to deterministic 0-99
  const hash = simpleHash(requestId) % 100;
  let cumulative = 0;

  // Find primary based on weights
  const totalWeight = chain.reduce((sum, p) => sum + p.weight, 0);
  if (totalWeight === 0) return chain; // all weight 0 → fallback only

  let primaryIdx = 0;
  for (let i = 0; i < chain.length; i++) {
    cumulative += (chain[i].weight / totalWeight) * 100;
    if (hash < cumulative) {
      primaryIdx = i;
      break;
    }
  }

  // Reorder: primary first, then rest in original order (excluding primary)
  return [
    chain[primaryIdx],
    ...chain.filter((_, i) => i !== primaryIdx),
  ];
}
```

### Integration points

- US-028 (registry).
- US-029 (adapters).
- US-031 (cost tracking — recibirá metadata.cost_usd).

### Notas técnicas

- Hash determinístico permite reproducir A/B routing post-hoc para
  debugging.
- Permanent errors (validation, auth) no deben fallback — fallar
  rápido evita gastar budget en intentos fútiles.
- Total timeout inclusive: si primary toma 80% del budget, secondary
  tiene solo 20%.

## Definition of Done

- [ ] `invokeWithFallback` implementado.
- [ ] Weight-based selection determinístico testeado.
- [ ] Fallback con transient errors funcional.
- [ ] Fail-fast con permanent errors funcional.
- [ ] Timeout enforcement.
- [ ] Telemetry events emitidos.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit con mocked adapters (1000 requests para verificar
  weight distribution).
- [ ] Tests integration con simulated provider failures.
- [ ] Validation contra spec `ai-gateway-strategy.md` §3 + §8.
- [ ] PR aprobada y mergeada.

---

*Depende de US-027/028/029. Bloqueante para US-031 + tasks
con multi-provider chain.*

# US-027: AI Gateway Worker skeleton + routing

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 0-1
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Establecer el **Cloudflare Worker** que actúa como AI Gateway:
- Single endpoint público interno: `POST /ai/invoke`.
- Routing por `task_id` al handler correspondiente.
- Auth: solo accesible desde otros Workers de la org (no público).
- Rate limiting básico por user (anti-abuse).
- Skeleton sin lógica de tasks específicas (eso viene en stories
  posteriores).

Esta story es la **base** sobre la cual se montan todas las
stories US-028 a US-043.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§3 (arquitectura).

## Scope

### In

- Worker `apps/workers/ai-gateway` con structure base:
  ```
  apps/workers/ai-gateway/
  ├── src/
  │   ├── index.ts              # entry point + router
  │   ├── handlers/             # uno por task (placeholders)
  │   ├── adapters/             # provider adapters (vacío inicial)
  │   ├── registry/             # task registry (US-028)
  │   ├── shared/
  │   │   ├── auth.ts           # internal auth check
  │   │   ├── rate-limit.ts     # Durable Object básico
  │   │   └── error-handler.ts
  │   └── types/
  └── wrangler.toml
  ```
- Endpoint `POST /ai/invoke` con shape:
  ```typescript
  interface InvokeRequest {
    task_id: string;          // 'score_pronunciation', etc.
    input: Record<string, any>;
    context?: {
      user_id?: string;
      request_id?: string;
    };
  }

  interface InvokeResponse {
    output: any;
    metadata: {
      provider_used: string;
      latency_ms: number;
      cost_usd: number;
      task_version: string;
    };
  }
  ```
- Auth: header `X-Internal-Auth` con shared secret (verificado
  contra Cloudflare secret `INTERNAL_AUTH_TOKEN`).
- Rate limiting básico via Durable Object: 100 invocations / min /
  user_id (anti-abuse, no budget).
- Error responses estandarizados:
  ```typescript
  interface ErrorResponse {
    error: 'task_not_found' | 'invalid_input' | 'auth_failed'
         | 'rate_limited' | 'provider_failed' | 'budget_exceeded';
    message: string;
    request_id: string;
  }
  ```
- Logging básico de cada request (request_id, task_id, latency,
  status).

### Out

- Task Registry persistence (US-028).
- Provider adapters (US-029).
- Fallback logic (US-030).
- Cost tracking (US-031).
- Tasks específicas (US-032+).

## Acceptance criteria

- **Given** otro Worker de la org con `X-Internal-Auth` válido
  hace POST `/ai/invoke` con `task_id: "noop"`, **When** Worker
  responde, **Then** retorna 200 con response stub
  `{ output: { ok: true }, metadata: { ... } }`.
- **Given** un request sin `X-Internal-Auth`, **When** llega,
  **Then** retorna 401 con
  `{ error: 'auth_failed', message: 'Missing internal auth' }`.
- **Given** un request con `task_id` inexistente, **When** routing
  busca handler, **Then** retorna 404 con
  `{ error: 'task_not_found', task_id }`.
- **Given** mismo `user_id` envía 101 requests en 1 minuto, **When**
  el 101vo, **Then** retorna 429 con
  `{ error: 'rate_limited', retry_after_seconds: <int> }`.
- **Given** un request válido, **When** se procesa, **Then** se
  loguea con `request_id`, `task_id`, `user_id`, `latency_ms`.
- **Given** un handler interno throws unexpected error, **When**
  error handler lo captura, **Then** retorna 500 con
  `{ error: 'provider_failed', request_id }` y se loguea SEV-2.
- **Given** Worker está deployed, **When** GET `/health` se llama,
  **Then** retorna `{ status: 'ok', version: '<git-sha>' }`.

## Developer details

### Owning service

`apps/workers/ai-gateway` (nuevo Worker).

### Dependencies

- Cloudflare account + Workers + Durable Objects + KV setup.
- `wrangler` CLI configurado.
- Secret `INTERNAL_AUTH_TOKEN` generado y guardado.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §3 — arquitectura.
- [`decisions/ADR-001-ai-gateway.md`](../../decisions/ADR-001-ai-gateway.md)
  — decisión.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/index.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const requestId = crypto.randomUUID();
    const url = new URL(request.url);

    if (url.pathname === '/health' && request.method === 'GET') {
      return jsonResponse({ status: 'ok', version: env.GIT_SHA });
    }

    if (url.pathname !== '/ai/invoke' || request.method !== 'POST') {
      return jsonResponse({ error: 'not_found' }, 404);
    }

    // Auth
    const auth = request.headers.get('X-Internal-Auth');
    if (auth !== env.INTERNAL_AUTH_TOKEN) {
      return errorResponse('auth_failed', 'Missing internal auth', 401, requestId);
    }

    // Parse
    let body: InvokeRequest;
    try {
      body = await request.json();
    } catch {
      return errorResponse('invalid_input', 'Body must be JSON', 400, requestId);
    }

    // Rate limit
    const userId = body.context?.user_id ?? 'anonymous';
    const rateLimiter = env.RATE_LIMITER.get(env.RATE_LIMITER.idFromName(userId));
    const allowed = await rateLimiter.fetch('/check', { method: 'POST' });
    if (!allowed.ok) {
      return errorResponse('rate_limited', 'Too many requests', 429, requestId);
    }

    // Route to handler
    const handler = HANDLERS[body.task_id];
    if (!handler) {
      return errorResponse('task_not_found', `Task ${body.task_id} not registered`, 404, requestId);
    }

    const startTime = Date.now();
    try {
      const output = await handler(body.input, { ...body.context, requestId, env });
      const latencyMs = Date.now() - startTime;
      console.log({ requestId, taskId: body.task_id, userId, latencyMs, status: 'ok' });
      return jsonResponse({
        output,
        metadata: {
          provider_used: 'placeholder',
          latency_ms: latencyMs,
          cost_usd: 0,
          task_version: '1.0.0',
        },
      });
    } catch (error) {
      console.error({ requestId, taskId: body.task_id, error: error.message });
      return errorResponse('provider_failed', error.message, 500, requestId);
    }
  },
};

// Placeholder noop handler para testing
const HANDLERS: Record<string, TaskHandler> = {
  noop: async () => ({ ok: true }),
};
```

### Rate Limiter Durable Object

```typescript
// apps/workers/ai-gateway/src/shared/rate-limit.ts
export class AIGatewayRateLimiter implements DurableObject {
  state: DurableObjectState;

  constructor(state: DurableObjectState) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const minute = Math.floor(Date.now() / 60_000);
    const key = `count_${minute}`;
    const count = (await this.state.storage.get<number>(key)) ?? 0;

    if (count >= 100) {
      return new Response('rate_limited', { status: 429 });
    }

    await this.state.storage.put(key, count + 1);
    await this.state.storage.setAlarm(Date.now() + 120_000); // cleanup en 2 min
    return new Response('ok', { status: 200 });
  }

  async alarm() {
    // Cleanup buckets viejos
    const now = Math.floor(Date.now() / 60_000);
    const keys = await this.state.storage.list({ prefix: 'count_' });
    for (const [key] of keys) {
      const minute = parseInt(key.replace('count_', ''));
      if (now - minute > 1) await this.state.storage.delete(key);
    }
  }
}
```

### Integration points

- Otros Workers de la org (consumers del Gateway).
- Cloudflare Durable Objects (rate limiting state).
- Cloudflare KV (task registry, US-028).
- Logging: console + Cloudflare Logs.

### Notas técnicas

- `X-Internal-Auth` shared secret se rota cada 90 días (manual MVP,
  automated post-MVP).
- `request_id` permite trazar cross-Worker.
- `provider_used = 'placeholder'` en esta story; real values vienen
  con US-029.

## Definition of Done

- [ ] Worker deployed en dev environment con endpoint `/ai/invoke`.
- [ ] Endpoint `/health` funcional.
- [ ] Auth check funcional.
- [ ] Rate limiting Durable Object operacional.
- [ ] Error handler estandariza responses.
- [ ] Logging básico funcional.
- [ ] Handler `noop` para testing pasa AC.
- [ ] 7 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec `ai-gateway-strategy.md` §3.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-028, US-029 (toda la pila del AI Gateway).*

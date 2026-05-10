# US-045: Sparks balance read endpoint + cache

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 1
**Story points:** 2
**Persona:** Estudiante (consume balance display)
**Owner:** —

---

## Contexto

El balance se consulta frecuentemente:
- Home screen muestra balance.
- Antes de cada exercise (decidir si tiene Sparks).
- Paywall + low balance prompts.
- Background polling.

Sin cache, cada request hits Postgres → load innecesaria. Con
cache (KV con TTL corto), latencia <50ms y BD aliviada.

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §10.

## Scope

### In

- Endpoint `GET /sparks/balance`:
  - Valida JWT.
  - Lee de KV cache (TTL 30s).
  - Si cache miss: query Postgres + populate cache.
  - Retorna `{ current, cycle_allotment, cycle_ends_at,
    lifetime_earned, lifetime_spent, expiring_soon: [...] }`.
- `expiring_soon`: array de packs que expiran en próximos 7 días
  (de transactions con `expires_at`).
- Cache invalidation:
  - Al hacer charge / refund / award: invalidate (delete from KV).
  - Cache se rebuilds en próximo read.
- Headers: `Cache-Control: max-age=30, private`.
- Telemetry: `sparks.balance_read`,
  `sparks.balance_cache_hit`, `sparks.balance_cache_miss`.

### Out

- Real-time push de balance changes (post-MVP, requires websockets).
- History endpoint (post-MVP, retrieve transactions).
- Endpoint admin para inspect balance de otros users (post-MVP).

## Acceptance criteria

- **Given** user logged in con balance 50, **When** GET
  `/sparks/balance`, **Then** retorna 200 con
  `{ current: 50, ... }`.
- **Given** mismo user request 2x consecutivo en <30s, **When**
  segundo, **Then** sirve de cache (latencia <30ms,
  `cache_hit` event).
- **Given** user hace charge → balance baja a 30, **When** próximo
  GET, **Then** retorna `current: 30` (cache invalidated).
- **Given** user con pack que expira en 5 días, **When** GET,
  **Then** `expiring_soon` incluye el pack con su amount + date.
- **Given** user sin profile (raro post-signup), **When** GET,
  **Then** retorna 404 `{ error: 'balance_not_found' }`.
- **Given** request sin JWT, **When** GET, **Then** retorna 401.
- **Given** Postgres down + cache hit, **When** GET, **Then** sirve
  de cache OK (resilience).
- **Given** Postgres down + cache miss, **When** GET, **Then**
  retorna 503 `{ error: 'service_unavailable' }`.

## Developer details

### Owning service

`apps/workers/api/handlers/sparks-balance.ts`.

### Dependencies

- US-044: schema.
- US-007: JWT middleware.
- KV namespace `SPARKS_KV`.

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §10 —
  API contracts.

### Implementación esperada

```typescript
// apps/workers/api/handlers/sparks-balance.ts
export async function handleGetBalance(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);
  if (!userId) return jsonResponse({ error: 'user_not_found' }, 404);

  const cacheKey = `sparks:balance:${userId}`;
  const cached = await env.SPARKS_KV.get(cacheKey);
  if (cached) {
    track('sparks.balance_cache_hit');
    track('sparks.balance_read');
    return new Response(cached, {
      headers: {
        'Content-Type': 'application/json',
        'Cache-Control': 'max-age=30, private',
      },
    });
  }

  track('sparks.balance_cache_miss');

  const balance = await env.DB.query(`
    SELECT current, cycle_allotment, cycle_started_at, cycle_ends_at,
           lifetime_earned, lifetime_spent
    FROM sparks_balances WHERE user_id = $1
  `, [userId]);

  if (balance.length === 0) {
    return jsonResponse({ error: 'balance_not_found' }, 404);
  }

  // Expiring soon: packs that expire in next 7 days, not yet expired
  const expiring = await env.DB.query(`
    SELECT amount, expires_at, related_id
    FROM sparks_transactions
    WHERE user_id = $1
      AND expires_at IS NOT NULL
      AND expired = false
      AND expires_at < now() + interval '7 days'
    ORDER BY expires_at ASC
  `, [userId]);

  const response = {
    ...balance[0],
    expiring_soon: expiring,
  };

  await env.SPARKS_KV.put(cacheKey, JSON.stringify(response), {
    expirationTtl: 30,
  });

  track('sparks.balance_read');
  return new Response(JSON.stringify(response), {
    headers: {
      'Content-Type': 'application/json',
      'Cache-Control': 'max-age=30, private',
    },
  });
}

export async function invalidateBalanceCache(userId: string, env: Env) {
  await env.SPARKS_KV.delete(`sparks:balance:${userId}`);
}
```

### Integration points

- Mobile app (consumer principal en home + paywall).
- US-047/048/050 invalidan cache después de transactions.
- US-031 cost tracking (orthogonal — Sparks es "currency",
  cost_tracking es "USD real").

### Notas técnicas

- TTL 30s balance entre freshness y carga: balance display puede
  ser estimate stale por 30s; transacciones (charge) invalidate
  inmediato.
- `private` en Cache-Control evita CDN cache (data sensitive
  per-user).
- Resiliencia: si Postgres down + cache hit, sirve OK; sin cache:
  503 explícito (mejor que data incorrecta).

## Definition of Done

- [ ] Endpoint implementado con cache.
- [ ] Cache invalidation function exportada.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Performance: P50 < 30ms con cache, < 200ms sin.
- [ ] Validation contra spec `sparks-system.md` §10.
- [ ] PR aprobada y mergeada.

---

*Depende de US-044, US-007. Bloqueante para US-052 (UI).*

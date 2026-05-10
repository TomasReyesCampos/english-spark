# US-047: Sparks charge endpoint (cobro previo idempotente)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 1
**Story points:** 5
**Persona:** Admin (consumido por pedagogical engine + AI Gateway)
**Owner:** —

---

## Contexto

**Regla de oro del sistema económico** ([reglas.md](../../reglas.md)):

> "Cobro de Sparks ANTES de invocar operación. Refund automático si
> falla."

Esta story implementa el cobro previo con:
- Idempotency (mismo `idempotency_key` cobrado solo 1x).
- Race condition safety (advisory locks).
- Negative balance prevention (CHECK constraint + helper function).
- Audit trail completo.

Es la **primera línea de defensa** contra costos LLM
descontrolados.

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §5
(cobro pattern).

## Scope

### In

- Endpoint `POST /sparks/charge`:
  - Valida JWT.
  - Body: `{ amount, reason, related_id, idempotency_key }`.
  - Validación: `amount > 0`, `reason` from enum,
    `idempotency_key` mandatory.
  - Llama `sparks_log` con `type='charge'`, `amount=-X`.
  - Si `insufficient_sparks` exception: retorna 402 Payment
    Required con balance actual.
  - Si `idempotency_key` duplicado: retorna transaction original
    (no doble cargo).
  - Invalidate balance cache.
  - Emit `sparks.charged` event con `{ user_id, amount,
    transaction_id, related_id }`.
- Internal endpoint (no para mobile cliente directo): solo otros
  Workers.
- Auth: dual — JWT del user + `X-Internal-Auth` (porque es
  internal).
- Telemetry: `sparks.charge_requested`, `.charge_succeeded`,
  `.charge_failed_insufficient`.

### Out

- UI de notificación al user de cargos (lo maneja el flow del
  caller — pedagogical engine).
- Refund (US-048).
- Premium plan handling (los plans no descuentan Sparks por
  exercise scoring; eso se chequea antes de llamar charge).

## Acceptance criteria

- **Given** user con balance 50, **When** POST charge con
  `amount: 10, idempotency_key: 'ex_attempt_001'`, **Then** balance
  baja a 40, transaction insertada, retorna `{ success: true,
  new_balance: 40, transaction_id: '<uuid>' }`.
- **Given** mismo idempotency_key se manda 2x, **When** segundo,
  **Then** retorna mismo transaction_id, balance NO cambia
  (idempotent).
- **Given** user con balance 5, **When** POST charge con
  `amount: 10`, **Then** retorna 402 con
  `{ error: 'insufficient_sparks', current_balance: 5,
  required: 10 }`.
- **Given** request sin idempotency_key, **When** POST, **Then**
  retorna 400 `{ error: 'idempotency_key_required' }`.
- **Given** request con `amount: 0` o negative, **When** valida,
  **Then** retorna 400 `{ error: 'invalid_amount' }`.
- **Given** request con reason no en enum, **When** valida, **Then**
  retorna 400 `{ error: 'invalid_reason' }`.
- **Given** 2 requests concurrent con idempotency_keys distintos
  para mismo user con balance 30 (cada uno amount 20), **When**
  ambos compiten, **Then** uno succeeds (balance → 10), otro fail
  con insufficient. NO race resulting en negative balance.
- **Given** charge exitoso, **When** completa, **Then** event
  `sparks.charged` se emite + cache invalidated.

## Developer details

### Owning service

`apps/workers/api/handlers/sparks-charge.ts`.

### Dependencies

- US-044: schema + helper `sparks_log`.
- US-045: cache invalidation.
- US-007: JWT middleware.

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §5 —
  cobro pattern.
- [`reglas.md`](../../reglas.md) — cobro previo + refund.

### Reasons enum permitidos

```typescript
const VALID_CHARGE_REASONS = [
  'exercise_scoring',
  'pronunciation_assessment',
  'free_response_evaluation',
  'roleplay_session_minute',
  'roleplay_session_extended',
  'insight_generation',
  'roadmap_regeneration',
  'streak_freeze_recovery',
  'pack_purchase',           // si user paga pack con balance (raro)
] as const;
```

### Implementación esperada

```typescript
// apps/workers/api/handlers/sparks-charge.ts
import { z } from 'zod';

const ChargeSchema = z.object({
  amount: z.number().int().positive(),
  reason: z.enum(VALID_CHARGE_REASONS),
  related_id: z.string().optional(),
  idempotency_key: z.string().min(1),
  metadata: z.record(z.any()).optional(),
});

export async function handleSparksCharge(request: AuthedRequest, env: Env) {
  // Internal auth (this is server-to-server)
  if (request.headers.get('X-Internal-Auth') !== env.INTERNAL_AUTH_TOKEN) {
    return jsonResponse({ error: 'unauthorized' }, 401);
  }

  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const body = await request.json();
  const parsed = ChargeSchema.safeParse(body);
  if (!parsed.success) {
    const issue = parsed.error.issues[0];
    return jsonResponse({ error: issue.message ?? 'invalid_input', path: issue.path }, 400);
  }

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  track('sparks.charge_requested', { amount: parsed.data.amount, reason: parsed.data.reason });

  // Check idempotency: if same key exists, return existing transaction
  const existing = await env.DB.query(`
    SELECT id, balance_after FROM sparks_transactions
    WHERE idempotency_key = $1 AND user_id = $2
  `, [parsed.data.idempotency_key, userId]);

  if (existing.length > 0) {
    return jsonResponse({
      success: true,
      transaction_id: existing[0].id,
      new_balance: existing[0].balance_after,
      idempotent_replay: true,
    });
  }

  // Atomic: charge via sparks_log helper
  try {
    const result = await env.DB.query(`
      SELECT * FROM sparks_log($1, 'charge', $2, $3, $4, $5)
    `, [
      userId,
      -parsed.data.amount, // negative amount = debit
      parsed.data.reason,
      parsed.data.related_id ?? null,
      parsed.data.idempotency_key,
    ]);

    const { transaction_id, new_balance } = result[0];

    await invalidateBalanceCache(userId, env);

    await emitEvent('sparks.charged', {
      user_id: userId,
      amount: parsed.data.amount,
      transaction_id,
      reason: parsed.data.reason,
      related_id: parsed.data.related_id,
    });

    track('sparks.charge_succeeded', { amount: parsed.data.amount });

    return jsonResponse({
      success: true,
      transaction_id,
      new_balance,
    });
  } catch (error: any) {
    if (error.message?.includes('insufficient_sparks')) {
      const current = await env.DB.query(`
        SELECT current FROM sparks_balances WHERE user_id = $1
      `, [userId]);
      track('sparks.charge_failed_insufficient', { requested: parsed.data.amount });
      return jsonResponse({
        error: 'insufficient_sparks',
        current_balance: current[0]?.current ?? 0,
        required: parsed.data.amount,
      }, 402);
    }
    throw error;
  }
}
```

### Race condition strategy

PostgreSQL row-level locking via `UPDATE ... RETURNING` en
`sparks_log` function (US-044) garantiza:
- Solo un statement puede modificar balance row a la vez.
- CHECK constraint `current >= 0` previene negative balance aún
  bajo race.
- Si concurrent: el segundo ve balance updated del primero +
  decide.

### Integration points

- Pedagogical engine (caller principal — antes de scoring).
- AI Gateway (caller — antes de invocar tasks costosas).
- US-048 (refund usa related transaction_id).
- Notifications (consumer de `sparks.charged` para low balance
  trigger).

### Notas técnicas

- `idempotency_key` debe ser unique cross-user pero same key per
  user retorna mismo transaction. Convención sugerida:
  `<feature>_<id>` (ej: `ex_attempt_${exercise_attempt_id}`).
- Endpoint internal (X-Internal-Auth) porque solo servers llaman
  directamente. Mobile NUNCA debería llamar charge directly.
- 402 Payment Required es el HTTP code correcto para "not enough
  Sparks".

## Definition of Done

- [ ] Endpoint implementado.
- [ ] Idempotency funcional.
- [ ] Race condition safe (test concurrent charges).
- [ ] CHECK constraint negative balance verificado.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration (incluyendo concurrent test con
  Promise.all de 10 charges).
- [ ] Performance: P50 < 50ms, P95 < 150ms.
- [ ] Validation contra spec `sparks-system.md` §5.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-048 (refund) y para EPIC-03 pedagogical scoring.*

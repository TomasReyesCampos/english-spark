# US-050: Sparks awardBonus (logros, referrals, comeback)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 2
**Story points:** 2
**Persona:** Admin (consumido por motivation, referrals, re-engagement)
**Owner:** —

---

## Contexto

Múltiples sistemas otorgan Sparks como bonus:
- **Achievements** (de motivation-and-achievements): 5-1000 Sparks
  por logro desbloqueado.
- **Referrals**: 20-250 Sparks (post-MVP).
- **Re-engagement D14**: 20 Sparks comeback (decisión cerrada en
  push-notifications-copy-bank §5.3).
- **Eventos especiales**: bonus comunitarios.

Esta story implementa el endpoint genérico
`POST /sparks/award-bonus` que cualquier sistema puede invocar.

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §4.

## Scope

### In

- Endpoint `POST /sparks/award-bonus`:
  - Auth: dual JWT user + X-Internal-Auth (interno only).
  - Body: `{ amount, reason, related_id, idempotency_key,
    expires_at? }`.
  - Validation: `amount > 0`, `reason` from extended enum.
  - Llama `sparks_log` con `type='award_bonus'`,
    `amount=+X`.
  - Idempotency: clave única previene duplicate awards.
  - Si `expires_at` provisto: bonus caduca (caso packs/promo
    temporales).
  - Invalidate cache.
  - Emit `sparks.bonus_awarded` event.
- Reasons enum extendido:
  - `achievement_${achievement_id}` (ej: `achievement_streak_7`).
  - `referral_signup`.
  - `referral_paid`.
  - `comeback_d14`.
  - `event_${event_id}`.
  - `support_compensation`.
  - `manual_admin_grant` (último recurso, audit-tracked).
- Cap diario por user (anti-abuse): max 5000 Sparks bonus / día.
  Si excede: rechaza con `daily_bonus_cap_exceeded`.

### Out

- UI para admin granting manual (post-MVP).
- Bonus tiered por plan (post-MVP).
- Streak-based bonus engine (eso es responsabilidad de motivation
  system; este endpoint solo recibe la award request).

## Acceptance criteria

- **Given** user con balance 50, **When** POST award-bonus con
  `amount: 20, reason: 'achievement_streak_7'`, **Then** balance
  sube a 70, transaction insertada con type='award_bonus'.
- **Given** mismo idempotency_key se manda 2x, **When** segundo,
  **Then** retorna mismo transaction_id sin doble award.
- **Given** request con `amount: 10000`, **When** valida, **Then**
  400 `{ error: 'invalid_amount', max_per_call: 5000 }`.
- **Given** user ya recibió 4500 bonus hoy, **When** pide otros
  600, **Then** 429 con `{ error: 'daily_bonus_cap_exceeded',
  awarded_today: 4500, cap: 5000 }`.
- **Given** request con `expires_at: '+30 days'`, **When**
  award completa, **Then** transaction tiene `expires_at`
  populated y se incluirá en cron expiration.
- **Given** request sin idempotency_key, **When** valida, **Then**
  400 `{ error: 'idempotency_key_required' }`.
- **Given** request sin X-Internal-Auth, **When** llega, **Then**
  401.
- **Given** award exitoso, **When** completa, **Then** event
  `sparks.bonus_awarded` emitido con
  `{ user_id, amount, reason, related_id }`.

## Developer details

### Owning service

`apps/workers/api/handlers/sparks-award-bonus.ts`.

### Dependencies

- US-044/045/047 (foundation + cache + helper sparks_log).

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §4.
- [`motivation-and-achievements.md`](../../product/motivation-and-achievements.md)
  §6.2 — catálogo de 80 logros con sparks_reward.

### Reasons enum permitidos

```typescript
const VALID_BONUS_REASONS = [
  // Pattern: achievement_<id> dynamically allowed
  'referral_signup',
  'referral_paid',
  'comeback_d14',
  'event_holiday',
  'event_anniversary',
  'support_compensation',
  'manual_admin_grant',
] as const;

function isValidBonusReason(reason: string): boolean {
  if (reason.startsWith('achievement_')) return true; // dynamic
  if (reason.startsWith('event_')) return true;
  return VALID_BONUS_REASONS.includes(reason as any);
}
```

### Implementación esperada

```typescript
// apps/workers/api/handlers/sparks-award-bonus.ts
const MAX_PER_CALL = 5000;
const DAILY_BONUS_CAP = 5000;

const AwardSchema = z.object({
  amount: z.number().int().positive().max(MAX_PER_CALL),
  reason: z.string().min(1).max(100),
  related_id: z.string().optional(),
  idempotency_key: z.string().min(1),
  expires_at: z.string().datetime().optional(),
  metadata: z.record(z.any()).optional(),
});

export async function handleSparksAwardBonus(request: AuthedRequest, env: Env) {
  if (request.headers.get('X-Internal-Auth') !== env.INTERNAL_AUTH_TOKEN) {
    return jsonResponse({ error: 'unauthorized' }, 401);
  }

  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const body = await request.json();
  const parsed = AwardSchema.safeParse(body);
  if (!parsed.success) {
    return jsonResponse({ error: parsed.error.issues[0].message }, 400);
  }

  if (!isValidBonusReason(parsed.data.reason)) {
    return jsonResponse({ error: 'invalid_reason', reason: parsed.data.reason }, 400);
  }

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  // Idempotency check
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

  // Daily cap check
  const todayAwarded = await env.DB.query(`
    SELECT COALESCE(SUM(amount), 0) AS today_total
    FROM sparks_transactions
    WHERE user_id = $1 AND type = 'award_bonus'
      AND created_at > date_trunc('day', now())
  `, [userId]);

  const totalToday = Number(todayAwarded[0].today_total);
  if (totalToday + parsed.data.amount > DAILY_BONUS_CAP) {
    return jsonResponse({
      error: 'daily_bonus_cap_exceeded',
      awarded_today: totalToday,
      cap: DAILY_BONUS_CAP,
    }, 429);
  }

  // Award via sparks_log
  const result = await env.DB.query(`
    SELECT * FROM sparks_log($1, 'award_bonus', $2, $3, $4, $5, $6, $7)
  `, [
    userId,
    parsed.data.amount,
    parsed.data.reason,
    parsed.data.related_id ?? null,
    parsed.data.idempotency_key,
    parsed.data.expires_at ?? null,
    JSON.stringify(parsed.data.metadata ?? {}),
  ]);

  const { transaction_id, new_balance } = result[0];

  await invalidateBalanceCache(userId, env);

  await emitEvent('sparks.bonus_awarded', {
    user_id: userId,
    amount: parsed.data.amount,
    reason: parsed.data.reason,
    related_id: parsed.data.related_id,
    transaction_id,
  });

  track('sparks.bonus_awarded', { reason: parsed.data.reason, amount: parsed.data.amount });

  return jsonResponse({
    success: true,
    transaction_id,
    new_balance,
  });
}
```

### Integration points

- Motivation system (consumer principal — cada `achievement.unlocked`
  triggers award).
- Re-engagement campaigns (D14 comeback bonus).
- Future referrals system.
- US-045 cache invalidation.

### Notas técnicas

- Daily cap 5000 es defensive — no debería alcanzar en uso normal
  (max logro legendario es 1000 Sparks).
- Endpoint internal: solo Workers llaman, no clientes mobile.
- `expires_at` opcional: trial Sparks lo tienen, achievements
  generalmente no (permanent).

## Definition of Done

- [ ] Endpoint implementado.
- [ ] Daily cap funcional.
- [ ] Idempotency funcional.
- [ ] Event emitido correctamente.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-044/045/047. Consumido por motivation system.*

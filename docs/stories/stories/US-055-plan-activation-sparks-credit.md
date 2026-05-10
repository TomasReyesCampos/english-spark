# US-055: Plan activation + Sparks credit on payment success

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 2
**Story points:** 2
**Persona:** Estudiante (recibe plan + Sparks)
**Owner:** —

---

## Contexto

Consumer único de eventos `payment.succeeded` (de Stripe US-053 +
RevenueCat US-054). Activa el plan del user y acredita los Sparks
del cycle.

**Operación atómica:** plan activation + Sparks credit en una
transacción. Si una falla, ambas rollback.

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §13 +
§4.

## Scope

### In

- Inngest function `activate-plan-on-payment-success`:
  - Triggered by event `payment.succeeded`.
  - Idempotent via `stripe_session_id` o `revenuecat_subscription_id`
    como key.
  - Lookup `plan_id` → config (allotment Sparks, name, billing
    cycle).
  - Atomic transaction:
    - Update `users.plan` o create row en `user_subscriptions`.
    - Update `sparks_balances.cycle_allotment`,
      `cycle_started_at`, `cycle_ends_at`.
    - Insert `sparks_transactions` type `award_subscription_cycle`,
      `amount = plan.allotment`, `expires_at = cycle_ends_at`.
  - Si `is_renewal = true`:
    - Sparks no usados del cycle anterior **se acumulan** (carry
      forward) hasta cap del plan_allotment * 2.
    - O sea: cycle_allotment 200 + 80 unused = 280 disponibles
      (capped at 400).
  - Invalidate cache.
  - Emit `subscription.activated` event.
- Plan config (hardcoded en MVP, post-MVP en BD):
  ```typescript
  const PLANS = {
    basic:   { sparks_allotment: 30,  cycle_days: 30 },
    pro:     { sparks_allotment: 200, cycle_days: 30 },
    premium: { sparks_allotment: 600, cycle_days: 30 },
  };
  ```
- Tabla `user_subscriptions`:
  ```sql
  CREATE TABLE user_subscriptions (
    user_id            UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    plan_id            TEXT NOT NULL,
    status             TEXT NOT NULL CHECK (status IN ('active', 'cancelled', 'expired', 'past_due')),
    cycle_starts_at    TIMESTAMPTZ NOT NULL,
    cycle_ends_at      TIMESTAMPTZ NOT NULL,
    cancelled_at       TIMESTAMPTZ,
    source             TEXT NOT NULL CHECK (source IN ('stripe', 'revenuecat')),
    external_subscription_id TEXT NOT NULL,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT now()
  );
  ```

### Out

- Plan downgrade prorate (post-MVP).
- Welcome flow post-paywall trigger (eso es US-024 de EPIC-01 +
  spec welcome-flow-post-paywall.md, separate trigger).

## Acceptance criteria

- **Given** event `payment.succeeded` con
  `plan_id: 'pro', is_renewal: false`, **When** Inngest function
  ejecuta, **Then** se crea/updates `user_subscriptions` con
  status='active', `sparks_balances.cycle_allotment = 200`,
  current se actualiza con award.
- **Given** event re-emitido (idempotency), **When** segunda
  ejecución, **Then** detect duplicate via key y skip
  re-activación.
- **Given** is_renewal=true, current=80, **When** renewal
  acredita 200, **Then** new current = 80 + 200 = 280 (carry
  forward).
- **Given** is_renewal=true, current=300, plan_allotment=200,
  **When** renewal, **Then** new current = min(300+200, 400)
  = 400 (cap).
- **Given** activación exitosa, **When** completa, **Then**
  emit `subscription.activated` con
  `{ user_id, plan_id, cycle_ends_at }`.
- **Given** error mid-transaction (DB timeout), **When** Inngest
  function falla, **Then** rollback completo (ni plan activated
  ni Sparks credited) + retry.
- **Given** plan_id desconocido (typo en config), **When** lookup
  falla, **Then** alert SEV-1 + no procesar (mejor errar que
  acreditar wrongly).
- **Given** event `payment.succeeded` para user que ya tiene
  Premium, **When** procesa nuevo pago Pro, **Then** downgrade:
  `plan_id` actualiza a 'pro', cycle reset, **Sparks no se
  reducen** (user pagó por ambos).

## Developer details

### Owning service

`apps/workers/inngest-functions/activate-plan-on-payment-success.ts`.

### Dependencies

- US-053 + US-054 emit events.
- US-044 schema + helper sparks_log.
- US-045 cache invalidation.

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §13 +
  §4.

### Implementación esperada

```typescript
// apps/workers/inngest-functions/activate-plan-on-payment-success.ts
const PLANS: Record<string, { sparks_allotment: number; cycle_days: number; name: string }> = {
  basic:   { sparks_allotment: 30,  cycle_days: 30, name: 'Plan Básico' },
  pro:     { sparks_allotment: 200, cycle_days: 30, name: 'Plan Pro' },
  premium: { sparks_allotment: 600, cycle_days: 30, name: 'Plan Premium' },
};

const CARRY_FORWARD_CAP_MULTIPLIER = 2;

export const activatePlanOnPaymentSuccessFn = inngest.createFunction(
  { id: 'activate-plan-on-payment-success', retries: 3 },
  { event: 'payment.succeeded' },
  async ({ event, step }) => {
    const { user_id, plan_id, is_renewal, source,
            stripe_subscription_id, revenuecat_subscription_id } = event.data;

    const plan = PLANS[plan_id];
    if (!plan) {
      throw new NonRetriableError(`unknown_plan: ${plan_id}`);
    }

    const externalSubId = stripe_subscription_id ?? revenuecat_subscription_id;
    const idempotencyKey = `plan_activation_${externalSubId}_${event.data.invoice_id ?? Date.now()}`;

    // Idempotency check
    const existing = await step.run('check-idempotency', () =>
      db.query(`SELECT id FROM sparks_transactions WHERE idempotency_key = $1`, [idempotencyKey])
    );
    if (existing.length > 0) {
      return { skipped: true, reason: 'duplicate' };
    }

    const cycleStart = new Date();
    const cycleEnd = new Date(cycleStart.getTime() + plan.cycle_days * 24 * 3600 * 1000);

    const result = await step.run('activate-plan-and-credit', () =>
      db.transaction(async (tx) => {
        // 1. Upsert subscription
        await tx.execute(`
          INSERT INTO user_subscriptions
            (user_id, plan_id, status, cycle_starts_at, cycle_ends_at, source, external_subscription_id)
          VALUES ($1, $2, 'active', $3, $4, $5, $6)
          ON CONFLICT (user_id) DO UPDATE
            SET plan_id = EXCLUDED.plan_id,
                status = 'active',
                cycle_starts_at = EXCLUDED.cycle_starts_at,
                cycle_ends_at = EXCLUDED.cycle_ends_at,
                source = EXCLUDED.source,
                external_subscription_id = EXCLUDED.external_subscription_id,
                cancelled_at = NULL,
                updated_at = now()
        `, [user_id, plan_id, cycleStart, cycleEnd, source, externalSubId]);

        // 2. Update balance metadata
        const currentBalance = await tx.query(`
          SELECT current FROM sparks_balances WHERE user_id = $1
        `, [user_id]);
        const existingCurrent = currentBalance[0]?.current ?? 0;

        // Carry forward logic
        const cap = plan.sparks_allotment * CARRY_FORWARD_CAP_MULTIPLIER;
        const newCurrent = is_renewal
          ? Math.min(existingCurrent + plan.sparks_allotment, cap)
          : plan.sparks_allotment;
        const sparksToAward = newCurrent - existingCurrent;

        await tx.execute(`
          INSERT INTO sparks_balances
            (user_id, current, cycle_allotment, cycle_started_at, cycle_ends_at, lifetime_earned)
          VALUES ($1, $2, $3, $4, $5, $2)
          ON CONFLICT (user_id) DO UPDATE
            SET cycle_allotment = EXCLUDED.cycle_allotment,
                cycle_started_at = EXCLUDED.cycle_started_at,
                cycle_ends_at = EXCLUDED.cycle_ends_at,
                lifetime_earned = sparks_balances.lifetime_earned + $6
        `, [user_id, newCurrent, plan.sparks_allotment, cycleStart, cycleEnd, sparksToAward]);

        // 3. Insert transaction (uses sparks_log helper for atomic balance update)
        const txResult = await tx.query(`
          INSERT INTO sparks_transactions
            (user_id, type, amount, balance_after, reason, idempotency_key, expires_at)
          VALUES ($1, 'award_subscription_cycle', $2, $3, $4, $5, $6)
          RETURNING id
        `, [user_id, sparksToAward, newCurrent, `cycle_${plan_id}`, idempotencyKey, cycleEnd]);

        return { subscription_id: txResult[0].id, new_balance: newCurrent };
      })
    );

    await step.run('invalidate-cache', () => invalidateBalanceCache(user_id, env));

    await step.run('emit-event', () =>
      emitEvent('subscription.activated', {
        user_id, plan_id, cycle_ends_at: cycleEnd.toISOString(),
        is_renewal, source,
      })
    );

    track('subscription.activated', { plan_id, is_renewal });
    return result;
  }
);
```

### Integration points

- US-053/054 (event source).
- US-044 (schema + transactions).
- US-045 (cache).
- Welcome flow post-paywall (mobile app detecta plan activated +
  navega).

### Notas técnicas

- Carry forward incentiva uso del producto (no perder Sparks
  comprados).
- Cap 2x previene hoarding indefinido.
- Plan downgrade preserves Sparks (user pagó, los conserva).

## Definition of Done

- [ ] Inngest function implementada.
- [ ] Schema `user_subscriptions` migration aplicada.
- [ ] Plan configs hardcoded.
- [ ] Carry forward logic correcta (test 3+ scenarios).
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Cierra el flow de payment → user con plan activo + Sparks
acreditados.*

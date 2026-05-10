# US-046: Sparks initial trial award (50 al signup)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 1
**Story points:** 2
**Persona:** Estudiante (recibe los 50 trial Sparks)
**Owner:** —

---

## Contexto

Cuando user completa signup (event `user.signed_up` de US-008), se
le acreditan **50 Sparks gratuitos** que duran los 7 días del
trial (decisión cerrada en
[ADR-006](../../decisions/ADR-006-trial-assessment.md)).

Esta story es el **handler del event** que crea el balance row +
inserta la transaction inicial.

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §4.3.

## Scope

### In

- Inngest function `award-trial-sparks-on-signup`:
  - Triggered by event `user.signed_up`.
  - Idempotency: usar `user_id` como key (no doble award).
  - Transaction:
    - Insert row en `sparks_balances` con `current = 50`,
      `cycle_allotment = 50`,
      `cycle_started_at = now()`,
      `cycle_ends_at = now() + 7 days`.
    - Insert transaction `award_trial`, `amount = 50`,
      `reason = 'trial_signup_award'`,
      `idempotency_key = 'trial_award_${user_id}'`.
  - Si balance row ya existe (re-trigger del event): NO award,
    log warning.
  - Invalidate balance cache.
  - Emit `sparks.trial_awarded` event.
- Failure handling: retry 3x con backoff exponential. Si falla
  3x: alert SEV-2 (user no recibió Sparks).

### Out

- Logic de extension del trial (post-MVP).
- Anonymous user trial Sparks (mismo comportamiento — anonymous
  también recibe; no diferenciation en MVP).

## Acceptance criteria

- **Given** event `user.signed_up` con
  `user_id = '<uuid>'`, **When** Inngest function ejecuta, **Then**
  se inserta `sparks_balances.current = 50` y transaction
  `award_trial`.
- **Given** event re-emitido para mismo user (race condition),
  **When** función ejecuta segunda vez, **Then** detecta
  duplicado vía idempotency_key y NO doble award.
- **Given** función completa exitosamente, **When** user llama
  GET /sparks/balance, **Then** retorna `current: 50`.
- **Given** función ejecuta, **When** completa, **Then** event
  `sparks.trial_awarded` se emite con `{ user_id, amount: 50 }`.
- **Given** Postgres timeout, **When** primera attempt falla,
  **Then** Inngest retry 1, 2, 3 con backoff. Si falla 3x: SEV-2
  alert.
- **Given** event llega para user que ya tiene balance row
  pre-existente, **When** función verifica, **Then** log warning +
  skip award + NO insert.
- **Given** anonymous user signup, **When** event triggered,
  **Then** mismo flow (50 Sparks acreditados).
- **Given** función completa, **When** balance cache existe,
  **Then** invalidate cache para que próxima read tenga balance
  fresh.

## Developer details

### Owning service

`apps/workers/inngest-functions/award-trial-sparks-on-signup.ts`.

### Dependencies

- US-008: emit event `user.signed_up`.
- US-044: schema sparks.
- US-045: cache invalidation function.

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §4.3 —
  trial Sparks economics.
- [`decisions/ADR-006-trial-assessment.md`](../../decisions/ADR-006-trial-assessment.md)
  — decisión de 50 Sparks.

### Implementación esperada

```typescript
// apps/workers/inngest-functions/award-trial-sparks-on-signup.ts
import { Inngest } from 'inngest';

const TRIAL_SPARKS = 50;
const TRIAL_DAYS = 7;

export const awardTrialSparksOnSignupFn = inngest.createFunction(
  { id: 'award-trial-sparks-on-signup', retries: 3 },
  { event: 'user.signed_up' },
  async ({ event, step }) => {
    const { user_id } = event.data;
    const idempotencyKey = `trial_award_${user_id}`;

    // Check if already awarded
    const existing = await step.run('check-existing', async () => {
      const result = await db.query(`
        SELECT id FROM sparks_transactions WHERE idempotency_key = $1
      `, [idempotencyKey]);
      return result.length > 0;
    });

    if (existing) {
      console.warn({ event: 'sparks.trial_award_skipped_duplicate', user_id });
      return { skipped: true, reason: 'duplicate' };
    }

    // Atomic: insert balance + transaction
    const result = await step.run('insert-balance-and-tx', async () => {
      return await db.transaction(async (tx) => {
        const cycleStart = new Date();
        const cycleEnd = new Date(cycleStart.getTime() + TRIAL_DAYS * 24 * 3600 * 1000);

        await tx.execute(`
          INSERT INTO sparks_balances
            (user_id, current, cycle_allotment, cycle_started_at, cycle_ends_at,
             lifetime_earned)
          VALUES ($1, $2, $2, $3, $4, $2)
          ON CONFLICT (user_id) DO NOTHING
        `, [user_id, TRIAL_SPARKS, cycleStart, cycleEnd]);

        await tx.execute(`
          INSERT INTO sparks_transactions
            (user_id, type, amount, balance_after, reason, idempotency_key,
             expires_at)
          VALUES ($1, 'award_trial', $2, $2, 'trial_signup_award', $3, $4)
        `, [user_id, TRIAL_SPARKS, idempotencyKey, cycleEnd]);

        return { user_id, awarded: TRIAL_SPARKS };
      });
    });

    await step.run('invalidate-cache', () =>
      invalidateBalanceCache(user_id, env)
    );

    await step.run('emit-event', () =>
      emitEvent('sparks.trial_awarded', { user_id, amount: TRIAL_SPARKS })
    );

    return result;
  }
);
```

### Integration points

- Inngest event bus.
- Postgres.
- KV cache invalidation.
- Notifications system (eventually consumes
  `sparks.trial_awarded` para welcome notif).

### Notas técnicas

- Idempotency key formato `trial_award_${user_id}` garantiza único
  award per user, ever.
- `cycle_ends_at = now() + 7 days` matchea trial duration de
  ADR-006.
- Trial Sparks expiran al final del trial (`expires_at`), pero
  expiration cron (US-057) los marca como expired post-fact.
- `ON CONFLICT (user_id) DO NOTHING` en balances evita
  duplicar row.

## Definition of Done

- [ ] Inngest function implementada.
- [ ] Idempotency verificada con duplicate event.
- [ ] Transaction atómica (ambas tablas o ninguna).
- [ ] Cache invalidation post-award.
- [ ] Event `sparks.trial_awarded` emitido.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration (con event simulado).
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-008 (event emit), US-044 (schema), US-045 (cache).
Cierra el flujo de signup → user con 50 Sparks listos.*

# US-056: Pack purchase flow (one-time Sparks pack)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 2
**Story points:** 2
**Persona:** Estudiante (compra packs)
**Owner:** —

---

## Contexto

Además de subscriptions, el producto vende **packs one-time** de
Sparks (sin renovación). Útil para users que necesitan más Sparks
del cycle pero no quieren upgradear plan.

3 packs MVP:
- Small: 50 Sparks por $2.99
- Medium: 130 Sparks por $6.99
- Large: 350 Sparks por $17.99

Esta story consume el evento `pack.purchased` (emitted por US-054
RevenueCat para mobile, o por US-053 Stripe para web vía
`checkout.session.completed` con metadata `is_pack=true`).

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §3
(packs).

## Scope

### In

- Inngest function `credit-pack-on-purchase`:
  - Triggered by event `pack.purchased`.
  - Idempotent via `external_transaction_id`.
  - Award Sparks via `sparks_log` con
    `type='award_pack'`,
    `amount=pack.sparks_amount`,
    `expires_at = now() + 6 months` (decisión cerrada en
    `sparks-system.md`),
    `reason='pack_${pack_id}_purchase'`.
  - Invalidate cache.
  - Emit `pack.credited` event.
- Pack configs hardcoded:
  ```typescript
  const PACKS = {
    small:  { sparks: 50,  expires_months: 6 },
    medium: { sparks: 130, expires_months: 6 },
    large:  { sparks: 350, expires_months: 6 },
  };
  ```
- Tracking visible al user: pack expiration date en balance
  endpoint (US-045 ya retorna `expiring_soon`).

### Out

- Pack discount/promo logic (post-MVP).
- Gift packs (post-MVP, decisión rechazada en pendientes).
- Custom pack sizes (post-MVP).

## Acceptance criteria

- **Given** event `pack.purchased` con
  `pack_id: 'medium', user_id, external_transaction_id`,
  **When** function ejecuta, **Then** balance sube +130,
  transaction insertada con type='award_pack',
  expires_at = +6 months.
- **Given** mismo event re-emitido, **When** segundo, **Then**
  detect duplicate via external_transaction_id, NO doble award.
- **Given** pack_id desconocido en config, **When** lookup,
  **Then** error log SEV-1 + no award (mejor errar).
- **Given** award exitoso, **When** completa, **Then** event
  `pack.credited` emitido.
- **Given** próximo GET /sparks/balance, **When** user consulta,
  **Then** `expiring_soon` incluye el pack si está dentro de
  7 días de expiration.
- **Given** user con balance 30 + pack large credited,
  **When** balance final, **Then** current = 30 + 350 = 380.
- **Given** Postgres timeout, **When** Inngest retry, **Then**
  retry 3x con backoff. Falla 3x: SEV-1 (user pagó, no recibió).
- **Given** event `pack.purchased` con `source: 'stripe'`
  (web), **When** function procesa, **Then** mismo flow funciona.

## Developer details

### Owning service

`apps/workers/inngest-functions/credit-pack-on-purchase.ts`.

### Dependencies

- US-053/054 emit events.
- US-044 (schema + sparks_log).
- US-045 (cache invalidation).

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §3.
- [`i18n.md`](../../cross-cutting/i18n.md) §4.1 — pricing
  localizado.

### Implementación esperada

```typescript
// apps/workers/inngest-functions/credit-pack-on-purchase.ts
const PACKS: Record<string, { sparks: number; expires_months: number }> = {
  small:  { sparks: 50,  expires_months: 6 },
  medium: { sparks: 130, expires_months: 6 },
  large:  { sparks: 350, expires_months: 6 },
};

export const creditPackOnPurchaseFn = inngest.createFunction(
  { id: 'credit-pack-on-purchase', retries: 3 },
  { event: 'pack.purchased' },
  async ({ event, step }) => {
    const { user_id, pack_id, external_transaction_id, source } = event.data;

    const pack = PACKS[pack_id];
    if (!pack) {
      throw new NonRetriableError(`unknown_pack: ${pack_id}`);
    }

    const idempotencyKey = `pack_${external_transaction_id}`;

    const expiresAt = new Date();
    expiresAt.setMonth(expiresAt.getMonth() + pack.expires_months);

    const result = await step.run('credit-pack', () =>
      db.query(`
        SELECT * FROM sparks_log($1, 'award_pack', $2, $3, $4, $5, $6)
      `, [
        user_id,
        pack.sparks,
        `pack_${pack_id}_purchase`,
        external_transaction_id,
        idempotencyKey,
        expiresAt,
      ])
    );

    if (result[0]?.transaction_id === null) {
      // Already processed
      return { skipped: true, reason: 'duplicate' };
    }

    await step.run('invalidate-cache', () => invalidateBalanceCache(user_id, env));

    await step.run('emit-event', () =>
      emitEvent('pack.credited', {
        user_id, pack_id,
        sparks_amount: pack.sparks,
        expires_at: expiresAt.toISOString(),
        source,
      })
    );

    track('pack.credited', { pack_id, source });
    return { credited: pack.sparks, expires_at: expiresAt.toISOString() };
  }
);
```

### Integration points

- US-053 + US-054 (event source).
- US-044 (sparks_log).
- US-057 (expiration cron procesa packs caducados).

### Notas técnicas

- 6 months expiration es decisión cerrada del producto (balance
  entre fairness y prevent hoarding indefinido).
- `expires_soon` notification trigger (post-MVP) avisará al user
  7 días antes.

## Definition of Done

- [ ] Inngest function implementada.
- [ ] 3 pack configs.
- [ ] Idempotency.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-053/054. Bloqueante para US-057 (expiration).*

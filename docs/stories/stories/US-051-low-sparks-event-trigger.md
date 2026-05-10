# US-051: Low Sparks event trigger (delega a notifications)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 2
**Story points:** 2
**Persona:** Estudiante (recibe notif) / Admin (lo dispara)
**Owner:** —

---

## Contexto

Cuando el balance de un user baja a < 20% de su `cycle_allotment`,
queremos triggerear notification de low balance (notification spec
en `push-notifications-copy-bank.md` §8.1).

Esta story emite el event apropiado; el send real lo maneja el
notifications-system (futuro EPIC-06).

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §11 +
[`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
§8.1.

## Scope

### In

- Inngest function `detect-low-sparks-on-charge`:
  - Triggered by event `sparks.charged`.
  - Lee balance post-charge del payload.
  - Lee `cycle_allotment` del balance row.
  - Si `current / cycle_allotment < 0.20` (20%) y `cycle_allotment
    > 0` (skip free trial users): emit `sparks.balance_low`.
- Cooldown logic: max 1 evento `sparks.balance_low` per user
  per 7 días (anti-spam).
  - Track via key en KV `low_sparks_alert:${user_id}` con TTL
    7 days.
  - Si key existe: skip emit.
- También trigger a `sparks.balance_depleted` si `current = 0`
  específicamente (más urgente).
- Telemetry: `sparks.low_balance_detected`,
  `sparks.low_balance_alert_skipped_cooldown`.

### Out

- Send real de la notif (eso es notifications-system US futura).
- Send via email (post-MVP).
- Retry logic si event delivery falla (Inngest lo maneja).

## Acceptance criteria

- **Given** user con `cycle_allotment: 200` (Plan Pro), balance
  `current: 50` (25%), **When** charge baja a `current: 30`
  (15% < 20%), **Then** emit `sparks.balance_low` event.
- **Given** user con `cycle_allotment: 200`, **When** charge baja
  a `current: 0`, **Then** emit BOTH `sparks.balance_low` y
  `sparks.balance_depleted`.
- **Given** evento `sparks.balance_low` ya emitido para user en
  últimos 7 días (key en KV), **When** próximo charge pasa
  threshold, **Then** NO emit (cooldown) + log skip.
- **Given** trial user con `cycle_allotment: 50` y balance baja a
  10 (20%), **When** function evalúa, **Then** emit
  `sparks.balance_low` (trial counts).
- **Given** balance baja pero sigue ≥ 20%, **When** function
  evalúa, **Then** NO emit.
- **Given** `cycle_allotment = 0` (raro, edge case), **When**
  function evalúa, **Then** NO emit (división por cero
  prevenida).
- **Given** Inngest retry tras failure, **When** mismo event
  procesado 2x, **Then** cooldown KV previene doble emit.
- **Given** user paga upgrade a Premium → `cycle_allotment: 600`,
  **When** próximo charge, **Then** threshold reset (20% de 600
  = 120; balance 50 todavía low → emit si no en cooldown).

## Developer details

### Owning service

`apps/workers/inngest-functions/detect-low-sparks-on-charge.ts`.

### Dependencies

- US-047: emite `sparks.charged`.
- US-044: schema balances (lectura cycle_allotment).
- KV namespace `SPARKS_KV` (cooldown tracking).

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §11.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §8.1 — copy de low_sparks notification.

### Implementación esperada

```typescript
// apps/workers/inngest-functions/detect-low-sparks-on-charge.ts
const LOW_THRESHOLD_PCT = 0.20;
const COOLDOWN_DAYS = 7;

export const detectLowSparksOnChargeFn = inngest.createFunction(
  { id: 'detect-low-sparks-on-charge' },
  { event: 'sparks.charged' },
  async ({ event, step }) => {
    const { user_id } = event.data;

    const balance = await step.run('fetch-balance', async () => {
      const result = await db.query(`
        SELECT current, cycle_allotment FROM sparks_balances WHERE user_id = $1
      `, [user_id]);
      return result[0];
    });

    if (!balance || balance.cycle_allotment === 0) {
      return { skipped: true, reason: 'no_allotment' };
    }

    const ratio = balance.current / balance.cycle_allotment;
    const isLow = ratio < LOW_THRESHOLD_PCT;
    const isDepleted = balance.current === 0;

    if (!isLow && !isDepleted) {
      return { skipped: true, reason: 'above_threshold' };
    }

    track('sparks.low_balance_detected', {
      user_id, current: balance.current, ratio,
    });

    // Cooldown check
    const cooldownKey = `low_sparks_alert:${user_id}`;
    const inCooldown = await step.run('check-cooldown', () =>
      env.SPARKS_KV.get(cooldownKey).then(v => !!v)
    );

    if (inCooldown) {
      track('sparks.low_balance_alert_skipped_cooldown', { user_id });
      return { skipped: true, reason: 'cooldown' };
    }

    // Emit events
    await step.run('emit-low', () =>
      emitEvent('sparks.balance_low', {
        user_id,
        current_balance: balance.current,
        cycle_allotment: balance.cycle_allotment,
        percent_remaining: Math.floor(ratio * 100),
      })
    );

    if (isDepleted) {
      await step.run('emit-depleted', () =>
        emitEvent('sparks.balance_depleted', { user_id })
      );
    }

    // Set cooldown
    await step.run('set-cooldown', () =>
      env.SPARKS_KV.put(cooldownKey, '1', {
        expirationTtl: COOLDOWN_DAYS * 24 * 3600,
      })
    );

    return { emitted: true, depleted: isDepleted };
  }
);
```

### Integration points

- US-047 emite trigger event.
- Notifications system (futuro EPIC-06) consume `sparks.balance_low`
  y `sparks.balance_depleted`.
- Telemetry.

### Notas técnicas

- Single trigger event (`sparks.charged`) en lugar de polling cron
  reduce latencia (notif llega en minutos no horas).
- Cooldown de 7 días alineado con `notifications-system.md` §4.6
  rate limit de transactional notifs.
- `sparks.balance_depleted` es separate event para que notifs
  pueda usar copy distinto (más urgente).

## Definition of Done

- [ ] Inngest function implementada.
- [ ] Cooldown KV-based funcional.
- [ ] Both events (`balance_low` + `balance_depleted`) emitidos
  correctamente.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit con event simulado.
- [ ] Test integration verifica cooldown.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-047. Consumido por notifications-system (post-MVP).*

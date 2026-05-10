# US-067: Payment events handler (failed/restriction)

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 3
**Story points:** 2
**Persona:** Estudiante (recibe payment / restriction notifs)
**Owner:** —

---

## Contexto

Consumer de events transactional críticos:
- `payment.failed` → notif `payment_failed`, 3 variants según
  attempt_count.
- `restriction.applied` → notif `restriction_applied`, 4 variants
  según restriction_type.

Ambos son **transactional críticos** — bypassan rate limit
estándar.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§4.6.

## Scope

### In

- Inngest functions:
  - `send-payment-failed-push`:
    - Trigger: event `payment.failed`.
    - Routing por `attempt_count`:
      - 1 → variant `payment_failed.first.01`.
      - 2 → variant `payment_failed.retry_2.01`.
      - 3 → variant `payment_failed.final.01`.
    - **Bypass rate limit** (crítico).
  - `send-restriction-applied-push`:
    - Trigger: event `restriction.applied` (de anti-fraud).
    - Routing por `restriction_type`:
      - `no_trial_sparks`, `rate_limited`, `suspended`, `banned`
        → variant correspondiente.
    - Bypass rate limit.
- Telemetry: `notif.payment_failed_push_sent`,
  `notif.restriction_applied_push_sent`.

### Out

- Email notifications (Stripe/RevenueCat ya envían sus propios
  receipts).
- SMS (post-MVP).

## Acceptance criteria

- **Given** event `payment.failed` con `attempt_count: 1`, **When**
  procesa, **Then** envía notif con variant `.first.01` ("Tu pago
  no pudo procesarse...").
- **Given** event con `attempt_count: 3` (último), **When** procesa,
  **Then** envía variant `.final.01` ("Última oportunidad...").
- **Given** event con `attempt_count` desconocido (>3 o 0), **When**
  procesa, **Then** SEV-2 alert + fallback a variant final.
- **Given** event `restriction.applied` con
  `restriction_type: 'banned'`, **When** procesa, **Then** envía
  notif con variant correspondiente del copy_bank.
- **Given** user con `push_enabled = false`, **When** payment_failed
  llega, **Then** **SÍ** envía (transactional crítico bypass).
- **Given** rate limit cap reminders excedido, **When** payment
  event, **Then** **SÍ** envía (transactional crítico).
- **Given** user banned recibió notif `restriction_applied
  (banned)`, **When** próximos events normales lleguen, **Then**
  US-061 anti-fraud check los bloquea — pero esta notif inicial
  SÍ se envía.
- **Given** copy_bank lookup falla, **When** procesa, **Then**
  SEV-2 + fallback hardcoded en código.

## Developer details

### Owning service

`apps/workers/inngest-functions/send-payment-events-push.ts`.

### Dependencies

- US-053/054 emit `payment.failed`.
- Anti-fraud system emit `restriction.applied`.
- US-059/060/061.

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §4.6.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §8.3 + §8.4.

### Implementación esperada

```typescript
// apps/workers/inngest-functions/send-payment-events-push.ts
const PAYMENT_FAILED_VARIANTS: Record<number, string> = {
  1: 'first',
  2: 'retry_2',
  3: 'final',
};

export const sendPaymentFailedPushFn = inngest.createFunction(
  { id: 'send-payment-failed-push', retries: 2 },
  { event: 'payment.failed' },
  async ({ event }) => {
    const { user_id, attempt_count, source } = event.data;

    const variant = PAYMENT_FAILED_VARIANTS[attempt_count] ?? 'final';
    if (!PAYMENT_FAILED_VARIANTS[attempt_count]) {
      console.warn({ event: 'notif.unknown_attempt_count', attempt_count });
    }

    // Auth check: only push_enabled (transactional override)
    const prefs = await db.query(`
      SELECT push_enabled FROM user_notification_preferences WHERE user_id = $1
    `, [user_id]);
    if (!prefs[0]?.push_enabled) return { skipped: 'push_disabled' };

    const copy = await db.query(`
      SELECT title_template, body_template, copy_id
      FROM notifications_copy_bank
      WHERE notification_id = 'payment_failed' AND variant = $1 AND is_active = true
      LIMIT 1
    `, [variant]);

    if (copy.length === 0) {
      throw new NonRetriableError(`copy_not_found: payment_failed.${variant}`);
    }

    await sendFcmNotification({
      user_id,
      notification_id: 'payment_failed',
      category: 'transactional',
      title: copy[0].title_template,
      body: copy[0].body_template,
      copy_id: copy[0].copy_id,
      data: { deeplink: 'englishspark://billing/retry' },
    }, env);

    track('notif.payment_failed_push_sent', { variant, source });
    return { sent: true };
  }
);

export const sendRestrictionAppliedPushFn = inngest.createFunction(
  { id: 'send-restriction-applied-push', retries: 2 },
  { event: 'restriction.applied' },
  async ({ event }) => {
    const { user_id, restriction_type } = event.data;

    const prefs = await db.query(`
      SELECT push_enabled FROM user_notification_preferences WHERE user_id = $1
    `, [user_id]);
    if (!prefs[0]?.push_enabled) return { skipped: 'push_disabled' };

    const copy = await db.query(`
      SELECT title_template, body_template, copy_id
      FROM notifications_copy_bank
      WHERE notification_id = 'restriction_applied' AND variant = $1 AND is_active = true
      LIMIT 1
    `, [restriction_type]);

    if (copy.length === 0) {
      throw new NonRetriableError(`copy_not_found: restriction_applied.${restriction_type}`);
    }

    await sendFcmNotification({
      user_id,
      notification_id: 'restriction_applied',
      category: 'transactional',
      title: copy[0].title_template,
      body: copy[0].body_template,
      copy_id: copy[0].copy_id,
      data: { deeplink: 'englishspark://settings/restrictions' },
    }, env);

    track('notif.restriction_applied_push_sent', { restriction_type });
    return { sent: true };
  }
);
```

### Integration points

- US-053/054 (payment.failed source).
- Anti-fraud system (restriction.applied source).
- US-061 sender.

### Notas técnicas

- Payment failed bypass total: crítico que user vea esto incluso si
  silenció todo (porque pierde el plan si no actúa).
- Restriction applied es uno-shot (no se re-envía aunque user
  bloquee push).

## Definition of Done

- [ ] 2 Inngest functions implementadas.
- [ ] Routing por attempt_count + restriction_type.
- [ ] Transactional bypass.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Consumer de EPIC-04 + anti-fraud.*

# US-066: Sparks events handler (low/depleted/expiring)

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 3
**Story points:** 3
**Persona:** Estudiante (recibe Sparks alerts)
**Owner:** —

---

## Contexto

EPIC-04 emite varios events relacionados a Sparks. Esta story es
el consumer único que envía push según el event:
- `sparks.balance_low` → notif `low_sparks`.
- `sparks.balance_depleted` → notif `low_sparks` con tono más
  urgente.
- `sparks.expiring_soon` → notif `pack_expiring`.

Categorías: estos son **transactional** según copy bank §1.5 — se
envían aunque user tenga `reminders_enabled = false`.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§4.6.

## Scope

### In

- Inngest function `send-sparks-events-push`:
  - Triggers múltiples events: `sparks.balance_low`,
    `sparks.balance_depleted`, `sparks.expiring_soon`.
  - Routing por event_type a copy específico del copy_bank:
    - `balance_low` → `low_sparks` notification_id, variant elegido
      por % remaining (de spec §8.1).
    - `balance_depleted` → `low_sparks` con variant "depleted"
      tone más urgente.
    - `expiring_soon` → `pack_expiring`.
  - Auth check: solo respeta `push_enabled = false` (no
    `reminders_enabled` porque es transactional).
  - sendFcmNotification.
  - Rate limit transactional: max 5/día/user (del bank §1.5).
- Telemetry: `notif.sparks_event_push_sent`,
  `notif.sparks_event_suppressed`.

### Out

- Notif de "Sparks acreditados" cuando user compra pack (post-MVP,
  podría ser nice-to-have).

## Acceptance criteria

- **Given** event `sparks.balance_low` con `current_balance: 30,
  cycle_allotment: 200, percent_remaining: 15`, **When** function
  procesa, **Then** envía notif `low_sparks` con body que menciona
  "30 Sparks" y "15%".
- **Given** event `sparks.balance_depleted`, **When** procesa,
  **Then** envía notif con tono urgente del copy bank.
- **Given** user con `push_enabled = false`, **When** event llega,
  **Then** skip (push global off).
- **Given** user con `reminders_enabled = false`, **When** event,
  **Then** **SÍ** envía (transactional override).
- **Given** event `sparks.expiring_soon` con
  `{ amount: 50, expires_at: '+5 days' }`, **When** procesa,
  **Then** notif `pack_expiring` con "5 días" y "50 Sparks".
- **Given** user ya recibió 5 transactional notifs hoy, **When**
  llega 6to event, **Then** rate limiter rechaza, telemetry
  suppress.
- **Given** user en exercise session, **When** event llega,
  **Then** **SÍ** envía (transactional no se suprime por
  sesión).
- **Given** copy_bank lookup falla para variant, **When**
  procesa, **Then** SEV-2 + skip.

## Developer details

### Owning service

`apps/workers/inngest-functions/send-sparks-events-push.ts`.

### Dependencies

- US-051 emit `sparks.balance_low`.
- US-057 emit `sparks.expiring_soon`.
- US-059/060/061.

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §4.6.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §8.1 + §8.2.

### Implementación esperada

```typescript
// apps/workers/inngest-functions/send-sparks-events-push.ts
export const sendSparksLowFn = inngest.createFunction(
  { id: 'send-sparks-low-push', retries: 2 },
  { event: 'sparks.balance_low' },
  async ({ event }) => handleSparksEvent(event, 'low_sparks', 'low'),
);

export const sendSparksDepletedFn = inngest.createFunction(
  { id: 'send-sparks-depleted-push', retries: 2 },
  { event: 'sparks.balance_depleted' },
  async ({ event }) => handleSparksEvent(event, 'low_sparks', 'depleted'),
);

export const sendSparksExpiringFn = inngest.createFunction(
  { id: 'send-sparks-expiring-push', retries: 2 },
  { event: 'sparks.expiring_soon' },
  async ({ event }) => handleSparksEvent(event, 'pack_expiring', 'default'),
);

async function handleSparksEvent(event: any, notifId: string, variant: string) {
  const { user_id } = event.data;

  // Auth check: only push_enabled matters for transactional
  const prefs = await db.query(`
    SELECT push_enabled, paused_until FROM user_notification_preferences WHERE user_id = $1
  `, [user_id]);
  if (!prefs[0]?.push_enabled) return { skipped: 'push_disabled' };

  // Look up copy
  const copy = await db.query(`
    SELECT title_template, body_template, copy_id
    FROM notifications_copy_bank
    WHERE notification_id = $1 AND variant = $2 AND is_active = true
    ORDER BY random() LIMIT 1
  `, [notifId, variant]);

  if (copy.length === 0) {
    console.error({ event: 'notif.copy_missing', notif_id: notifId, variant });
    throw new NonRetriableError('copy_not_found');
  }

  // Render with event payload vars
  const vars = {
    sparks_remaining: event.data.current_balance ?? event.data.amount ?? 0,
    percent_remaining: event.data.percent_remaining ?? 0,
    pack_expiry_days: event.data.expires_at
      ? Math.ceil((new Date(event.data.expires_at).getTime() - Date.now()) / (1000 * 60 * 60 * 24))
      : 0,
  };

  const title = renderTemplate(copy[0].title_template, vars);
  const body = renderTemplate(copy[0].body_template, vars);

  await sendFcmNotification({
    user_id,
    notification_id: notifId,
    category: 'transactional',
    title, body,
    copy_id: copy[0].copy_id,
    data: { deeplink: notifId === 'pack_expiring'
      ? 'englishspark://sparks/balance'
      : 'englishspark://sparks/buy' },
  }, env);

  track('notif.sparks_event_push_sent', { type: event.name });
  return { sent: true };
}
```

### Integration points

- US-051, US-057 (event sources).
- US-059 schema.
- US-061 sender.
- US-070 rate limiter.

### Notas técnicas

- 3 Inngest functions separadas (1 per event) para clarity y
  independent retries.
- Transactional category bypasses `reminders_enabled` opt-out.
- Aún respeta `push_enabled = false` (master kill switch).

## Definition of Done

- [ ] 3 Inngest functions implementadas.
- [ ] Variant routing correcto.
- [ ] Transactional override respetado.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Consumer de EPIC-04 events.*

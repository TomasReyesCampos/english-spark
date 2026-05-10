# US-068: Welcome series cron (D1/D3/D5/D7)

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 3
**Story points:** 2
**Persona:** Estudiante trial
**Owner:** —

---

## Contexto

Welcome series envía 4 notifs durante los primeros 7 días del
trial:
- D1 (24h post-signup): "¡Buen primer día, María!"
- D3: progress mention.
- D5: pre-assessment expectation.
- D7: assessment trigger.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§4 +
[`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
§4.

## Scope

### In

- Cron Worker `apps/workers/crons/welcome-series.ts`:
  - Trigger: cada hora UTC.
  - Para cada hora identifica users elegibles:
    - D1: signed_up entre 23-25 horas atrás, no recibió welcome_d1
      yet.
    - D3: signed_up entre 71-73 horas atrás.
    - D5: signed_up entre 119-121 horas atrás.
    - D7: signed_up entre 167-169 horas atrás.
  - Idempotency check: `notifications_log` ya tiene entry para
    ese notification_id?
  - Variants:
    - D3: with_progress si practicó al menos 1 vez, no_progress
      si no.
    - D7: variant fijo (assessment trigger).
  - sendFcmNotification.
- Telemetry: `notif.welcome_series_sent` con day.

### Out

- D1 IA-personalizado (post-MVP, MVP usa template).
- Email welcome series (post-MVP).

## Acceptance criteria

- **Given** user signed_up hace 24h, **When** welcome cron corre,
  **Then** envía welcome_d1 con su display_name.
- **Given** user ya recibió welcome_d1, **When** cron evalúa
  segunda vez, **Then** skip (idempotency via notifications_log
  check).
- **Given** user D3 que practicó al menos 1 vez, **When** procesa,
  **Then** envía variant `welcome_d3.with_progress.*`.
- **Given** user D3 sin practicar, **When** procesa, **Then**
  variant `welcome_d3.no_progress.*`.
- **Given** user D7, **When** procesa, **Then** envía
  welcome_d7 con deeplink al assessment.
- **Given** user con `onboarding_enabled = false`, **When** cron,
  **Then** skip.
- **Given** user banned, **When** cron, **Then** skip.
- **Given** user que ya pagó (no trial active), **When** D5 cron,
  **Then** skip welcome_d5 (welcome series solo aplica durante
  trial).

## Developer details

### Owning service

`apps/workers/crons/welcome-series.ts`.

### Dependencies

- US-008 emit `user.signed_up`.
- US-059/060/061.

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §4.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §4 — copy variants.

### Implementación esperada

```typescript
// apps/workers/crons/welcome-series.ts
const WELCOME_HOURS = [
  { day: 1, notif_id: 'welcome_d1', min_hours: 23, max_hours: 25 },
  { day: 3, notif_id: 'welcome_d3', min_hours: 71, max_hours: 73 },
  { day: 5, notif_id: 'welcome_d5', min_hours: 119, max_hours: 121 },
  { day: 7, notif_id: 'welcome_d7', min_hours: 167, max_hours: 169 },
];

export async function runWelcomeSeriesCron(env: Env) {
  let totalSent = 0;

  for (const config of WELCOME_HOURS) {
    const eligible = await env.DB.query(`
      SELECT u.id AS user_id, u.display_name, u.signup_at,
             COUNT(ea.id) AS exercise_count
      FROM users u
      JOIN user_notification_preferences np ON np.user_id = u.id
      LEFT JOIN exercise_attempts ea ON ea.user_id = u.id
        AND ea.completed_at IS NOT NULL
      WHERE u.deleted_at IS NULL
        AND np.push_enabled = true AND np.onboarding_enabled = true
        AND u.signup_at BETWEEN now() - interval '${config.max_hours} hours'
                              AND now() - interval '${config.min_hours} hours'
        AND NOT EXISTS (
          SELECT 1 FROM notifications_log nl
          WHERE nl.user_id = u.id AND nl.notification_id = $1
        )
        AND NOT EXISTS (
          SELECT 1 FROM user_restrictions ur
          WHERE ur.user_id = u.id AND ur.restriction_type IN ('suspended', 'banned')
        )
        AND NOT EXISTS (
          SELECT 1 FROM user_subscriptions us
          WHERE us.user_id = u.id AND us.status = 'active'
        )
      GROUP BY u.id, u.display_name, u.signup_at
    `, [config.notif_id]);

    for (const user of eligible) {
      const variant = pickVariant(config.day, user.exercise_count > 0);

      const copy = await env.DB.query(`
        SELECT title_template, body_template, copy_id
        FROM notifications_copy_bank
        WHERE notification_id = $1 AND variant = $2 AND is_active = true
        ORDER BY random() LIMIT 1
      `, [config.notif_id, variant]);

      if (copy.length === 0) {
        console.error({ event: 'notif.welcome_copy_missing', notif: config.notif_id, variant });
        continue;
      }

      const vars = { name: user.display_name?.split(' ')[0] ?? null, minutes: 15 };
      const title = renderTemplate(copy[0].title_template, vars);
      const body = renderTemplate(copy[0].body_template, vars);

      await sendFcmNotification({
        user_id: user.user_id,
        notification_id: config.notif_id,
        category: 'onboarding',
        title, body,
        copy_id: copy[0].copy_id,
        data: { deeplink: config.day === 7
          ? 'englishspark://assessment/start'
          : config.day === 5 ? 'englishspark://assessment/preview'
          : 'englishspark://home' },
      }, env);

      totalSent++;
    }
  }

  track('notif.welcome_series_sent', { total: totalSent });
  return { sent: totalSent };
}

function pickVariant(day: number, practiced: boolean): string {
  if (day === 3) return practiced ? 'with_progress' : 'no_progress';
  return 'default';
}
```

### Cron trigger

```toml
[triggers]
crons = ["0 * * * *"]
```

### Integration points

- US-059 notifications_log (idempotency).
- US-061 sender.
- US-008 user.signed_up timestamp.

### Notas técnicas

- ±1 hour window evita doble send incluso si cron timing drifts.
- Welcome no aplica a users que ya pagaron (`user_subscriptions
  active`) — coordinar con flow de conversion.
- D7 timing alineado con assessment trigger Day 7 (ADR-006).

## Definition of Done

- [ ] Cron implementado + trigger.
- [ ] Idempotency via notifications_log.
- [ ] Variant routing D3 with/without progress.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-008, US-059/060/061. Driver de retention temprana.*

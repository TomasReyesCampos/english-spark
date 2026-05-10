# US-064: Streak at risk cron

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 2
**Story points:** 2
**Persona:** Estudiante (con streak ≥ 5)
**Owner:** —

---

## Contexto

Cron que detecta users con streak en riesgo (no practicaron hoy +
streak ≥ 5) y envía notificación urgente 4 horas antes de
medianoche en su timezone.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§4.4.

## Scope

### In

- Cron Worker `apps/workers/crons/streak-at-risk.ts`:
  - Trigger: cada hora UTC.
  - Para cada hora: identifica timezones donde **ahora es 20:00
    local** (4h antes de medianoche).
  - Query users con:
    - timezone matching (donde local hour = 20).
    - `streak >= 5`.
    - NO exercise_attempt completed hoy en su TZ.
    - `reminders_enabled = true`.
    - No en pause.
  - Para cada user matching:
    - Lee scheduled `streak_at_risk` content (de US-063 batch) o
      fallback.
    - sendFcmNotification.
- Telemetry: `notif.streak_at_risk_cron_run`,
  `.streak_at_risk_eligible`, `.streak_at_risk_sent`.

### Out

- Streak < 5 (decisión del producto — no notif para rachas cortas).
- Multiple alerts por noche (single shot).

## Acceptance criteria

- **Given** user con streak 8, timezone='America/Mexico_City'
  (UTC-6), local time 20:00 (UTC 02:00), **When** cron corre UTC
  02:00, **Then** match y notification enviada.
- **Given** user con streak 3, **When** cron evalúa, **Then** skip
  (under threshold).
- **Given** user ya practicó hoy, **When** cron evalúa, **Then**
  skip.
- **Given** scheduled content existe del batch nocturno, **When**
  cron lo usa, **Then** marca sent.
- **Given** sin scheduled, **When** cron, **Then** fallback a
  template de copy_bank (variant según length de streak).
- **Given** user practicó entre batch nocturno y cron, **When**
  cron re-verifica (no envió), **Then** detecta exercise_attempt
  reciente y skip.
- **Given** user banned, **When** cron evalúa, **Then** skip.
- **Given** rate limit cap diario reminders alcanzado para user,
  **When** cron intenta enviar, **Then** US-070 rate limiter
  rechaza.

## Developer details

### Owning service

`apps/workers/crons/streak-at-risk.ts`.

### Dependencies

- US-059/060/061.
- US-062 (mismo patrón).
- US-063 (provee scheduled).

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §4.4.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §3 — copy variants por streak length.

### Implementación esperada

```typescript
// apps/workers/crons/streak-at-risk.ts
const STREAK_HOUR_LOCAL = 20; // 4h antes de medianoche
const MIN_STREAK = 5;

export async function runStreakAtRiskCron(env: Env) {
  const eligible = await env.DB.query(`
    SELECT
      u.id AS user_id, u.display_name,
      np.timezone,
      st.current_streak,
      sp.daily_minutes_available AS minutes
    FROM users u
    JOIN user_notification_preferences np ON np.user_id = u.id
    JOIN streaks st ON st.user_id = u.id
    LEFT JOIN student_profiles sp ON sp.user_id = u.id
    WHERE u.deleted_at IS NULL
      AND np.push_enabled = true AND np.reminders_enabled = true
      AND (np.paused_until IS NULL OR np.paused_until < now())
      AND st.current_streak >= $1
      AND EXTRACT(HOUR FROM (now() AT TIME ZONE np.timezone)) = $2
      AND NOT EXISTS (
        SELECT 1 FROM exercise_attempts ea
        WHERE ea.user_id = u.id
          AND ea.completed_at > date_trunc('day', now() AT TIME ZONE np.timezone)
      )
      AND NOT EXISTS (
        SELECT 1 FROM user_restrictions ur
        WHERE ur.user_id = u.id
          AND ur.restriction_type IN ('suspended', 'banned')
      )
  `, [MIN_STREAK, STREAK_HOUR_LOCAL]);

  track('notif.streak_at_risk_eligible', { count: eligible.length });

  let sent = 0;
  for (const user of eligible) {
    const scheduled = await env.DB.query(`
      SELECT id, title, body, copy_id FROM notifications_scheduled
      WHERE user_id = $1 AND notification_id = 'streak_at_risk'
        AND status = 'pending'
      LIMIT 1
    `, [user.user_id]);

    let title: string, body: string, copyId: string | null;

    if (scheduled.length > 0) {
      ({ title, body, copy_id: copyId } = scheduled[0]);
    } else {
      const variant = user.current_streak >= 30 ? 'long'
                    : user.current_streak >= 10 ? 'medium'
                    : 'short';
      const tmpl = await env.DB.query(`
        SELECT * FROM notifications_copy_bank
        WHERE notification_id = 'streak_at_risk' AND variant = $1
          AND is_active = true ORDER BY random() LIMIT 1
      `, [variant]);
      title = renderTemplate(tmpl[0].title_template, user);
      body = renderTemplate(tmpl[0].body_template, user);
      copyId = tmpl[0].copy_id;
    }

    const result = await sendFcmNotification({
      user_id: user.user_id,
      notification_id: 'streak_at_risk',
      category: 'reminder',
      title, body,
      copy_id: copyId,
      data: { deeplink: 'englishspark://exercise/today?reason=streak' },
    }, env);

    if (result.delivered > 0) {
      sent++;
      if (scheduled.length > 0) {
        await env.DB.execute(`
          UPDATE notifications_scheduled SET status = 'sent' WHERE id = $1
        `, [scheduled[0].id]);
      }
    }
  }

  track('notif.streak_at_risk_sent', { sent });
  return { eligible: eligible.length, sent };
}
```

### Cron trigger

```toml
[triggers]
crons = ["0 * * * *"]  # Mismo schedule que daily-reminders
```

### Integration points

- US-061 sendFcmNotification.
- US-063 (provee scheduled).
- US-070 rate limiter.

### Notas técnicas

- 20:00 local = 4h antes de medianoche. Sweet spot: suficiente
  tiempo para reaccionar, no demasiado temprano para spam.
- Variants por length: short (5-9), medium (10-29), long (30+) ya
  definidos en `push-notifications-copy-bank.md` §3.2.

## Definition of Done

- [ ] Cron implementado + trigger.
- [ ] Variant selection por streak length.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] QA con device en TZ distinta.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-059/060/061. Métrica clave: streaks salvadas /
notif enviada.*

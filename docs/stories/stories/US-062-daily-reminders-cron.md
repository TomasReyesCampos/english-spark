# US-062: Cron daily reminders timezone-aware

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 2
**Story points:** 5
**Persona:** Estudiante (recibe), Admin (cron operativo)
**Owner:** —

---

## Contexto

Daily reminders es **la notification más importante del producto**
para retention. Esta story implementa:
- Cron que corre cada hora UTC.
- Para cada hora UTC: identifica usuarios cuya hora local
  preferida coincide (TZ-aware).
- Filter: solo users que NO practicaron hoy + `reminders_enabled =
  true` + no en pause/cooldown.
- Lee contenido del `notifications_scheduled` (pre-generated por
  batch nocturno US-063) o fallback a template hardcoded.
- Envía via `sendFcmNotification` (US-061).

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§7.3.

## Scope

### In

- Cron Worker `apps/workers/crons/daily-reminders.ts`:
  - Trigger: cada hora UTC (`0 * * * *`).
  - Para hora UTC actual: query users con
    `preferred_reminder_hour` matching local hour calculated desde
    timezone.
  - Filters:
    - `push_enabled = true`.
    - `reminders_enabled = true`.
    - `paused_until IS NULL OR paused_until < now()`.
    - NO exercise_attempt completado hoy (en TZ del user).
    - Anti-fraud: no user `banned` ni `suspended`.
  - Para cada user matching:
    - Lee scheduled content del user para este `notification_id =
      'daily_reminder'` y `scheduled_for` < now() + 1 hour.
    - Si no hay scheduled: usar fallback template del
      `notifications_copy_bank`.
    - Render variables ({name}, {streak}, {focus_today}, etc.).
    - sendFcmNotification.
    - Mark scheduled como `status='sent'`.
- Helper `get_user_local_hour(timezone, utc_now)` SQL function
  (o computed Worker-side con Intl).
- Telemetry: `notif.daily_reminder_cron_run`,
  `.daily_reminder_eligible_count`, `.daily_reminder_sent`.

### Out

- Content generation per se (US-063 batch nocturno).
- A/B testing entre variants (post-MVP).

## Acceptance criteria

- **Given** UTC hour = 23:00 y user con
  `preferred_reminder_hour=19, timezone='America/Mexico_City'`,
  **When** cron corre, **Then** local time del user es 17:00 →
  NO match (queremos 19:00 local).
- **Given** UTC hour = 01:00 y user con
  `preferred_reminder_hour=19, timezone='America/Mexico_City'`
  (que está a UTC-6), **When** local time es 19:00, **Then**
  match.
- **Given** user ya practicó hoy (exercise_attempt en últimas
  24h en su TZ), **When** cron evalúa, **Then** skip.
- **Given** user con `paused_until = tomorrow`, **When** cron,
  **Then** skip.
- **Given** user banned, **When** cron evalúa, **Then** skip.
- **Given** scheduled content existe (`notifications_scheduled`
  pending), **When** cron lo usa, **Then** marca como sent y NO
  recae en fallback.
- **Given** sin scheduled (cron AI batch falló), **When** cron,
  **Then** usa fallback de copy_bank con vars resueltas.
- **Given** 1000 users eligible, **When** cron procesa, **Then**
  completa en <5 min con telemetry counts correctos.

## Developer details

### Owning service

`apps/workers/crons/daily-reminders.ts`.

### Dependencies

- US-059, US-060, US-061.
- US-063 (provee content batch).

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §7.3.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §2 — copy de daily_reminder.

### Implementación esperada

```typescript
// apps/workers/crons/daily-reminders.ts
export async function runDailyRemindersCron(env: Env) {
  const startTime = Date.now();
  const currentUtcHour = new Date().getUTCHours();

  // Query users matching preferred hour in their TZ
  const eligible = await env.DB.query(`
    SELECT
      u.id AS user_id, u.display_name,
      np.preferred_reminder_hour, np.timezone,
      sp.cycle_allotment, sb.current AS sparks_current,
      st.current_streak
    FROM users u
    JOIN user_notification_preferences np ON np.user_id = u.id
    LEFT JOIN sparks_balances sb ON sb.user_id = u.id
    LEFT JOIN streaks st ON st.user_id = u.id
    WHERE u.deleted_at IS NULL
      AND np.push_enabled = true
      AND np.reminders_enabled = true
      AND (np.paused_until IS NULL OR np.paused_until < now())
      AND EXTRACT(HOUR FROM (now() AT TIME ZONE np.timezone)) = np.preferred_reminder_hour
      AND NOT EXISTS (
        SELECT 1 FROM exercise_attempts ea
        WHERE ea.user_id = u.id
          AND ea.completed_at > date_trunc('day', now() AT TIME ZONE np.timezone)
      )
      AND NOT EXISTS (
        SELECT 1 FROM user_restrictions ur
        WHERE ur.user_id = u.id
          AND ur.restriction_type IN ('suspended', 'banned')
          AND (ur.ends_at IS NULL OR ur.ends_at > now())
      )
  `);

  track('notif.daily_reminder_eligible_count', { count: eligible.length });

  let sent = 0;
  for (const user of eligible) {
    // Try scheduled first
    const scheduled = await env.DB.query(`
      SELECT id, title, body, copy_id FROM notifications_scheduled
      WHERE user_id = $1 AND notification_id = 'daily_reminder'
        AND status = 'pending' AND scheduled_for <= now() + interval '1 hour'
      LIMIT 1
    `, [user.user_id]);

    let title: string, body: string, copyId: string | null;

    if (scheduled.length > 0) {
      ({ title, body, copy_id: copyId } = scheduled[0]);
    } else {
      // Fallback to template
      const tmpl = await selectFallbackTemplate('daily_reminder', user, env);
      title = renderTemplate(tmpl.title_template, user);
      body = renderTemplate(tmpl.body_template, user);
      copyId = tmpl.copy_id;
    }

    const result = await sendFcmNotification({
      user_id: user.user_id,
      notification_id: 'daily_reminder',
      category: 'reminder',
      title, body,
      copy_id: copyId ?? undefined,
      data: { deeplink: 'englishspark://exercise/today' },
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

  const elapsed = Date.now() - startTime;
  track('notif.daily_reminder_sent', { sent, elapsed_ms: elapsed });
  return { eligible: eligible.length, sent };
}
```

### Cron config

```toml
[triggers]
crons = ["0 * * * *"]  # Cada hora UTC
```

### Fallback template selection

```typescript
function selectFallbackTemplate(notifId: string, user: any, env: Env) {
  // Lógica: con/sin streak, con/sin focus_today, día de semana
  // Usar las variants definidas en push-notifications-copy-bank.md §2
  const variant = user.current_streak >= 3
    ? (user.focus_today ? 'streak_focus' : 'streak_generic')
    : (user.focus_today ? 'focus' : 'generic');

  return env.DB.query(`
    SELECT * FROM notifications_copy_bank
    WHERE notification_id = $1 AND variant = $2 AND is_active = true
    ORDER BY random() LIMIT 1
  `, [notifId, variant]);
}
```

### Integration points

- US-061 sendFcmNotification.
- US-063 (provee scheduled content).
- US-070 rate limiter (CALLER side).

### Notas técnicas

- TZ matching SQL: `EXTRACT(HOUR FROM (now() AT TIME ZONE
  user_tz))` resuelve correctamente DST.
- Cron cada hora aceptable: ±60 min de drift es UX-tolerable.
- Para escalar: 100k users eligible en 1h → ~28 req/s →
  Worker handles fácil.

## Definition of Done

- [ ] Cron Worker implementado + trigger configurado.
- [ ] TZ-aware filter correcto.
- [ ] Fallback template logic.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration con TZ simulados.
- [ ] QA: test con device en device TZ distinta a UTC.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-059/060/061. Consumer principal del producto.*

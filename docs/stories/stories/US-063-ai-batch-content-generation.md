# US-063: AI batch nocturno content generation

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 2
**Story points:** 2
**Persona:** Admin
**Owner:** —

---

## Contexto

Cron 02:00 UTC que pre-genera contenido personalizado IA para
**todas las daily_reminder + streak_at_risk** del día siguiente,
usando la task `generate_notification_content` (US-036).

Pre-generation evita:
- Latencia en cron horario US-062 (no espera LLM).
- Burst de costos AI gateway en hora pico de envío.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§8.1.

## Scope

### In

- Cron Worker `apps/workers/crons/notif-content-batch.ts`:
  - Trigger: diario 02:00 UTC.
  - Query users elegibles para daily_reminder mañana:
    - `push_enabled = true AND reminders_enabled = true`.
    - Activos en últimos 7 días (signed_in OR exercise_attempt).
    - Sin restricción.
  - Para cada user:
    - Recopilar context: name, streak, focus_today (de roadmap),
      recent_highlight, plan.
    - Determinar si necesita `streak_at_risk` (streak >= 5).
    - Invocar AI Gateway task `generate_notification_content`
      (US-036).
    - Persistir output en `notifications_scheduled` con
      `scheduled_for = mañana hora preferida en TZ del user`,
      `generated_by = 'ai'`.
  - Batch concurrency control: max 10 invocations LLM concurrentes
    (Anthropic rate limit + cost burst).
  - Telemetry: `notif.batch_started`, `.batch_user_processed`,
    `.batch_completed`.

### Out

- A/B testing entre variantes (post-MVP).
- Re-generation si user cambió focus mid-day (post-MVP).

## Acceptance criteria

- **Given** cron triggered 02:00 UTC, **When** ejecuta, **Then**
  itera todos los users activos elegibles.
- **Given** user con streak 8, **When** batch procesa, **Then**
  genera daily_reminder + streak_at_risk (porque streak >= 5).
- **Given** user con streak 2, **When** batch procesa, **Then**
  genera solo daily_reminder (no streak_at_risk).
- **Given** AI Gateway task falla para un user, **When** batch
  procesa, **Then** skip ese user (US-036 ya tiene fallback al
  template; batch no inserta scheduled para ese user, cron horario
  US-062 hará fallback).
- **Given** scheduled ya existe para mañana (re-run del batch),
  **When** batch corre, **Then** detect duplicate y skip (no
  doble generation).
- **Given** 10k users elegibles, **When** batch procesa, **Then**
  completa en <30 min con concurrency=10.
- **Given** budget total del batch excede $20 (10k × $0.001 + 30%
  buffer), **When** monitor alert, **Then** SEV-2 (revisar prompt
  o concurrency).
- **Given** batch completa, **When** próximo cron horario US-062
  corre, **Then** encuentra scheduled rows pendientes y las usa.

## Developer details

### Owning service

`apps/workers/crons/notif-content-batch.ts`.

### Dependencies

- US-036: AI Gateway task `generate_notification_content`.
- US-059: schema notifications_scheduled.
- US-060: preferences read.

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §8.1.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §10 — reglas IA personalization.

### Implementación esperada

```typescript
// apps/workers/crons/notif-content-batch.ts
import pLimit from 'p-limit';

const CONCURRENCY = 10;
const limit = pLimit(CONCURRENCY);

export async function runContentBatchCron(env: Env) {
  const startTime = Date.now();

  // Active users elegibles (signed in OR exercised last 7 days)
  const users = await env.DB.query(`
    SELECT DISTINCT u.id AS user_id, u.display_name,
      np.preferred_reminder_hour, np.timezone,
      sp.active_track, sp.current_focus,
      sb.cycle_allotment,
      st.current_streak,
      sub.plan_id
    FROM users u
    JOIN user_notification_preferences np ON np.user_id = u.id
    LEFT JOIN student_profiles sp ON sp.user_id = u.id
    LEFT JOIN sparks_balances sb ON sb.user_id = u.id
    LEFT JOIN streaks st ON st.user_id = u.id
    LEFT JOIN user_subscriptions sub ON sub.user_id = u.id
    WHERE u.deleted_at IS NULL
      AND np.push_enabled = true AND np.reminders_enabled = true
      AND (u.last_signin_at > now() - interval '7 days'
           OR EXISTS (SELECT 1 FROM exercise_attempts ea
                      WHERE ea.user_id = u.id
                        AND ea.completed_at > now() - interval '7 days'))
      AND NOT EXISTS (
        SELECT 1 FROM user_restrictions ur
        WHERE ur.user_id = u.id
          AND ur.restriction_type IN ('suspended', 'banned')
      )
  `);

  let processed = 0;
  let skipped = 0;

  await Promise.all(users.map(user => limit(async () => {
    // Compute scheduled_for: tomorrow at preferred_reminder_hour in user's TZ
    const scheduledFor = computeNextLocalTime(
      user.timezone, user.preferred_reminder_hour
    );

    // Skip if already scheduled
    const existing = await env.DB.query(`
      SELECT id FROM notifications_scheduled
      WHERE user_id = $1 AND notification_id = 'daily_reminder'
        AND scheduled_for >= $2 AND scheduled_for < $2 + interval '24 hours'
        AND status = 'pending'
    `, [user.user_id, scheduledFor]);
    if (existing.length > 0) {
      skipped++;
      return;
    }

    const notifsToGenerate = ['daily_reminder'];
    if (user.current_streak >= 5) notifsToGenerate.push('streak_at_risk');

    try {
      const generated = await aiGateway.invoke('generate_notification_content', {
        user: {
          name: user.display_name?.split(' ')[0] ?? null,
          streak: user.current_streak ?? 0,
          plan: user.plan_id ?? 'trial',
        },
        tomorrow: { focus: user.current_focus ?? null },
        notifications_to_generate: notifsToGenerate,
      }, { user_id: user.user_id });

      for (const notifId of notifsToGenerate) {
        const copy = generated.output[notifId];
        if (!copy) continue;

        await env.DB.execute(`
          INSERT INTO notifications_scheduled
            (user_id, notification_id, scheduled_for, title, body, copy_id, generated_by)
          VALUES ($1, $2, $3, $4, $5, $6, 'ai')
        `, [user.user_id, notifId, scheduledFor, copy.title, copy.body, copy.copy_id]);
      }
      processed++;
      track('notif.batch_user_processed', { user_id: user.user_id });
    } catch (error: any) {
      console.warn({ event: 'notif.batch_user_failed', user_id: user.user_id, error: error.message });
    }
  })));

  const elapsed = Date.now() - startTime;
  track('notif.batch_completed', {
    total_users: users.length,
    processed, skipped, elapsed_ms: elapsed,
  });

  return { total: users.length, processed, skipped };
}

function computeNextLocalTime(timezone: string, hour: number): Date {
  // Compute "tomorrow at hour:00 in timezone" returned as UTC Date
  // Uses Intl.DateTimeFormat for TZ math
  const now = new Date();
  // Simplified: real implementation handles DST + timezone-aware date construction
  return new Date(/* ... */);
}
```

### Cron trigger

```toml
[triggers]
crons = ["0 2 * * *"]  # Diario 02:00 UTC
```

### Integration points

- US-036 AI Gateway task.
- US-059 schema scheduled.
- US-062 (consumer del scheduled output).

### Notas técnicas

- Concurrency 10 balance entre throughput y Anthropic rate limit.
- Si batch toma > 2 horas: el cron horario empezará a fallback a
  templates para users no-batched yet. OK degradation.
- Cost target: $0.001 per user × 10k users = $10/día max.

## Definition of Done

- [ ] Cron implementado + trigger.
- [ ] Concurrency control.
- [ ] Idempotency check.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration con AI mocked.
- [ ] Cost verification post-deploy.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-036, US-059. Provee content para US-062.*

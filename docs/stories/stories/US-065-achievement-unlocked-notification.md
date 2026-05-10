# US-065: Achievement unlocked notification handler

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 3
**Story points:** 3
**Persona:** Estudiante (recibe celebration)
**Owner:** —

---

## Contexto

Cuando el motivation system desbloquea un logro emite event
`achievement.unlocked`. Esta story es el consumer que envía push
push con el copy específico del achievement (de los 80 del
catalog).

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§4 + `motivation-and-achievements.md` §6.2.

## Scope

### In

- Inngest function `send-achievement-unlocked-push`:
  - Triggered by event `achievement.unlocked` con payload
    `{ user_id, achievement_id, sparks_awarded }`.
  - Idempotency: skip si ya envió push para este
    `achievement_id + user_id` (clave en KV).
  - Lee copy específico de
    `notifications_copy_bank` por `copy_id = achievement_id`.
  - Skip si user tiene `achievements_enabled = false`.
  - Skip si user está en sesión activa
    (exercise_attempts.in_progress=true en últimos 5 min) — UI
    maneja in-app celebration en ese caso.
  - sendFcmNotification con notification_id =
    `achievement_unlocked`, category = `achievement`.
- Rate limiting: max 3 achievement notifs por user por día (del
  catalog del copy_bank §1.5). Si excede: suprime silently.
- Telemetry: `notif.achievement_push_sent`,
  `notif.achievement_suppressed_in_session`,
  `notif.achievement_suppressed_cap`.

### Out

- In-app celebration UI (responsabilidad del mobile app).
- Achievement screen / collection (responsabilidad del producto).

## Acceptance criteria

- **Given** event `achievement.unlocked` con
  `achievement_id: 'streak_7'`, **When** function ejecuta, **Then**
  envía push con título "🔥 Primera semana" + body de copy_bank.
- **Given** mismo event re-emitido (idempotency), **When** segunda
  vez, **Then** detecta key en KV y skip.
- **Given** user con `achievements_enabled = false`, **When** event,
  **Then** skip silently (sin push pero sí log).
- **Given** user con exercise_attempt active (en sesión), **When**
  event, **Then** skip + telemetry
  `achievement_suppressed_in_session`.
- **Given** user ya recibió 3 achievement notifs hoy, **When** 4to
  llega, **Then** skip + telemetry `cap`. NO error.
- **Given** achievement_id no existe en copy_bank, **When** lookup,
  **Then** SEV-2 alert (catalog gap) + skip.
- **Given** push enviado exitosamente, **When** completa, **Then**
  notifications_log row insertada.
- **Given** rate limit Durable Object (US-070) rechaza, **When**
  attempt, **Then** log + skip.

## Developer details

### Owning service

`apps/workers/inngest-functions/send-achievement-unlocked-push.ts`.

### Dependencies

- US-059/060/061.
- US-070 rate limiter.
- Motivation system emit `achievement.unlocked`.

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §4.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §6 — copy de los 80 logros.
- [`motivation-and-achievements.md`](../../product/motivation-and-achievements.md)
  §6.2 — catalog autoritativo.

### Implementación esperada

```typescript
// apps/workers/inngest-functions/send-achievement-unlocked-push.ts
const MAX_ACHIEVEMENT_NOTIFS_PER_DAY = 3;

export const sendAchievementUnlockedPushFn = inngest.createFunction(
  { id: 'send-achievement-unlocked-push', retries: 2 },
  { event: 'achievement.unlocked' },
  async ({ event, step }) => {
    const { user_id, achievement_id, sparks_awarded } = event.data;

    const idempotencyKey = `achievement_notif_${user_id}_${achievement_id}`;
    const alreadySent = await step.run('check-idempotency', () =>
      env.NOTIF_KV.get(idempotencyKey).then(v => !!v)
    );
    if (alreadySent) return { skipped: 'duplicate' };

    // Check user preferences
    const prefs = await step.run('check-prefs', () => db.query(`
      SELECT achievements_enabled, push_enabled, paused_until
      FROM user_notification_preferences WHERE user_id = $1
    `, [user_id]));
    if (!prefs[0]?.push_enabled || !prefs[0]?.achievements_enabled) {
      return { skipped: 'preferences' };
    }

    // Check if user in active session
    const inSession = await step.run('check-session', () => db.query(`
      SELECT 1 FROM exercise_attempts
      WHERE user_id = $1 AND in_progress = true
        AND started_at > now() - interval '5 minutes'
      LIMIT 1
    `, [user_id]));
    if (inSession.length > 0) {
      track('notif.achievement_suppressed_in_session');
      return { skipped: 'in_session' };
    }

    // Check daily cap
    const todayCount = await step.run('check-cap', () => db.query(`
      SELECT COUNT(*) AS count FROM notifications_log
      WHERE user_id = $1 AND category = 'achievement'
        AND sent_at > date_trunc('day', now())
    `, [user_id]));
    if (Number(todayCount[0].count) >= MAX_ACHIEVEMENT_NOTIFS_PER_DAY) {
      track('notif.achievement_suppressed_cap');
      return { skipped: 'daily_cap' };
    }

    // Look up copy
    const copy = await step.run('lookup-copy', () => db.query(`
      SELECT title_template, body_template, copy_id
      FROM notifications_copy_bank
      WHERE notification_id = 'achievement_unlocked'
        AND copy_id LIKE $1 AND is_active = true LIMIT 1
    `, [`%${achievement_id}%`]));

    if (copy.length === 0) {
      console.error({ event: 'notif.achievement_copy_missing', achievement_id });
      throw new NonRetriableError('copy_not_found');
    }

    const title = copy[0].title_template; // no vars needed for achievements
    const body = renderTemplate(copy[0].body_template, { sparks_awarded });

    await step.run('send', () => sendFcmNotification({
      user_id,
      notification_id: 'achievement_unlocked',
      category: 'achievement',
      title, body,
      copy_id: copy[0].copy_id,
      data: { deeplink: `englishspark://achievements/${achievement_id}` },
    }, env));

    await step.run('mark-sent', () =>
      env.NOTIF_KV.put(idempotencyKey, '1', { expirationTtl: 30 * 24 * 3600 })
    );

    track('notif.achievement_push_sent', { achievement_id });
    return { sent: true };
  }
);
```

### Integration points

- Motivation system (event emit).
- US-059 schema (copy_bank seeded con achievements).
- US-061 sender.
- US-070 rate limiter.

### Notas técnicas

- Idempotency 30 días TTL: previene re-send si event re-emit
  semanas después (raro pero defensive).
- Suppression in-session: importante porque UI ya muestra
  celebration. Push duplicado sería molesto.

## Definition of Done

- [ ] Function implementada.
- [ ] Suppression logic (preferences, in_session, daily cap).
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Consumer de motivation system. Bloqueante para retention via
gamification.*

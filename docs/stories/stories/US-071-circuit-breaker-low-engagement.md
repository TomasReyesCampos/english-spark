# US-071: Circuit breaker para low-engagement users

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 3
**Story points:** 2
**Persona:** Admin (anti-fatigue + anti-spam reputation)
**Owner:** —

---

## Contexto

Si un user no abre push consistentemente, seguir enviándole es:
- Mala UX (notifs irrelevantes).
- Daño a reputation del sender (Apple/Google penalizan low
  engagement → entrega futura sufre).
- Costo desperdiciado (FCM gratis pero AI generation no).

Circuit breaker pausa notifs (excepto transactional críticos) si
detecta signals de low engagement.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§9.3.

## Scope

### In

- Cron Worker `apps/workers/crons/circuit-breaker.ts`:
  - Trigger: diario 04:00 UTC.
  - 3 reglas de evaluación per user:

    1. **3 dismisses consecutivos** sin opens → pause 7 días.
    2. **Open rate < 5% en últimos 30 días** (con ≥ 20 notifs sent) →
       pause 14 días.
    3. **Inactive 30 días** (sin opens ni exercise) → pause 30
       días.
  - Update `user_notification_preferences.paused_until` y
    `pause_reason`.
- Cron unpause: si user vuelve a engagear (open o exercise), unpause
  inmediato.
- Emit event `user.notifications_paused` (audit + analytics).
- Telemetry: `notif.circuit_breaker_paused`,
  `notif.circuit_breaker_unpaused`.

### Out

- Per-category circuit breaker (post-MVP — MVP es global).
- ML-based engagement scoring (post-MVP).

## Acceptance criteria

- **Given** user con last 3 notifs dismissed (no opened),
  **When** cron evalúa, **Then** `paused_until = now() + 7 days`,
  `pause_reason = 'user_dismissed_3_in_row'`.
- **Given** user con 20 notifs en últimos 30 días pero solo 1
  opened (5%), **When** cron evalúa, **Then** pause 14 días
  con reason `open_rate_under_5pct_in_30d`.
- **Given** user inactivo 30 días, **When** cron, **Then** pause
  30 días con reason `inactive_30_days`.
- **Given** user con paused_until = future, **When** envío de
  notif intenta (US-061), **Then** sender check `paused_until >
  now()` y skip.
- **Given** user paused abre app (signs in), **When** detection
  cron (separate o on event), **Then** unpause inmediato.
- **Given** user paused recibe transactional crítico
  (payment_failed), **When** US-067 evalúa, **Then** **SÍ** envía
  (transactional bypass).
- **Given** user con múltiples signals (dismisses + low open rate),
  **When** cron evalúa, **Then** aplica el pause MÁS LARGO (14
  days > 7).
- **Given** evento `user.notifications_paused`, **When** emitido,
  **Then** payload incluye `{ user_id, reason, paused_until }`
  para audit.

## Developer details

### Owning service

`apps/workers/crons/circuit-breaker.ts` +
`unpause-on-engagement.ts` (Inngest function event-driven).

### Dependencies

- US-059 (preferences).
- US-061 (sender respeta paused_until).

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §9.3.

### Implementación esperada

```typescript
// apps/workers/crons/circuit-breaker.ts
export async function runCircuitBreakerCron(env: Env) {
  let paused = 0;

  // Rule 1: 3 dismisses consecutivos
  const dismissed3 = await env.DB.query(`
    SELECT user_id, MAX(sent_at) AS last_sent FROM (
      SELECT user_id, sent_at,
        opened_at IS NULL AS dismissed,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY sent_at DESC) AS rn
      FROM notifications_log
      WHERE sent_at > now() - interval '14 days'
    ) sub
    WHERE rn <= 3 AND dismissed = true
    GROUP BY user_id
    HAVING COUNT(*) = 3
  `);
  await applyPause(dismissed3, 7, 'user_dismissed_3_in_row', env);
  paused += dismissed3.length;

  // Rule 2: open rate < 5%
  const lowOpenRate = await env.DB.query(`
    SELECT user_id,
      COUNT(*) AS total,
      SUM(CASE WHEN opened_at IS NOT NULL THEN 1 ELSE 0 END) AS opened
    FROM notifications_log
    WHERE sent_at > now() - interval '30 days'
    GROUP BY user_id
    HAVING COUNT(*) >= 20
      AND (SUM(CASE WHEN opened_at IS NOT NULL THEN 1 ELSE 0 END)::float / COUNT(*)) < 0.05
  `);
  await applyPause(lowOpenRate, 14, 'open_rate_under_5pct_in_30d', env);
  paused += lowOpenRate.length;

  // Rule 3: inactive 30 days
  const inactive30 = await env.DB.query(`
    SELECT u.id AS user_id
    FROM users u
    WHERE u.deleted_at IS NULL
      AND u.last_signin_at < now() - interval '30 days'
      AND NOT EXISTS (
        SELECT 1 FROM exercise_attempts ea
        WHERE ea.user_id = u.id AND ea.completed_at > now() - interval '30 days'
      )
  `);
  await applyPause(inactive30, 30, 'inactive_30_days', env);
  paused += inactive30.length;

  track('notif.circuit_breaker_paused', { count: paused });
  return { paused };
}

async function applyPause(users: any[], days: number, reason: string, env: Env) {
  for (const { user_id } of users) {
    const pausedUntil = new Date(Date.now() + days * 24 * 3600 * 1000);
    await env.DB.execute(`
      UPDATE user_notification_preferences
      SET paused_until = GREATEST(paused_until, $1), pause_reason = $2
      WHERE user_id = $3
    `, [pausedUntil, reason, user_id]);
    await emitEvent('user.notifications_paused', { user_id, reason, paused_until: pausedUntil });
  }
}

// apps/workers/inngest-functions/unpause-on-engagement.ts
export const unpauseOnEngagementFn = inngest.createFunction(
  { id: 'unpause-on-engagement' },
  [
    { event: 'user.signed_in' },
    { event: 'exercise.completed' },
    { event: 'notification.opened' },
  ],
  async ({ event }) => {
    const { user_id } = event.data;
    const result = await db.query(`
      UPDATE user_notification_preferences
      SET paused_until = NULL, pause_reason = NULL
      WHERE user_id = $1 AND paused_until > now()
      RETURNING user_id
    `, [user_id]);

    if (result.length > 0) {
      track('notif.circuit_breaker_unpaused', { user_id });
      await emitEvent('user.notifications_unpaused', { user_id });
    }
  }
);
```

### Cron trigger

```toml
[triggers]
crons = ["0 4 * * *"]  # Diario 04:00 UTC
```

### Integration con US-061

```typescript
// En sendFcmNotification, before FCM:
const prefs = await db.query(`
  SELECT paused_until FROM user_notification_preferences WHERE user_id = $1
`, [job.user_id]);
if (prefs[0]?.paused_until && prefs[0].paused_until > new Date()
    && job.category !== 'transactional') {
  track('notif.suppressed_paused', { user_id: job.user_id });
  return { delivered: 0, failed: 0, suppressed: true };
}
```

### Integration points

- US-061 (sender respeta pause).
- US-067 (transactional bypass).
- Auth + exercise systems (emit engagement events).

### Notas técnicas

- Pause es **soft** — transactional siempre bypass.
- Unpause es **immediate** al engagement (no wait for cron).
- `GREATEST(paused_until, ...)` extiende pause si nueva regla
  dispara antes de que actual termine.

## Definition of Done

- [ ] Cron + trigger.
- [ ] 3 rules implementadas.
- [ ] Inngest unpause on engagement.
- [ ] US-061 respeta paused_until.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Cierra EPIC-06. Protege reputation del sender + UX del user.*

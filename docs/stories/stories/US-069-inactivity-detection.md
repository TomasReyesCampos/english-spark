# US-069: Inactivity detection (D3/D7/D14)

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 3
**Story points:** 3
**Persona:** Estudiante inactivo (re-engagement)
**Owner:** —

---

## Contexto

Re-engagement de users inactivos con 3 niveles de intensidad:
- **D3** (3 días sin actividad): suave, motivador.
- **D7**: personal, mencionar progreso perdido.
- **D14**: último intento con incentivo (20 Sparks bonus).

Después de D14 sin retorno: no más notifs (user churned).

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§4.5.

## Scope

### In

- Cron Worker `apps/workers/crons/inactivity-detector.ts`:
  - Trigger: diario 13:00 UTC (mid-afternoon en LatAm).
  - Para cada threshold (D3, D7, D14):
    - Query users con:
      - No `notification.opened` desde X días (no exercise tampoco).
      - `reengagement_enabled = true`.
      - No recibió notif `inactivity_dN` aún para este threshold.
      - No paid plan active (no spam paying users).
  - sendFcmNotification con variant correspondiente.
  - Para D14: trigger event `sparks.award_bonus_due` (20 Sparks
    cuando user vuelve — handled by motivation).
  - Telemetry: `notif.inactivity_dN_sent`.
- Anti-loop: tracking via `notifications_log` previene re-send
  (próxima inactividad reinicia clock desde último opened).

### Out

- WhatsApp para D30+ (post-MVP).
- A/B test de timings (post-MVP).

## Acceptance criteria

- **Given** user con último opened/exercise hace 3 días, **When**
  cron corre, **Then** envía notif inactivity_d3 con variant
  motivador.
- **Given** user D7 sin actividad y con
  recent_improvement detectado en tracker_metric, **When** procesa,
  **Then** variant `with_progress.01` mencionando que mejoraba.
- **Given** user D14 sin actividad, **When** procesa, **Then**
  envía variant con "20 Sparks bonus" + flag para credit al
  reabrir.
- **Given** user ya recibió inactivity_d7 hace 4 días, **When** cron
  procesa hoy (todavía sin actividad), **Then** NO re-envía d7
  (ya enviado), pero SÍ envía d14 si pasaron 14 días desde último
  activity.
- **Given** user con `reengagement_enabled = false`, **When** cron,
  **Then** skip todos los thresholds.
- **Given** user con plan paid active, **When** cron, **Then** skip
  (no spam paying users con re-engagement).
- **Given** user vuelve a abrir app entre D3 y D7, **When** abre,
  **Then** clock se resetea — próxima inactividad detecta D3 desde
  esa última apertura.
- **Given** user D14 abre app, **When** trigger
  `sparks.award_bonus_due`, **Then** motivation system acredita 20
  Sparks via US-050.

## Developer details

### Owning service

`apps/workers/crons/inactivity-detector.ts`.

### Dependencies

- US-059/060/061.
- US-050 (award bonus).

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §4.5.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §5.

### Implementación esperada

```typescript
// apps/workers/crons/inactivity-detector.ts
const THRESHOLDS = [
  { days: 3,  notif_id: 'inactivity_d3' },
  { days: 7,  notif_id: 'inactivity_d7' },
  { days: 14, notif_id: 'inactivity_d14' },
];

export async function runInactivityDetectorCron(env: Env) {
  let totalSent = 0;

  for (const threshold of THRESHOLDS) {
    const eligible = await env.DB.query(`
      WITH last_activity AS (
        SELECT u.id AS user_id,
          GREATEST(
            COALESCE(MAX(ea.completed_at), '1970-01-01'),
            COALESCE(MAX(nl.opened_at), '1970-01-01'),
            u.signup_at
          ) AS last_active
        FROM users u
        LEFT JOIN exercise_attempts ea ON ea.user_id = u.id
        LEFT JOIN notifications_log nl ON nl.user_id = u.id
        WHERE u.deleted_at IS NULL
        GROUP BY u.id
      )
      SELECT
        la.user_id, u.display_name,
        EXTRACT(DAY FROM now() - la.last_active) AS days_inactive
      FROM last_activity la
      JOIN users u ON u.id = la.user_id
      JOIN user_notification_preferences np ON np.user_id = la.user_id
      WHERE EXTRACT(DAY FROM now() - la.last_active)
            BETWEEN $1 AND $1 + 0.99
        AND np.push_enabled = true AND np.reengagement_enabled = true
        AND NOT EXISTS (
          SELECT 1 FROM notifications_log nl
          WHERE nl.user_id = la.user_id
            AND nl.notification_id = $2
            AND nl.sent_at > la.last_active
        )
        AND NOT EXISTS (
          SELECT 1 FROM user_subscriptions us
          WHERE us.user_id = la.user_id AND us.status = 'active'
        )
        AND NOT EXISTS (
          SELECT 1 FROM user_restrictions ur
          WHERE ur.user_id = la.user_id AND ur.restriction_type IN ('suspended', 'banned')
        )
    `, [threshold.days, threshold.notif_id]);

    for (const user of eligible) {
      const variant = threshold.notif_id === 'inactivity_d7'
        ? await pickD7Variant(user.user_id, env)
        : 'default';

      const copy = await fetchCopy(threshold.notif_id, variant, env);
      const vars = { name: user.display_name?.split(' ')[0] ?? null };

      await sendFcmNotification({
        user_id: user.user_id,
        notification_id: threshold.notif_id,
        category: 'reengagement',
        title: renderTemplate(copy.title_template, vars),
        body: renderTemplate(copy.body_template, vars),
        copy_id: copy.copy_id,
        data: { deeplink: 'englishspark://home?reason=comeback' },
      }, env);

      if (threshold.days === 14) {
        // Flag for bonus when user returns (motivation system reads this)
        await env.DB.execute(`
          INSERT INTO pending_bonuses (user_id, reason, amount, expires_at)
          VALUES ($1, 'comeback_d14', 20, now() + interval '7 days')
          ON CONFLICT (user_id, reason) DO NOTHING
        `, [user.user_id]);
      }

      totalSent++;
      track(`notif.inactivity_d${threshold.days}_sent`);
    }
  }

  return { sent: totalSent };
}

async function pickD7Variant(userId: string, env: Env): Promise<string> {
  const hasProgress = await env.DB.query(`
    SELECT 1 FROM user_subskill_mastery
    WHERE user_id = $1 AND updated_at > now() - interval '21 days'
      AND current_score > 60
    LIMIT 1
  `, [userId]);
  return hasProgress.length > 0 ? 'with_progress' : 'no_progress';
}
```

### Schema extension

```sql
CREATE TABLE pending_bonuses (
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  reason      TEXT NOT NULL,
  amount      INT NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL,
  claimed_at  TIMESTAMPTZ,
  PRIMARY KEY (user_id, reason)
);
```

### Integration points

- US-050 (motivation reads pending_bonuses, awards on app open).
- US-059 notifications_log.
- US-061 sender.

### Notas técnicas

- "Reset clock" semantics: anti-loop logic usa
  `nl.sent_at > la.last_active`. Si user abre app (genera opened),
  `last_active` se actualiza → próxima notif sería para next
  threshold después de nueva inactivity.
- Cron 13:00 UTC = 07:00-09:00 LatAm (mid-morning). Buen timing
  para que llegue mientras user revisa phone.

## Definition of Done

- [ ] Cron + trigger.
- [ ] 3 thresholds (D3/D7/D14) operativos.
- [ ] D7 variant logic with/without progress.
- [ ] D14 pending bonus persisted.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-059/060/061, US-050.*

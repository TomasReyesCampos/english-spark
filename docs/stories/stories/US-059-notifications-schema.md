# US-059: Schema notifications (tokens + preferences + log + scheduled)

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 2
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Schema base del sistema de notifications:
- `user_fcm_tokens`: tokens por device, manejo de invalidation.
- `user_notification_preferences`: opt-out granular, hora
  preferida, timezone, circuit breaker.
- `notifications_log`: historial de envíos (delivery + opens) con
  retention 6 meses.
- `notifications_scheduled`: contenido pre-generado por batch
  nocturno, picked up por cron horario.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§5.

## Scope

### In

- Migration `0XXX_notifications_schema.sql`:
  - `user_fcm_tokens` table (de spec §5.1).
  - `user_notification_preferences` table (§5.2).
  - `notifications_log` table (§5.3).
  - `notifications_scheduled` table (§5.4).
  - `notifications_copy_bank` table (de
    `push-notifications-copy-bank.md` §13.4) para templates de
    fallback.
  - Índices definidos en spec.
- Seed inicial de `notifications_copy_bank` con los templates
  hardcoded del copy bank (variants de daily_reminder,
  streak_at_risk, welcome series, etc.).
- Trigger `updated_at` automático.
- Función helper `get_user_in_timezone_now(tz)` para queries
  timezone-aware.
- Migration down + idempotency tests.

### Out

- Endpoints CRUD (US-060).
- FCM sender (US-061).
- Crons (US-062+).

## Acceptance criteria

- **Given** migration aplicada, **When** se ejecuta, **Then** las
  5 tablas existen con columns + indexes + constraints de spec.
- **Given** un user nuevo, **When** registra primer token,
  **Then** row se inserta sin error.
- **Given** mismo `token` se intenta insertar 2x, **When** segunda,
  **Then** unique constraint violation o ON CONFLICT depending on
  strategy.
- **Given** user borrado (CASCADE), **When** elimina, **Then**
  tokens + preferences + log + scheduled se eliminan.
- **Given** `notifications_log.error` NOT NULL, **When** se intenta
  setear `delivered_at` también, **Then** CHECK constraint rechaza
  (XOR semántico).
- **Given** `notifications_scheduled.status = 'sent'`, **When**
  cron intenta procesar de nuevo, **Then** WHERE status='pending'
  excluye (no double send).
- **Given** seed inicial aplicado, **When**
  SELECT FROM notifications_copy_bank, **Then** ≥ 30 rows
  (templates hardcoded del copy bank).
- **Given** migration idempotente, **When** corre 2x, **Then** no
  error (IF NOT EXISTS).

## Developer details

### Owning service

Database / migrations.

### Dependencies

- US-002: schema users (FK target).

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §5 — schemas autoritativos.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §13.4 — schema notifications_copy_bank.

### Schema esperado (resumen, ver specs para detalle completo)

```sql
-- migrations/0XXX_notifications_schema.sql

CREATE TABLE IF NOT EXISTS user_fcm_tokens (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token           TEXT NOT NULL UNIQUE,
  platform        TEXT NOT NULL CHECK (platform IN ('ios', 'android')),
  device_id       TEXT,
  app_version     TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_used_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_active       BOOLEAN NOT NULL DEFAULT true,
  last_error      TEXT,
  last_error_at   TIMESTAMPTZ
);

CREATE INDEX idx_fcm_tokens_user_active
  ON user_fcm_tokens(user_id) WHERE is_active = true;

CREATE TABLE IF NOT EXISTS user_notification_preferences (
  user_id                 UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  push_enabled            BOOLEAN NOT NULL DEFAULT true,
  reminders_enabled       BOOLEAN NOT NULL DEFAULT true,
  achievements_enabled    BOOLEAN NOT NULL DEFAULT true,
  onboarding_enabled      BOOLEAN NOT NULL DEFAULT true,
  reengagement_enabled    BOOLEAN NOT NULL DEFAULT true,
  preferred_reminder_hour INT NOT NULL DEFAULT 19
                          CHECK (preferred_reminder_hour BETWEEN 0 AND 23),
  timezone                TEXT NOT NULL DEFAULT 'America/Mexico_City',
  paused_until            TIMESTAMPTZ,
  pause_reason            TEXT,
  updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS notifications_log (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  notification_id TEXT NOT NULL,
  category        TEXT NOT NULL,
  title           TEXT NOT NULL,
  body            TEXT NOT NULL,
  data            JSONB NOT NULL DEFAULT '{}',
  copy_id         TEXT,
  fcm_message_id  TEXT,
  sent_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  delivered_at    TIMESTAMPTZ,
  opened_at       TIMESTAMPTZ,
  error           TEXT,
  CONSTRAINT log_status_xor CHECK (
    error IS NULL OR (delivered_at IS NULL AND opened_at IS NULL)
  )
);

CREATE INDEX idx_notif_log_user_date ON notifications_log(user_id, sent_at DESC);
CREATE INDEX idx_notif_log_recent_opens
  ON notifications_log(user_id, opened_at DESC)
  WHERE opened_at IS NOT NULL;

CREATE TABLE IF NOT EXISTS notifications_scheduled (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  notification_id TEXT NOT NULL,
  scheduled_for   TIMESTAMPTZ NOT NULL,
  title           TEXT NOT NULL,
  body            TEXT NOT NULL,
  data            JSONB NOT NULL DEFAULT '{}',
  copy_id         TEXT,
  status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                    'pending', 'sent', 'cancelled', 'failed'
                  )),
  generated_by    TEXT NOT NULL DEFAULT 'system',
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scheduled_pending ON notifications_scheduled(scheduled_for)
  WHERE status = 'pending';

CREATE TABLE IF NOT EXISTS notifications_copy_bank (
  copy_id           TEXT PRIMARY KEY,
  notification_id   TEXT NOT NULL,
  variant           TEXT NOT NULL,
  locale            TEXT NOT NULL DEFAULT 'es-MX',
  title_template    TEXT NOT NULL,
  body_template     TEXT NOT NULL,
  body_expanded     TEXT,
  required_vars     TEXT[] NOT NULL DEFAULT '{}',
  optional_vars     TEXT[] NOT NULL DEFAULT '{}',
  is_fallback       BOOLEAN NOT NULL DEFAULT false,
  is_active         BOOLEAN NOT NULL DEFAULT true,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_copy_bank_lookup
  ON notifications_copy_bank(notification_id, variant, locale)
  WHERE is_active = true;
```

### Seed inicial de copy_bank

Insertar al menos:
- 5 variants de `daily_reminder` (de copy bank §2.2-§2.7).
- 3 variants de `streak_at_risk` (§3.2).
- 4 variants de welcome series (§4.1-§4.4).
- 3 variants de inactivity (§5).
- 80 entries de achievements (§6.3, todos los logros).
- 4 transactional (§8).

Total seed: ~100 rows.

### Integration points

- US-060 (consumer del schema).
- US-061 (FCM sender lee tokens activos).
- US-062+ (crons).

### Notas técnicas

- `notifications_scheduled.status = 'pending'` index parcial es
  crítico para performance del cron dispatcher (US-062).
- `notifications_log` retention 6 meses via cron mensual (cubierto
  en US-070 cleanup).
- `copy_bank` permite hot-swap de templates sin deploy (cambio
  is_active).

## Definition of Done

- [ ] Migration aplicada en dev.
- [ ] Down migration funcional.
- [ ] 5 tablas + indexes + constraints verificados.
- [ ] Seed inicial copy_bank con 100+ rows.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests SQL: constraints, CASCADE, idempotency.
- [ ] Validation contra spec
  `notifications-system.md` §5 +
  `push-notifications-copy-bank.md` §13.4.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para todas las stories de EPIC-06.*

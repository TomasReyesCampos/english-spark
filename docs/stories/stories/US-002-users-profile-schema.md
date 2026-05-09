# US-002: Schema `users` + `student_profiles`

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Toda story de auth y onboarding necesita persistir datos en
Postgres (Supabase). Esta story crea las **2 tablas core** que
sirven de base para todo lo demás:

- `users`: cuenta minimal con Firebase UID + email + signup
  metadata.
- `student_profiles`: perfil completo del aprendiz (1-to-1 con
  users) con todos los campos de declarativo + observado + medido.

Schemas autoritativos vienen de:
- `architecture/authentication-system.md` §5 (`users`).
- `product/student-profile-and-assessment.md` §3.2 (`student_profiles`).

Esta story es **bloqueante** para US-003 a US-009 (todas necesitan
persistir).

## Scope

### In

- Migration SQL inicial (`migrations/0001_users_profiles.sql`):
  - `users` table con Firebase UID, email, sign-up metadata.
  - `student_profiles` table con todos los campos definidos en spec.
  - Foreign key `student_profiles.user_id → users.id ON DELETE CASCADE`.
  - Índices definidos en spec (country, trial_active,
    pending_assessment).
- Triggers de `updated_at` automático.
- Seed de configuración: timezone default, locale default, trial
  duration default.
- Tests de migration: up + down + idempotency.
- Documentación del schema en `docs/db/schema.md` (a crear como
  parte de esta story).

### Out

- Endpoints de CRUD sobre estas tablas (van en US-008).
- Relaciones a otras tablas (sparks_balances, roadmaps, etc.) —
  son stories propias.
- Implementación del trigger
  `student_profile auto-creado on user.signed_up` (eso es lógica
  de negocio, va en US-008).

## Acceptance criteria

- **Given** una BD Postgres limpia, **When** se ejecuta
  `migrations/0001_users_profiles.sql`, **Then** las tablas `users`
  y `student_profiles` existen con todos los columnas y constraints
  del spec.
- **Given** la migration aplicada, **When** se inserta un user con
  email único, **Then** se crea sin error.
- **Given** un user existente, **When** se intenta crear otro user
  con el mismo email, **Then** retorna constraint violation
  `unique_email`.
- **Given** un user con `student_profile` asociado, **When** se
  borra el user, **Then** el `student_profile` se borra
  automáticamente (CASCADE).
- **Given** un `student_profile` con `target_english_variant =
  'klingon'`, **When** se intenta insert, **Then** retorna check
  constraint violation (solo permite 'american', 'british', 'neutral').
- **Given** la migration aplicada, **When** se corre
  `migrations/0001_users_profiles_down.sql`, **Then** las 2 tablas
  desaparecen sin errors.
- **Given** la migration es idempotent, **When** se aplica 2 veces
  consecutivas, **Then** la segunda no produce error (uses `IF
  NOT EXISTS`).

## Developer details

### Owning service

Database / migrations.

### Dependencies

- Cuenta Supabase con proyecto `english-spark` configurado.
- Postgres ≥ 15 (para `gen_random_uuid()` sin extension explícita,
  aunque agregamos `pgcrypto` por compat).
- Tooling de migrations: pendiente decidir entre Drizzle, Kysely o
  raw SQL. Para esta story: **raw SQL** + un wrapper simple
  (pendiente decidir tooling completo en story separada).

### Specs referenciados

- [`architecture/authentication-system.md`](../../architecture/authentication-system.md)
  §5 — `users` schema.
- [`product/student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §3.2 — `student_profiles` schema autoritativo.

### Schema esperado (resumen)

```sql
-- migrations/0001_users_profiles.sql

CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE IF NOT EXISTS users (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  firebase_uid    TEXT NOT NULL UNIQUE,
  email           TEXT NOT NULL UNIQUE,
  display_name    TEXT,
  is_anonymous    BOOLEAN NOT NULL DEFAULT false,
  signup_provider TEXT NOT NULL,           -- 'google', 'apple', 'email', 'anonymous'
  signup_country  TEXT,                     -- ISO 3166-1 alpha-2
  signup_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_signin_at  TIMESTAMPTZ,
  deleted_at      TIMESTAMPTZ,              -- soft delete
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_firebase_uid ON users(firebase_uid)
  WHERE deleted_at IS NULL;

CREATE TABLE IF NOT EXISTS student_profiles (
  user_id                  UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  -- (todos los campos de student-profile-and-assessment.md §3.2)
  -- ... ver spec autoritativo ...
);

CREATE INDEX idx_profiles_country ON student_profiles(country);
CREATE INDEX idx_profiles_trial_active ON student_profiles(trial_ends_at)
  WHERE trial_status = 'active';
CREATE INDEX idx_profiles_pending_assessment ON student_profiles(trial_ends_at)
  WHERE trial_status = 'active' AND assessment_completed_at IS NULL;

-- Trigger updated_at
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER profiles_updated_at BEFORE UPDATE ON student_profiles
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

(Schema completo de `student_profiles` en spec referenciado §3.2.)

### Integration points

- Supabase project (run migrations vía Supabase CLI o connection
  directa).
- Cloudflare Workers (eventually conectarán a la BD via Supabase
  pooler).

### Notas técnicas

- `firebase_uid` es UNIQUE pero NO PRIMARY KEY: usamos UUID propio
  para evitar acoplamiento con Firebase (si migramos auth
  provider).
- Soft delete (`deleted_at`) en users: 30 días antes de hard delete
  por cron (regla del producto, ver `reglas.md`).
- `student_profile` se crea automáticamente cuando user se
  registra: lógica en handler de `user.signed_up` event (US-008,
  no en esta story).
- `trial_started_at` default `now()` y `trial_ends_at` default
  `now() + interval '7 days'` (matchea ADR-006).

## Definition of Done

- [ ] Migration `0001_users_profiles.sql` creada y aplicada en
  dev environment.
- [ ] Migration down (`0001_users_profiles_down.sql`) creada y
  testeada.
- [ ] Tests unit de schema (constraints, triggers) pasan.
- [ ] Idempotency verificada (re-run no rompe).
- [ ] Documentación `docs/db/schema.md` creada con ERD y
  descripción de tablas.
- [ ] Validation contra spec
  `student-profile-and-assessment.md` §3.2.
- [ ] Migration revisada (rollback documented).
- [ ] PR aprobada y mergeada.

---

*Bloqueante de US-003 a US-009.*

# US-072: Schema roadmaps + roadmap_blocks + block_completions

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 2
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Schema central del roadmap engine:
- `roadmaps`: 1 row por user con plan activo (initial o
  definitivo).
- `roadmap_levels`: agrupación temática (4-6 levels por roadmap).
- `roadmap_blocks`: bloques individuales en orden.
- `block_completions`: tracking per-user de bloques (status,
  scores, attempts).
- `learning_blocks`: catálogo global de bloques (175 MVP de
  track-*-blocks.md).

US-021 ya creó roadmaps básicas en EPIC-01; esta story consolida
+ expande para todas las features post-MVP del epic.

Backend en
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §7.

## Scope

### In

- Migration `0XXX_roadmap_schema.sql`:
  - `learning_blocks` (catálogo): id, title, description, cefr,
    track, target_subskills[], prerequisites[],
    estimated_minutes, mastery_criteria, asset_sequence.
  - `roadmaps`: id, user_id, roadmap_type (initial/definitive),
    active_track, cefr_at_generation, ai_summary,
    estimated_completion_weeks, is_active, archived_at,
    archived_reason, generated_by.
  - `roadmap_levels`: roadmap_id, level_index, name, ai_reasoning.
  - `roadmap_blocks`: roadmap_id, level_id, block_id, order,
    personalization_reason.
  - `block_completions`: user_id, block_id, status (pending /
    in_progress / completed / skipped_test_out / abandoned),
    started_at, completed_at, exercise_attempts_count,
    score_avg, mastery_evidence.
- Índices críticos:
  - `idx_roadmaps_user_active`: `WHERE is_active = true`.
  - `idx_block_completions_user`.
- Seed inicial de `learning_blocks` con los 175 bloques MVP
  (Job Ready 74 + Travel 39 + Daily Conversation 62).
- Migration down.

### Out

- Endpoints (US-073).
- Assessment session schema (US-080).

## Acceptance criteria

- **Given** migration aplicada, **When** ejecutada, **Then** las
  5 tablas existen con FK + indexes.
- **Given** seed aplicado, **When**
  `SELECT COUNT(*) FROM learning_blocks`, **Then** retorna 175
  rows.
- **Given** user con roadmap activo, **When** se crea segundo
  roadmap activo para mismo user, **Then** se permite pero
  primer roadmap debe ser marcado `is_active = false` antes
  (constraint via partial unique index).
- **Given** block_id de roadmap_blocks no existe en
  learning_blocks, **When** insert, **Then** FK constraint
  violation.
- **Given** user delete CASCADE, **When** elimina, **Then**
  roadmaps, levels, blocks, completions cascade.
- **Given** `block_completions.status = 'completed'`, **When**
  intentar updates de `score_avg`, **Then** permitido (status
  doesn't lock score).
- **Given** seed idempotente, **When** corre 2x, **Then** sin
  error (uses ON CONFLICT).
- **Given** down migration, **When** ejecuta, **Then** 5 tablas se
  borran limpio.

## Developer details

### Owning service

Database / migrations.

### Dependencies

- US-002: users schema (FK target).
- 3 docs de tracks (seed source).

### Specs referenciados

- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §7
  — schemas.
- [`track-job-ready-blocks.md`](../../product/track-job-ready-blocks.md),
  [`track-travel-confident-blocks.md`](../../product/track-travel-confident-blocks.md),
  [`track-daily-conversation-blocks.md`](../../product/track-daily-conversation-blocks.md).

### Schema esperado (resumen)

```sql
-- migrations/0XXX_roadmap_schema.sql

CREATE TABLE IF NOT EXISTS learning_blocks (
  id              TEXT PRIMARY KEY,
  track           TEXT NOT NULL CHECK (track IN (
                    'job_ready', 'travel_confident', 'daily_conversation'
                  )),
  cefr            TEXT NOT NULL,
  title           TEXT NOT NULL,
  description     TEXT NOT NULL,
  target_subskills TEXT[] NOT NULL DEFAULT '{}',
  prerequisites   TEXT[] NOT NULL DEFAULT '{}',
  estimated_minutes INT NOT NULL,
  mastery_criteria JSONB NOT NULL DEFAULT '{}',
  asset_sequence  JSONB NOT NULL DEFAULT '[]',
  is_capstone     BOOLEAN NOT NULL DEFAULT false,
  testable_out    BOOLEAN NOT NULL DEFAULT true,
  variant_of      TEXT,                       -- ej: jr_b1_industry_vocab_intro_tech
  approved_for_production BOOLEAN NOT NULL DEFAULT false,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS roadmaps (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  roadmap_type    TEXT NOT NULL CHECK (roadmap_type IN ('initial', 'definitive')),
  active_track    TEXT NOT NULL,
  cefr_at_generation TEXT NOT NULL,
  ai_summary      TEXT,
  estimated_completion_weeks INT,
  is_active       BOOLEAN NOT NULL DEFAULT true,
  archived_at     TIMESTAMPTZ,
  archived_reason TEXT,
  generated_by    TEXT NOT NULL DEFAULT 'ai_haiku',
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_roadmap_user_active_unique
  ON roadmaps(user_id) WHERE is_active = true;

CREATE TABLE IF NOT EXISTS roadmap_levels (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  roadmap_id      UUID NOT NULL REFERENCES roadmaps(id) ON DELETE CASCADE,
  level_index     INT NOT NULL,
  name            TEXT NOT NULL,
  ai_reasoning    TEXT,
  UNIQUE (roadmap_id, level_index)
);

CREATE TABLE IF NOT EXISTS roadmap_blocks (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  roadmap_id      UUID NOT NULL REFERENCES roadmaps(id) ON DELETE CASCADE,
  level_id        UUID NOT NULL REFERENCES roadmap_levels(id) ON DELETE CASCADE,
  block_id        TEXT NOT NULL REFERENCES learning_blocks(id),
  order_in_level  INT NOT NULL,
  personalization_reason TEXT,
  UNIQUE (roadmap_id, block_id)
);

CREATE TABLE IF NOT EXISTS block_completions (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  block_id        TEXT NOT NULL REFERENCES learning_blocks(id),
  status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                    'pending', 'in_progress', 'completed',
                    'skipped_test_out', 'abandoned'
                  )),
  started_at      TIMESTAMPTZ,
  completed_at    TIMESTAMPTZ,
  exercise_attempts_count INT NOT NULL DEFAULT 0,
  score_avg       NUMERIC(5,2),
  mastery_evidence JSONB DEFAULT '{}',
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, block_id)
);

CREATE INDEX idx_block_completions_user ON block_completions(user_id, completed_at DESC);
CREATE INDEX idx_block_completions_in_progress
  ON block_completions(user_id) WHERE status = 'in_progress';
```

### Seed de learning_blocks

```sql
-- Seed loader (script aparte o COPY desde CSV generado)
-- 175 rows de los 3 tracks docs.
INSERT INTO learning_blocks (id, track, cefr, title, description, target_subskills, prerequisites, estimated_minutes, mastery_criteria, asset_sequence, is_capstone) VALUES
('jr_b1_intro_self_professional', 'job_ready', 'B1', 'Presentarte profesionalmente', '...', ARRAY['vocab_business_general', 'flu_speaking_pace'], ARRAY[]::text[], 14, '{"score_min": 70}'::jsonb, '[...]'::jsonb, false),
-- ... 174 more rows
ON CONFLICT (id) DO NOTHING;
```

### Integration points

- US-021 ya crea roadmaps en EPIC-01 — ahora valida contra
  catálogo formal.
- US-073, US-077, US-078, US-080, US-085 (consumers).

### Notas técnicas

- Partial unique index `WHERE is_active = true` permite múltiples
  roadmaps archivados pero solo uno activo por user.
- `asset_sequence` JSONB permite flexibilidad — schema dentro
  matches `content-creation-system.md` §6.1.
- `mastery_criteria` JSONB para flexibilidad.

## Definition of Done

- [ ] Migration aplicada + down funcional.
- [ ] Seed 175 bloques.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests SQL: constraints, indexes, CASCADE.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para todo EPIC-07 + EPIC-03.*

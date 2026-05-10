# US-080: Assessment session start + orchestration 4 partes

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3
**Story points:** 3
**Persona:** Estudiante (Day 7+)
**Owner:** —

---

## Contexto

Implementa el **arranque** del assessment Day 7: crea session,
selecciona ejercicios de las 4 partes (perfil + skills activas +
skills receptivas + aspiraciones), retorna estructura completa al
cliente.

Backend en
[`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
§6 +
[`assessment-content-bank.md`](../../product/assessment-content-bank.md).

## Scope

### In

- Endpoint `POST /assessment/start`:
  - Valida JWT.
  - Body: `{ reason: 'trial_day_7' | 'sparks_runout_early' | 'user_initiated' | 'periodic_reassessment' }`.
  - Validation timing (§13.1 student-profile):
    - Day 1-2: rechaza salvo `sparks_runout_early`.
    - Day 3-4: optional.
    - Day 5-6: soft mandatory.
    - Day 7+: firm mandatory.
  - CEFR sampling (hybrid logic de §13.2):
    - Compare self_perceived_level vs initial_test_results.cefr_estimate.
    - distance 0: usa ese.
    - distance 1: user_choice (frontend pregunta).
    - distance 2+: usa max + show_explanation.
  - Selecciona items del `assessment_content_bank` para 4 partes:
    - Parte 1: 3-5 preguntas de profile deepening.
    - Parte 2: 5 ejercicios activos (reading aloud, roleplay,
      pronunciation, listening, free).
    - Parte 3: 18 items receptivos (vocab + grammar MC).
    - Parte 4: 4 preguntas aspiraciones.
  - Si user completó sporadic questions: skip redundantes en
    Parte 1 (de §7.1.8).
  - Crea `assessment_sessions` con `selected_items` JSONB.
  - Retorna `{ session_id, parts: [...], estimated_duration_minutes: 20 }`.
- Schema `assessment_sessions` + `assessment_attempts`.
- Telemetry: `assessment.started`, `.timing_validation_failed`.

### Out

- Submit exercises (US-081).
- Finalize (US-082).

## Acceptance criteria

- **Given** user Day 7, **When** POST /assessment/start con
  `reason='trial_day_7'`, **Then** retorna session con 4 partes
  + ~25 items.
- **Given** user Day 2 sin Sparks runout, **When** POST, **Then**
  400 `{ error: 'too_early', current_day: 2, earliest: 3 }`.
- **Given** user Day 1 con Sparks runout, **When** POST con
  `reason='sparks_runout_early'`, **Then** se permite (exception).
- **Given** user con self_perceived='B1+' y measured='B1+',
  **When** start, **Then** assessment_cefr='B1+', items sampled
  para ese nivel.
- **Given** difference 1 nivel, **When** start, **Then** retorna
  flag `cefr_choice_required: { options: ['B1+', 'B2'] }` —
  frontend pregunta antes de continuar.
- **Given** user completó sporadic_questions con audio captures,
  **When** start, **Then** Parte 2 reduce items redundantes
  (algoritmo de §7.1.8).
- **Given** ya hay session active del user, **When** POST start,
  **Then** retorna existente (idempotent) o nueva si la previa
  expiró.
- **Given** request sin JWT, **When** llega, **Then** 401.

## Developer details

### Owning service

`apps/workers/api/handlers/assessment-start.ts`.

### Dependencies

- US-072.
- `assessment_content_bank.md` items en seed table.

### Specs referenciados

- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §6.1, §13.1, §13.2.
- [`assessment-content-bank.md`](../../product/assessment-content-bank.md).

### Schema

```sql
CREATE TABLE assessment_sessions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  assessment_cefr TEXT NOT NULL,
  reason          TEXT NOT NULL,
  selected_items  JSONB NOT NULL,
  status          TEXT NOT NULL DEFAULT 'in_progress' CHECK (status IN (
                    'in_progress', 'completed', 'cancelled', 'expired'
                  )),
  started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at    TIMESTAMPTZ,
  expires_at      TIMESTAMPTZ NOT NULL DEFAULT (now() + interval '1 hour')
);

CREATE TABLE assessment_attempts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id      UUID NOT NULL REFERENCES assessment_sessions(id) ON DELETE CASCADE,
  part            INT NOT NULL CHECK (part BETWEEN 1 AND 4),
  exercise_index  INT NOT NULL,
  item_id         TEXT NOT NULL,
  payload         JSONB NOT NULL,
  score           JSONB,
  submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (session_id, part, exercise_index)
);
```

### Implementación esperada

```typescript
// apps/workers/api/handlers/assessment-start.ts
export async function handleAssessmentStart(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const { reason } = await request.json();
  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);
  const profile = await db.getProfile(userId);

  // Timing validation
  const trialDay = Math.floor((Date.now() - new Date(profile.trial_started_at).getTime()) / (24*3600*1000));
  if (trialDay < 3 && reason !== 'sparks_runout_early') {
    return jsonResponse({ error: 'too_early', current_day: trialDay, earliest: 3 }, 400);
  }

  // Existing active session?
  const existing = await db.query(`
    SELECT * FROM assessment_sessions
    WHERE user_id = $1 AND status = 'in_progress' AND expires_at > now()
  `, [userId]);
  if (existing.length > 0) return jsonResponse(existing[0]);

  // Hybrid CEFR sampling
  const cefrSelection = determineCefrForAssessment(profile);
  if (cefrSelection.show_choice) {
    return jsonResponse({
      cefr_choice_required: true,
      options: cefrSelection.options,
    });
  }

  // Item selection
  const items = await selectAssessmentItems(cefrSelection.cefr, profile, env);

  const session = await db.query(`
    INSERT INTO assessment_sessions
      (user_id, assessment_cefr, reason, selected_items)
    VALUES ($1, $2, $3, $4) RETURNING *
  `, [userId, cefrSelection.cefr, reason, JSON.stringify(items)]);

  track('assessment.started', { cefr: cefrSelection.cefr, reason });
  return jsonResponse({
    session_id: session[0].id,
    parts: items,
    estimated_duration_minutes: 20,
  });
}
```

### Integration points

- US-072 (schemas).
- assessment_content_bank seed.
- US-081 (consumer del session_id).

### Notas técnicas

- Session expira en 1 hour: si user pausa >1h, debe reiniciar (de
  spec §11).
- Idempotency via existing active session.

## Definition of Done

- [ ] Endpoint implementado.
- [ ] Schemas migration.
- [ ] Timing + CEFR sampling logic.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] PR aprobada y mergeada.

---

*Depende de US-072. Bloqueante para US-081-083.*

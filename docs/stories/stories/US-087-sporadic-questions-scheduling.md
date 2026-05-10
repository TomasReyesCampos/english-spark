# US-087: Sporadic questions scheduling + presentation

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 4
**Story points:** 3
**Persona:** Estudiante (Day 1-7 pre-assessment)
**Owner:** —

---

## Contexto

Sporadic questions: micro-preguntas (5-30s, max 1-2 por sesión)
durante el pre-assessment phase (Day 1 hasta
assessment_completed_at). Promoted from exploration in 2026-05.

Lifecycle: Day 1 hasta assessment completion. Después: no más
sporadic.

Backend en
[`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
§7.1.

## Scope

### In

- Schema `sporadic_questions` (pool de items) + `sporadic_responses`
  + `sporadic_responses_ml_training` (de §7.1.4 spec).
- Seed inicial con ~58 items distribuidos:
  - 7 self_assessment (listening, reading, speaking, vocab, grammar,
    pronunciation, frustración).
  - 10 micro_audio (frases con phonemes target).
  - 20 vocab MC.
  - 20 grammar correct/wrong.
  - 1 goal validation.
- Endpoint `GET /sporadic/next` que:
  - Verifica user está en pre-assessment phase
    (assessment_completed_at IS NULL).
  - Aplica throttle (max 1-2 per session, 3 per day, 8-12 totales).
  - Skip si user dismissed 3 consecutive recientes (cooldown 24h
    via `sporadic_paused_until`).
  - Selecciona item del pool weighted por (no usado por user, no
    redundant).
  - Retorna `{ question_id, category, content, estimated_seconds }`
    o null si nothing to show.
- Endpoint `POST /sporadic/:question_id/respond`:
  - Body: response_data + skipped flag.
  - Detecta fake patterns (§7.1.5) y persists con flag
    `flagged_likely_fake`.
  - Persist en `sporadic_responses`.
  - Si fake: also persist row en
    `sporadic_responses_ml_training`.
- Telemetry: `sporadic.question_shown`, `.responded`,
  `.skipped`, `.fake_detected`.

### Out

- Roadmap reorder per responses (US-088).
- UI flow específico (responsibility del mobile app integration).

## Acceptance criteria

- **Given** user Day 2 sin assessment, **When** GET /sporadic/next,
  **Then** retorna item válido del pool con throttling
  considerado.
- **Given** user post-assessment, **When** GET /sporadic/next,
  **Then** retorna null + reason='post_assessment_phase'.
- **Given** user con 3 skips consecutivos, **When** próximo GET,
  **Then** retorna null + reason='cooldown_active', y
  `sporadic_paused_until = now() + 24h`.
- **Given** user con 12 sporadic responses ya, **When** GET,
  **Then** retorna null + reason='total_cap_reached'.
- **Given** POST respond con audio_silence > 70% del audio,
  **When** procesa, **Then** persist con
  `flagged_likely_fake=true` + log para ML training.
- **Given** POST respond con tiempo respuesta < 2s para 5
  opciones, **When** procesa, **Then** flagged_likely_fake.
- **Given** POST respond multiple choice, **When** procesa,
  **Then** persist con response_data + score si applicable.
- **Given** items pool agotado para este user, **When** GET, **Then**
  retorna null + reason='pool_exhausted'.

## Developer details

### Owning service

`apps/workers/api/handlers/sporadic-*.ts`.

### Dependencies

- US-072 schemas.
- assessment_content_bank seed.

### Specs referenciados

- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §7.1 — spec autoritativo.
- [`docs/explorations/sporadic-questions.md`](../../explorations/sporadic-questions.md)
  — exploration que se promovió.

### Schema

(Ya definido en spec §7.1.4 — replicar.)

```sql
CREATE TABLE sporadic_questions (
  id              TEXT PRIMARY KEY,
  category        TEXT NOT NULL,
  cefr_level      TEXT,
  goal_relevance  TEXT[] DEFAULT '{}',
  target_dimension TEXT,
  content         JSONB NOT NULL,
  estimated_seconds INT NOT NULL,
  weight_in_calibration NUMERIC(3,2) DEFAULT 0.5,
  approved        BOOLEAN NOT NULL DEFAULT false,
  archived        BOOLEAN NOT NULL DEFAULT false,
  use_count       INT NOT NULL DEFAULT 0,
  skip_count      INT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sporadic_responses (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  question_id     TEXT NOT NULL REFERENCES sporadic_questions(id),
  asked_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  responded_at    TIMESTAMPTZ,
  skipped         BOOLEAN NOT NULL DEFAULT false,
  response_data   JSONB,
  audio_atomic_id TEXT REFERENCES media_atomics(id),
  scoring_result  JSONB,
  session_id      TEXT,
  trigger_context TEXT,
  flagged_likely_fake BOOLEAN NOT NULL DEFAULT false,
  flag_reason     TEXT
);

CREATE TABLE sporadic_responses_ml_training (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  response_id     UUID NOT NULL REFERENCES sporadic_responses(id),
  flag_signals    JSONB NOT NULL,
  user_state_snapshot JSONB,
  logged_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Implementación esperada

```typescript
const MAX_DAILY = 3;
const MAX_TOTAL = 12;

export async function handleSporadicNext(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);
  const profile = await db.getProfile(userId);

  if (profile.assessment_completed_at) {
    return jsonResponse({ next: null, reason: 'post_assessment_phase' });
  }

  if (profile.sporadic_paused_until && new Date(profile.sporadic_paused_until) > new Date()) {
    return jsonResponse({ next: null, reason: 'cooldown_active' });
  }

  // Throttle checks
  const total = await db.query(`SELECT COUNT(*) AS count FROM sporadic_responses WHERE user_id = $1`, [userId]);
  if (Number(total[0].count) >= MAX_TOTAL) {
    return jsonResponse({ next: null, reason: 'total_cap_reached' });
  }

  const todayCount = await db.query(`
    SELECT COUNT(*) AS count FROM sporadic_responses
    WHERE user_id = $1 AND asked_at > date_trunc('day', now())
  `, [userId]);
  if (Number(todayCount[0].count) >= MAX_DAILY) {
    return jsonResponse({ next: null, reason: 'daily_cap_reached' });
  }

  // Pick next from pool (weighted, exclude already used)
  const item = await db.query(`
    SELECT * FROM sporadic_questions
    WHERE approved = true AND archived = false
      AND NOT EXISTS (
        SELECT 1 FROM sporadic_responses sr
        WHERE sr.user_id = $1 AND sr.question_id = sporadic_questions.id
      )
    ORDER BY weight_in_calibration DESC, random() LIMIT 1
  `, [userId]);

  if (item.length === 0) {
    return jsonResponse({ next: null, reason: 'pool_exhausted' });
  }

  track('sporadic.question_shown', { category: item[0].category });

  // Reserve question (insert pending response with no responded_at)
  await db.execute(`
    INSERT INTO sporadic_responses (user_id, question_id, session_id)
    VALUES ($1, $2, $3)
  `, [userId, item[0].id, request.headers.get('X-Session-Id')]);

  return jsonResponse({
    next: {
      question_id: item[0].id,
      category: item[0].category,
      content: item[0].content,
      estimated_seconds: item[0].estimated_seconds,
    },
  });
}

export async function handleSporadicRespond(
  request: AuthedRequest, env: Env, questionId: string
) {
  // ... validation ...
  const { response_data, skipped } = await request.json();

  const flags = detectFakePatterns(response_data, skipped);

  await db.execute(`
    UPDATE sporadic_responses
    SET responded_at = now(), skipped = $1, response_data = $2,
        flagged_likely_fake = $3, flag_reason = $4
    WHERE user_id = $5 AND question_id = $6
  `, [skipped, JSON.stringify(response_data), flags.fake, flags.reason ?? null, userId, questionId]);

  if (flags.fake) {
    await db.execute(`
      INSERT INTO sporadic_responses_ml_training (response_id, flag_signals)
      VALUES ((SELECT id FROM sporadic_responses WHERE user_id = $1 AND question_id = $2), $3)
    `, [userId, questionId, JSON.stringify(flags.signals)]);
  }

  // ... throttle pause logic (3 skips consecutivos → cooldown) ...

  track(skipped ? 'sporadic.skipped' : 'sporadic.responded');
  if (flags.fake) track('sporadic.fake_detected');

  return jsonResponse({ ok: true });
}
```

### Integration points

- US-088 (consumer de responses).
- Mobile app (UI integration in exercise flow).

### Notas técnicas

- Pool de 58 items + 12 max per user = ~80% probabilidad de
  cobertura completa antes de assessment Day 7.
- Fake detection heurísticas de spec §7.1.5.

## Definition of Done

- [ ] Schema + seed.
- [ ] 2 endpoints.
- [ ] Throttle logic.
- [ ] Fake detection.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-088.*

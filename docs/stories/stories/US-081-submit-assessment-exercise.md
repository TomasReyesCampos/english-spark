# US-081: Submit assessment exercise + per-exercise scoring

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3
**Story points:** 3
**Persona:** Estudiante / Admin
**Owner:** —

---

## Contexto

Cliente envía cada ejercicio del assessment a medida que lo
completa. Backend invoca scoring vía AI Gateway tasks
correspondientes y persiste `assessment_attempts`. Finalize (US-082)
agrega para resultado total.

Backend en
[`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
§10.8.

## Scope

### In

- Endpoint `POST /assessment/:session_id/submit-exercise`:
  - Valida JWT + ownership de session.
  - Body: `{ part, exercise_index, payload }` donde payload
    matchea el shape del exercise type.
  - Verifica session `status='in_progress'` y no expirada.
  - Routes scoring por exercise type:
    - Pronunciation drill → AI Gateway `score_pronunciation`
      (US-033).
    - Free response → `transcribe_user_audio` + `score_fluency`
      (US-034/035).
    - Roleplay → `score_fluency` extended.
    - Listening MC → match correct answer locally.
    - Vocab MC → match locally.
    - Grammar → `detect_grammar_errors` (US-040).
  - Persiste row en `assessment_attempts` con score JSONB.
  - **Assessment NO cobra Sparks** (decisión cerrada, free).
  - Retorna `{ score_partial, next_exercise_index, is_last_of_part }`.
- Telemetry: `assessment.exercise_submitted`,
  `assessment.exercise_score_low` (si score bajo).

### Out

- Finalize (US-082).
- UI flow del assessment (incluida en post-assessment flow doc).

## Acceptance criteria

- **Given** session active y exercise pronunciation drill, **When**
  POST submit con audio_url payload, **Then** invoca
  score_pronunciation, persiste attempt, retorna score parcial.
- **Given** session expirada, **When** submit, **Then** 410 Gone
  con `{ error: 'session_expired' }`.
- **Given** session de otro user, **When** intent submit, **Then**
  403.
- **Given** mismo (part, exercise_index) submit 2x, **When** segunda,
  **Then** UPSERT actualiza el attempt (allow re-submit).
- **Given** exercise free_response con audio, **When** procesa,
  **Then** transcribe primero, después fluency score, persist
  con ambos resultados.
- **Given** AI Gateway timeout, **When** scoring falla, **Then**
  persist attempt con `score: null`, retorna 200 con flag
  `scoring_pending: true`. Finalize (US-082) re-intenta.
- **Given** vocab MC con expected_answer 'B' y user_answer 'B',
  **When** procesa, **Then** score 100, persist.
- **Given** assessment NO cobra Sparks, **When** procesa, **Then**
  NO se invoca sparks.charge (verify con query post).

## Developer details

### Owning service

`apps/workers/api/handlers/assessment-submit.ts`.

### Dependencies

- US-080.
- US-033/034/035/040 (scoring tasks).

### Specs referenciados

- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §10.8.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §3 — scoring por dimension.

### Implementación esperada

```typescript
export async function handleSubmitAssessmentExercise(
  request: AuthedRequest, env: Env, sessionId: string
) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const session = await db.query(`
    SELECT * FROM assessment_sessions
    WHERE id = $1 AND status = 'in_progress' AND expires_at > now()
  `, [sessionId]);
  if (session.length === 0) {
    return jsonResponse({ error: 'session_expired_or_not_found' }, 410);
  }

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);
  if (session[0].user_id !== userId) {
    return jsonResponse({ error: 'not_your_session' }, 403);
  }

  const { part, exercise_index, payload } = await request.json();
  const item = lookupItemFromSession(session[0].selected_items, part, exercise_index);

  let score: any = null;
  try {
    switch (item.exercise_type) {
      case 'pronunciation_drill':
        score = await aiGateway.invoke('score_pronunciation', {
          audio_url: payload.audio_url,
          target_text: item.target_text,
          language: 'en-US',
          granularity: 'phoneme',
        }, { user_id: userId });
        break;
      case 'free_response':
        const transcript = await aiGateway.invoke('transcribe_user_audio', {
          audio_url: payload.audio_url,
          expected_language: 'en',
        }, { user_id: userId });
        score = await aiGateway.invoke('score_fluency', {
          transcript: transcript.text,
          duration_seconds: payload.duration_seconds,
          target_topic: item.topic,
        }, { user_id: userId });
        score.transcript = transcript.text;
        break;
      case 'listening_mc':
      case 'vocab_mc':
        score = { correct: payload.answer === item.correct_answer, score: payload.answer === item.correct_answer ? 100 : 0 };
        break;
      case 'grammar_mc':
        score = await aiGateway.invoke('detect_grammar_errors', {
          transcript: payload.user_response,
          target_cefr: session[0].assessment_cefr,
        }, { user_id: userId });
        break;
      // ... otros types
    }
  } catch (error) {
    console.error({ event: 'assessment.scoring_failed', error: error.message });
    score = null;
  }

  await db.execute(`
    INSERT INTO assessment_attempts (session_id, part, exercise_index, item_id, payload, score)
    VALUES ($1, $2, $3, $4, $5, $6)
    ON CONFLICT (session_id, part, exercise_index) DO UPDATE
      SET payload = EXCLUDED.payload, score = EXCLUDED.score, submitted_at = now()
  `, [sessionId, part, exercise_index, item.id, JSON.stringify(payload), JSON.stringify(score)]);

  const total = countItemsInPart(session[0].selected_items, part);
  const isLast = exercise_index === total - 1;

  track('assessment.exercise_submitted', { part, exercise_index, type: item.exercise_type });

  return jsonResponse({
    score_partial: score,
    next_exercise_index: isLast ? null : exercise_index + 1,
    is_last_of_part: isLast,
    scoring_pending: score === null,
  });
}
```

### Integration points

- US-033/034/035/040 (AI Gateway tasks).
- US-080 (session).
- US-082 (finalize agrega).

### Notas técnicas

- Free response es 2-step (transcribe → score). Total latency
  puede exceeder 10s — frontend muestra loading state.
- Si scoring falla: persist con score=null. Finalize re-intentará.
- UPSERT permite re-submit del mismo ejercicio (user puede regrabar).

## Definition of Done

- [ ] Endpoint implementado.
- [ ] Routing por exercise type.
- [ ] Persistencia + UPSERT.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-080. Bloqueante para US-082.*

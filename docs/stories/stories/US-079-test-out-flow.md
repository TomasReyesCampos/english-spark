# US-079: Test-out flow (skip via mini-assessment)

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3
**Story points:** 3
**Persona:** Estudiante (skip bloques que ya domina)
**Owner:** —

---

## Contexto

Test-out permite al user **saltarse un bloque** si demuestra
mastery via mini-assessment (2-3 ejercicios cortos). Útil cuando:
- User cambió de track y sub-skills mastered transfirieron.
- User entró en CEFR alto y bloques básicos son trivialmente
  fáciles.

Sin test-out, user con sub-skills mastered se aburre repitiendo.

Backend en
[`pedagogical-system.md`](../../product/pedagogical-system.md) §4.5
+
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §11.

## Scope

### In

- Endpoint `POST /roadmap/blocks/:block_id/test-out/start`:
  - Verifica block está en roadmap + `testable_out=true`.
  - Capstones NO son testable_out → 400 si intenta.
  - Genera mini-assessment de 2-3 ejercicios seleccionados de
    `asset_sequence` del bloque (subset crítico).
  - Cap: max 3 test-outs por día (anti-abuse).
  - Retorna `{ test_out_session_id, exercises }`.
- Endpoint `POST /test-out/:session_id/submit-exercise`:
  - Submit ejercicio con audio/text response.
  - Score via pedagogical engine (US-033/034/035 tasks).
  - Persist en `test_out_attempts`.
- Endpoint `POST /test-out/:session_id/finalize`:
  - Evalúa scores promedio vs threshold (75 para test-out,
    stricter que regular completion 70).
  - Si pasa: bloque marcado `status='skipped_test_out'`,
    mastery propagated igual que completion.
  - Si no pasa: session marcada `status='failed'`, user must do
    block normal.
- Schema `test_out_sessions` + `test_out_attempts`.
- Telemetry: `test_out.started`, `.exercise_submitted`,
  `.finalized_passed`, `.finalized_failed`,
  `.daily_cap_reached`.

### Out

- UI del flow (incluido en pedagogical UI, EPIC-03).
- Test-out de múltiples bloques en bulk (post-MVP).

## Acceptance criteria

- **Given** block con `testable_out=true`, **When** POST start,
  **Then** retorna session con 2-3 ejercicios seleccionados.
- **Given** block capstone (testable_out=false), **When** start,
  **Then** 400 `{ error: 'block_not_testable_out' }`.
- **Given** user ya hizo 3 test-outs hoy, **When** 4to start,
  **Then** 429 `{ error: 'daily_cap_reached' }`.
- **Given** session active con 3 ejercicios, **When** submit los
  3, **Then** persist test_out_attempts con scores.
- **Given** scores avg 80 (≥ 75 threshold), **When** finalize,
  **Then** block status='skipped_test_out', mastery propagada,
  emit `block.completed` con `via_test_out: true`.
- **Given** scores avg 70 (below 75), **When** finalize, **Then**
  session status='failed', block sigue pending, user debe hacer
  block normal.
- **Given** session timeout 30 min sin finalize, **When** cron
  evalúa, **Then** session auto-cancelled, user puede start
  nueva.
- **Given** session ya finalized, **When** retry submit, **Then**
  409 `{ error: 'session_finalized' }`.

## Developer details

### Owning service

`apps/workers/api/handlers/test-out-*.ts`.

### Dependencies

- US-072/077/078.
- AI Gateway scoring tasks (US-033/034/035).
- Pedagogical engine (EPIC-03).

### Specs referenciados

- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §4.5 — test-out criteria.
- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §11.

### Schema

```sql
CREATE TABLE test_out_sessions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  block_id        TEXT NOT NULL REFERENCES learning_blocks(id),
  status          TEXT NOT NULL DEFAULT 'in_progress' CHECK (status IN (
                    'in_progress', 'finalized_passed', 'finalized_failed', 'cancelled'
                  )),
  selected_assets JSONB NOT NULL,
  started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  finalized_at    TIMESTAMPTZ
);

CREATE TABLE test_out_attempts (
  session_id      UUID NOT NULL REFERENCES test_out_sessions(id) ON DELETE CASCADE,
  asset_id        TEXT NOT NULL,
  score           NUMERIC(5,2),
  submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (session_id, asset_id)
);
```

### Implementación esperada

```typescript
const TEST_OUT_THRESHOLD = 75;
const DAILY_CAP = 3;

export async function handleTestOutStart(
  request: AuthedRequest, env: Env, blockId: string
) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  // Daily cap
  const todayCount = await env.DB.query(`
    SELECT COUNT(*) AS count FROM test_out_sessions
    WHERE user_id = $1 AND started_at > date_trunc('day', now())
  `, [userId]);
  if (Number(todayCount[0].count) >= DAILY_CAP) {
    track('test_out.daily_cap_reached');
    return jsonResponse({ error: 'daily_cap_reached' }, 429);
  }

  // Block validation
  const block = await env.DB.query(`
    SELECT id, asset_sequence, testable_out, is_capstone
    FROM learning_blocks WHERE id = $1
  `, [blockId]);
  if (block.length === 0 || !block[0].testable_out) {
    return jsonResponse({ error: 'block_not_testable_out' }, 400);
  }

  // Select 2-3 critical assets (subset)
  const selectedAssets = pickCriticalAssets(block[0].asset_sequence, 3);

  const session = await env.DB.query(`
    INSERT INTO test_out_sessions (user_id, block_id, selected_assets)
    VALUES ($1, $2, $3) RETURNING id
  `, [userId, blockId, JSON.stringify(selectedAssets)]);

  track('test_out.started', { block_id: blockId });
  return jsonResponse({
    test_out_session_id: session[0].id,
    exercises: selectedAssets,
  });
}

export async function handleTestOutFinalize(
  request: AuthedRequest, env: Env, sessionId: string
) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const session = await env.DB.query(`
    SELECT s.*, AVG(a.score) AS avg_score, COUNT(a.asset_id) AS attempts_count
    FROM test_out_sessions s
    LEFT JOIN test_out_attempts a ON a.session_id = s.id
    WHERE s.id = $1 AND s.status = 'in_progress'
    GROUP BY s.id
  `, [sessionId]);

  if (session.length === 0) {
    return jsonResponse({ error: 'session_finalized_or_not_found' }, 409);
  }

  const avgScore = Number(session[0].avg_score ?? 0);
  const totalRequired = JSON.parse(session[0].selected_assets).length;
  const completed = Number(session[0].attempts_count) >= totalRequired;

  if (!completed) {
    return jsonResponse({ error: 'incomplete_session', required: totalRequired }, 400);
  }

  const passed = avgScore >= TEST_OUT_THRESHOLD;

  await env.DB.execute(`
    UPDATE test_out_sessions
    SET status = $1, finalized_at = now()
    WHERE id = $2
  `, [passed ? 'finalized_passed' : 'finalized_failed', sessionId]);

  if (passed) {
    // Mark block as skipped_test_out + propagate mastery
    await env.DB.execute(`
      INSERT INTO block_completions
        (user_id, block_id, status, completed_at, score_avg)
      VALUES ($1, $2, 'skipped_test_out', now(), $3)
      ON CONFLICT (user_id, block_id) DO UPDATE
        SET status = 'skipped_test_out',
            completed_at = now(),
            score_avg = EXCLUDED.score_avg
    `, [session[0].user_id, session[0].block_id, avgScore]);

    await emitEvent('block.completed', {
      user_id: session[0].user_id,
      block_id: session[0].block_id,
      score_avg: avgScore,
      via_test_out: true,
    });

    await env.ROADMAP_KV.delete(`roadmap_active:${session[0].user_id}`);
    track('test_out.finalized_passed', { block_id: session[0].block_id });
  } else {
    track('test_out.finalized_failed', { block_id: session[0].block_id, score: avgScore });
  }

  return jsonResponse({
    status: passed ? 'passed' : 'failed',
    score_avg: avgScore,
    threshold: TEST_OUT_THRESHOLD,
  });
}
```

### Integration points

- US-072 schemas.
- US-078 (consume `block.completed` event con
  `via_test_out: true` para diferenciar en achievements).
- AI Gateway scoring tasks.

### Notas técnicas

- Threshold 75 (vs regular 70) refleja que test-out es
  oportunidad de demonstrar más fuerte.
- 2-3 ejercicios subset: selecciona los más diagnósticos
  (free_response + pronunciation drill usuario).
- Daily cap 3 previene abuse (alguien intentando skip todo el
  roadmap).

## Definition of Done

- [ ] 3 endpoints implementados.
- [ ] Schema `test_out_*` migrations.
- [ ] Daily cap + capstone exclusion.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-072/077/078, AI Gateway scoring.*

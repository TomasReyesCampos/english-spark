# US-082: Finalize assessment + roadmap definitivo generation

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3
**Story points:** 5
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Cierra el assessment Day 7:
- Agrega scores de los ~25 ejercicios en
  `assessment_results` JSONB.
- Detecta "broken assessment" (silence excesivo, random
  responses).
- Persiste en `student_profiles.assessment_results +
  assessment_completed_at`.
- Trigger generación del **roadmap definitivo** (más detallado y
  específico que el initial).
- Emit `assessment.completed` event (consumed por motivation,
  notifications, paywall).

Backend en
[`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
§6.2-§6.6 + `ai-roadmap-system.md` §6.

## Scope

### In

- Endpoint `POST /assessment/:session_id/finalize`:
  - Valida JWT + ownership + session in_progress.
  - Verifica que **todos** los items del session han sido
    submitted (else 400 con `incomplete: missing_count`).
  - Agrega scores:
    - `pronunciation`: avg de pronunciation drills.
    - `fluency`: de free_responses.
    - `grammar`: de detect_grammar (% error rate inverted).
    - `vocabulary`: de vocab_mc + reading.
    - `listening`: de listening_mc + comprensión.
    - `overall_speaking`: weighted (pron 25% + fluency 25% +
      grammar 20% + vocab 15% + listening 15%, per §2.5).
  - Detecta broken assessment:
    - Audio silence > 80% en al menos 30% de exercises.
    - Multiple choice todos misma respuesta.
    - Tiempo total < 5 min.
    - Si broken: persist con flag `low_confidence: true` + return
      prompt al user para retry.
  - Computa measured_cefr_level desde scores.
  - Identifica strongest_areas, weakest_areas.
  - Persist `student_profiles.assessment_results = {...},
    assessment_completed_at = now()`.
  - Archive initial roadmap, trigger Inngest function para
    generate_definitive_roadmap.
  - Cambia trial_status a `'converted'` o `'completed_assessment'`.
  - Emit `assessment.completed` con full results.
  - Retorna estructura completa de results.
- Telemetry: `assessment.finalized`, `.broken_detected`,
  `.results_persisted`.

### Out

- 8-screen reveal flow (US-083).
- Paywall integration (post-MVP UI, EPIC-04 ya tiene Sparks).

## Acceptance criteria

- **Given** session completa (todos submitted), **When** finalize,
  **Then** retorna assessment_results con 5 dimension scores +
  measured_cefr.
- **Given** session incompleta (3 items pendientes), **When**
  finalize, **Then** 400 `{ error: 'incomplete', missing_count: 3 }`.
- **Given** session con audio_silence en 5 de 6 free_responses,
  **When** finalize evalúa, **Then** flag `low_confidence: true`
  + retorna prompt para retry.
- **Given** scores: pronunciation 75, fluency 80, grammar 70,
  vocab 65, listening 70, **When** computes overall, **Then**
  weighted = 73.25 → cefr_level = 'B1+'.
- **Given** finalize exitoso, **When** completa, **Then**
  initial_roadmap archived, Inngest function triggered, event
  `assessment.completed` emitido.
- **Given** weakest_areas detection, **When** procesa, **Then**
  retorna ordered list (lowest dimension first).
- **Given** session ya finalized previamente, **When** retry,
  **Then** 409 `{ error: 'already_finalized' }`.
- **Given** AI Gateway timeout en re-scoring de attempts pending,
  **When** finalize, **Then** usa scores existentes + log warning;
  no bloquea finalization.

## Developer details

### Owning service

`apps/workers/api/handlers/assessment-finalize.ts` +
`inngest-functions/generate-definitive-roadmap.ts`.

### Dependencies

- US-080/081.
- US-021 (pattern roadmap generation).

### Specs referenciados

- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §6.2-§6.6.
- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §6.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §2.5 — overall scoring formula.

### Implementación esperada

```typescript
const BROKEN_AUDIO_THRESHOLD = 0.30; // 30% de exercises silenciosos = broken
const BROKEN_TIME_THRESHOLD_MIN = 5;

export async function handleFinalizeAssessment(
  request: AuthedRequest, env: Env, sessionId: string
) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const session = await db.query(`SELECT * FROM assessment_sessions WHERE id = $1`, [sessionId]);
  if (session.length === 0) return jsonResponse({ error: 'not_found' }, 404);
  if (session[0].status === 'completed') {
    return jsonResponse({ error: 'already_finalized' }, 409);
  }

  // Verify all items submitted
  const total = countAllItems(session[0].selected_items);
  const submitted = await db.query(`
    SELECT COUNT(*) AS count FROM assessment_attempts WHERE session_id = $1
  `, [sessionId]);
  const missing = total - Number(submitted[0].count);
  if (missing > 0) {
    return jsonResponse({ error: 'incomplete', missing_count: missing }, 400);
  }

  // Aggregate scores
  const attempts = await db.query(`
    SELECT * FROM assessment_attempts WHERE session_id = $1
  `, [sessionId]);

  const scores = aggregateScoresByDimension(attempts);
  const measuredCefr = mapDimensionsToCefr(scores);
  const { strongest, weakest } = identifyStrengthsWeaknesses(scores);

  // Broken detection
  const isBroken = detectBrokenAssessment(attempts, session[0].started_at);

  const assessmentResults = {
    completed_at: new Date().toISOString(),
    assessment_version: 'v1.0',
    duration_seconds: Math.floor((Date.now() - new Date(session[0].started_at).getTime()) / 1000),
    measured_cefr_level: measuredCefr,
    scores,
    phonetic_error_patterns: extractPhoneticPatterns(attempts),
    grammar_error_patterns: extractGrammarPatterns(attempts),
    strongest_areas: strongest,
    weakest_areas: weakest,
    low_confidence: isBroken,
  };

  // Persist
  await db.transaction(async (tx) => {
    await tx.execute(`
      UPDATE student_profiles
      SET assessment_results = $1, assessment_completed_at = now(),
          trial_status = CASE WHEN trial_status = 'active' THEN 'completed_assessment'
                              ELSE trial_status END
      WHERE user_id = $2
    `, [JSON.stringify(assessmentResults), session[0].user_id]);

    await tx.execute(`
      UPDATE assessment_sessions
      SET status = 'completed', completed_at = now() WHERE id = $1
    `, [sessionId]);

    // Archive initial roadmap
    await tx.execute(`
      UPDATE roadmaps SET is_active = false, archived_at = now(),
        archived_reason = 'replaced_by_definitive'
      WHERE user_id = $1 AND is_active = true AND roadmap_type = 'initial'
    `, [session[0].user_id]);
  });

  // Trigger definitive roadmap generation
  await emitEvent('assessment.completed', {
    user_id: session[0].user_id,
    assessment_results: assessmentResults,
  });

  if (isBroken) {
    track('assessment.broken_detected', { session_id: sessionId });
  }
  track('assessment.finalized', { cefr: measuredCefr, low_confidence: isBroken });

  return jsonResponse({
    assessment_results: assessmentResults,
    broken_prompt_required: isBroken,
  });
}

// Inngest function
export const generateDefinitiveRoadmapFn = inngest.createFunction(
  { id: 'generate-definitive-roadmap', retries: 2 },
  { event: 'assessment.completed' },
  async ({ event, step }) => {
    const { user_id, assessment_results } = event.data;
    const profile = await db.getProfile(user_id);

    const availableBlocks = await filterBlocksForUser({
      ...profile,
      cefr_estimate: assessment_results.measured_cefr_level,
    });

    const llmOutput = await aiGateway.invoke('generate_initial_roadmap', {
      // Reuse task but with definitive-specific prompt context
      profile: { ...profile, assessment_results },
      availableBlocks,
      mode: 'definitive',
    });

    const roadmapId = await persistRoadmap(user_id, llmOutput, 'definitive');
    await env.ROADMAP_KV.delete(`roadmap_active:${user_id}`);
    await emitEvent('roadmap.generated', { user_id, roadmap_id: roadmapId, type: 'definitive' });
  }
);
```

### Integration points

- US-080/081.
- US-021 pattern reused.
- Motivation system (consume assessment.completed for achievements).
- Notifications (US-067 patterns).

### Notas técnicas

- Broken detection sigue spec §6.6.
- Definitive roadmap más profundo: 50-80 bloques vs 30-40 del
  initial.
- Reuse de task `generate_initial_roadmap` con flag `mode=definitive`
  para diferenciar prompt template.

## Definition of Done

- [ ] Endpoint + Inngest function implementados.
- [ ] Aggregation logic correcta (matchea spec §2.5).
- [ ] Broken detection.
- [ ] Roadmap definitivo generation.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-080/081. Climax del assessment flow.*

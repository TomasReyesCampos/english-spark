# US-020: Procesamiento screen + initial_test_results consolidado

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Después de los 3 ejercicios del mini-test (US-017 a US-019), el
backend tiene 3 partial results en
`student_profile.initial_test_results.exercise_{1,2,3}`. Esta
story consolida los 3 en un objeto unificado con CEFR estimado y
displays una pantalla de procesamiento mientras se computa.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §14
(procesamiento). Backend en
[`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
§3.4 (`InitialTestResults` shape).

## Scope

### In

- Pantalla "Estamos analizando..." con animación cíclica + mensajes
  educativos rotativos:
  - "Analizando tu pronunciación..."
  - "Midiendo tu fluidez..."
  - "Identificando tus fortalezas..."
  - "Armando tu plan inicial..."
- Cada mensaje 2-3 segundos. Total visible 8-12 segundos.
- Endpoint `POST /pedagogical/finalize-minitest`:
  - Lee
    `initial_test_results.exercise_{1,2,3}` del profile.
  - Computa scores agregados:
    - `pronunciation_score` (0-10): promedio del ej1 (frase) + ej3
      (palabras matched).
    - `fluency_score` (0-10): de ej2 (free response).
    - `cefr_estimate`: heuristic mapping de scores combinados.
  - Detecta `error_patterns` foundational del ej1
    (phoneme_scores).
  - Persiste `student_profile.initial_test_results` (el shape
    final).
  - Trigger event `initial_test.completed`.
  - Retorna full `InitialTestResults`.
- Manejo del flow skipped (US-016 skip path):
  - Si `consent_audio_processing = false` y mini-test no completado:
    `cefr_estimate` se asume del `self_perceived_level` con offset
    -1 (más conservador).
- Si user llegó hasta acá: navega automáticamente a US-022 (roadmap
  reveal) cuando finalize termina.
- Telemetry: `minitest.processing_started`, `.processing_completed`,
  `.cefr_estimated` con valor.

### Out

- Roadmap initial generation per se (US-021).
- Roadmap reveal screen (US-022).
- Storage retention policy (cross-cutting cleanup).

## Acceptance criteria

- **Given** user terminó los 3 ejercicios del mini-test, **When**
  llega a la pantalla de procesamiento, **Then** ve animación con
  mensajes rotativos.
- **Given** procesamiento toma 8-12 segundos, **When** completa,
  **Then** automáticamente navega a US-022.
- **Given** `exercise_1.pronunciation_score = 75` (de 100),
  `exercise_2.fluency_score = 65`, `exercise_3.vocabulary_score
  = 100`, **When** se computa,
  **Then** `cefr_estimate` corresponde a B1+ (ver mapping abajo).
- **Given** user skipped el mini-test (no consent), **When** se
  llama finalize, **Then** retorna
  `cefr_estimate = max(A2, self_perceived_level - 1)` y demás
  scores son null.
- **Given** procesamiento toma >15 segundos (timeout backend),
  **When** falla, **Then** muestra "Esto está tardando más de lo
  esperado..." con retry button. Si retry falla 2x: fallback a
  CEFR del self_perceived_level.
- **Given** `exercise_2.language_mismatch = true` (user habló
  español), **When** finalize procesa, **Then** flag se propaga al
  `initial_test_results.confidence = 'low'` y el roadmap inicial
  use ese flag.
- **Given** finalize completa, **When** event
  `initial_test.completed` emitido, **Then** US-021 (AI Roadmap)
  consume y empieza generación.
- **Given** finalize falla con backend error 500, **When** intenta
  consolidar, **Then** se loguea SEV-2 y user ve fallback con
  CEFR del self_perceived_level + skip flag.

## Developer details

### Owning service

Mobile app + backend
`/pedagogical/finalize-minitest`.

### Dependencies

- US-017, US-018, US-019: ejercicios completados.
- US-015: profile update.
- AI Gateway tasks ya invocadas en stories anteriores.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §14.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §3.4 — `InitialTestResults` shape autoritativo.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §2.5 — mapping scores → CEFR.

### CEFR estimation heuristic

```typescript
// apps/workers/api/lib/cefr-from-minitest.ts
function estimateCefrFromMiniTest(results: PartialResults): CefrLevel {
  const pronScore = (results.exercise_1?.pronunciation_score ?? 0) / 100;
  const fluencyScore = (results.exercise_2?.fluency_score ?? 0) / 100;
  const vocabScore = (results.exercise_3?.vocabulary_score ?? 0) / 100;

  // Speaking-weighted (matchea pedagogical-system.md §2.5)
  const weighted =
    pronScore * 0.40 +
    fluencyScore * 0.40 +
    vocabScore * 0.20;
  const composite = weighted * 100;

  if (composite < 35) return 'A2';
  if (composite < 55) return 'B1';
  if (composite < 70) return 'B1+';
  if (composite < 82) return 'B2';
  if (composite < 90) return 'B2+';
  return 'C1';
}
```

### Implementación esperada

```typescript
// apps/workers/api/handlers/finalize-minitest.ts
export async function handleFinalizeMinitest(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const profile = await db.getProfile(request.user!.firebase_uid);
  if (!profile) return jsonResponse({ error: 'profile_not_found' }, 404);

  const partialResults = profile.initial_test_results;
  let consolidated: InitialTestResults;

  if (!profile.consent_audio_processing || !partialResults?.exercise_1) {
    // Skipped path
    const fallbackCefr = downgradeOneLevel(profile.self_perceived_level);
    consolidated = {
      cefr_estimate: fallbackCefr,
      fluency_score: null,
      pronunciation_score: null,
      detected_error_patterns: [],
      recorded_at: new Date().toISOString(),
      confidence: 'self_reported_only',
    };
  } else {
    const cefr = estimateCefrFromMiniTest(partialResults);
    consolidated = {
      cefr_estimate: cefr,
      pronunciation_score: avgPronScore(partialResults),
      fluency_score: partialResults.exercise_2.fluency_score / 10,
      detected_error_patterns: extractPhonemicPatterns(partialResults.exercise_1),
      recorded_at: new Date().toISOString(),
      confidence: partialResults.exercise_2?.language_mismatch ? 'low' : 'normal',
    };
  }

  await db.updateProfile(request.user!.firebase_uid, {
    initial_test_results: consolidated,
  });

  await emitEvent('initial_test.completed', {
    user_id: profile.user_id,
    cefr_estimate: consolidated.cefr_estimate,
    confidence: consolidated.confidence,
  });

  track('minitest.cefr_estimated', { cefr: consolidated.cefr_estimate });

  return jsonResponse(consolidated);
}
```

### Integration points

- AI Gateway tasks ya invocados en US-017/18/19.
- Profile update.
- Event bus → US-021 listener.

## Definition of Done

- [ ] Pantalla con animación y mensajes rotativos.
- [ ] Endpoint `/pedagogical/finalize-minitest` implementado.
- [ ] Heuristic CEFR estimation testeado con 5+ casos.
- [ ] Skip path funciona.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] QA manual end-to-end (mini-test completo + skip).
- [ ] Validation contra spec
  `student-profile-and-assessment.md` §3.4.
- [ ] PR aprobada y mergeada.

---

*Depende de US-017, US-018, US-019. Bloqueante para US-021.*

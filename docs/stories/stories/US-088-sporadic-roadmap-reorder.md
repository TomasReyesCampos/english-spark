# US-088: Sporadic responses scoring + roadmap reorder

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 4
**Story points:** 2
**Persona:** Admin / Estudiante (recibe roadmap reordered)
**Owner:** —

---

## Contexto

Procesa responses sporadic non-fake para:
- Actualizar `self_reported_dimensions` JSONB en profile.
- Detectar `sporadic_mismatch_flags` (self-perception vs measured
  divergence).
- Reorder blocks del roadmap provisional Day 1-2 según insights
  (NO regenera roadmap entero).
- Audio captures contribuyen a `user_subskill_mastery` con peso
  0.5x (de §2.6 pedagogical-system).

Backend en
[`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
§7.1.6-§7.1.7.

## Scope

### In

- Inngest function `process-sporadic-response`:
  - Triggered post-respond (US-087).
  - Skip si `flagged_likely_fake = true`.
  - Por category:
    - `self_assessment`: update `self_reported_dimensions[dim]`.
    - `micro_audio`: trigger pronunciation scoring (US-033),
      contribute a `user_subskill_mastery` con weight 0.5.
    - `vocab_check`, `grammar_check`: score local, update
      `user_subskill_mastery` con weight 0.5.
    - `goal_validation`: si user cambió goal, trigger
      `profile.updated` event (que US-084 evalúa significant).
  - Detect mismatch: si self-perception ≥ 4 pero measured ≤ 60
    (o inverso), flag en `sporadic_mismatch_flags`.
- Cron `reorder-provisional-roadmap`:
  - Diario, evalúa users en Day 1-2 con
    self_reported_dimensions populated.
  - Llama a un internal helper que **reordena** blocks del
    roadmap activo según foco detectado (más weak first).
  - NO regenera (no LLM call) — solo cambia order_in_level.
- Telemetry: `sporadic.response_processed`,
  `.mismatch_flagged`, `.roadmap_reordered`.

### Out

- LLM-driven full regeneration (eso solo en significant_change
  US-085).
- Multi-user batch processing (post-MVP if needed).

## Acceptance criteria

- **Given** sporadic response category='self_assessment'
  dimension='speaking' score=4, **When** procesa, **Then**
  `student_profiles.self_reported_dimensions.speaking = 4`.
- **Given** mismatch detected (self=5, measured=40 en vocab),
  **When** procesa, **Then** flag `vocab_mismatch_high_to_low`
  added a `sporadic_mismatch_flags`.
- **Given** response micro_audio con score 65, **When** procesa,
  **Then** user_subskill_mastery actualiza con learning_rate
  ajustado × 0.5 weight.
- **Given** response con `flagged_likely_fake = true`, **When**
  function evalúa, **Then** skip (no update).
- **Given** response goal_validation con goal nuevo, **When**
  procesa, **Then** emit `profile.updated` con
  `goal_change_via_sporadic=true`.
- **Given** cron reorder corre, user con
  `self_reported_dimensions.speaking=1` (very weak), **When**
  procesa, **Then** blocks con `target_subskills` related a
  speaking suben en order_in_level.
- **Given** reorder applied, **When** próximo GET /roadmap/active,
  **Then** retorna roadmap con new order (cache invalidated).
- **Given** user no Day 1-2, **When** cron evalúa, **Then** skip
  (only provisional reorder Day 1-2).

## Developer details

### Owning service

`apps/workers/inngest-functions/process-sporadic-response.ts` +
`crons/reorder-provisional-roadmap.ts`.

### Dependencies

- US-087.
- US-072 (roadmap schemas).
- US-033 (pronunciation scoring para micro_audio).

### Specs referenciados

- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §7.1.6-§7.1.8.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §2.6 — source_weight for sporadic = 0.5.

### Implementación esperada

```typescript
// process-sporadic-response.ts
export const processSporadicResponseFn = inngest.createFunction(
  { id: 'process-sporadic-response' },
  { event: 'sporadic.responded' },
  async ({ event, step }) => {
    const { response_id, user_id, question_id } = event.data;

    const response = await step.run('fetch-response', () => db.query(`
      SELECT sr.*, sq.category, sq.target_dimension, sq.content
      FROM sporadic_responses sr
      JOIN sporadic_questions sq ON sq.id = sr.question_id
      WHERE sr.id = $1
    `, [response_id]));

    if (response[0].flagged_likely_fake) {
      return { skipped: 'fake' };
    }

    const responseData = response[0].response_data;

    switch (response[0].category) {
      case 'self_assessment':
        await db.execute(`
          UPDATE student_profiles
          SET self_reported_dimensions = jsonb_set(
            COALESCE(self_reported_dimensions, '{}'::jsonb),
            $1, $2::jsonb
          )
          WHERE user_id = $3
        `, [`{${response[0].target_dimension}}`, JSON.stringify(responseData.score), user_id]);
        break;

      case 'micro_audio':
        const score = await aiGateway.invoke('score_pronunciation', { /* ... */ });
        await updateSubskillMasteryWithWeight(user_id, response[0].target_dimension, score.overall, 0.5);

        // Mismatch detection
        const selfScore = (await getProfile(user_id)).self_reported_dimensions?.[response[0].target_dimension];
        if (selfScore && Math.abs(selfScore * 20 - score.overall) > 30) {
          await db.execute(`
            UPDATE student_profiles
            SET sporadic_mismatch_flags = sporadic_mismatch_flags || $1
            WHERE user_id = $2
          `, [[`${response[0].target_dimension}_mismatch`], user_id]);
        }
        break;

      case 'vocab_check':
      case 'grammar_check':
        const correct = responseData.answer === response[0].content.correct;
        await updateSubskillMasteryWithWeight(user_id, response[0].target_dimension, correct ? 100 : 0, 0.5);
        break;

      case 'goal_validation':
        if (responseData.new_goal) {
          await emitEvent('profile.updated', {
            user_id,
            fields_changed: ['primary_goals'],
            goal_change_via_sporadic: true,
            significant_change: true, // delegates to US-084 to confirm
          });
        }
        break;
    }

    track('sporadic.response_processed', { category: response[0].category });
  }
);

// crons/reorder-provisional-roadmap.ts
export async function runReorderProvisional(env: Env) {
  const day1to2Users = await db.query(`
    SELECT user_id, self_reported_dimensions
    FROM student_profiles
    WHERE trial_started_at > now() - interval '2 days'
      AND trial_started_at <= now() - interval '1 day'
      AND self_reported_dimensions IS NOT NULL
      AND assessment_completed_at IS NULL
  `);

  for (const profile of day1to2Users) {
    await reorderRoadmapByWeakest(profile.user_id, profile.self_reported_dimensions, env);
  }

  return { processed: day1to2Users.length };
}
```

### Integration points

- US-087 (event source).
- US-033 (scoring).
- US-072 (mastery + roadmap schemas).
- US-073 (cache invalidation post-reorder).

### Notas técnicas

- Source weight 0.5 para sporadic alineado con
  `pedagogical-system.md` §2.6.
- Reorder NO regenera = sin LLM cost.
- Mismatch threshold (Δ 30 score) heurístico.

## Definition of Done

- [ ] Inngest function + cron implementados.
- [ ] Weight 0.5 aplicado correctamente.
- [ ] Reorder logic.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec §7.1.
- [ ] PR aprobada y mergeada.

---

*Cierra EPIC-07. Depende de US-087.*

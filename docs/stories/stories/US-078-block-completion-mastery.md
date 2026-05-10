# US-078: Block completion + mastery propagation

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3
**Story points:** 3
**Persona:** Estudiante (consumer del progress) / Admin
**Owner:** —

---

## Contexto

Detecta cuándo un bloque está "completado" basado en su
`mastery_criteria` (definido en learning_blocks) y los scores de
exercise_attempts del user.

Cuando block completes:
- Update `block_completions.status = 'completed'`,
  `completed_at = now()`.
- Propagate mastery a `user_subskill_mastery` (track-orthogonal).
- Emit `block.completed` event (consumed by motivation system para
  detección de achievements).
- Invalidate roadmap cache.

Backend en
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §11
+ [`pedagogical-system.md`](../../product/pedagogical-system.md) §6.

## Scope

### In

- Inngest function `evaluate-block-completion`:
  - Triggered by event `exercise_attempt.completed` (de
    EPIC-03 futuro).
  - Para el block del exercise_attempt:
    - Cuenta total exercise_attempts del bloque para este user.
    - Aggregate score_avg, mastery_evidence.
    - Compara contra `mastery_criteria` del bloque:
      - `score_min`: score promedio mínimo (typically 70).
      - `attempts_min`: mínimo de exercises completados (typically
        todos los obligatorios del asset_sequence).
      - `subskill_thresholds`: opcional, scores específicos por
        sub-skill.
    - Si cumple: status='completed', completed_at=now().
- Propagation a `user_subskill_mastery`:
  - Para cada target_subskill del block, update mastery score
    via algoritmo de
    [`pedagogical-system.md`](../../product/pedagogical-system.md) §2.5
    (learning_rate por total_attempts).
- Emit `block.completed` event con `{ user_id, block_id,
  score_avg, mastery_changes: [...] }`.
- Invalidate roadmap cache.
- Telemetry: `block.completion_evaluated`, `.completed`,
  `.criteria_not_met`.

### Out

- Exercise scoring per se (EPIC-03).
- Achievement detection (motivation system consume event).
- Score calculations specifics (delegated to pedagogical engine).

## Acceptance criteria

- **Given** event `exercise_attempt.completed`, **When** function
  ejecuta, **Then** lee exercise_attempts del block y compara
  contra criteria.
- **Given** block con criteria `score_min: 70, attempts_min: 4`,
  user tiene 4 attempts con avg 75, **When** evalúa, **Then**
  status='completed', emit `block.completed`.
- **Given** user tiene 4 attempts con avg 65 (below 70), **When**
  evalúa, **Then** status sigue 'in_progress', telemetry
  `criteria_not_met`.
- **Given** block con target_subskills `['pron_th_voiceless']`,
  **When** completes, **Then** user_subskill_mastery row para
  `pron_th_voiceless` se actualiza con delta según learning_rate.
- **Given** capstone block con criteria stricter (score_min:75),
  **When** evalúa con avg 73, **Then** NO complete (capstones
  exigen más).
- **Given** block ya `completed`, **When** llega event más,
  **Then** skip (no re-complete, no re-emit event).
- **Given** event sin block_id (anomalous), **When** function,
  **Then** log warning + skip.
- **Given** completion exitoso, **When** próximo GET
  /roadmap/active, **Then** ese block aparece como completed.

## Developer details

### Owning service

`apps/workers/inngest-functions/evaluate-block-completion.ts`.

### Dependencies

- US-072 (schemas).
- US-077 (block status).
- EPIC-03 emite `exercise_attempt.completed`.

### Specs referenciados

- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §11.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §2.5, §6 — mastery aggregation.

### Implementación esperada

```typescript
// apps/workers/inngest-functions/evaluate-block-completion.ts
export const evaluateBlockCompletionFn = inngest.createFunction(
  { id: 'evaluate-block-completion', retries: 2 },
  { event: 'exercise_attempt.completed' },
  async ({ event, step }) => {
    const { user_id, block_id, exercise_attempt_id } = event.data;

    const block = await step.run('fetch-block', () => db.query(`
      SELECT id, target_subskills, mastery_criteria, asset_sequence, is_capstone
      FROM learning_blocks WHERE id = $1
    `, [block_id]));
    if (block.length === 0) return { skipped: 'block_not_found' };

    const current = await step.run('fetch-completion', () => db.query(`
      SELECT status FROM block_completions
      WHERE user_id = $1 AND block_id = $2
    `, [user_id, block_id]));
    if (current[0]?.status === 'completed') return { skipped: 'already_completed' };

    // Aggregate exercise_attempts for this block
    const attempts = await step.run('aggregate-attempts', () => db.query(`
      SELECT
        COUNT(*) AS total_attempts,
        AVG(score) AS avg_score,
        json_agg(json_build_object(
          'asset_id', asset_id,
          'score', score,
          'completed_at', completed_at
        )) AS attempts_detail
      FROM exercise_attempts
      WHERE user_id = $1 AND block_id = $2 AND completed_at IS NOT NULL
    `, [user_id, block_id]));

    const criteria = block[0].mastery_criteria;
    const totalAttempts = Number(attempts[0].total_attempts);
    const avgScore = Number(attempts[0].avg_score ?? 0);

    // Compute required attempts (obligatorios del asset_sequence)
    const requiredAttempts = block[0].asset_sequence
      .filter((a: any) => !a.optional).length;

    const meetsAttempts = totalAttempts >= (criteria.attempts_min ?? requiredAttempts);
    const meetsScore = avgScore >= (criteria.score_min ?? 70);

    track('block.completion_evaluated', {
      block_id, total_attempts: totalAttempts, avg_score: avgScore,
      meets_attempts: meetsAttempts, meets_score: meetsScore,
    });

    if (!meetsAttempts || !meetsScore) {
      track('block.criteria_not_met', { block_id });
      // Update progress but not completion
      await db.execute(`
        UPDATE block_completions
        SET exercise_attempts_count = $1, score_avg = $2, updated_at = now()
        WHERE user_id = $3 AND block_id = $4
      `, [totalAttempts, avgScore, user_id, block_id]);
      return { completed: false };
    }

    // COMPLETED
    await step.run('mark-completed', () => db.execute(`
      UPDATE block_completions
      SET status = 'completed', completed_at = now(),
          exercise_attempts_count = $1, score_avg = $2, updated_at = now()
      WHERE user_id = $3 AND block_id = $4
    `, [totalAttempts, avgScore, user_id, block_id]));

    // Propagate mastery to target_subskills
    const masteryChanges = [];
    for (const subskill of block[0].target_subskills) {
      const change = await step.run(`update-mastery-${subskill}`, async () => {
        const result = await db.query(`
          SELECT current_score, total_attempts FROM user_subskill_mastery
          WHERE user_id = $1 AND subskill_id = $2
        `, [user_id, subskill]);

        const current = result[0]?.current_score ?? 50; // default neutral
        const totalAtt = result[0]?.total_attempts ?? 0;

        // Learning rate (de pedagogical-system.md §2.5)
        const learningRate = totalAtt < 5 ? 0.30
                           : totalAtt < 20 ? 0.15
                           : 0.08;
        const delta = (avgScore - current) * learningRate;
        const newScore = Math.max(0, Math.min(100, current + delta));

        await db.execute(`
          INSERT INTO user_subskill_mastery
            (user_id, subskill_id, current_score, total_attempts, updated_at)
          VALUES ($1, $2, $3, $4, now())
          ON CONFLICT (user_id, subskill_id) DO UPDATE
            SET current_score = EXCLUDED.current_score,
                total_attempts = user_subskill_mastery.total_attempts + 1,
                updated_at = now()
        `, [user_id, subskill, newScore, totalAtt + 1]);

        return { subskill, from: current, to: newScore, delta };
      });
      masteryChanges.push(change);
    }

    await step.run('invalidate-cache', () => env.ROADMAP_KV.delete(`roadmap_active:${user_id}`));

    await step.run('emit-event', () => emitEvent('block.completed', {
      user_id, block_id,
      score_avg: avgScore,
      mastery_changes: masteryChanges,
      is_capstone: block[0].is_capstone,
    }));

    track('block.completed', { block_id, is_capstone: block[0].is_capstone });

    return { completed: true, mastery_changes: masteryChanges };
  }
);
```

### Integration points

- EPIC-03 emit `exercise_attempt.completed`.
- US-077 (block status updates).
- US-072 schema user_subskill_mastery.
- Motivation system (consume `block.completed` para achievements).

### Notas técnicas

- Capstones detection via `is_capstone` flag → threshold más alto
  ya está en criteria del block.
- Learning rate logic alineado con
  `pedagogical-system.md` §2.5.
- Idempotency: si event re-emitted post-completion, skip via
  `already_completed`.

## Definition of Done

- [ ] Inngest function implementada.
- [ ] Mastery propagation correcta.
- [ ] Idempotency.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-072/077, EPIC-03 future. Driver de progreso.*

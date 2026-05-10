# US-085: Roadmap regeneration on significant_change

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3-4
**Story points:** 3
**Persona:** Estudiante (recibe roadmap actualizado)
**Owner:** —

---

## Contexto

Consumer del event `profile.updated` con
`significant_change: true`. Regenera roadmap manteniendo block
completions (no resets).

Backend en
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §14.3.

## Scope

### In

- Inngest function `regenerate-roadmap-on-significant-change`:
  - Triggered by event `profile.updated` con `significant_change=true`.
  - Mantiene block_completions del user (transfer cross-track via
    sub-skill mastery).
  - Genera nuevo roadmap con el active_track (que puede haber
    cambiado).
  - Archive roadmap actual con
    `archived_reason='significant_change_profile'`.
  - Marca el nuevo roadmap como active.
  - Emit `roadmap.regenerated` event.
- Cliente UI prompt: si user_facing significant change (cambio de
  primary_goals manual desde Settings), mobile app muestra modal
  "¿Quieres regenerar tu plan?" antes de PATCH (decisión del user
  controla emit del flag).
- Idempotency via event_id de Inngest (auto).
- Telemetry: `roadmap.regeneration_triggered`,
  `.regeneration_completed`.

### Out

- UI prompt en mobile app (story propia menor; o lectura del flag
  ya provee la info).
- Manual regeneration endpoint (post-MVP).

## Acceptance criteria

- **Given** event `profile.updated` con `significant_change=true`,
  **When** Inngest function ejecuta, **Then** archive current
  roadmap + genera nuevo con active_track actualizado.
- **Given** user con block_completions, **When** regenera, **Then**
  completions se preservan (no reset).
- **Given** sub-skills mastered en track anterior, **When** new
  roadmap incluye blocks con esas sub-skills, **Then** esos blocks
  son candidatos automáticos a test-out (frontend puede prompt).
- **Given** event sin significant_change flag o false, **When**
  function evalúa, **Then** skip (no regenera).
- **Given** AI Gateway falla mid-regeneration, **When** retry,
  **Then** fallback a roadmap genérico para
  (active_track, cefr).
- **Given** mismo event re-emitted (Inngest retry), **When** segunda
  vez, **Then** detect duplicate via Inngest id + skip.
- **Given** regeneration exitosa, **When** completes, **Then**
  `roadmap.regenerated` event emitido con
  `{ user_id, old_roadmap_id, new_roadmap_id, reason }`.
- **Given** cache active roadmap invalidated, **When** próximo
  GET, **Then** retorna nuevo roadmap.

## Developer details

### Owning service

`apps/workers/inngest-functions/regenerate-roadmap-on-significant-change.ts`.

### Dependencies

- US-084 emit event con flag.
- US-021/032 pattern de generation.

### Specs referenciados

- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md)
  §14.3.

### Implementación esperada

```typescript
export const regenerateRoadmapFn = inngest.createFunction(
  { id: 'regenerate-roadmap-on-significant-change', retries: 2 },
  { event: 'profile.updated' },
  async ({ event, step }) => {
    if (!event.data.significant_change) {
      return { skipped: 'not_significant' };
    }

    const { user_id } = event.data;
    const profile = await step.run('fetch-profile', () => db.getProfile(user_id));

    // Available blocks filtered for new context
    const availableBlocks = await step.run('filter-blocks', () =>
      filterBlocksForUser(profile)
    );

    // Archive current
    const oldRoadmap = await step.run('archive-current', async () => {
      const result = await db.query(`
        UPDATE roadmaps SET is_active = false, archived_at = now(),
          archived_reason = 'significant_change_profile'
        WHERE user_id = $1 AND is_active = true
        RETURNING id
      `, [user_id]);
      return result[0]?.id;
    });

    // Generate new
    let newRoadmap;
    try {
      const output = await aiGateway.invoke('generate_initial_roadmap', {
        profile, availableBlocks,
        mode: profile.assessment_completed_at ? 'definitive' : 'initial',
      }, { user_id });
      newRoadmap = await persistRoadmap(user_id, output,
        profile.assessment_completed_at ? 'definitive' : 'initial');
    } catch (error) {
      console.warn({ event: 'roadmap.regeneration_fallback', user_id });
      newRoadmap = await persistGenericFallbackRoadmap(user_id,
        profile.active_track, profile.cefr_estimate);
    }

    await env.ROADMAP_KV.delete(`roadmap_active:${user_id}`);

    await emitEvent('roadmap.regenerated', {
      user_id,
      old_roadmap_id: oldRoadmap,
      new_roadmap_id: newRoadmap,
      reason: 'significant_change',
    });

    track('roadmap.regeneration_completed');
    return { new_roadmap_id: newRoadmap };
  }
);
```

### Integration points

- US-084 event source.
- US-021/032 pattern.
- US-073 cache invalidation.

### Notas técnicas

- Block completions preservation: completions table no se toca; new
  roadmap simplemente referencia blocks que están en
  `roadmap_blocks` separately.
- Sub-skill mastery cross-track: ya garantizado por ADR-007
  (mastery es track-orthogonal).

## Definition of Done

- [ ] Inngest function implementada.
- [ ] Fallback handling.
- [ ] Atomic archive + new active.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-084. Driver de roadmap freshness.*

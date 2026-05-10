# US-077: Block start + status tracking

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3
**Story points:** 2
**Persona:** Estudiante / Admin (consumido por exercise sessions)
**Owner:** —

---

## Contexto

Tracking del lifecycle de un bloque para un user:
- `pending` → `in_progress` cuando user empieza el primer asset.
- `in_progress` → `completed` cuando alcanza mastery_criteria.
- `in_progress` → `abandoned` después de 14 días sin progreso.

Esta story implementa el **start** del bloque. La completion va
en US-078.

Backend en
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §11
+ [`pedagogical-system.md`](../../product/pedagogical-system.md).

## Scope

### In

- Endpoint `POST /roadmap/blocks/:block_id/start`:
  - Valida JWT.
  - Verifica block está en roadmap activo del user.
  - Verifica prerequisites cumplidos (mastery o block completion).
  - UPSERT en `block_completions` con `status = 'in_progress'`,
    `started_at = now()`.
  - Si ya estaba `in_progress`: no change, retorna current state.
  - Si ya `completed`: retorna current state sin re-iniciar.
  - Emit event `block.started`.
  - Invalidate cache.
- Cron `mark-abandoned-blocks`:
  - Trigger: diario.
  - Bloques con `status='in_progress'` y `started_at < now() - 14
    days` → mark `abandoned`.
- Telemetry: `block.start_requested`, `.start_succeeded`,
  `.start_prereqs_missing`.

### Out

- Completion logic (US-078).
- Test-out (US-079).

## Acceptance criteria

- **Given** user con block X pending y prereqs completed, **When**
  POST start, **Then** status='in_progress', started_at populated,
  retorna current state.
- **Given** mismo POST llamado 2x, **When** segundo, **Then**
  retorna current state sin reset (idempotent).
- **Given** block con prereqs NO completed, **When** start, **Then**
  400 `{ error: 'prerequisites_missing', missing: [...] }`.
- **Given** block ya `completed`, **When** start, **Then** 200 con
  current state (no re-inicia).
- **Given** block no está en roadmap del user, **When** start,
  **Then** 403.
- **Given** event `block.started` emitido, **When** consumed,
  **Then** payload incluye `{ user_id, block_id, started_at }`.
- **Given** cron mark-abandoned, **When** corre, **Then** bloques
  in_progress con started_at < 14 días → status='abandoned'.
- **Given** block abandoned, **When** user re-tries POST start,
  **Then** status vuelve a `in_progress` con nuevo started_at.

## Developer details

### Owning service

`apps/workers/api/handlers/block-start.ts` +
`apps/workers/crons/mark-abandoned-blocks.ts`.

### Dependencies

- US-072 + US-073.

### Specs referenciados

- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §11.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §4 — completion criteria.

### Implementación esperada

```typescript
// apps/workers/api/handlers/block-start.ts
export async function handleStartBlock(
  request: AuthedRequest, env: Env, blockId: string
) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  // Verify block in user's roadmap
  const inRoadmap = await env.DB.query(`
    SELECT lb.prerequisites FROM roadmap_blocks rb
    JOIN roadmaps r ON r.id = rb.roadmap_id
    JOIN learning_blocks lb ON lb.id = rb.block_id
    WHERE rb.block_id = $1 AND r.user_id = $2 AND r.is_active = true
  `, [blockId, userId]);

  if (inRoadmap.length === 0) {
    return jsonResponse({ error: 'block_not_in_roadmap' }, 403);
  }

  // Check prerequisites
  const prereqs = inRoadmap[0].prerequisites ?? [];
  if (prereqs.length > 0) {
    const completed = await env.DB.query(`
      SELECT block_id FROM block_completions
      WHERE user_id = $1 AND block_id = ANY($2::text[])
        AND status IN ('completed', 'skipped_test_out')
    `, [userId, prereqs]);
    const completedIds = new Set(completed.map((r: any) => r.block_id));
    const missing = prereqs.filter((p: string) => !completedIds.has(p));
    if (missing.length > 0) {
      track('block.start_prereqs_missing', { block_id: blockId, missing });
      return jsonResponse({ error: 'prerequisites_missing', missing }, 400);
    }
  }

  // Check current status
  const current = await env.DB.query(`
    SELECT status, started_at FROM block_completions
    WHERE user_id = $1 AND block_id = $2
  `, [userId, blockId]);

  if (current.length > 0 && current[0].status === 'completed') {
    return jsonResponse({
      status: current[0].status,
      message: 'already_completed',
    });
  }

  // UPSERT: in_progress
  track('block.start_requested', { block_id: blockId });
  await env.DB.execute(`
    INSERT INTO block_completions (user_id, block_id, status, started_at)
    VALUES ($1, $2, 'in_progress', now())
    ON CONFLICT (user_id, block_id) DO UPDATE
      SET status = 'in_progress',
          started_at = CASE
            WHEN block_completions.status = 'abandoned' THEN now()
            ELSE block_completions.started_at
          END,
          updated_at = now()
  `, [userId, blockId]);

  await env.ROADMAP_KV.delete(`roadmap_active:${userId}`);
  await emitEvent('block.started', {
    user_id: userId, block_id: blockId,
    started_at: new Date().toISOString(),
  });

  track('block.start_succeeded');
  return jsonResponse({ status: 'in_progress', started_at: new Date().toISOString() });
}

// apps/workers/crons/mark-abandoned-blocks.ts
export async function runMarkAbandonedCron(env: Env) {
  const result = await env.DB.query(`
    UPDATE block_completions
    SET status = 'abandoned', updated_at = now()
    WHERE status = 'in_progress'
      AND started_at < now() - interval '14 days'
    RETURNING user_id, block_id
  `);
  for (const row of result) {
    await emitEvent('block.abandoned', row);
  }
  track('blocks.abandoned_marked', { count: result.length });
  return { abandoned: result.length };
}
```

### Cron trigger

```toml
[triggers]
crons = ["0 5 * * *"]  # Diario 05:00 UTC
```

### Integration points

- US-072 (schema).
- US-073 (cache invalidation).
- US-074 (resolve consumer).
- US-078 (consume `block.started` para tracking exercise_attempts).

### Notas técnicas

- Idempotent UPSERT permite re-llamar safely.
- Re-start de abandoned reinicia started_at (clock reset).
- Cron 14 días alineado con session abandonment definition.

## Definition of Done

- [ ] Endpoint + cron implementados.
- [ ] Idempotency verificada.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-072. Bloqueante para US-078.*

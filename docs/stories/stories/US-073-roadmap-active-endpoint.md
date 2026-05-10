# US-073: GET /roadmap/active + listing endpoints

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 2
**Story points:** 2
**Persona:** Estudiante (consumido por home + reveal screens)
**Owner:** —

---

## Contexto

Endpoints HTTP que el cliente mobile usa para consultar:
- El roadmap activo del user.
- Lista de bloques con su status (completed, in_progress,
  pending).
- Próximo bloque sugerido.

Backend en
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §10.

## Scope

### In

- `GET /roadmap/active`:
  - Valida JWT.
  - Retorna roadmap activo del user con levels + blocks anidados
    + status de completion.
  - Incluye `next_block_id`: primero pending sin prereqs gap.
  - Incluye stats: `total_blocks`, `completed_blocks`,
    `percent_complete`, `estimated_remaining_weeks`.
  - Cache KV TTL 60s.
- `GET /roadmap/next-block`:
  - Retorna next block sugerido + asset_sequence + sparks_cost
    estimate.
- `GET /roadmap/blocks/:block_id`:
  - Detalle de bloque específico + status del user.
- Empty state: si user no tiene roadmap (raro post-onboarding):
  retorna `{ roadmap: null, message: 'roadmap_pending_generation' }`.
- Telemetry: `roadmap.viewed`, `roadmap.next_block_viewed`,
  `roadmap.block_viewed`.

### Out

- Endpoints de modificación (US-077+).
- Roadmap history endpoint (post-MVP).

## Acceptance criteria

- **Given** user con roadmap activo, **When** GET /roadmap/active,
  **Then** retorna estructura completa con levels[] + blocks[]
  + stats.
- **Given** user con 5 blocks completados de 50, **When** stats,
  **Then** `percent_complete: 10`, `completed_blocks: 5`.
- **Given** mismo request 2x consecutivo, **When** segundo,
  **Then** sirve de cache KV (latencia <30ms).
- **Given** user sin roadmap (post-signup pre-generation), **When**
  GET, **Then** retorna 200 con `roadmap: null,
  message: 'roadmap_pending_generation'`.
- **Given** roadmap con blocks pending pero prereqs no met, **When**
  next-block, **Then** retorna primero whose prereqs están
  completed (no gap).
- **Given** todos los blocks completed, **When** next-block,
  **Then** retorna `{ next_block_id: null, status:
  'roadmap_completed' }`.
- **Given** block_id no existe en roadmap del user, **When** GET
  /roadmap/blocks/:id, **Then** 404.
- **Given** request sin JWT, **When** llega, **Then** 401.

## Developer details

### Owning service

`apps/workers/api/handlers/roadmap-*.ts`.

### Dependencies

- US-072: schemas.
- US-007: JWT middleware.

### Specs referenciados

- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §10
  — API contracts.

### Implementación esperada

```typescript
// apps/workers/api/handlers/roadmap-active.ts
export async function handleGetActiveRoadmap(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  const cacheKey = `roadmap_active:${userId}`;
  const cached = await env.ROADMAP_KV.get(cacheKey);
  if (cached) {
    return new Response(cached, {
      headers: {
        'Content-Type': 'application/json',
        'Cache-Control': 'max-age=60, private',
      },
    });
  }

  const roadmap = await env.DB.query(`
    SELECT * FROM roadmaps WHERE user_id = $1 AND is_active = true
  `, [userId]);

  if (roadmap.length === 0) {
    return jsonResponse({ roadmap: null, message: 'roadmap_pending_generation' });
  }

  const roadmapId = roadmap[0].id;

  const levels = await env.DB.query(`
    SELECT l.*,
      json_agg(json_build_object(
        'block_id', rb.block_id,
        'order', rb.order_in_level,
        'personalization_reason', rb.personalization_reason,
        'block', json_build_object(
          'title', lb.title,
          'description', lb.description,
          'cefr', lb.cefr,
          'estimated_minutes', lb.estimated_minutes,
          'target_subskills', lb.target_subskills,
          'prerequisites', lb.prerequisites
        ),
        'completion', json_build_object(
          'status', COALESCE(bc.status, 'pending'),
          'started_at', bc.started_at,
          'completed_at', bc.completed_at,
          'score_avg', bc.score_avg
        )
      ) ORDER BY rb.order_in_level) AS blocks
    FROM roadmap_levels l
    LEFT JOIN roadmap_blocks rb ON rb.level_id = l.id
    LEFT JOIN learning_blocks lb ON lb.id = rb.block_id
    LEFT JOIN block_completions bc ON bc.block_id = rb.block_id AND bc.user_id = $1
    WHERE l.roadmap_id = $2
    GROUP BY l.id
    ORDER BY l.level_index
  `, [userId, roadmapId]);

  // Stats
  const allBlocks = levels.flatMap((l: any) => l.blocks);
  const completed = allBlocks.filter((b: any) =>
    b.completion.status === 'completed' || b.completion.status === 'skipped_test_out'
  ).length;
  const total = allBlocks.length;

  // Next block: first pending with all prereqs completed
  const completedIds = new Set(allBlocks
    .filter((b: any) => b.completion.status === 'completed' || b.completion.status === 'skipped_test_out')
    .map((b: any) => b.block_id));

  const nextBlock = allBlocks.find((b: any) =>
    b.completion.status === 'pending'
    && b.block.prerequisites.every((p: string) => completedIds.has(p))
  );

  const response = {
    roadmap: {
      ...roadmap[0],
      levels,
      stats: {
        total_blocks: total,
        completed_blocks: completed,
        percent_complete: total > 0 ? Math.floor((completed / total) * 100) : 0,
        estimated_remaining_weeks: Math.ceil(
          (total - completed) / Math.max(1, total / roadmap[0].estimated_completion_weeks)
        ),
      },
      next_block_id: nextBlock?.block_id ?? null,
    },
  };

  await env.ROADMAP_KV.put(cacheKey, JSON.stringify(response), {
    expirationTtl: 60,
  });

  track('roadmap.viewed', { user_id: userId });
  return jsonResponse(response);
}

// apps/workers/api/handlers/roadmap-next-block.ts
export async function handleGetNextBlock(request: AuthedRequest, env: Env) {
  // Similar logic but returns just the next block + asset_sequence
  // Cache: don't cache (changes too frequently as user completes blocks)
}
```

### Integration points

- US-022 (roadmap reveal screen consumer).
- US-024 (first exercise intro consumer).
- Home screen del producto (consumer principal).
- US-077, US-078 (invalidate cache al cambiar status).

### Notas técnicas

- Cache TTL 60s: balance entre freshness y carga (estructura
  cambia lenta).
- JSON aggregation en single query previene N+1.
- Next block computed server-side para mantener single source.

## Definition of Done

- [ ] 3 endpoints implementados.
- [ ] Cache funcional.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Performance: P50 < 50ms con cache, < 200ms sin.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-072. Bloqueante para US-077+ y consumers del
producto.*

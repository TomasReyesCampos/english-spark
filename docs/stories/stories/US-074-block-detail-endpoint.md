# US-074: Block detail + asset references endpoint

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 2
**Story points:** 1
**Persona:** Estudiante (consume al abrir bloque)
**Owner:** —

---

## Contexto

Endpoint que resuelve un bloque específico con todos sus assets
listos para presentar. Cliente lo llama al "abrir" un bloque para
empezar ejercicios.

Backend en
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §10.

## Scope

### In

- `GET /roadmap/blocks/:block_id/resolve`:
  - Valida JWT.
  - Verifica que el block está en el roadmap activo del user.
  - Resuelve cada asset del `asset_sequence` con sus URLs
    finales (R2 signed URLs para audio/image/video).
  - Retorna:
    - `block`: detalle del bloque.
    - `assets`: array resuelto con URLs accesibles.
    - `current_completion`: status del user (pending /
      in_progress / etc).
    - `sparks_cost_estimate`: costo total estimado del bloque.
- Signed URLs con TTL 30 min (suficiente para session de
  ejercicios).
- Telemetry: `block.resolved`.

### Out

- Streaming de assets (post-MVP — MVP retorna URLs static).

## Acceptance criteria

- **Given** user con block X en su roadmap, **When** GET resolve,
  **Then** retorna block + assets con signed URLs accesibles
  desde R2.
- **Given** block_id no está en roadmap del user, **When** GET,
  **Then** 403 `{ error: 'block_not_in_roadmap' }`.
- **Given** block aún tiene prereqs sin completar, **When** GET,
  **Then** retorna OK pero con flag
  `prerequisites_missing: ['block_x']` (cliente decide si
  bloquea entry o solo warning).
- **Given** signed URL fetched, **When** se llama el storage_key
  desde R2, **Then** retorna binary content (audio/image/etc).
- **Given** TTL signed URL = 30 min, **When** user vuelve después
  de 1h, **Then** URLs expiraron — re-call resolve genera nuevas.
- **Given** block_id no existe en learning_blocks, **When** GET,
  **Then** 404.
- **Given** request sin JWT, **When** llega, **Then** 401.
- **Given** atomic referenciado en asset_sequence no está
  approved, **When** resolve, **Then** SEV-2 log + retorna error
  (no servir atomic no aprobado).

## Developer details

### Owning service

`apps/workers/api/handlers/roadmap-block-resolve.ts`.

### Dependencies

- US-072 + US-073.
- Storage signed URL helper (de US-017 ya implementado).

### Specs referenciados

- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §10.
- [`content-creation-system.md`](../../product/content-creation-system.md)
  §6 — estructura learning_block.

### Implementación esperada

```typescript
// apps/workers/api/handlers/roadmap-block-resolve.ts
export async function handleResolveBlock(
  request: AuthedRequest, env: Env, blockId: string
) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  // Verify block is in user's active roadmap
  const inRoadmap = await env.DB.query(`
    SELECT 1 FROM roadmap_blocks rb
    JOIN roadmaps r ON r.id = rb.roadmap_id
    WHERE rb.block_id = $1 AND r.user_id = $2 AND r.is_active = true
  `, [blockId, userId]);

  if (inRoadmap.length === 0) {
    return jsonResponse({ error: 'block_not_in_roadmap' }, 403);
  }

  // Fetch block details
  const block = await env.DB.query(`
    SELECT * FROM learning_blocks WHERE id = $1 AND approved_for_production = true
  `, [blockId]);
  if (block.length === 0) {
    return jsonResponse({ error: 'block_not_found' }, 404);
  }

  // Resolve assets
  const assetSequence = block[0].asset_sequence; // JSONB array
  const resolvedAssets = [];
  for (const asset of assetSequence) {
    const composite = await env.DB.query(`
      SELECT * FROM learning_assets WHERE id = $1 AND approved_for_production = true
    `, [asset.asset_id]);

    if (composite.length === 0) {
      console.error({ event: 'block.asset_not_approved', asset_id: asset.asset_id });
      return jsonResponse({ error: 'asset_unavailable', asset_id: asset.asset_id }, 500);
    }

    // Resolve atomic URLs
    const atomicRefs = composite[0].atomic_refs ?? []; // assume schema
    const resolvedAtomics: any[] = [];
    for (const atomicId of atomicRefs) {
      const atomic = await env.DB.query(`
        SELECT id, media_format, storage_key, duration_ms, width, height
        FROM media_atomics
        WHERE id = $1 AND approved = true AND archived = false
      `, [atomicId]);
      if (atomic.length === 0) continue;
      const signedUrl = await generateR2SignedUrl(atomic[0].storage_key, 1800, env);
      resolvedAtomics.push({ ...atomic[0], url: signedUrl });
    }

    resolvedAssets.push({
      ...composite[0],
      order: asset.order,
      optional: asset.optional,
      atomics: resolvedAtomics,
    });
  }

  // Current completion status
  const completion = await env.DB.query(`
    SELECT * FROM block_completions WHERE user_id = $1 AND block_id = $2
  `, [userId, blockId]);

  // Check prerequisites
  const prereqs = block[0].prerequisites ?? [];
  const completedPrereqs = await env.DB.query(`
    SELECT block_id FROM block_completions
    WHERE user_id = $1 AND block_id = ANY($2::text[])
      AND status IN ('completed', 'skipped_test_out')
  `, [userId, prereqs]);
  const completedIds = new Set(completedPrereqs.map((r: any) => r.block_id));
  const missingPrereqs = prereqs.filter((p: string) => !completedIds.has(p));

  track('block.resolved', { block_id: blockId, user_id: userId });

  return jsonResponse({
    block: block[0],
    assets: resolvedAssets,
    current_completion: completion[0] ?? { status: 'pending' },
    prerequisites_missing: missingPrereqs,
    sparks_cost_estimate: computeSparksCost(resolvedAssets),
  });
}

function computeSparksCost(assets: any[]): number {
  // Roughly: 1 Sparks per assets that require AI scoring (free_response, roleplay)
  return assets.reduce((sum, a) => {
    const interaction = a.labels?.interaction_type;
    if (['free_response', 'roleplay_structured', 'roleplay_free'].includes(interaction)) {
      return sum + 1;
    }
    return sum;
  }, 0);
}
```

### Integration points

- US-073 (consumed por flow del cliente).
- US-077 (consumer al start de bloque).
- R2 signed URL generation.

### Notas técnicas

- Resolución de assets puede ser cara (múltiples queries). Cache
  por composite si volumen lo requiere (post-MVP).
- Signed URLs 30 min TTL: suficiente para session normal.
- `prerequisites_missing` es info (cliente decide flow), no
  bloquea endpoint.

## Definition of Done

- [ ] Endpoint implementado.
- [ ] Signed URLs funcionales.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-072 + US-073. Consumer principal: exercise session
del producto.*

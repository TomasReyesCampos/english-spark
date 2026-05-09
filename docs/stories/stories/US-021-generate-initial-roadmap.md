# US-021: Generate initial roadmap (AI Gateway integration)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 5
**Persona:** Estudiante (consumido a través del onboarding)
**Owner:** —

---

## Contexto

Después del mini-test consolidado (US-020), generamos el **roadmap
inicial provisional** del user usando IA. Este roadmap es:
- **Provisional:** se updateará al Day 7 con el assessment formal.
- **Personalizado:** usa `student_profile` completo (goals, CEFR
  estimate, anxiety, deadline, professional_field) + el track
  activo para seleccionar bloques.
- **Acotado:** 30-40 bloques, ~4 semanas estimated, foco general.

UX del reveal en US-022. Esta story cubre la **generación
backend**.

Backend en
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §5
(initial roadmap generation).

## Scope

### In

- Endpoint backend `POST /roadmap/generate-initial` (interno,
  triggered por event `initial_test.completed`):
  - Lee profile completo del user.
  - Filtra bloques disponibles del `active_track` por:
    - CEFR ≤ user.cefr_estimate + 1.
    - prerequisites cumplidos (intersección con sub-skills
      mastered).
    - variantes por `professional_field` cuando aplique.
  - Llama AI Gateway task `generate_initial_roadmap` con:
    - Profile.
    - Available blocks filtered.
    - Output schema (estructura JSON esperada).
  - Valida output con Zod.
  - Persiste en tabla `roadmaps` con `roadmap_type = 'initial'`.
  - Persiste en `roadmap_blocks` (orden + niveles temáticos).
  - Trigger event `roadmap.generated` con
    `{ user_id, roadmap_id }`.
- Schema sugerido para `roadmaps` y `roadmap_blocks` (a definir si
  no existe ya en migrations previas).
- Manejo de errors:
  - LLM timeout: retry 1x con primary, fallback a secondary.
  - LLM output inválido (Zod fail): retry 1x. Si falla 2x:
    fallback a roadmap genérico hardcoded para ese CEFR + track.
  - Profile incompleto: error 400 con `{ error: 'profile_incomplete' }`.
- Telemetry: `roadmap.initial_generation_started`,
  `.initial_generation_completed`, `.initial_generation_failed`,
  `.initial_generation_fallback_used`.

### Out

- Roadmap reveal UI (US-022).
- Roadmap update post-assessment Day 7 (story propia).
- Edición manual del roadmap por el user (post-MVP, decision
  cerrada en `pendientes.md`).

## Acceptance criteria

- **Given** event `initial_test.completed` emitido para un user
  con profile completo, **When** handler procesa, **Then** se
  genera un roadmap, persiste en BD, y emite `roadmap.generated`.
- **Given** user con `active_track = 'job_ready'` y
  `cefr_estimate = 'B1'`, **When** se generan los blocks, **Then**
  el roadmap incluye bloques de `track-job-ready-blocks.md` slices
  B1 + B1+ relevantes (no B2+ todavía).
- **Given** `professional_field = 'tech'`, **When** se asigna la
  variante de `jr_b1_industry_vocab_intro`, **Then** se asigna el
  variant `_tech` (no `_finance`, etc.).
- **Given** profile con `language_anxiety = 5` (alta), **When** se
  generan blocks, **Then** los primeros 5 bloques son de baja
  presión (categoría easier-end, no roleplays con AI directos).
- **Given** AI Gateway timeout 1ra vez, **When** retry, **Then**
  segundo intento usa secondary provider (configurado en
  `ai-gateway-strategy.md`).
- **Given** AI Gateway falla 2x, **When** fallback, **Then** se usa
  roadmap genérico hardcoded para `(B1, job_ready)` y user no se
  bloquea.
- **Given** profile incompleto (falta `cefr_estimate`), **When**
  endpoint, **Then** retorna 400 con error claro.
- **Given** roadmap generado, **When** se persiste, **Then** la
  estructura matchea `roadmaps` y `roadmap_blocks` schemas con
  `roadmap_type = 'initial'`, `is_active = true`.

## Developer details

### Owning service

`apps/workers/api` (handler) +
`apps/workers/inngest-functions` (event consumer).

### Dependencies

- US-020: trigger event.
- AI Gateway con task `generate_initial_roadmap`.
- Bloques catalogados en
  `learning_blocks` (parte de la creación de contenido — fuera de
  EPIC-01, pero el seed mínimo para los 3 tracks B1 debe estar
  para test).
- Tabla `roadmaps` y `roadmap_blocks` (story de schema separada;
  asumimos disponibles).

### Specs referenciados

- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §5.
- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §5.1
  — prompt template autoritativo.
- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  — task registry y fallback.
- [`decisions/ADR-007-multi-track-strategy.md`](../../decisions/ADR-007-multi-track-strategy.md)
  — single-track filtering.
- [`track-job-ready-blocks.md`](../../product/track-job-ready-blocks.md),
  [`track-travel-confident-blocks.md`](../../product/track-travel-confident-blocks.md),
  [`track-daily-conversation-blocks.md`](../../product/track-daily-conversation-blocks.md).

### Implementación esperada

```typescript
// apps/workers/inngest-functions/on-initial-test-completed.ts
import { Inngest } from 'inngest';

export const generateInitialRoadmapFn = inngest.createFunction(
  { id: 'generate-initial-roadmap', retries: 2 },
  { event: 'initial_test.completed' },
  async ({ event, step }) => {
    const { user_id } = event.data;

    const profile = await step.run('fetch-profile', () =>
      db.getProfileById(user_id)
    );
    if (!isProfileComplete(profile)) {
      throw new NonRetriableError('profile_incomplete');
    }

    const availableBlocks = await step.run('filter-blocks', () =>
      filterBlocksForUser(profile)
    );

    const llmOutput = await step.run('generate-roadmap-llm', async () => {
      try {
        return await aiGateway.invoke('generate_initial_roadmap', {
          profile, availableBlocks,
        });
      } catch (err) {
        // Fallback a generic
        return getGenericFallbackRoadmap(profile.active_track, profile.cefr_estimate);
      }
    });

    const validated = RoadmapSchema.parse(llmOutput);

    const roadmapId = await step.run('persist-roadmap', () =>
      db.persistInitialRoadmap(user_id, validated)
    );

    await step.run('emit-event', () =>
      emitEvent('roadmap.generated', { user_id, roadmap_id: roadmapId }));

    track('roadmap.initial_generation_completed', { user_id });
    return { roadmap_id: roadmapId };
  }
);
```

### Filtering logic

```typescript
function filterBlocksForUser(profile: StudentProfile): Block[] {
  const allBlocks = getBlocksForTrack(profile.active_track);

  return allBlocks.filter(block => {
    // CEFR filter
    const blockLevelOrder = cefrToOrder(block.cefr_level);
    const userLevelOrder = cefrToOrder(profile.initial_test_results.cefr_estimate);
    if (blockLevelOrder > userLevelOrder + 1) return false;

    // Prerequisites
    const prereqsMet = block.prerequisites.every(prereqId =>
      profile.completed_blocks?.includes(prereqId) ?? false
    );
    if (!prereqsMet && blockLevelOrder > userLevelOrder) return false;

    // Field variants
    if (block.variants && profile.professional_field) {
      return block.variants.some(v => v.field === profile.professional_field);
    }

    return true;
  });
}
```

### Schemas afectados

```sql
-- migrations/0XXX_roadmaps.sql (puede ser story propia anterior)
CREATE TABLE roadmaps (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  roadmap_type    TEXT NOT NULL CHECK (roadmap_type IN ('initial', 'definitive')),
  active_track    TEXT NOT NULL,
  cefr_at_generation TEXT NOT NULL,
  ai_summary      TEXT,
  estimated_completion_weeks INT,
  is_active       BOOLEAN NOT NULL DEFAULT true,
  archived_at     TIMESTAMPTZ,
  archived_reason TEXT,
  generated_by    TEXT NOT NULL,            -- 'ai_haiku' | 'fallback_generic'
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roadmap_blocks (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  roadmap_id      UUID NOT NULL REFERENCES roadmaps(id) ON DELETE CASCADE,
  block_id        TEXT NOT NULL,
  level_index     INT NOT NULL,             -- agrupación en niveles temáticos
  level_name      TEXT,
  order_in_level  INT NOT NULL,
  personalization_reason TEXT,
  status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'in_progress', 'completed', 'skipped'))
);
```

### Integration points

- Inngest functions (event-driven generation).
- AI Gateway.
- Postgres (roadmaps + roadmap_blocks).
- Event bus → US-022 (consumer del `roadmap.generated`).

### Notas técnicas

- Retry budget: 2 retries con exponential backoff. Fallback nunca
  falla (es deterministic generic).
- Fallback genérico se debe pre-poblar con 1 roadmap por
  `(track, cefr)` combination — fuera de scope de esta story
  pero coordinar.
- Schema persist es **transactional** — o se persiste todo o
  ninguno.

## Definition of Done

- [ ] Inngest function implementada.
- [ ] AI Gateway task `generate_initial_roadmap` registrada.
- [ ] Filtering logic correcto.
- [ ] Fallback genérico funciona.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit de filtering + Zod parsing.
- [ ] Test integration end-to-end (event triggers function,
  roadmap persisted).
- [ ] Performance: generation completes <12s P95 (timeout 15s).
- [ ] Validation contra spec `ai-roadmap-system.md` §5.
- [ ] PR aprobada y mergeada.

---

*Depende de US-020. Bloqueante para US-022.*

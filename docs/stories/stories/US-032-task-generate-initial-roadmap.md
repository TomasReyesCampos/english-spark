# US-032: Task `generate_initial_roadmap`

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 1
**Story points:** 5
**Persona:** Admin (consumido por Estudiante via roadmap engine)
**Owner:** —

---

## Contexto

Primera task LLM-style del Registry: genera el roadmap inicial
provisional cuando user completa onboarding (US-021 de EPIC-01).
Recibe profile + bloques disponibles, retorna estructura JSON con
30-40 bloques organizados en 4-6 niveles temáticos.

Esta task **desbloquea US-021** (Initial roadmap generation) de
EPIC-01.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4.2.4 (tasks de roadmap) +
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §5.1
(prompt template autoritativo).

## Scope

### In

- Registrar task `generate_initial_roadmap` en `task_registry`:
  - Provider chain: `[{anthropic, claude-haiku-4-5, weight: 100},
                      {openai, gpt-4o-mini, weight: 0}]`.
  - Budget: $0.05 per call.
  - Timeout: 12 segundos.
  - Activate (`is_active = true`).
- Prompt template en `apps/workers/ai-gateway/prompts/generate_initial_roadmap/v1_anthropic.txt`
  basado en spec `ai-roadmap-system.md` §5.1.
- Input schema (Zod):
  ```typescript
  {
    profile: { active_track, cefr_estimate, primary_goals, language_anxiety,
               daily_minutes_available, professional_field, deadline_date },
    available_blocks: Array<{ id, title, cefr, target_subskills, prerequisites }>,
  }
  ```
- Output schema (Zod):
  ```typescript
  {
    levels: Array<{
      index: number,
      name: string,                  // "Foundation"
      ai_reasoning: string,
      blocks: Array<{
        block_id: string,
        order: number,
        personalization_reason: string,
      }>,
    }>,
    estimated_completion_weeks: number,
    ai_summary: string,              // 2-3 frases user-facing
  }
  ```
- Handler `apps/workers/ai-gateway/src/handlers/generate-initial-roadmap.ts`:
  - Carga prompt template.
  - Renderiza con placeholders del input.
  - Invoca `invokeWithFallback`.
  - Parsea response JSON.
  - Valida output con Zod.
  - Retorna structured output.
- Si LLM retorna JSON malformado: retry 1x. Si segundo fail:
  throws `validation_failed`.
- Tests con golden datasets:
  - 5 perfiles tipo (B1 job seeker, B2 traveler, B1+ casual, etc.).
  - Validar que output incluye bloques apropiados al CEFR + track.

### Out

- Generación del roadmap definitivo Day 7 (task propia post-MVP,
  `generate_definitive_roadmap`).
- Persistence en BD del roadmap generado (eso es US-021 de
  EPIC-01).

## Acceptance criteria

- **Given** task registrada y activa, **When** se invoca `POST /ai/invoke`
  con `task_id: 'generate_initial_roadmap'` y profile válido,
  **Then** retorna 200 con output que cumple Zod schema.
- **Given** profile con `active_track = 'job_ready', cefr =
  'B1'`, **When** task ejecuta, **Then** todos los `block_id` del
  output están en `available_blocks` del input (no inventa
  bloques).
- **Given** profile con `language_anxiety = 5`, **When** generación,
  **Then** los primeros 5 bloques del primer nivel tienen
  `target_subskills` que NO son roleplay-heavy (anxiety-aware).
- **Given** profile con `daily_minutes_available = 5`, **When**
  generación, **Then** `estimated_completion_weeks` es ≥ 6 weeks
  (slow track).
- **Given** profile con `professional_field = 'tech'`, **When**
  Job Ready track tiene variantes industry, **Then** la variante
  `_tech` se selecciona si existe en available_blocks.
- **Given** LLM retorna JSON parseable pero con block_ids inventados,
  **When** validation runs, **Then** retorna error `invalid_output`
  y NO se persiste.
- **Given** LLM rate-limited 429, **When** orchestrator gestiona,
  **Then** fallback a OpenAI gpt-4o-mini con mismo prompt.
- **Given** task se invoca 100x con perfiles distintos, **When**
  budget tracking, **Then** spending se registra correctamente y
  promedia ~$0.03-$0.05 per call.

## Developer details

### Owning service

`apps/workers/ai-gateway/src/handlers/generate-initial-roadmap.ts`.

### Dependencies

- US-027/028/029/030/031 (foundation completa).
- Catálogo de bloques disponibles (parte de EPIC-02).

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.2.4 — task registration.
- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §5.1
  — prompt template autoritativo.
- [`track-job-ready-blocks.md`](../../product/track-job-ready-blocks.md),
  [`track-travel-confident-blocks.md`](../../product/track-travel-confident-blocks.md),
  [`track-daily-conversation-blocks.md`](../../product/track-daily-conversation-blocks.md).

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/handlers/generate-initial-roadmap.ts
const InputSchema = z.object({
  profile: z.object({
    active_track: z.enum(['job_ready', 'travel_confident', 'daily_conversation']),
    cefr_estimate: z.enum(['A2', 'B1', 'B1+', 'B2', 'B2+', 'C1']),
    primary_goals: z.array(z.string()).max(3),
    language_anxiety: z.number().int().min(1).max(5),
    daily_minutes_available: z.number(),
    professional_field: z.string().nullable(),
    deadline_date: z.string().datetime().nullable(),
  }),
  available_blocks: z.array(z.object({
    id: z.string(),
    title: z.string(),
    cefr: z.string(),
    target_subskills: z.array(z.string()),
    prerequisites: z.array(z.string()),
  })),
});

const OutputSchema = z.object({
  levels: z.array(z.object({
    index: z.number().int(),
    name: z.string().max(50),
    ai_reasoning: z.string().max(200),
    blocks: z.array(z.object({
      block_id: z.string(),
      order: z.number().int(),
      personalization_reason: z.string().max(150),
    })).min(3).max(15),
  })).min(3).max(6),
  estimated_completion_weeks: z.number().int().min(2).max(20),
  ai_summary: z.string().max(300),
});

export async function handleGenerateInitialRoadmap(
  rawInput: any,
  context: HandlerContext
): Promise<z.infer<typeof OutputSchema>> {
  const input = InputSchema.parse(rawInput);

  const promptTemplate = await loadPromptTemplate(
    'generate_initial_roadmap', 'v1_anthropic', context.env
  );
  const renderedPrompt = renderTemplate(promptTemplate, {
    profile: JSON.stringify(input.profile, null, 2),
    available_blocks: JSON.stringify(input.available_blocks, null, 2),
  });

  let attempts = 0;
  while (attempts <= 1) {
    const { output, metadata } = await invokeWithFallback(
      'generate_initial_roadmap_llm_call',
      {
        system_prompt: renderedPrompt.system,
        messages: [{ role: 'user', content: renderedPrompt.user }],
        temperature: 0.3,
        max_tokens: 3000,
        response_format: 'json',
      },
      context
    );

    try {
      const parsed = JSON.parse(output.text);
      const validated = OutputSchema.parse(parsed);

      // Verify all block_ids exist in available_blocks
      const availableIds = new Set(input.available_blocks.map(b => b.id));
      const invalidBlocks = validated.levels
        .flatMap(l => l.blocks)
        .filter(b => !availableIds.has(b.block_id));
      if (invalidBlocks.length > 0) {
        throw new Error(`hallucinated_blocks: ${invalidBlocks.map(b => b.block_id).join(',')}`);
      }

      return validated;
    } catch (error) {
      if (attempts < 1) {
        attempts++;
        continue;
      }
      throw new ProviderError('invalid_output', `LLM output validation failed: ${error.message}`);
    }
  }
  throw new Error('unreachable');
}
```

### Prompt template (resumen, ver spec autoritativo)

`apps/workers/ai-gateway/prompts/generate_initial_roadmap/v1_anthropic.txt`:

```
SYSTEM:
Eres un experto en pedagogía de inglés para hispanohablantes
latinoamericanos. Generas roadmaps personalizados respetando:
- Solo selecciona blocks de la BIBLIOTECA DISPONIBLE (no inventes).
- Respeta prerequisites de cada block.
- Si language_anxiety >= 4: primeros bloques de baja presión.
- Si daily_minutes_available <= 10: estimated_completion_weeks
  multiplicado x2.5.

USER:
PERFIL DEL USUARIO:
{{profile}}

BIBLIOTECA DISPONIBLE:
{{available_blocks}}

INSTRUCCIONES:
1. Selecciona entre 30 y 50 bloques que cubran los objetivos.
2. Organízalos en 4-6 niveles temáticos con nombres motivadores.
3. Respeta los prerequisites.
4. Prioriza bloques que ataquen los errores detectados.
5. Para cada bloque escribe UNA frase de personalization_reason.
6. Para cada nivel escribe UNA frase de ai_reasoning.
7. Genera ai_summary de 2-3 frases user-facing.

DEVUELVE JSON exacto siguiendo este schema:
{{output_schema}}
```

### Integration points

- US-021 de EPIC-01 (consumer principal).
- Task Registry (US-028).
- Orchestrator (US-030).
- Cost tracking (US-031).

### Notas técnicas

- `temperature: 0.3` da consistencia (no es generación creativa).
- `response_format: 'json'` fuerza JSON estructurado en Anthropic.
- Si el output_schema es muy grande, considerar incluirlo como
  string en el prompt (no hardcoded).
- Golden datasets para regression testing post-MVP.

## Definition of Done

- [ ] Task registrada en BD con `is_active = true`.
- [ ] Handler implementado.
- [ ] Prompt template versionado en repo.
- [ ] Zod schemas input + output.
- [ ] Hallucination detection (block_ids no inventados).
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit del handler con LLM mocked.
- [ ] Tests integration con golden datasets (5 perfiles).
- [ ] Cost real ≤ $0.05 promedio (verificable post-deploy).
- [ ] Validation contra spec
  `ai-roadmap-system.md` §5.1.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-021 de EPIC-01.*

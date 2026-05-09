# US-040: Task `detect_grammar_errors`

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 1-2
**Story points:** 3
**Persona:** Admin (consumido por pedagogical engine)
**Owner:** —

---

## Contexto

Detectar errores gramaticales en transcripts del user durante
ejercicios. Devuelve lista estructurada de errores con tipo,
posición, y sugerencia de corrección.

Esta task es consumida por **EPIC-03 Pedagogical engine** para
scoring de grammar dimension. También es útil para feedback
constructivo en roleplays.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4.2.3.

## Scope

### In

- Implementar parte LLM de **OpenAIAdapter.invokeLLM** (completa el
  stub de US-034).
- Registrar task `detect_grammar_errors`:
  - Provider chain: `[{anthropic, claude-haiku-4-5, weight: 100},
                      {openai, gpt-4o-mini, weight: 0}]`.
  - Budget: $0.003 per call.
  - Timeout: 10s.
  - Active: true.
- Input schema:
  ```typescript
  {
    transcript: string,
    target_cefr?: string,            // 'B1', 'B2' — adjust strictness
    target_subskills?: string[],     // ['gram_present_perfect', ...]
  }
  ```
- Output schema:
  ```typescript
  {
    error_count: number,
    errors: Array<{
      error_type: string,            // 'verb_tense', 'subject_verb_agreement', etc.
      excerpt: string,               // "I have went"
      position_start: number,
      position_end: number,
      correction: string,            // "I have gone" / "I went"
      explanation: string,            // brief learner-friendly explanation in Spanish
      severity: 'minor' | 'major',
    }>,
    error_density: number,           // errors per 100 words
  }
  ```
- Prompt template adaptado al CEFR target:
  - B1: solo flagear errores major; ignorar minor (preserva
    confianza).
  - B2+: flagear minor también.
- Si target_subskills provistos: priorizar errores de esas
  sub-skills en el output.

### Out

- Real-time correction during recording (post-MVP).
- Spelling correction (out of scope — solo grammar).
- Style suggestions (post-MVP, ver `pedagogical-system.md` §3.3).

## Acceptance criteria

- **Given** task registrada, **When** se invoca con transcript "I
  have went to the store yesterday", **Then** retorna
  `error_count >= 1` con error tipo verb_tense (mixing have
  + past simple "yesterday").
- **Given** transcript perfecto, **When** task analiza, **Then**
  retorna `error_count: 0, errors: []`.
- **Given** target_cefr='B1' y transcript con minor error
  (article missing donde es opcional), **When** task evalúa,
  **Then** ese error NO aparece en output (B1 leniency).
- **Given** target_cefr='B2+' con mismo transcript, **When**
  evalúa, **Then** SÍ aparece el minor error.
- **Given** target_subskills=['gram_present_perfect'], **When**
  output, **Then** errores de present perfect aparecen primero en
  el array.
- **Given** transcript en español ("Yo fue al mercado"), **When**
  procesa, **Then** retorna error_density alto y errors descriptivos
  ("not English").
- **Given** invocation con transcript de 50 palabras, **When**
  cost, **Then** `cost_usd ≈ 0.001-0.003`.
- **Given** Anthropic falla 429, **When** orchestrator, **Then**
  fallback a OpenAI gpt-4o-mini (provider chain weight 0 = fallback
  only).

## Developer details

### Owning service

`apps/workers/ai-gateway/src/handlers/detect-grammar-errors.ts` +
extender OpenAIAdapter de US-034.

### Dependencies

- US-027/028/029/030/031.
- AnthropicAdapter (US-029).
- OpenAIAdapter (US-034 + extensión LLM aquí).

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.2.3.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §3.3 — grammar scoring.

### Implementación OpenAIAdapter LLM (extiende US-034)

```typescript
// apps/workers/ai-gateway/src/adapters/openai.ts (extension)
async invokeLLM(input: LLMInput, config: ProviderConfig): Promise<LLMOutput> {
  const start = Date.now();

  try {
    const response = await this.client.chat.completions.create({
      model: config.model,
      messages: [
        ...(input.system_prompt ? [{ role: 'system', content: input.system_prompt }] : []),
        ...input.messages,
      ],
      temperature: input.temperature ?? 0.7,
      max_tokens: input.max_tokens ?? 2048,
      response_format: input.response_format === 'json'
        ? { type: 'json_object' } : undefined,
    });

    const text = response.choices[0].message.content ?? '';
    return {
      text,
      usage_input_tokens: response.usage?.prompt_tokens ?? 0,
      usage_output_tokens: response.usage?.completion_tokens ?? 0,
      latency_ms: Date.now() - start,
    };
  } catch (error: any) {
    if (error.status === 429) throw new ProviderError('rate_limited', error.message);
    if (error.status === 503) throw new ProviderError('provider_unavailable', error.message);
    if (error.status === 400) throw new ProviderError('invalid_request', error.message);
    throw new ProviderError('unknown_provider_error', error.message);
  }
}

estimateCost(input: any, config: ProviderConfig): number {
  if (config.model === 'gpt-4o-mini') {
    const pricing = { input: 0.15 / 1_000_000, output: 0.60 / 1_000_000 };
    return (input.usage_input_tokens ?? 0) * pricing.input
         + (input.usage_output_tokens ?? 0) * pricing.output;
  }
  // ... otros modelos
  return 0;
}
```

### Handler

```typescript
// apps/workers/ai-gateway/src/handlers/detect-grammar-errors.ts
export async function handleDetectGrammarErrors(input: any, context: HandlerContext) {
  const validated = InputSchema.parse(input);
  const promptTemplate = await loadPromptTemplate('detect_grammar_errors', 'v1_anthropic', context.env);

  const userPrompt = renderTemplate(promptTemplate.user, {
    transcript: validated.transcript,
    target_cefr: validated.target_cefr ?? 'B1',
    leniency: validated.target_cefr === 'B1' ? 'major_only' : 'all_errors',
    subskill_priority: validated.target_subskills?.join(', ') ?? 'none',
    output_schema: JSON.stringify(OutputSchema.shape, null, 2),
  });

  const result = await invokeWithFallback('detect_grammar_errors', {
    system_prompt: promptTemplate.system,
    messages: [{ role: 'user', content: userPrompt }],
    temperature: 0.2,
    max_tokens: 1500,
    response_format: 'json',
  }, context);

  const parsed = OutputSchema.parse(JSON.parse(result.output.text));

  // Compute error_density
  const wordCount = validated.transcript.split(/\s+/).filter(Boolean).length;
  const errorDensity = wordCount > 0 ? (parsed.error_count / wordCount) * 100 : 0;

  // Re-order: subskill priority first
  if (validated.target_subskills) {
    parsed.errors.sort((a, b) => {
      const aPriority = validated.target_subskills.some(s => a.error_type.includes(s.replace('gram_', '')));
      const bPriority = validated.target_subskills.some(s => b.error_type.includes(s.replace('gram_', '')));
      return (bPriority ? 1 : 0) - (aPriority ? 1 : 0);
    });
  }

  return { ...parsed, error_density: errorDensity };
}
```

### Prompt template (resumen)

```
SYSTEM:
You are a grammar checker for English language learners (Spanish
speakers in Latin America). Detect grammatical errors in the
provided transcript.

USER:
TRANSCRIPT:
{{transcript}}

TARGET CEFR LEVEL: {{target_cefr}}
LENIENCY: {{leniency}}  // 'major_only' for B1, 'all_errors' for B2+
SUBSKILL PRIORITY: {{subskill_priority}}

INSTRUCTIONS:
1. Identify grammatical errors only (not spelling, not style).
2. For B1 students: flag only major errors. Skip minor stylistic issues.
3. For each error: provide error_type (e.g., 'verb_tense',
   'subject_verb_agreement'), excerpt, correction, and a brief
   explanation IN SPANISH for the learner (~1 sentence,
   mexicano-tuteo).
4. Set severity: 'major' (changes meaning, blocks comprehension)
   vs 'minor' (clarity issue but understandable).

Reply with JSON matching this schema:
{{output_schema}}
```

### Integration points

- US-034 (transcribe) provee transcript input.
- EPIC-03 pedagogical engine (consumer principal post-MVP).
- US-031 cost tracking.

### Notas técnicas

- Anthropic Haiku es preferido por calidad para grammar análisis.
- OpenAI gpt-4o-mini fallback es similar en calidad pero más
  variabilidad.
- Explanations en español mexicano-tuteo (consistencia con
  ADR-008).

## Definition of Done

- [ ] OpenAIAdapter LLM completo (extends US-034).
- [ ] Task registrada y activa.
- [ ] Handler con CEFR-aware leniency + subskill prioritization.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit con 10 transcripts (errors variados).
- [ ] Test integration con LLM real.
- [ ] Validation contra spec
  `ai-gateway-strategy.md` §4.2.3.
- [ ] PR aprobada y mergeada.

---

*Consumed by EPIC-03 pedagogical engine.*

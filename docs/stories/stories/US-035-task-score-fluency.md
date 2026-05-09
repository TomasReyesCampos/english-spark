# US-035: Task `score_fluency`

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 1
**Story points:** 3
**Persona:** Admin (consumido por mini-test, ejercicios)
**Owner:** —

---

## Contexto

Score de fluidez del speech del user. Métricas:
- WPM (words per minute).
- Pause count + average pause duration.
- Filler word rate (um, uh, like, you know).
- Coherence score (vía LLM evaluation).

Esta task **desbloquea US-018** (mini-test free response) y todos
los free response exercises post-MVP.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4.2.1 +
[`pedagogical-system.md`](../../product/pedagogical-system.md) §3.2.

## Scope

### In

- Registrar task `score_fluency`:
  - Provider chain: `[{anthropic, claude-haiku-4-5, weight: 100}]`
    (LLM-based para coherence; metrics primitivas como WPM se
    computan localmente sin LLM).
  - Budget: $0.005 per call.
  - Timeout: 10s.
  - Active: true.
- Input schema:
  ```typescript
  {
    transcript: string,
    duration_seconds: number,
    segments?: Array<{ start: number; end: number; text: string }>,
    target_topic?: string,           // para coherence eval
  }
  ```
- Output schema:
  ```typescript
  {
    overall: number,                  // 0-100
    wpm: number,                      // words per minute
    pause_count: number,              // pauses > 1s
    average_pause_seconds: number,
    filler_count: number,
    filler_rate: number,              // fillers per 100 words
    coherence_score: number,          // 0-10 from LLM
    detected_fillers: string[],       // ['um', 'like', 'you know']
  }
  ```
- Helpers locales (sin LLM):
  - WPM: count words in transcript / (duration / 60).
  - Pause detection: gap entre segments > 1s.
  - Filler detection: regex match contra lista
    `['um', 'uh', 'like', 'you know', 'i mean', 'so', 'well']`.
- LLM call para coherence (Anthropic Haiku):
  - Prompt corto: "Rate this response 0-10 for coherence and
    relevance. Topic: {topic}. Response: {transcript}".
  - Cost esperado < $0.001.
- Aggregation overall = weighted average de WPM normalized,
  coherence, filler penalty.

### Out

- Real-time fluency scoring streaming (post-MVP).
- Self-correction tracking (post-MVP).
- Pronunciation included en fluency (separado en US-033).

## Acceptance criteria

- **Given** transcript de 60 palabras en 30 segundos, **When** task
  computes WPM, **Then** `wpm = 120` (60 words / 0.5 min).
- **Given** transcript con 3 fillers ("um", "like", "uh"), **When**
  detect_fillers, **Then** `filler_count = 3`,
  `detected_fillers = ['um', 'like', 'uh']`.
- **Given** segments con gap de 3s entre dos consecutivos, **When**
  pause detection, **Then** `pause_count` incluye ese gap.
- **Given** transcript coherente sobre travel, **When** LLM evalúa
  con `target_topic = 'travel'`, **Then** `coherence_score >= 7`.
- **Given** transcript no relacionado al topic ("I had pizza for
  lunch") con topic "describe your job", **When** LLM evalúa,
  **Then** `coherence_score <= 4`.
- **Given** transcript vacío, **When** task procesa, **Then**
  retorna defaults (overall=0, wpm=0, sin throw).
- **Given** segments no provistos (solo transcript), **When**
  procesa, **Then** pause metrics son null pero WPM y fillers
  funcionan.
- **Given** invocation total cost ~$0.001, **When** persist,
  **Then** `cost_usd ≈ 0.001` (mostly LLM coherence call).

## Developer details

### Owning service

`apps/workers/ai-gateway/src/handlers/score-fluency.ts`.

### Dependencies

- US-027/028/029/030/031 (Anthropic adapter listo).

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.2.1.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §3.2 — fluency dimension definitions.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/handlers/score-fluency.ts
const FILLERS = ['um', 'uh', 'like', 'you know', 'i mean', 'so well', 'well'];

export async function handleScoreFluency(
  input: any,
  context: HandlerContext
): Promise<any> {
  const validated = InputSchema.parse(input);
  const { transcript, duration_seconds, segments, target_topic } = validated;

  const wordCount = transcript.split(/\s+/).filter(Boolean).length;
  const wpm = duration_seconds > 0 ? Math.round((wordCount / duration_seconds) * 60) : 0;

  // Filler detection
  const detected: string[] = [];
  const lowerTranscript = transcript.toLowerCase();
  for (const filler of FILLERS) {
    const regex = new RegExp(`\\b${filler.replace(/ /g, '\\s+')}\\b`, 'gi');
    const matches = lowerTranscript.match(regex);
    if (matches) detected.push(...matches);
  }
  const fillerCount = detected.length;
  const fillerRate = wordCount > 0 ? (fillerCount / wordCount) * 100 : 0;

  // Pause detection
  let pauseCount = 0;
  let totalPauseSeconds = 0;
  if (segments && segments.length > 1) {
    for (let i = 1; i < segments.length; i++) {
      const gap = segments[i].start - segments[i - 1].end;
      if (gap > 1) {
        pauseCount++;
        totalPauseSeconds += gap;
      }
    }
  }
  const avgPauseSeconds = pauseCount > 0 ? totalPauseSeconds / pauseCount : 0;

  // Coherence via LLM (only if transcript non-trivial)
  let coherenceScore = 0;
  if (wordCount >= 5) {
    const llmResult = await invokeWithFallback('score_fluency', {
      system_prompt: 'You are a language learning evaluator. Rate the response coherence (0-10) and relevance to the topic. Reply with JSON only.',
      messages: [{
        role: 'user',
        content: `Topic: ${target_topic ?? '(no specific topic)'}\nResponse: "${transcript}"\n\nReply with JSON: {"coherence_score": <0-10>, "reasoning": "<one sentence>"}`,
      }],
      temperature: 0.2,
      max_tokens: 200,
      response_format: 'json',
    }, context);

    try {
      const parsed = JSON.parse(llmResult.output.text);
      coherenceScore = Math.max(0, Math.min(10, parsed.coherence_score));
    } catch {
      coherenceScore = 5; // fallback to neutral if parse fails
    }
  }

  // Overall: weighted average
  const wpmNormalized = Math.min(100, (wpm / 130) * 100);   // 130 wpm = 100%
  const fillerPenalty = Math.max(0, 100 - fillerRate * 10); // 10% fillers = 0 score
  const overall = Math.round(
    wpmNormalized * 0.4 +
    fillerPenalty * 0.3 +
    (coherenceScore * 10) * 0.3
  );

  return {
    overall,
    wpm,
    pause_count: pauseCount,
    average_pause_seconds: avgPauseSeconds,
    filler_count: fillerCount,
    filler_rate: fillerRate,
    coherence_score: coherenceScore,
    detected_fillers: [...new Set(detected.map(f => f.toLowerCase()))],
  };
}
```

### Integration points

- US-018 de EPIC-01 (consumer principal).
- US-034 (transcribe) precede esta task en pipeline.
- Future: ejercicios free_response (EPIC-03).

### Notas técnicas

- Helpers locales (WPM, fillers, pauses) son **deterministas** —
  útil para tests unitarios sin LLM mocks.
- Coherence LLM con `temperature 0.2` para consistencia.
- Si transcript es tan corto que LLM no agrega valor (<5 palabras):
  skip LLM call y default coherence.

## Definition of Done

- [ ] Task registrada y activa.
- [ ] Handler con helpers locales + LLM coherence.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit del filler detection (10+ casos).
- [ ] Tests unit de WPM + pause detection.
- [ ] Test integration con transcript real + LLM.
- [ ] Validation contra spec
  `pedagogical-system.md` §3.2.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-018 de EPIC-01.*

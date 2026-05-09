# US-033: Task `score_pronunciation` (Azure)

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 1
**Story points:** 3
**Persona:** Admin (consumido por mini-test, ejercicios)
**Owner:** —

---

## Contexto

Task crítica para el producto: scoring de pronunciation con detalle
**fonema-por-fonema**. Azure Pronunciation Assessment es el
proveedor estándar (best-in-class para forced alignment + scores
por fonema vs alternativas open-source).

Esta task **desbloquea US-017** (mini-test repetir frase) y
generalmente todos los pronunciation drills del producto.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4.2.2 +
[`pedagogical-system.md`](../../product/pedagogical-system.md) §3.1.

## Scope

### In

- Implementar **AzureSpeechAdapter** (extiende skeleton de US-029).
- Registrar task `score_pronunciation`:
  - Provider chain: `[{azure, pronunciation_assessment_v1, weight: 100}]`.
    Sin fallback en MVP (Azure es único proveedor con calidad
    fonema-level — alternativas post-MVP).
  - Budget: $0.005 per call.
  - Timeout: 15s.
  - Active: true.
- Input schema:
  ```typescript
  {
    audio_url: string,           // R2 signed URL
    target_text: string,         // "I would love to travel to Japan someday"
    language: 'en-US' | 'en-GB',
    granularity: 'phoneme' | 'word' | 'sentence',
  }
  ```
- Output schema:
  ```typescript
  {
    overall: number,              // 0-100
    accuracy: number,
    fluency: number,
    completeness: number,
    pronunciation: number,
    words: Array<{
      word: string,
      score: number,
      phonemes: Array<{ phoneme: string, score: number }>,
    }>,
    detected_error_patterns: string[],   // ['/θ/_voiceless_substitution', ...]
  }
  ```
- Error patterns inference: post-process scores fonema-level para
  detectar patterns conocidos del hispanohablante (ej: /θ/ → /s/
  substitution si score de /θ/ es < 50 consistentemente).
- Handler `apps/workers/ai-gateway/src/handlers/score-pronunciation.ts`.
- Integration con AI Gateway orchestrator (aunque no usa fallback,
  pasa por orchestrator para cost tracking + logging).

### Out

- Pronunciation scoring con modelo propio (post-MVP, ver
  `pedagogical-system.md` §11).
- Streaming pronunciation feedback (post-MVP).
- Multi-language scoring (solo EN en MVP).

## Acceptance criteria

- **Given** task registrada y activa, **When** se invoca con
  `audio_url` válido + `target_text = "I would love to travel"`,
  **Then** retorna 200 con output schema completo (overall,
  accuracy, fluency, completeness, pronunciation, words[]).
- **Given** audio nítido y user dice frase target perfectamente,
  **When** Azure scores, **Then** `overall ≥ 85`.
- **Given** audio donde user pronuncia /θ/ como /s/ (think → "sink"),
  **When** Azure detecta, **Then** `detected_error_patterns`
  incluye `'/θ/_voiceless_substitution'`.
- **Given** audio con partial coverage (user solo dice 50% del
  target), **When** scores, **Then** `completeness ≈ 50`.
- **Given** audio_url inaccesible (404, signed URL expired),
  **When** Azure intenta fetch, **Then** retorna error
  `audio_fetch_failed` (no se cobra al user en este caso).
- **Given** target_text contains emoji o caracteres no-ASCII,
  **When** task valida input, **Then** strip o throws con
  `invalid_input` claro.
- **Given** invocation cuesta ~$0.004, **When** se persiste en
  ai_invocations, **Then** `cost_usd ≈ 0.004` (verificable contra
  Azure pricing).
- **Given** Azure timeout, **When** orchestrator detecta, **Then**
  retorna error `provider_unavailable` (sin fallback en MVP).

## Developer details

### Owning service

`apps/workers/ai-gateway/src/handlers/score-pronunciation.ts` +
adapters/azure-speech.ts.

### Dependencies

- US-027/028/029/030/031.
- Cuenta Azure con Speech Services habilitado.
- Secret `AZURE_SPEECH_KEY` configurado.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.2.2.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §3.1 — pronunciation scoring detalles.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/adapters/azure-speech.ts
export class AzureSpeechAdapter implements ProviderAdapter {
  readonly providerId = 'azure';
  readonly supportedModels = ['pronunciation_assessment_v1'];

  constructor(private env: Env) {}

  async invokeASR(
    input: { audio_url: string; target_text: string; language: string; granularity: string },
    config: ProviderConfig
  ): Promise<any> {
    const startTime = Date.now();

    // Fetch audio from R2
    const audioResponse = await fetch(input.audio_url);
    if (!audioResponse.ok) {
      throw new ProviderError('audio_fetch_failed', `Status ${audioResponse.status}`);
    }
    const audioBuffer = await audioResponse.arrayBuffer();

    // Build Azure Pronunciation Assessment request
    const pronunciationConfig = {
      ReferenceText: input.target_text,
      GradingSystem: 'HundredMark',
      Granularity: input.granularity,
      EnableMiscue: true,
    };
    const configB64 = btoa(JSON.stringify(pronunciationConfig));

    const url = `https://westus.stt.speech.microsoft.com/speech/recognition/conversation/cognitiveservices/v1?language=${input.language}&format=detailed`;

    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Ocp-Apim-Subscription-Key': this.env.AZURE_SPEECH_KEY,
        'Content-Type': 'audio/wav; codecs=audio/pcm; samplerate=22050',
        'Pronunciation-Assessment': configB64,
      },
      body: audioBuffer,
    });

    if (!response.ok) {
      throw new ProviderError('provider_unavailable', `Azure ${response.status}`);
    }

    const data = await response.json();
    const assessment = data.NBest?.[0]?.PronunciationAssessment;
    if (!assessment) {
      throw new ProviderError('no_assessment', 'Azure returned no assessment');
    }

    return {
      overall: assessment.PronScore,
      accuracy: assessment.AccuracyScore,
      fluency: assessment.FluencyScore,
      completeness: assessment.CompletenessScore,
      pronunciation: assessment.PronScore,
      words: data.NBest[0].Words?.map((w: any) => ({
        word: w.Word,
        score: w.PronunciationAssessment.AccuracyScore,
        phonemes: w.Phonemes?.map((p: any) => ({
          phoneme: p.Phoneme,
          score: p.PronunciationAssessment.AccuracyScore,
        })) ?? [],
      })) ?? [],
      _latency_ms: Date.now() - startTime,
    };
  }

  estimateCost(): number {
    return 0.004; // approximate per-call Azure cost
  }

  async isHealthy(): Promise<boolean> {
    try {
      const response = await fetch('https://westus.stt.speech.microsoft.com/healthz', {
        headers: { 'Ocp-Apim-Subscription-Key': this.env.AZURE_SPEECH_KEY },
      });
      return response.ok;
    } catch { return false; }
  }
}

// apps/workers/ai-gateway/src/handlers/score-pronunciation.ts
const ERROR_PATTERN_RULES = [
  { phoneme: 'θ', threshold: 50, pattern: '/θ/_voiceless_substitution' },
  { phoneme: 'ð', threshold: 50, pattern: '/ð/_voiced_substitution' },
  { phoneme: 'v', threshold: 50, pattern: '/v/_vs_b_confusion' },
  { phoneme: 'ɪ', threshold: 50, pattern: '/ɪ/_short_long_i_confusion' },
  // ... más patterns
];

export async function handleScorePronunciation(
  input: any,
  context: HandlerContext
): Promise<any> {
  const validated = InputSchema.parse(input);

  const result = await invokeWithFallback(
    'score_pronunciation',
    validated,
    context
  );

  // Post-process: infer error patterns
  const detectedPatterns = inferErrorPatterns(result.output);
  return { ...result.output, detected_error_patterns: detectedPatterns };
}

function inferErrorPatterns(scoringResult: any): string[] {
  const patterns = new Set<string>();
  for (const word of scoringResult.words ?? []) {
    for (const phoneme of word.phonemes ?? []) {
      for (const rule of ERROR_PATTERN_RULES) {
        if (phoneme.phoneme === rule.phoneme && phoneme.score < rule.threshold) {
          patterns.add(rule.pattern);
        }
      }
    }
  }
  return [...patterns];
}
```

### Integration points

- US-017, US-019 de EPIC-01 (consumers).
- Future: ejercicios de pronunciation drill del producto (EPIC-03).
- Cost tracking (US-031).

### Notas técnicas

- Azure Speech Services region: `westus` para latencia óptima desde
  Cloudflare.
- Audio format esperado: WAV PCM 22050Hz mono. Si cliente envía
  MP3: convertir on-the-fly o documentar requirement.
- Pronunciation assessment cuesta similar a STT regular en Azure
  pricing.
- Single provider en MVP — alternativas post-MVP cuando volumen
  justifique evaluar.

## Definition of Done

- [ ] AzureSpeechAdapter implementado.
- [ ] Task registrada y activa.
- [ ] Handler con error pattern inference.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit con audio fixtures (audio bueno + audio con
  /θ/→/s/).
- [ ] Test integration end-to-end con audio real subido a R2.
- [ ] Validation contra spec
  `pedagogical-system.md` §3.1.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-017, US-019 de EPIC-01.*

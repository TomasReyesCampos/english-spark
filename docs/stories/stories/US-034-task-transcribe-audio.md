# US-034: Task `transcribe_user_audio` (Whisper)

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 1
**Story points:** 2
**Persona:** Admin (consumido por mini-test, ejercicios)
**Owner:** —

---

## Contexto

Speech-to-text del audio del user. Necesario para:
- Mini-test ejercicio 2 (free response) — US-018.
- Mini-test ejercicio 3 (image naming) — US-019.
- Ejercicios de free response del producto (EPIC-03).

OpenAI Whisper es el estándar de calidad. Google Speech-to-Text V2
como fallback (cost lower, quality slightly lower).

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4.2.1.

## Scope

### In

- Implementar **OpenAIAdapter** completo (Whisper + LLM, replacing
  el stub de US-029).
- Implementar **GoogleAdapter** stub para Speech-to-Text V2 (basic).
- Registrar task `transcribe_user_audio`:
  - Provider chain: `[{openai, whisper-1, weight: 100},
                      {google, speech-to-text-v2, weight: 0}]`.
  - Budget: $0.006 per call (Whisper $0.006/min).
  - Timeout: 30s.
  - Active: true.
- Input schema:
  ```typescript
  {
    audio_url: string,             // R2 signed URL
    expected_language?: string,    // 'en' default; undefined = auto-detect
    prompt?: string,               // hint para Whisper (opcional)
  }
  ```
- Output schema:
  ```typescript
  {
    text: string,
    language: string,              // detected
    duration_seconds: number,
    segments?: Array<{ start: number; end: number; text: string }>,
  }
  ```
- Handler con language detection: si `expected_language = 'en'` y
  Whisper detecta otro: flag `language_mismatch = true` (consumido
  por mini-test US-018).
- Audio format support: MP3, WAV, M4A, OGG (Whisper acepta
  multiples).

### Out

- Real-time streaming transcription (post-MVP).
- Custom vocabulary / phrase hints (post-MVP).
- Speaker diarization (post-MVP).

## Acceptance criteria

- **Given** task registrada y activa, **When** se invoca con audio
  válido (10s en inglés), **Then** retorna 200 con transcript +
  language='en'.
- **Given** audio en español, **When** task con
  `expected_language='en'`, **Then** retorna transcript con
  `language='es'` y `language_mismatch=true`.
- **Given** audio inaccesible (404), **When** Whisper intenta fetch,
  **Then** retorna error `audio_fetch_failed`.
- **Given** Whisper timeout (>30s), **When** orchestrator detecta,
  **Then** fallback a Google Speech-to-Text.
- **Given** Whisper rate-limited 429, **When** retry 1x falla,
  **Then** fallback a Google.
- **Given** Google adapter no implementado completamente, **When**
  fallback intenta, **Then** retorna `not_implemented` y task
  fail final con detalle de attempts.
- **Given** audio silencioso (mostly silence), **When** Whisper
  procesa, **Then** retorna `text: ""` sin error (no es bug
  per se).
- **Given** audio MP3 22050Hz, **When** se transcribe, **Then**
  funciona idénticamente a WAV (formato agnóstico).

## Developer details

### Owning service

`apps/workers/ai-gateway/src/adapters/openai.ts` +
`handlers/transcribe-user-audio.ts`.

### Dependencies

- US-027/028/029/030/031.
- Secret `OPENAI_API_KEY`.
- (Post-MVP fallback) `GOOGLE_AI_API_KEY` para Google STT.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.2.1 — task config.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/adapters/openai.ts
import OpenAI from 'openai';

export class OpenAIAdapter implements ProviderAdapter {
  readonly providerId = 'openai';
  readonly supportedModels = ['gpt-4o-mini', 'gpt-4o', 'whisper-1'];

  private client: OpenAI;

  constructor(env: Env) {
    this.client = new OpenAI({ apiKey: env.OPENAI_API_KEY });
  }

  async invokeASR(
    input: { audio_url: string; expected_language?: string; prompt?: string },
    config: ProviderConfig
  ): Promise<any> {
    const start = Date.now();

    const audioResponse = await fetch(input.audio_url);
    if (!audioResponse.ok) {
      throw new ProviderError('audio_fetch_failed', `Status ${audioResponse.status}`);
    }
    const audioBlob = await audioResponse.blob();

    const file = new File([audioBlob], 'audio.mp3', { type: 'audio/mpeg' });

    try {
      const transcription = await this.client.audio.transcriptions.create({
        file,
        model: 'whisper-1',
        language: input.expected_language,
        prompt: input.prompt,
        response_format: 'verbose_json',
      });

      return {
        text: transcription.text,
        language: transcription.language,
        duration_seconds: transcription.duration,
        segments: transcription.segments?.map(s => ({
          start: s.start, end: s.end, text: s.text,
        })),
        _latency_ms: Date.now() - start,
      };
    } catch (error: any) {
      if (error.status === 429) throw new ProviderError('rate_limited', error.message);
      if (error.status === 503) throw new ProviderError('provider_unavailable', error.message);
      throw new ProviderError('unknown_provider_error', error.message);
    }
  }

  async invokeLLM(input: any, config: ProviderConfig): Promise<any> {
    // OpenAI LLM (gpt-4o-mini, gpt-4o) — stub for now, completed in US-040
    throw new NotImplementedError('OpenAI LLM (full impl in US-040)');
  }

  estimateCost(input: any, config: ProviderConfig): number {
    if (config.model === 'whisper-1') {
      const minutes = (input.duration_seconds ?? 30) / 60;
      return minutes * 0.006;
    }
    return 0;
  }

  async isHealthy(): Promise<boolean> {
    try {
      await this.client.models.list();
      return true;
    } catch { return false; }
  }
}

// apps/workers/ai-gateway/src/handlers/transcribe-user-audio.ts
export async function handleTranscribe(
  input: any,
  context: HandlerContext
): Promise<any> {
  const validated = InputSchema.parse(input);
  const result = await invokeWithFallback('transcribe_user_audio', validated, context);

  return {
    ...result.output,
    language_mismatch: validated.expected_language
      ? result.output.language !== validated.expected_language
      : false,
  };
}
```

### Integration points

- US-018, US-019 de EPIC-01 (consumers).
- Cost tracking (US-031).
- Future: free response exercises (EPIC-03).

### Notas técnicas

- Whisper acepta files hasta 25MB. Audios del producto típicamente
  son < 1MB (<30s mp3 22kHz mono).
- `verbose_json` retorna timestamps por segmento — útil para
  ejercicios que requieren timing.
- Post-MVP: agregar `prompt` con context del producto para mejorar
  accuracy ("This is an English language learning exercise...").

## Definition of Done

- [ ] OpenAIAdapter implementado (al menos invokeASR + estimateCost).
- [ ] Task registrada y activa.
- [ ] Handler con language_mismatch detection.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit con mocked Whisper.
- [ ] Test integration con audio real (10s en inglés).
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-018, US-019 de EPIC-01.*

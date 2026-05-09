# US-039: Task `validate_image_no_text` (OCR)

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 1-2
**Story points:** 2
**Persona:** Admin (consumido por content pipeline)
**Owner:** —

---

## Contexto

DALL-E 3 a veces inserta texto involuntario en imágenes (típicamente
texto garabato o palabras parciales). Esto es inaceptable para
ejercicios `image_description` (donde no queremos pistas textuales).
Esta task valida si la imagen contiene texto detectable usando OCR.

Si detecta texto: la imagen se rechaza y se re-genera (post-MVP) o
se flagea para review manual.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4.2.9 +
[`content-creation-system.md`](../../product/content-creation-system.md)
§8.1 (`is_text_in_image` field).

## Scope

### In

- Implementar lookup contra **Google Cloud Vision API** (OCR).
- Registrar task `validate_image_no_text`:
  - Provider chain: `[{google, vision_text_detection_v1, weight: 100}]`.
  - Budget: $0.0015 per call (Google Vision pricing).
  - Timeout: 10s.
  - Active: true.
- Input schema:
  ```typescript
  { image_url: string }
  ```
- Output schema:
  ```typescript
  {
    has_text: boolean,
    detected_strings: string[],         // textos detectados
    confidence: number,                 // 0-1, max confidence
    is_text_in_image: boolean,          // alias for has_text, matches schema field
  }
  ```
- Threshold: si confidence > 0.7 y al menos 3 chars detectados:
  `has_text = true`.
- Excepciones: ignorar single chars sueltos (probable artifact).
- Telemetry: `image_ocr.checked`,
  `image_ocr.text_detected`.

### Out

- Auto-regeneration cuando text detected (post-MVP — MVP es solo
  flagging para review humana).
- OCR multilenguaje específico (Vision auto-detecta).

## Acceptance criteria

- **Given** task registrada, **When** se invoca con imagen clean
  (no text), **Then** retorna `has_text: false`,
  `detected_strings: []`.
- **Given** imagen DALL-E con palabra "Coffee" garabateada, **When**
  Vision detecta, **Then** retorna `has_text: true`,
  `detected_strings: ['Coffee']`.
- **Given** imagen con single char "@" en esquina, **When** task
  procesa, **Then** retorna `has_text: false` (excepción de
  single chars).
- **Given** imagen con 5 chars detectados pero confidence 0.5,
  **When** task evalúa threshold, **Then** retorna `has_text:
  false` (below threshold).
- **Given** imagen inaccesible (404), **When** Vision intenta
  fetch, **Then** retorna error `image_fetch_failed`.
- **Given** Vision API rate-limited, **When** falla 429, **Then**
  retry 1x con backoff. Sin fallback.
- **Given** invocation cuesta $0.0015, **When** persist, **Then**
  cost tracking refleja correctamente.
- **Given** content pipeline batch ejecuta 100 OCR validations,
  **When** total cost, **Then** ≤ $0.15 (within budget).

## Developer details

### Owning service

`apps/workers/ai-gateway/src/adapters/google-vision.ts` (extiende
GoogleAdapter de US-034 con métodos image-related) +
`handlers/validate-image-no-text.ts`.

### Dependencies

- US-027/028/029/030/031.
- US-038 (genera las imágenes a validar).
- Google Cloud Vision API key configurado en
  `GOOGLE_AI_API_KEY` o secret separado.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.2.9.
- [`content-creation-system.md`](../../product/content-creation-system.md)
  §8.1 — campo `is_text_in_image`.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/adapters/google-vision.ts (extend GoogleAdapter)
export class GoogleAdapter implements ProviderAdapter {
  readonly providerId = 'google';
  readonly supportedModels = ['vision_text_detection_v1', 'speech-to-text-v2'];

  constructor(private env: Env) {}

  async invokeImageOCR(
    input: { image_url: string },
    config: ProviderConfig
  ): Promise<{ texts: string[]; confidence: number; latency_ms: number }> {
    const start = Date.now();

    const response = await fetch(
      `https://vision.googleapis.com/v1/images:annotate?key=${this.env.GOOGLE_VISION_KEY}`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          requests: [{
            image: { source: { imageUri: input.image_url } },
            features: [{ type: 'TEXT_DETECTION', maxResults: 10 }],
          }],
        }),
      }
    );

    if (!response.ok) {
      if (response.status === 429) throw new ProviderError('rate_limited', '');
      throw new ProviderError('provider_unavailable', `Status ${response.status}`);
    }

    const data = await response.json();
    const annotations = data.responses?.[0]?.textAnnotations ?? [];

    const texts = annotations.slice(1).map((a: any) => a.description); // [0] is full block
    const maxConfidence = annotations[0]?.confidence ?? 0;

    return {
      texts,
      confidence: maxConfidence,
      latency_ms: Date.now() - start,
    };
  }

  estimateCost(): number {
    return 0.0015; // Google Vision pricing
  }

  async isHealthy(): Promise<boolean> {
    return true; // simplified
  }
}

// apps/workers/ai-gateway/src/handlers/validate-image-no-text.ts
const MIN_CHARS = 3;
const CONFIDENCE_THRESHOLD = 0.7;

export async function handleValidateImageNoText(
  input: any,
  context: HandlerContext
): Promise<any> {
  const validated = InputSchema.parse(input);

  const result = await invokeWithFallback('validate_image_no_text', validated, context);
  const { texts, confidence } = result.output;

  // Filter: ignore single-char artifacts
  const meaningfulTexts = texts.filter((t: string) => t.trim().length >= MIN_CHARS);

  const hasText = meaningfulTexts.length > 0 && confidence >= CONFIDENCE_THRESHOLD;

  if (hasText) {
    track('image_ocr.text_detected', {
      detected_strings: meaningfulTexts,
      confidence,
    });
  } else {
    track('image_ocr.checked', { clean: true });
  }

  return {
    has_text: hasText,
    detected_strings: meaningfulTexts,
    confidence,
    is_text_in_image: hasText,
  };
}
```

### Integration points

- US-038 (generate_image) → US-039 corre after en pipeline.
- Content team manual review queue (si has_text=true → flag).
- Cost tracking (US-031).

### Notas técnicas

- Google Vision pricing: $0.0015 por TEXT_DETECTION call ≤ 1k/mes
  (free tier 1k). Beyond: $1.50 per 1k.
- Para 100 imágenes batch validation: $0.15 (within free tier
  monthly).
- Threshold 0.7 calibrado experimentalmente — ajustar si false
  positive rate es alto (cron monthly review).

## Definition of Done

- [ ] GoogleAdapter extendido con `invokeImageOCR`.
- [ ] Task registrada y activa.
- [ ] Handler con threshold filtering.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit con mocks Vision response.
- [ ] Test integration: validar 5 imágenes (3 clean + 2 with text).
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Consumed by EPIC-02 content pipeline post US-038.*

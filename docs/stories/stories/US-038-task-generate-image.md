# US-038: Task `generate_image` (DALL-E 3)

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 1-2
**Story points:** 3
**Persona:** Admin (consumido por content team batch)
**Owner:** —

---

## Contexto

Generar las ~100 imágenes del seed (70 scene atomics + 30 character
avatars). DALL-E 3 vía OpenAI API es el primary; Midjourney como
fallback opcional post-MVP.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4.2.9 +
[`atomics-catalog-seed.md`](../../product/atomics-catalog-seed.md)
§9.3.

## Scope

### In

- Implementar **DalleAdapter** con OpenAI Images API:
  - Modelo `dall-e-3`.
  - Quality: HD por default.
  - Size: `1024x1024` por default; `1792x1024` para wide scenes.
  - Style: `natural` (vs `vivid` por default).
- Registrar task `generate_image`:
  - Provider chain: `[{dalle3, dall-e-3, weight: 100}]`.
  - Budget: $0.05 per call (HD 1024 = $0.04).
  - Timeout: 60s (DALL-E 3 puede tardar).
  - Active: true.
- Input schema:
  ```typescript
  {
    prompt: string,
    size?: '1024x1024' | '1792x1024' | '1024x1792',
    style?: 'natural' | 'vivid',
    storage_key?: string,
  }
  ```
- Output schema:
  ```typescript
  {
    image_url: string,
    storage_key: string,
    width: number,
    height: number,
    generation_seed?: number,        // si proveedor lo retorna
    cost_usd: number,
  }
  ```
- Post-process:
  - Recibir PNG de DALL-E 3.
  - Convert a WebP (compress) usando `Sharp` o equivalente WASM.
  - Upload a R2.
- Telemetry: `image.generated`,
  `image.compression_ratio`.

### Out

- Midjourney fallback (post-MVP).
- Image variations / inpainting (post-MVP).
- Stable Diffusion local model (post-MVP).

## Acceptance criteria

- **Given** task registrada y activa, **When** se invoca con prompt
  válido, **Then** retorna 200 con `image_url` accesible y
  dimensions matching size requested.
- **Given** prompt que viola OpenAI policy (ej: violence), **When**
  DALL-E rechaza, **Then** retorna error `policy_violation` con
  detail.
- **Given** prompt en español ("Una persona feliz"), **When**
  DALL-E procesa, **Then** funciona (DALL-E 3 acepta multi-lang).
- **Given** rate limit OpenAI, **When** falla 429, **Then** retry
  1x con backoff. Sin fallback en MVP.
- **Given** PNG de 2MB recibido, **When** post-process compress a
  WebP, **Then** archivo final < 500KB sin pérdida visual notoria.
- **Given** invocation con HD 1024, **When** cost se persiste,
  **Then** `cost_usd ≈ 0.04`.
- **Given** R2 upload falla, **When** post-process termina,
  **Then** retorna error `storage_failed`.
- **Given** DALL-E timeout 60s, **When** orchestrator detecta,
  **Then** retorna error `timeout` (no fallback en MVP).

## Developer details

### Owning service

`apps/workers/ai-gateway/src/adapters/dalle.ts` +
`handlers/generate-image.ts`.

### Dependencies

- US-027/028/029/030/031.
- OpenAI API key (ya configurado en US-034).
- R2 bucket.
- Sharp / image processing WASM.

### Specs referenciados

- [`atomics-catalog-seed.md`](../../product/atomics-catalog-seed.md)
  §9.3.
- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.2.9.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/adapters/dalle.ts
import OpenAI from 'openai';

export class DalleAdapter implements ProviderAdapter {
  readonly providerId = 'dalle3';
  readonly supportedModels = ['dall-e-3'];

  private client: OpenAI;

  constructor(env: Env) {
    this.client = new OpenAI({ apiKey: env.OPENAI_API_KEY });
  }

  async invokeImage(
    input: { prompt: string; size?: string; style?: string },
    config: ProviderConfig
  ): Promise<{ image_url: string; latency_ms: number }> {
    const start = Date.now();

    try {
      const response = await this.client.images.generate({
        model: 'dall-e-3',
        prompt: input.prompt,
        size: (input.size as any) ?? '1024x1024',
        quality: 'hd',
        style: (input.style as any) ?? 'natural',
        n: 1,
      });

      const url = response.data[0].url;
      if (!url) throw new ProviderError('no_image_returned', 'DALL-E returned empty');

      return { image_url: url, latency_ms: Date.now() - start };
    } catch (error: any) {
      if (error.code === 'content_policy_violation') {
        throw new ProviderError('policy_violation', error.message);
      }
      if (error.status === 429) throw new ProviderError('rate_limited', error.message);
      if (error.status === 503) throw new ProviderError('provider_unavailable', error.message);
      throw new ProviderError('unknown_provider_error', error.message);
    }
  }

  estimateCost(input: { size?: string }): number {
    const size = input.size ?? '1024x1024';
    if (size === '1024x1024') return 0.04;   // HD
    if (size === '1792x1024' || size === '1024x1792') return 0.08;
    return 0.04;
  }

  async isHealthy(): Promise<boolean> {
    try {
      await this.client.models.list();
      return true;
    } catch { return false; }
  }
}

// apps/workers/ai-gateway/src/handlers/generate-image.ts
export async function handleGenerateImage(input: any, context: HandlerContext) {
  const validated = InputSchema.parse(input);

  const result = await invokeWithFallback('generate_image', validated, context);
  const { image_url } = result.output;

  // Fetch and compress
  const imageResponse = await fetch(image_url);
  const pngBuffer = await imageResponse.arrayBuffer();

  const webpBuffer = await convertToWebP(pngBuffer);
  const compressionRatio = pngBuffer.byteLength / webpBuffer.byteLength;
  track('image.compression_ratio', { ratio: compressionRatio });

  const storageKey = validated.storage_key
    ?? `atomics/v1/images/${crypto.randomUUID()}.webp`;
  await context.env.R2_BUCKET.put(storageKey, webpBuffer, {
    httpMetadata: { contentType: 'image/webp' },
  });

  const dimensions = parseDimensions(validated.size ?? '1024x1024');

  track('image.generated', {
    storage_key: storageKey,
    size_bytes: webpBuffer.byteLength,
  });

  return {
    image_url: `https://r2.englishspark.com/${storageKey}`,
    storage_key: storageKey,
    width: dimensions.width,
    height: dimensions.height,
    cost_usd: result.metadata.cost_usd,
  };
}
```

### Integration points

- EPIC-02 Content pipeline (consumer batch).
- R2 storage.
- OpenAIAdapter (compartido para Whisper en US-034).
- US-039 (validate_image_no_text) corre after esta task.

### Notas técnicas

- DALL-E 3 NO tiene seed control en su API pública (a diferencia
  de Midjourney). Para character consistency, reuse mismo prompt
  determinístico + generar N variants y curar manualmente la
  mejor.
- WebP compression típica 5-8x sobre PNG sin pérdida visual.
- Cost de batch: 100 imágenes × $0.04 = $4 total.

## Definition of Done

- [ ] DalleAdapter implementado.
- [ ] Task registrada y activa.
- [ ] Handler con WebP compression + R2 upload.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit del adapter (mocked).
- [ ] Test integration: generar 1 imagen real, verificar compression
  + R2.
- [ ] Validation contra spec
  `atomics-catalog-seed.md` §9.3.
- [ ] PR aprobada y mergeada.

---

*Consumed by EPIC-02 Content pipeline.*

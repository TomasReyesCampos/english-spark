# US-029: Provider adapter interface + Anthropic adapter

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 0-1
**Story points:** 5
**Persona:** Admin
**Owner:** —

---

## Contexto

Cada proveedor de LLM (Anthropic, OpenAI, Google, etc.) tiene API
distinta. El **Provider Adapter** abstrae estas diferencias detrás
de una interfaz común. El Worker de Gateway elige adapter desde el
provider_chain del task y le pasa input estandarizado.

Esta story implementa:
- Interfaz unificada (TypeScript).
- Adapter de **Anthropic** completo (primer proveedor en línea
  porque cubre la mayoría de tasks LLM-style: roadmap, notification,
  insights).
- Adapters skeleton de OpenAI, Google, Azure (interfaces sin
  implementación; se completan en stories de tasks específicas).

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§5 (Provider Adapters).

## Scope

### In

- Interfaz `ProviderAdapter`:
  ```typescript
  interface ProviderAdapter {
    readonly providerId: string;
    readonly supportedModels: string[];

    invokeLLM(input: LLMInput, config: ProviderConfig): Promise<LLMOutput>;
    invokeAudio?(input: AudioInput, config: ProviderConfig): Promise<AudioOutput>;
    invokeImage?(input: ImageInput, config: ProviderConfig): Promise<ImageOutput>;
    invokeASR?(input: ASRInput, config: ProviderConfig): Promise<ASROutput>;

    estimateCost(input: any, config: ProviderConfig): number;
    isHealthy(): Promise<boolean>;
  }
  ```
- **Anthropic adapter completo:**
  - `invokeLLM` con messages API.
  - Cost estimation desde input/output tokens.
  - `isHealthy` que pinga endpoint de status.
  - Retry simple (1x) en errores transient (429, 503).
  - Soporta `system_prompt`, `messages`, `temperature`,
    `max_tokens`, response format JSON cuando configurado.
- **Skeletons:** OpenAI, Google, Azure, ElevenLabs, DALL-E con
  interfaz pero sin lógica implementada (throws "not implemented"
  hasta que stories propias los completen).
- API keys de proveedores en Cloudflare secrets:
  - `ANTHROPIC_API_KEY`
  - `OPENAI_API_KEY`
  - `GOOGLE_AI_API_KEY`
  - `AZURE_SPEECH_KEY`
  - `ELEVENLABS_API_KEY`
- Factory `getAdapter(providerId)` retorna instancia singleton del
  adapter.
- Telemetry: `ai_gateway.provider_invoked` con
  `{ provider, model, latency_ms, cost_usd, status }`.

### Out

- Adapters concretos OpenAI, Google, Azure, ElevenLabs, DALL-E,
  HeyGen — van con sus tasks correspondientes (US-033, US-034,
  US-037, US-038, etc.).
- Multi-provider fallback chain (US-030).
- Cost aggregation per user (US-031).

## Acceptance criteria

- **Given** un Worker llama
  `getAdapter('anthropic').invokeLLM({ system, messages, model })`,
  **When** la API responde, **Then** retorna `LLMOutput` con
  `text`, `usage_input_tokens`, `usage_output_tokens`,
  `latency_ms`.
- **Given** Anthropic retorna 429 (rate limit), **When** adapter
  procesa, **Then** retry con exponential backoff (1s); si segundo
  fail: throws `ProviderError('rate_limited')`.
- **Given** Anthropic retorna 503, **When** adapter procesa,
  **Then** mismo retry pattern.
- **Given** Anthropic retorna 400 (bad request), **When** adapter,
  **Then** NO retry, throws `ProviderError('invalid_request')`.
- **Given** request con model unsupported, **When** factory
  valida, **Then** throws antes de llamar API.
- **Given** llamada exitosa con tokens 1500 input + 500 output con
  Haiku, **When** estimateCost, **Then** retorna correctamente
  ~$0.001 (precio Haiku).
- **Given** `isHealthy` ejecuta, **When** Anthropic responde 200,
  **Then** retorna `true`. Si timeout 5s o 5xx: retorna `false`.
- **Given** se intenta `getAdapter('openai').invokeLLM(...)` antes
  de que esté implementado, **When** se llama, **Then** throws
  `NotImplementedError` con mensaje claro.

## Developer details

### Owning service

`apps/workers/ai-gateway/src/adapters/`.

### Dependencies

- US-027, US-028.
- API keys configuradas.
- `@anthropic-ai/sdk` package (works in Cloudflare Workers).

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §5 — interfaz unificada.
- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §10.1 — métricas mínimas.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/adapters/types.ts
export interface LLMInput {
  system_prompt?: string;
  messages: Array<{ role: 'user' | 'assistant'; content: string }>;
  temperature?: number;
  max_tokens?: number;
  response_format?: 'text' | 'json';
}

export interface LLMOutput {
  text: string;
  usage_input_tokens: number;
  usage_output_tokens: number;
  latency_ms: number;
}

export class ProviderError extends Error {
  constructor(public code: string, message: string) {
    super(message);
  }
}

export class NotImplementedError extends Error {
  constructor(adapter: string) {
    super(`Adapter ${adapter} not implemented yet`);
  }
}

// apps/workers/ai-gateway/src/adapters/anthropic.ts
import Anthropic from '@anthropic-ai/sdk';

const PRICING = {
  'claude-haiku-4-5':  { input: 0.0008 / 1000, output: 0.004 / 1000 },
  'claude-sonnet-4-6': { input: 0.003 / 1000, output: 0.015 / 1000 },
  'claude-opus-4-7':   { input: 0.015 / 1000, output: 0.075 / 1000 },
};

export class AnthropicAdapter implements ProviderAdapter {
  readonly providerId = 'anthropic';
  readonly supportedModels = Object.keys(PRICING);

  private client: Anthropic;

  constructor(env: Env) {
    this.client = new Anthropic({ apiKey: env.ANTHROPIC_API_KEY });
  }

  async invokeLLM(input: LLMInput, config: ProviderConfig): Promise<LLMOutput> {
    if (!this.supportedModels.includes(config.model)) {
      throw new ProviderError('unsupported_model', config.model);
    }

    const start = Date.now();
    let attempt = 0;

    while (attempt <= 1) {
      try {
        const response = await this.client.messages.create({
          model: config.model,
          system: input.system_prompt,
          messages: input.messages,
          temperature: input.temperature ?? 0.7,
          max_tokens: input.max_tokens ?? 2048,
        });

        const text = response.content[0].type === 'text' ? response.content[0].text : '';
        return {
          text,
          usage_input_tokens: response.usage.input_tokens,
          usage_output_tokens: response.usage.output_tokens,
          latency_ms: Date.now() - start,
        };
      } catch (error: any) {
        if ((error.status === 429 || error.status === 503) && attempt < 1) {
          await new Promise(r => setTimeout(r, 1000));
          attempt++;
          continue;
        }
        if (error.status === 429) throw new ProviderError('rate_limited', error.message);
        if (error.status === 503) throw new ProviderError('provider_unavailable', error.message);
        if (error.status === 400) throw new ProviderError('invalid_request', error.message);
        throw new ProviderError('unknown_provider_error', error.message);
      }
    }
    throw new ProviderError('exhausted_retries', 'unreachable');
  }

  estimateCost(input: { usage_input_tokens: number; usage_output_tokens: number },
               config: ProviderConfig): number {
    const pricing = PRICING[config.model as keyof typeof PRICING];
    return input.usage_input_tokens * pricing.input
         + input.usage_output_tokens * pricing.output;
  }

  async isHealthy(): Promise<boolean> {
    try {
      await this.client.messages.create({
        model: 'claude-haiku-4-5',
        max_tokens: 1,
        messages: [{ role: 'user', content: 'ping' }],
      });
      return true;
    } catch {
      return false;
    }
  }
}
```

### Factory

```typescript
// apps/workers/ai-gateway/src/adapters/factory.ts
const adapters = new Map<string, ProviderAdapter>();

export function getAdapter(providerId: string, env: Env): ProviderAdapter {
  if (!adapters.has(providerId)) {
    switch (providerId) {
      case 'anthropic': adapters.set('anthropic', new AnthropicAdapter(env)); break;
      case 'openai':    adapters.set('openai',    new OpenAIAdapterStub()); break;
      case 'google':    adapters.set('google',    new GoogleAdapterStub()); break;
      case 'azure':     adapters.set('azure',     new AzureAdapterStub()); break;
      case 'elevenlabs':adapters.set('elevenlabs',new ElevenLabsAdapterStub()); break;
      case 'dalle3':    adapters.set('dalle3',    new DalleAdapterStub()); break;
      case 'heygen':    adapters.set('heygen',    new HeygenAdapterStub()); break;
      default: throw new Error(`Unknown provider: ${providerId}`);
    }
  }
  return adapters.get(providerId)!;
}
```

### Integration points

- US-027 Worker invokes adapters.
- US-028 Registry tells which adapter to use per task.
- US-030 Fallback chain orchestrates adapters.
- US-031 Cost tracking consume `estimateCost`.

### Notas técnicas

- Anthropic SDK funciona en Cloudflare Workers runtime.
- Singleton pattern en factory evita re-instanciar Anthropic
  client por request.
- Pricing values deben actualizarse cuando Anthropic cambie precios
  (cron mensual valida vs página de pricing).

## Definition of Done

- [ ] Interfaz `ProviderAdapter` definida.
- [ ] AnthropicAdapter completo con 3 modelos soportados.
- [ ] Skeletons de los otros 6 providers throwing
  `NotImplementedError`.
- [ ] Factory pattern singleton.
- [ ] Telemetry en cada invocation.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit del Anthropic adapter (con mocked SDK).
- [ ] Test integration: una invocación real a Haiku con prompt
  trivial.
- [ ] Validation contra spec `ai-gateway-strategy.md` §5.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-030 y todas las stories de tasks LLM.*

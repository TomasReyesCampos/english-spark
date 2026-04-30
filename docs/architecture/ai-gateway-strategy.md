# AI Gateway Strategy

> Capa única de acceso a LLMs y servicios de IA. Multi-proveedor con
> fallback, A/B testing, validation con Zod, cost tracking, prompt
> versioning. **Único path** para invocar IA desde el resto del sistema.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador. Toda llamada a LLM o
servicio de IA pasa por este Gateway. Mencionar nombres de proveedores
fuera de este sistema es bug de diseño.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 establece **principios** (regla de oro).
- §2 define **boundaries**.
- §3 define **arquitectura del Gateway**.
- §4 contiene el **Task Registry** completo (catálogo de tareas IA del
  sistema).
- §5 define **provider adapters**.
- §6 cubre **prompt versioning**.
- §7 cubre **golden datasets** y evaluación.
- §8 cubre **migración progresiva** entre modelos.
- §9 cubre **manejo de deprecaciones**.
- §10 cubre **cost tracking y observability**.
- §11 contiene **API contracts** (`executeTask`).
- §12 enumera **edge cases**.
- §13 contiene el **plan de implementación**.
- §14 lista **métricas** y alertas.

---

## 1. Principios rectores

### 1.1 Regla de oro

> **Si el código de la aplicación menciona "anthropic", "openai",
> "gemini", "azure" o cualquier nombre de proveedor fuera de la capa
> de abstracción, es un bug de diseño que debe corregirse.**

Toda invocación de IA usa `executeTask(taskId, input)`. El Gateway
internamente decide qué proveedor + modelo + prompt usar.

### 1.2 LLMs son commodities intercambiables

La arquitectura asume que cualquier LLM puede reemplazarse por otro
con esfuerzo mínimo. En la práctica no son equivalentes; pero la
arquitectura no es el obstáculo cuando se decide cambiar.

### 1.3 Decisiones basadas en datos, no hype

Cada migración a modelo nuevo está respaldada por métricas medidas
en datasets propios (golden datasets §7), no por benchmarks públicos
ni marketing del proveedor.

### 1.4 Multi-proveedor desde día uno

Aunque un proveedor sirva 95% del tráfico, mantener al menos un
secundario activo (5%) para cada tarea crítica. Costo despreciable;
beneficio en resiliencia y poder de negociación enorme.

### 1.5 Observabilidad antes que optimización

No se optimiza lo que no se mide. Cada llamada al Gateway registra
costo, latencia, calidad, validation result.

### 1.6 Cobro de Sparks antes de invocar

Operaciones que cuestan Sparks: el Gateway llama
`sparks.chargeOperation` antes de invocar el modelo. Si falla el
charge, no invoca. Si la invocación falla, llama `sparks.refundOperation`.

---

## 2. Boundaries

### 2.1 Es responsable de

- Routing de cada `taskId` a proveedor + modelo + prompt según
  Task Registry.
- Cargar el prompt versionado correcto.
- Ejecutar con retry y fallback.
- Validar output con Zod schema definido en el task.
- Cobrar Sparks (delegando a `sparks-system`).
- Registrar métricas (costo, latencia, validation result).
- A/B testing entre modelos via experimentos.
- Circuit breaking si proveedor falla mucho.
- Detección automática de deprecaciones de modelo.

### 2.2 NO es responsable de

- **Lógica de negocio:** no sabe qué es un "roadmap" o un "Spark".
- **Generar prompts dinámicamente:** los carga de archivos versionados
  (templates con placeholders).
- **Mantener estado de usuario:** stateless excepto métricas
  agregadas.
- **Decidir cuándo migrar de modelo:** humano decide (con base en
  reportes mensuales §7); Gateway ejecuta el rollout.
- **Rate limiting global del usuario:** eso es responsabilidad de
  Sparks (cap por plan) + Anti-fraud (caps por seguridad).

### 2.3 Tensiones con otros sistemas

| Tensión | Resolución |
|---------|-----------|
| Modelo nuevo cuesta más → ajustar Sparks por operation | Admin actualiza `sparks_operation_costs`. Gateway no decide costos. |
| Validación de output falla → ¿qué devolver al llamador? | Gateway maneja retry interno; al exponer error al llamador, error es claro y task-specific. |
| LLM es lento → ¿bloquear al usuario? | Tasks `realtime` tienen `timeout_ms`. Si excede, error explícito (no esperar). |
| Necesito invocar LLM para "algo nuevo" | NO invocar directo. Registrar nuevo task en Registry primero. |

---

## 3. Arquitectura

### 3.1 Diagrama lógico

```
┌──────────────────────────────────────────────────────────┐
│                  CÓDIGO DE NEGOCIO                       │
│      (no conoce proveedores ni modelos)                  │
└──────────────────────────────────────────────────────────┘
                          ↓
        executeTask({ task_id, input, user_id })
                          ↓
┌──────────────────────────────────────────────────────────┐
│                    AI GATEWAY                            │
│  ┌────────────────────────────────────────────────────┐ │
│  │  1. Resolver Task Registry → config actual         │ │
│  │  2. Validar input con Zod schema                   │ │
│  │  3. Cobrar Sparks (delegando a sparks-system)      │ │
│  │  4. Decidir variante (control / experimental %)    │ │
│  │  5. Cargar prompt versionado para modelo elegido   │ │
│  │  6. Ejecutar con retry exponencial                 │ │
│  │  7. Si primary falla: cae a fallback chain         │ │
│  │  8. Validar output con Zod schema                  │ │
│  │  9. Si validation falla: retry con feedback (1x)   │ │
│  │ 10. Si refund: invocar sparks.refundOperation      │ │
│  │ 11. Registrar métricas y emitir evento             │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
                          ↓
                ┌─────────┼─────────┐
                ↓         ↓         ↓
         ┌──────────┐ ┌──────────┐ ┌──────────┐
         │ Adapter  │ │ Adapter  │ │ Adapter  │
         │Anthropic │ │  Google  │ │  OpenAI  │
         └──────────┘ └──────────┘ └──────────┘
                ↓         ↓         ↓
            [APIs externas]
```

### 3.2 Componentes

| Componente | Para qué |
|-----------|----------|
| **Task Registry** | Catálogo central de tareas IA con primary + fallback + experimental |
| **Provider Adapters** | Wrappers de SDK por proveedor (Anthropic, Google, OpenAI, Azure, self-hosted) |
| **Prompt Loader** | Carga `prompts/<task>/<version>_<provider>.txt` según selección |
| **Validator** | Aplica Zod schema al output del LLM |
| **Retry Engine** | Backoff exponencial, fallback chain |
| **Cost Tracker** | Persiste costo por task, modelo, usuario |
| **Experiment Manager** | Routea % del tráfico a variante experimental |
| **Circuit Breaker** | Si proveedor X falla > N veces en M min, ruteo temporalmente a fallback |

### 3.3 Stack recomendado para empezar

(De ADR-001-ai-gateway.)

| Componente | Tool | Por qué |
|-----------|------|---------|
| Adapter multi-proveedor | LiteLLM (open source) | Interfaz unificada, gratis |
| Observability + logs | Langfuse | Logging y dashboards específicos LLMs, tier gratuito |
| Validation | Zod (TypeScript) | Type-safe, integración natural |
| Golden datasets | Promptfoo | Framework de evaluación, tier gratuito |
| A/B testing | Implementación propia sobre LiteLLM | Lógica simple en el Gateway |

---

## 4. Task Registry

### 4.1 Schema del Task

```typescript
interface TaskDefinition {
  // Identidad
  id: string;                        // ej: 'score_pronunciation'
  description: string;
  category: 'realtime' | 'batch' | 'background';

  // Schemas
  input_schema: ZodSchema;           // valida input antes de invocar
  output_schema: ZodSchema;          // valida output del LLM

  // Configuración primary (variant 'control')
  primary: ModelConfig;
  fallback_chain: ModelConfig[];     // intentar en orden si primary falla

  // Experimentación opcional
  experimental?: {
    config: ModelConfig;
    traffic_pct: number;             // 0-100
    mode: 'shadow' | 'canary';
    started_at: string;
    rollback_criteria: RollbackCriteria;
  };

  // Constraints
  max_cost_usd: number;              // por invocación; alerta si excede
  timeout_ms: number;                // total incluyendo retries
  retry_policy: { max_attempts: number; backoff_ms: number };
  quality_threshold: number;         // % de validations que deben pasar

  // Sparks
  sparks_operation_id?: string;      // ej: 'conversation_minute'; null si free

  // Metadata
  introduced_at: string;
  owner: string;
}

interface ModelConfig {
  provider: 'anthropic' | 'google' | 'openai' | 'azure' | 'self_hosted';
  model: string;                     // ej: 'claude-haiku-4-5'
  prompt_version: string;            // ref a prompts/<task>/<v>_<provider>.txt
  temperature?: number;
  max_tokens?: number;
  use_batch_api?: boolean;           // para Anthropic Batch API
  use_provisioned_throughput?: boolean;
}

interface RollbackCriteria {
  max_validation_rate_drop: number;
  max_latency_p95_increase_ratio: number;
  max_error_rate_increase: number;
  max_cost_increase_ratio: number;
}
```

### 4.2 Catálogo completo de tasks del MVP

#### 4.2.1 Tasks de transcripción

```typescript
'transcribe_user_audio': {
  description: 'STT del audio del usuario para análisis posterior',
  category: 'realtime',
  input_schema: z.object({
    audio_storage_key: z.string(),  // ref a R2
    expected_language: z.literal('en'),
  }),
  output_schema: z.object({
    text: z.string(),
    confidence: z.number().min(0).max(1),
    word_timings: z.array(z.object({
      word: z.string(), start_ms: z.number(), end_ms: z.number(),
    })),
  }),
  primary: {
    provider: 'azure',
    model: 'whisper-large-v3',
    prompt_version: 'n/a',
  },
  fallback_chain: [
    { provider: 'self_hosted', model: 'whisper-large-v3', prompt_version: 'n/a' },
  ],
  max_cost_usd: 0.05,
  timeout_ms: 15000,
  retry_policy: { max_attempts: 2, backoff_ms: 1000 },
  quality_threshold: 0.95,
  sparks_operation_id: null,        // incluido en charge previo de la operation
},
```

#### 4.2.2 Tasks de pronunciation

```typescript
'score_pronunciation': {
  description: 'Evaluar pronunciación de un audio',
  category: 'realtime',
  input_schema: z.object({
    audio_storage_key: z.string(),
    expected_text: z.string(),
    target_variant: z.enum(['en-US', 'en-GB']),
  }),
  output_schema: z.object({
    overall_score: z.number().min(0).max(100),
    word_scores: z.array(z.object({
      word: z.string(), score: z.number().min(0).max(100),
    })),
    phoneme_errors: z.array(z.object({
      phoneme: z.string(), expected: z.string(), produced: z.string(),
    })),
  }),
  primary: {
    provider: 'azure',
    model: 'pronunciation-assessment',
    prompt_version: 'n/a',
  },
  fallback_chain: [
    { provider: 'self_hosted', model: 'pronunciation-v3', prompt_version: 'n/a' },
  ],
  max_cost_usd: 0.04,
  timeout_ms: 10000,
  retry_policy: { max_attempts: 2, backoff_ms: 1000 },
  quality_threshold: 0.98,
},
```

#### 4.2.3 Tasks de grammar

```typescript
'detect_grammar_errors': {
  description: 'Detectar errores gramaticales en transcripción de speech',
  category: 'realtime',
  input_schema: z.object({
    transcription: z.string(),
    target_cefr: z.enum(['A2','B1','B1+','B2','B2+','C1']),
    exercise_context: z.enum([
      'free_response','roleplay_free','roleplay_structured',
      'image_description','translation','fill_blank',
      'sentence_ordering','read_aloud',
    ]),
  }),
  output_schema: z.object({
    errors: z.array(z.object({
      type: z.string(),
      severity: z.enum(['minor','moderate','major']),
      position: z.number(),
      excerpt: z.string(),
      suggestion: z.string(),
      affected_subskill: z.string(),
    })),
  }),
  primary: {
    provider: 'anthropic',
    model: 'claude-haiku-4-5',
    prompt_version: 'v1_claude',
    temperature: 0.1,
    max_tokens: 1500,
  },
  fallback_chain: [
    { provider: 'google', model: 'gemini-2.0-flash', prompt_version: 'v1_gemini', temperature: 0.1 },
    { provider: 'openai', model: 'gpt-4o-mini', prompt_version: 'v1_gpt4o_mini', temperature: 0.1 },
  ],
  max_cost_usd: 0.01,
  timeout_ms: 8000,
  retry_policy: { max_attempts: 2, backoff_ms: 1000 },
  quality_threshold: 0.95,
},
```

#### 4.2.4 Tasks de roadmap

```typescript
'generate_initial_roadmap': {
  description: 'Generar roadmap inicial al completar onboarding',
  category: 'realtime',
  input_schema: RoadmapInputSchema,    // ver ai-roadmap-system.md
  output_schema: RoadmapOutputSchema,
  primary: {
    provider: 'anthropic',
    model: 'claude-haiku-4-5',
    prompt_version: 'v2_claude',
    temperature: 0.3,
    max_tokens: 4000,
  },
  fallback_chain: [
    { provider: 'google', model: 'gemini-2.0-flash', prompt_version: 'v1_gemini' },
    { provider: 'openai', model: 'gpt-4o-mini', prompt_version: 'v1_gpt4o_mini' },
  ],
  max_cost_usd: 0.05,
  timeout_ms: 15000,
  retry_policy: { max_attempts: 2, backoff_ms: 1000 },
  quality_threshold: 0.95,
  sparks_operation_id: null,        // free, parte del onboarding
},

'nightly_roadmap_update': {
  description: 'Actualizar roadmap del usuario en job nocturno',
  category: 'batch',
  input_schema: RoadmapUpdateInputSchema,
  output_schema: RoadmapUpdateOutputSchema,
  primary: {
    provider: 'anthropic',
    model: 'claude-haiku-4-5',
    prompt_version: 'v1_claude',
    use_batch_api: true,            // 50% off
    temperature: 0.3,
    max_tokens: 2000,
  },
  fallback_chain: [
    { provider: 'google', model: 'gemini-2.0-flash', prompt_version: 'v1_gemini' },
  ],
  max_cost_usd: 0.02,
  timeout_ms: 60000,                  // batch puede ser lento
  retry_policy: { max_attempts: 3, backoff_ms: 5000 },
  quality_threshold: 0.95,
  sparks_operation_id: null,        // incluido en plan
},
```

#### 4.2.5 Tasks de mensajes humanizados

```typescript
'generate_morning_message': {
  description: 'Generar mensaje matutino personalizado',
  category: 'background',
  input_schema: MorningMessageInputSchema,
  output_schema: z.object({
    title: z.string().max(50),
    body: z.string().max(120),
  }),
  primary: {
    provider: 'anthropic',
    model: 'claude-haiku-4-5',
    prompt_version: 'v1_claude',
    use_batch_api: true,
    temperature: 0.7,
    max_tokens: 200,
  },
  fallback_chain: [
    { provider: 'google', model: 'gemini-2.0-flash', prompt_version: 'v1_gemini' },
  ],
  max_cost_usd: 0.001,
  timeout_ms: 30000,
  retry_policy: { max_attempts: 2, backoff_ms: 2000 },
  quality_threshold: 0.95,
  sparks_operation_id: null,
},

'generate_weekly_summary': {
  description: 'Generar resumen semanal del progreso',
  category: 'background',
  input_schema: WeeklySummaryInputSchema,
  output_schema: WeeklySummaryOutputSchema,
  primary: {
    provider: 'anthropic',
    model: 'claude-haiku-4-5',
    prompt_version: 'v1_claude',
    use_batch_api: true,
  },
  // ...
  sparks_operation_id: 'weekly_summary_generation',  // 3 Sparks
},
```

#### 4.2.6 Tasks de roleplay y conversación

```typescript
'conversation_turn': {
  description: 'Un turno de conversación 1:1 con la IA',
  category: 'realtime',
  input_schema: z.object({
    conversation_history: z.array(z.object({
      role: z.enum(['user', 'assistant']),
      content: z.string(),
    })),
    user_profile: UserProfileSnapshotSchema,
    block_context: z.string().optional(),
  }),
  output_schema: z.object({
    response: z.string(),
    target_subskills_practiced: z.array(z.string()),
  }),
  primary: {
    provider: 'anthropic',
    model: 'claude-sonnet-4-6',     // mejor calidad para conversación
    prompt_version: 'v1_claude',
    temperature: 0.7,
    max_tokens: 500,
  },
  fallback_chain: [
    { provider: 'google', model: 'gemini-2.0-flash', prompt_version: 'v1_gemini' },
  ],
  max_cost_usd: 0.05,
  timeout_ms: 8000,
  retry_policy: { max_attempts: 2, backoff_ms: 500 },
  quality_threshold: 0.97,
  sparks_operation_id: 'conversation_minute',  // 1 Spark/min
},

'generate_personalized_roleplay': {
  description: 'Generar roleplay custom cuando biblioteca no tiene match',
  category: 'realtime',
  // ...
  sparks_operation_id: 'generate_personalized_roleplay',  // 5 Sparks
},
```

#### 4.2.7 Tasks de notifications

```typescript
'generate_notification_content': {
  description: 'Generar contenido personalizado de daily reminder + streak alert',
  category: 'background',
  // ...
  primary: {
    provider: 'anthropic',
    model: 'claude-haiku-4-5',
    prompt_version: 'v1_claude',
    use_batch_api: true,
    temperature: 0.6,
    max_tokens: 300,
  },
  // ...
},
```

#### 4.2.8 Tasks de Customer Support

```typescript
'ai_assistant_response': {
  description: 'Generar respuesta del AI Assistant a consulta de soporte',
  category: 'realtime',
  input_schema: z.object({
    user_message: z.string(),
    conversation_history: z.array(/* ... */),
    user_context: z.object({/* ... */}),
    relevant_articles: z.array(z.string()),
  }),
  output_schema: z.object({
    response: z.string(),
    escalation_needed: z.boolean(),
    escalation_reason: z.string().optional(),
    create_ticket: z.boolean(),
    ticket_priority: z.enum(['low','normal','high','critical']).optional(),
  }),
  primary: {
    provider: 'anthropic',
    model: 'claude-haiku-4-5',
    prompt_version: 'v1_claude',
    temperature: 0.4,
    max_tokens: 500,
  },
  fallback_chain: [
    { provider: 'google', model: 'gemini-2.0-flash', prompt_version: 'v1_gemini' },
  ],
  max_cost_usd: 0.005,
  timeout_ms: 5000,
  retry_policy: { max_attempts: 1, backoff_ms: 1000 },
  quality_threshold: 0.99,
  sparks_operation_id: null,        // incluido en plan
},
```

### 4.3 Persistencia del Registry

Archivo TypeScript versionado en repo:

```
apps/workers/ai-gateway/
├── src/
│   ├── tasks/
│   │   ├── transcribe_user_audio.ts
│   │   ├── score_pronunciation.ts
│   │   ├── detect_grammar_errors.ts
│   │   └── ...
│   ├── registry.ts                ← exports merge de todos
│   ├── adapters/
│   ├── prompts/                   ← (a copiar de docs)
│   └── ...
└── prompts/
    ├── transcribe_user_audio/
    ├── score_pronunciation/
    └── ...
```

Cambios al Registry pasan por PR review. Después de merge, deploy
gradual con feature flags si el cambio es riesgoso.

---

## 5. Provider Adapters

### 5.1 Interfaz unificada

Cada adapter implementa:

```typescript
interface ProviderAdapter {
  provider_id: 'anthropic' | 'google' | 'openai' | 'azure' | 'self_hosted';

  invoke(request: ProviderRequest): Promise<ProviderResponse>;

  estimateCost(request: ProviderRequest, response?: ProviderResponse): number;

  isHealthy(): Promise<boolean>;

  getCapabilities(): {
    supports_batch_api: boolean;
    supports_streaming: boolean;
    supports_provisioned_throughput: boolean;
  };
}

interface ProviderRequest {
  model: string;
  prompt: string;
  temperature?: number;
  max_tokens?: number;
  metadata: { task_id: string; user_id_hash?: string };
}

interface ProviderResponse {
  output: string;
  usage: {
    input_tokens: number;
    output_tokens: number;
    cost_usd: number;
  };
  latency_ms: number;
  raw_response?: unknown;             // para debugging, no persistir
}
```

### 5.2 Adapters concretos

| Adapter | Implementación |
|---------|---------------|
| Anthropic | LiteLLM con `claude-*` models |
| Google | LiteLLM con `gemini-*` models |
| OpenAI | LiteLLM con `gpt-*` models |
| Azure (Pronunciation) | SDK directo, no via LiteLLM (es speech, no LLM) |
| Azure (Whisper) | LiteLLM con `azure/whisper-large-v3` |
| Self-hosted | HTTPS endpoint propio (Modal, GPU pod, etc.) |

### 5.3 Modelos propios

Tratamiento idéntico a APIs externas. Endpoint propio expuesto vía
HTTPS:

```typescript
{
  provider: 'self_hosted',
  model: 'pronunciation-v3',
  prompt_version: 'n/a',
  endpoint: 'https://ml.internal.example.com/pronunciation/v3',
}
```

Beneficios del tratamiento uniforme:
- Si modelo propio falla, fallback automático a API externa.
- Si API externa falla, modelo propio toma el tráfico.
- Migración gradual sigue mismo protocolo.
- A/B testing trivial.

---

## 6. Prompt versioning

### 6.1 Estructura de archivos

```
apps/workers/ai-gateway/prompts/
├── score_pronunciation/
│   └── (n/a — Azure Pronunciation no usa prompt)
├── detect_grammar_errors/
│   ├── v1_claude.txt
│   ├── v1_gemini.txt
│   ├── v1_gpt4o_mini.txt
│   └── README.md
├── generate_initial_roadmap/
│   ├── v1_claude.txt              (deprecado)
│   ├── v2_claude.txt              (actual)
│   ├── v1_gemini.txt              (actual Gemini)
│   ├── v1_gpt4o_mini.txt
│   └── README.md
└── ...
```

### 6.2 Convenciones

- Filename: `v<version>_<provider>.txt`.
- Versiones son monotónicas crecientes por task+provider.
- Old versions **se mantienen** (deprecadas) para auditoría y
  rollback rápido.
- README.md por task explica evolución y razones de cambio.

### 6.3 Por qué versionar prompt por modelo

Modelos tienen comportamientos sutilmente distintos:

- Claude tiende a respetar mejor estructura de XML tags.
- Gemini funciona mejor con instrucciones en formato numerado claro.
- GPT-4o-mini necesita reforzar reglas de formato JSON.

Optimizar el prompt para cada modelo extrae el mejor rendimiento
posible.

### 6.4 Templates con placeholders

```text
# detect_grammar_errors/v1_claude.txt

Eres un experto en gramática inglesa evaluando producción oral de
hispanohablantes latinoamericanos.

CONTEXTO:
- Nivel CEFR target: {{target_cefr}}
- Tipo de ejercicio: {{exercise_context}}

TRANSCRIPCIÓN A EVALUAR:
{{transcription}}

INSTRUCCIONES:
1. Identificá errores gramaticales reales (no estilísticos).
2. Para cada error, clasificá severidad:
   - minor: no afecta comprensión
   - moderate: notable pero comprensible
   - major: causa confusión
3. Para errores en {{exercise_context}}, sé tolerante con disfluencias
   normales del speech espontáneo.
4. Para B1, no marqués estructuras C1 como "esperadas".

DEVOLVÉ JSON SIGUIENDO EL SCHEMA:
{
  "errors": [
    {
      "type": "...",
      "severity": "minor|moderate|major",
      "position": ...,
      "excerpt": "...",
      "suggestion": "...",
      "affected_subskill": "..."
    }
  ]
}

Devolvé SOLO el JSON, sin markdown ni texto adicional.
```

### 6.5 Workflow de evolución

1. Identificar problema: ej. validation rate cae a 92%, debería ser
   ≥ 95%.
2. Crear `v3_claude.txt` con cambios.
3. Correr golden dataset (§7) contra ambas versiones.
4. Si v3 mejora: actualizar Task Registry para apuntar a v3.
5. Mantener v2 con `deprecated_at` para audit y rollback.

---

## 7. Golden datasets y evaluación

### 7.1 Concepto

Para cada task, dataset fijo de casos de prueba que sirve como verdad
de referencia para evaluar cualquier modelo o prompt.

### 7.2 Estructura

```
apps/workers/ai-gateway/golden_datasets/
├── detect_grammar_errors/
│   ├── case_001_b1_minor_errors.json
│   ├── case_002_b2_no_errors.json
│   ├── case_003_b1_major_misuse_perfect.json
│   ├── ...
│   └── case_100_edge_case_empty_input.json
├── generate_initial_roadmap/
│   └── ...
└── ...
```

### 7.3 Composición de un caso

```json
{
  "id": "case_002_b2_sales_urgent",
  "description": "B2 en sales con deadline en 3 semanas",
  "input": {
    "primary_goals": ["job_interview"],
    "deadline": "2026-05-15",
    "professional_context": "sales",
    "self_perceived_level": "intermediate",
    "self_perceived_anxiety": 3,
    "daily_minutes_available": 30,
    "initial_test_results": {
      "cefr_estimate": "B2",
      "fluency_score": 6.5,
      "pronunciation_score": 5.8,
      "detected_error_patterns": ["past_perfect", "th_pronunciation"]
    }
  },
  "validation_criteria": {
    "must_have_blocks_in_categories": ["interview_basics", "sales_specific"],
    "must_prioritize_pronunciation": true,
    "max_total_blocks": 35,
    "estimated_weeks_must_be_between": [2, 4],
    "all_block_ids_must_exist": true,
    "must_respect_prerequisites": true
  },
  "expected_output_sample": "..."
}
```

### 7.4 Cómo se construyen

- **Casos típicos:** derivados de usuarios reales (anonimizados) o
  generados manualmente cubriendo perfiles representativos.
- **Edge cases:** deadlines extremos, inputs mínimos, valores fuera
  de rango pero válidos.
- **Casos de regresión:** cualquier caso que falló en producción se
  agrega aquí para garantizar que no vuelva.

Tamaño objetivo: 30–100 casos por task. Más es mejor pero hay
rendimientos decrecientes.

### 7.5 Uso del dataset

```bash
# Evaluar modelo nuevo contra una task
pnpm eval --task detect_grammar_errors --model gemini-2.5-flash

# Output:
# ✓ 95/100 cases passed validation
# Average cost: $0.0002 per call
# Average latency: 1.2s
# Comparison vs current (claude-haiku-4-5):
#   Cost: -50%
#   Latency: -15%
#   Validation pass rate: -1%
```

Convierte decisión subjetiva en data-driven.

### 7.6 Job mensual de evaluación

Primer lunes del mes, automático:

```typescript
async function monthlyEvaluation() {
  for (const task of taskRegistry) {
    for (const model of availableModels) {
      const results = await runGoldenDataset(task, model);
      report.add({ task, model, results });
    }
  }
  generateReport(report);
  notifyTeam(report);
}
```

### 7.7 Formato del reporte

```
═══════════════════════════════════════════════════════════
   AI MODEL EVALUATION REPORT - 2026-05
═══════════════════════════════════════════════════════════

TASK: detect_grammar_errors
Current: claude-haiku-4-5 ($0.0004/call, 96.0% pass, 1.4s)

Alternatives evaluated:
┌──────────────────┬──────────┬──────────┬──────────┬──────────┐
│ Model            │ Cost     │ Quality  │ Latency  │ Verdict  │
├──────────────────┼──────────┼──────────┼──────────┼──────────┤
│ gemini-2.5-flash │ $0.0002  │ 95.0%    │ 1.1s     │ EVALUATE │
│ gpt-4o-mini-v2   │ $0.0003  │ 97.0%    │ 1.5s     │ EVALUATE │
│ claude-haiku-5   │ $0.0003  │ 98.0%    │ 1.3s     │ MIGRATE  │
└──────────────────┴──────────┴──────────┴──────────┴──────────┘

Recommendation: Initiate migration to claude-haiku-5
  - Same provider (low risk)
  - 25% cost reduction
  - +2% quality improvement

ESTIMATED MONTHLY SAVINGS IF ALL MIGRATIONS APPROVED: $1,240
```

Decisión humana, ejecución automatizada (rollout sigue protocolo §8).

---

## 8. Migración progresiva (rollout)

### 8.1 Fases

**Fase 0 — Evaluación offline (días 1-3):**
Correr golden dataset contra modelo candidato. Si pasa quality
threshold, continuar.

**Fase 1 — Shadow mode (días 4-10):**
- 100% del tráfico va al modelo actual y se le devuelve al usuario.
- Mismo input se envía en paralelo al candidato.
- Outputs del candidato se loggean pero NO se le muestran al usuario.
- Comparación automática.

**Fase 2 — Canary 5% (días 11-15):**
- 5% del tráfico real va al candidato.
- Métricas: costo, latencia, validation rate, error rate.
- Idealmente, métrica de producto (ej: tasa de completion del
  onboarding).

**Fase 3 — Ramp up (días 16-25):**
- 5% → 25%, esperar 2-3 días, evaluar.
- 25% → 50%, esperar, evaluar.
- 50% → 100%.

**Fase 4 — Modelo viejo en fallback (días 26+):**
- Modelo viejo queda en fallback chain.
- 30 días después, se retira.

### 8.2 Criterios de rollback automático

Rollback si **alguno** de:

- Validation rate cae > 3% respecto a baseline.
- Latencia p95 sube > 50% respecto a baseline.
- Error rate del proveedor sube > 2%.
- Costo por llamada sube inesperadamente > 20%.

### 8.3 Implementación en Registry

```typescript
'detect_grammar_errors': {
  primary: {
    provider: 'anthropic',
    model: 'claude-haiku-4-5',
    prompt_version: 'v2_claude',
  },
  experimental: {
    config: {
      provider: 'anthropic',
      model: 'claude-haiku-5',
      prompt_version: 'v3_claude',
    },
    traffic_pct: 5,
    mode: 'canary',
    started_at: '2026-04-25T00:00:00Z',
    rollback_criteria: {
      max_validation_rate_drop: 0.03,
      max_latency_p95_increase_ratio: 1.5,
      max_error_rate_increase: 0.02,
      max_cost_increase_ratio: 1.2,
    },
  },
  // ...
},
```

---

## 9. Manejo de deprecaciones

### 9.1 Detección automática

Job semanal de CI verifica estado de cada modelo en uso:

```typescript
for (const task of taskRegistry) {
  for (const config of [task.primary, ...task.fallback_chain]) {
    const status = await checkModelDeprecation(config);
    if (status.deprecated) {
      await createGitHubIssue({
        title: `Deprecation: ${config.model}`,
        body: `Sunset: ${status.sunset_date}, replacement: ${status.replacement}`,
        labels: ['ai-deprecation', 'priority-medium'],
      });
    }
  }
}
```

### 9.2 Proceso de respuesta

| Día | Acción |
|-----|--------|
| 0 | Issue automático creado, equipo notificado |
| 1–3 | Identificar modelo de reemplazo |
| 4–10 | Correr golden dataset, ajustar prompts |
| 11–30 | Rollout progresivo (§8) |
| 30 | Modelo nuevo sirve 100% |
| 30–90 | Modelo viejo en fallback |
| 90+ | Modelo viejo retirado |

Generalmente proveedores dan 6–12 meses de notice; el proceso debería
tomar < 30 días.

### 9.3 Plan de contingencia (deprecation acelerada)

Si proveedor anuncia deprecation con < 30 días:

1. Activar fallback inmediatamente.
2. Acelerar evaluación del reemplazo (golden dataset en 24-48h).
3. Skip shadow mode si urgencia lo requiere; ir directo a canary.
4. Migración completa en 2 semanas o menos.

Solo posible si arquitectura del Gateway está implementada y el
secundario está activo. De ahí la importancia de la inversión inicial.

---

## 10. Cost tracking y observability

### 10.1 Métricas mínimas por llamada

Cada invocación del Gateway registra:

```sql
CREATE TABLE ai_gateway_invocations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  task_id         TEXT NOT NULL,
  model_used      TEXT NOT NULL,
  prompt_version  TEXT NOT NULL,
  started_at      TIMESTAMPTZ NOT NULL,
  completed_at    TIMESTAMPTZ NOT NULL,
  latency_ms      INT NOT NULL,
  input_tokens    INT,
  output_tokens   INT,
  cost_usd        NUMERIC(10,6) NOT NULL,
  validation_passed BOOLEAN NOT NULL,
  validation_errors JSONB,
  retry_attempts  INT NOT NULL DEFAULT 0,
  fallback_used   BOOLEAN NOT NULL DEFAULT false,
  user_id_hash    TEXT,                 -- SHA-256
  experiment_variant TEXT,              -- si A/B test
  trace_id        TEXT
);

CREATE INDEX idx_ai_invocations_task_date ON ai_gateway_invocations(task_id, started_at DESC);
CREATE INDEX idx_ai_invocations_user ON ai_gateway_invocations(user_id_hash, started_at DESC) WHERE user_id_hash IS NOT NULL;
CREATE INDEX idx_ai_invocations_model ON ai_gateway_invocations(model_used, started_at DESC);
```

Retention: 90 días detallado; agregados mensuales indefinidos.

### 10.2 Dashboard interno

Página interna admin que muestra:

- **Por task:** modelo actual, costo promedio, validation rate,
  latencia p50/p95/p99 en 24h, 7d, 30d.
- **Experimentos activos:** tasks en shadow o canary, métricas
  comparativas en tiempo real.
- **Alertas activas:** deprecaciones anunciadas, picos de costo,
  fallos recientes.
- **Tendencias:** costo total mensual, distribución por task,
  evolución semana a semana.

### 10.3 Alertas críticas

- Costo del día > 2x promedio histórico.
- Validation rate de cualquier task < 90%.
- Latencia p95 > 2x baseline.
- Cualquier usuario individual con costo > $1 en 24h.
- Proveedor primario con error rate > 5%.
- Anuncio de deprecation detectado.

### 10.4 Cost tracking por usuario

```sql
CREATE TABLE ai_cost_per_user_daily (
  user_id_hash    TEXT NOT NULL,
  date            DATE NOT NULL,
  total_cost_usd  NUMERIC(10,4) NOT NULL,
  invocations     INT NOT NULL,
  PRIMARY KEY (user_id_hash, date)
);
```

Recalculado por cron diario. Permite identificar usuarios anómalos
(probable bug o abuso).

---

## 11. API contracts

### 11.1 `executeTask`

**Llamado por:** cualquier sistema que necesita IA.

**Request:**

```typescript
interface ExecuteTaskRequest<TInput> {
  task_id: string;
  input: TInput;                       // matchea task.input_schema
  user_id?: string;                    // para Sparks charge + cost tracking
  trace_id?: string;
  metadata?: Record<string, unknown>;
}
```

**Response:**

```typescript
interface ExecuteTaskResponse<TOutput> {
  output: TOutput;                     // matchea task.output_schema
  metadata: {
    invocation_id: string;
    model_used: string;
    cost_usd: number;
    latency_ms: number;
    fallback_used: boolean;
    experiment_variant: string | null;
  };
}
```

**Errores:**

| Código | Causa |
|--------|-------|
| `TASK_NOT_REGISTERED` | `task_id` no existe |
| `INPUT_VALIDATION_FAILED` | Input no matchea Zod schema |
| `INSUFFICIENT_SPARKS` | `chargeOperation` rechazó |
| `RESTRICTED_FEATURE` | User restriction deniega operation |
| `OUTPUT_VALIDATION_FAILED_PERSISTENT` | Output del LLM falla validation incluso después de retry + fallback |
| `ALL_PROVIDERS_FAILED` | Primary y todo el fallback chain fallaron |
| `TIMEOUT` | Excedió `task.timeout_ms` |
| `BUDGET_EXCEEDED` | `task.max_cost_usd` excedido (con fallback ya intentado) |

### 11.2 Flujo de `executeTask`

```typescript
async function executeTask<TInput, TOutput>(
  req: ExecuteTaskRequest<TInput>
): Promise<ExecuteTaskResponse<TOutput>> {
  const task = registry.get(req.task_id);
  if (!task) throw new Error('TASK_NOT_REGISTERED');

  // 1. Validar input
  const input = task.input_schema.parse(req.input);

  // 2. Cobrar Sparks (si aplica)
  let charge: ChargeOperationResponse | null = null;
  if (task.sparks_operation_id && req.user_id) {
    charge = await sparks.chargeOperation({
      user_id: req.user_id,
      operation_id: task.sparks_operation_id,
      idempotency_key: `task:${req.trace_id ?? randomUUID()}`,
    });
  }

  // 3. Decidir variante (control / experimental)
  const variant = decideVariant(task);

  // 4. Cargar prompt
  const prompt = await loadPrompt(task.id, variant.config.prompt_version);

  // 5. Ejecutar con retry y fallback
  let response: ProviderResponse | null = null;
  let lastError: Error | null = null;
  let fallbackUsed = false;

  const candidates = [variant.config, ...task.fallback_chain];

  for (const config of candidates) {
    try {
      response = await invokeWithRetry(config, prompt, input, task);

      // 6. Validar output
      const parsed = task.output_schema.safeParse(JSON.parse(response.output));
      if (parsed.success) {
        // ¡Listo!
        await logInvocation({ /* metrics */ });
        return {
          output: parsed.data as TOutput,
          metadata: {
            invocation_id: response.invocation_id,
            model_used: config.model,
            cost_usd: response.usage.cost_usd,
            latency_ms: response.latency_ms,
            fallback_used: fallbackUsed,
            experiment_variant: variant.experiment_variant,
          },
        };
      } else {
        // Validation falló, intentar 1 retry con feedback
        const retryPrompt = prompt + '\n\nERROR EN OUTPUT ANTERIOR: ' + parsed.error.message;
        const retryResp = await invokeWithRetry(config, retryPrompt, input, task);
        const retryParsed = task.output_schema.safeParse(JSON.parse(retryResp.output));
        if (retryParsed.success) {
          await logInvocation({ /* metrics with retry */ });
          return { output: retryParsed.data, metadata: { /* ... */ } };
        }
        // Fallback al siguiente provider
        lastError = new Error('OUTPUT_VALIDATION_FAILED');
        fallbackUsed = true;
        continue;
      }
    } catch (err) {
      lastError = err;
      fallbackUsed = true;
      continue;
    }
  }

  // 7. Refund de Sparks si todo falla
  if (charge) {
    await sparks.refundOperation({
      charge_id: charge.charge_id,
      reason: 'all_providers_failed',
    });
  }

  throw lastError ?? new Error('ALL_PROVIDERS_FAILED');
}
```

### 11.3 `getTaskMetrics`

**Llamado por:** dashboard interno, monitoring.

```typescript
interface GetTaskMetricsRequest {
  task_id: string;
  period: '1h' | '24h' | '7d' | '30d';
}

interface GetTaskMetricsResponse {
  invocations: number;
  validation_pass_rate: number;
  fallback_rate: number;
  avg_cost_usd: number;
  total_cost_usd: number;
  latency_p50_ms: number;
  latency_p95_ms: number;
  latency_p99_ms: number;
  by_model: Array<{
    model: string;
    invocations: number;
    avg_cost_usd: number;
  }>;
}
```

---

## 12. Edge cases (tests obligatorios)

### 12.1 Provider failures

1. **Primary 503:** automatic fallback a secondary. Sin error visible al
   llamador.
2. **Primary timeout:** mismo, fallback.
3. **Todos los providers fallan:** error `ALL_PROVIDERS_FAILED` al
   llamador. Sparks refunded.
4. **Provider returns 200 pero output vacío:** validation falla, retry
   1x con feedback, si vuelve a fallar: fallback.

### 12.2 Output validation

5. **Output JSON malformado:** retry 1x; si falla, fallback.
6. **Output JSON válido pero no matchea Zod schema:** retry 1x con
   feedback ("output anterior tenía estos errores: X"); si falla,
   fallback.
7. **Output dice "I cannot help with that":** validation falla
   (schema no matchea), retry/fallback.

### 12.3 Idempotency y trace

8. **executeTask llamado 2 veces con mismo idempotency_key:** segundo
   retorna cached result, no re-invoca.
9. **Trace_id propagado a logs:** verificable en
   `ai_gateway_invocations.trace_id`.

### 12.4 Sparks integration

10. **Sparks charge falla (insufficient):** `executeTask` retorna
    error `INSUFFICIENT_SPARKS` sin invocar al modelo.
11. **Modelo invoca exitoso pero Sparks refund falla:** alerta crítica.
    Operation result se devuelve al usuario; reconciliación posterior
    corrige Sparks balance.

### 12.5 Experiments

12. **5% canary:** verificable que aprox 5% de invocations tienen
    `experiment_variant != null`.
13. **Rollback automático:** dado validation rate del experimental
    cae > 3%, experiment_variant deja de routar tráfico.

### 12.6 Costos

14. **Invocation excede `max_cost_usd`:** alerta + log; operación se
    completa pero se marca para review.
15. **Usuario individual gasta > $1 en 24h:** alerta automática a
    Sentry para review humano (potencial bug o abuso).

---

## 13. Plan de implementación

### 13.1 Fase 1: Foundation (mes 1)

- AI Gateway básico con LiteLLM como adapter.
- Task Registry inicial con 3-5 tareas (transcribe, score_pronunciation,
  detect_grammar_errors, generate_initial_roadmap, generate_morning_message).
- Logging básico (cost, latency, validation result).
- Validation con Zod para cada task.
- Un solo proveedor por task (sin fallbacks aún).
- Integración con Sparks: `chargeOperation` antes de invocar.

### 13.2 Fase 2: Resilience (meses 2-3)

- Fallback chain a cada task.
- Retry con backoff exponencial.
- Setup de Langfuse para observabilidad.
- Primer dashboard interno.
- Idempotency keys.

### 13.3 Fase 3: Quality (meses 3-6)

- Golden datasets para tasks críticas (≥ 30 casos cada una).
- Setup de Promptfoo.
- Implementar shadow mode.
- Versionado de prompts por modelo (v1 → v2).

### 13.4 Fase 4: Optimization (meses 6-9)

- Canary mode con rampa progresiva.
- Job mensual de evaluación de modelos.
- Sistema de alertas críticas.
- Procesos de respuesta a deprecaciones.

### 13.5 Fase 5: Self-hosted (meses 9-12)

- Integrar primeros modelos propios al Gateway.
- Migración progresiva de tráfico de API a modelo propio.
- Métricas comparativas en producción.

---

## 14. Métricas

### 14.1 Métricas críticas

| Métrica | Target |
|---------|--------|
| Tiempo de migración a modelo nuevo (decisión → 100% rollout) | < 30 días |
| Outages causados por cambios de proveedor | 0 |
| Captura de ahorros (ahorro identificado / ahorro capturado) | ≥ 80% |
| Costo de IA como % del revenue | Decreciente trimestre a trimestre |
| Tiempo entre anuncio de deprecation y migración completa | < 60 días |
| Validation rate global | > 95% |
| Fallback rate global | < 5% |

### 14.2 Alertas

- Costo del día > 2x promedio: SEV-2.
- Validation rate de cualquier task < 90%: SEV-2.
- Latencia p95 > 2x baseline: SEV-2.
- Usuario individual > $1 en 24h: review.
- Provider primario con error rate > 5%: SEV-2.
- Anuncio de deprecation detectado: review.
- Todos los providers de un task failing: SEV-1.

---

## 15. Decisiones cerradas

### 15.1 Build vs Buy del Gateway: **LiteLLM + custom logic** ✓

**Razón:** LiteLLM cubre 70% gratis. Construir todo desde cero es 3-6
meses sin diferenciación. Servicios managed (Portkey, OpenRouter)
agregan vendor lock-in y costos por volumen.

### 15.2 Promptfoo vs Braintrust: **probar ambos en evaluación** ✓

**Razón:** ambos tienen tier gratuito. Decidir basado en qué framework
es más cómodo después de uso real. Default arranque: Promptfoo (más
simple).

### 15.3 Versionado de prompts: **archivos en repo (no servicio externo)** ✓

**Razón:** PromptLayer u otros agregan dependencia externa para algo
que git resuelve. Cambios pasan por PR review (auditoría natural).

### 15.4 Retención de logs de llamadas: **90 días detallado, agregados
indefinidos** ✓

**Razón:** detallados son grandes y privacidad; agregados son chicos
y útiles para análisis histórico. 90 días es suficiente para forensia
de incidentes.

### 15.5 % activo del secundario por categoría: **5% en realtime
crítico, solo fallback en batch/background** ✓

**Razón:** verificación de fallback + datos comparativos vale la pena
solo en realtime crítico. Batch/background pueden vivir con
fallback frío (acepta latencia adicional cuando se activa).

---

## 16. Aspectos de costo

### 16.1 Costos del stack

(De ai-roadmap-system §8.3, contextualizado para AI Gateway:)

A 100k usuarios activos, costo de IA es aproximadamente $1.000–$1.500/mes
distribuido entre todas las tasks.

### 16.2 Provisioned throughput

Para tasks críticas y de alto volumen, evaluar contratos de capacidad
reservada con proveedores que lo ofrecen (Anthropic Provisioned
Throughput a partir de cierto volumen). Da:

- Precio fijo por contrato.
- Capacidad garantizada (no rate limits).
- Aislamiento de cambios de precio durante el contrato.

Solo justificable a partir de cierto volumen (post-soft-launch).

---

## 17. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`sparks-system.md`](sparks-system.md) | Cobra Sparks antes de cada invocación. |
| [`../decisions/ADR-001-ai-gateway.md`](../decisions/ADR-001-ai-gateway.md) | Decisión arquitectónica. |
| [`../product/pedagogical-system.md`](../product/pedagogical-system.md) | Consume tasks: transcribe_user_audio, score_pronunciation, detect_grammar_errors, evaluate_listening_response. |
| [`../product/ai-roadmap-system.md`](../product/ai-roadmap-system.md) | Consume tasks: generate_initial_roadmap, nightly_roadmap_update, generate_definitive_roadmap. |
| [`../product/student-profile-and-assessment.md`](../product/student-profile-and-assessment.md) | Consume scoring tasks durante assessment. |
| [`notifications-system.md`](notifications-system.md) | Consume generate_notification_content y generate_morning_message en batch nocturno. |
| [`../business/customer-support-system.md`](../business/customer-support-system.md) | Consume ai_assistant_response. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.11 | Eventos `ai.task_completed`. |
| [`../cross-cutting/security-threat-model.md`](../cross-cutting/security-threat-model.md) §3.4 | Amenazas LLM-specific. |
| [`../cross-cutting/testing-strategy.md`](../cross-cutting/testing-strategy.md) §4.9 | Tests obligatorios. |

---

*Documento vivo. Actualizar cuando se agreguen tasks nuevas, cambien
proveedores o se aprenda de la implementación.*

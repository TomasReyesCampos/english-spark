# AI Gateway Strategy

> Estrategia para gestionar dependencias de modelos LLM, manejar deprecaciones,
> capturar mejoras de costo/calidad y mantener el sistema resiliente ante
> cambios del ecosistema de IA.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —

---

## 1. Problema que resuelve

Los proveedores de modelos LLM (Anthropic, OpenAI, Google, etc.) cambian con
frecuencia significativamente mayor que cualquier otra dependencia del stack:

- Deprecan modelos con aviso típico de 6-12 meses.
- Ajustan precios varias veces al año (generalmente a la baja).
- Lanzan modelos nuevos cada 2-3 meses, frecuentemente más baratos y mejores.
- Cambian APIs, rate limits y políticas de uso.

Sin una estrategia explícita, cada uno de estos eventos genera trabajo de
emergencia, riesgo de outages y pérdida de oportunidades de optimización.
Con la estrategia correcta, son eventos planificados que mejoran el sistema.

---

## 2. Principios rectores

### 2.1 Nunca acoplar código de negocio a un proveedor

Regla de oro: si el código de la aplicación menciona "anthropic", "openai",
"gemini" o cualquier nombre de proveedor fuera de la capa de abstracción,
es un bug de diseño que debe corregirse.

### 2.2 Tratar a los LLMs como commodities intercambiables

Toda la arquitectura debe asumir que cualquier LLM puede reemplazarse por
otro con esfuerzo mínimo. Esto no significa que en la práctica sean
equivalentes, significa que la arquitectura no debe ser un obstáculo cuando
se decide cambiar.

### 2.3 Decisiones basadas en datos, no en hype

Cada decisión de migración a un modelo nuevo debe estar respaldada por
métricas medidas en datasets propios, no por benchmarks públicos ni por
marketing del proveedor.

### 2.4 Multi-proveedor desde el día uno

Aunque un proveedor sirva el 95% del tráfico, mantener al menos un segundo
proveedor activo para cada tarea crítica. El costo es despreciable, el
beneficio en resiliencia y poder de negociación es enorme.

### 2.5 Observabilidad antes que optimización

No se puede optimizar lo que no se mide. Cada llamada a un LLM debe
registrar costo, latencia, calidad y resultado de validaciones.

---

## 3. Arquitectura: el AI Gateway

### 3.1 Diagrama lógico

```
┌────────────────────────────────────────────────────────────┐
│                  CÓDIGO DE NEGOCIO                         │
│  (no conoce proveedores ni modelos específicos)            │
└────────────────────────────────────────────────────────────┘
                          ↓
                  executeTask({
                    taskId: "generate_initial_roadmap",
                    inputs: {...}
                  })
                          ↓
┌────────────────────────────────────────────────────────────┐
│                    AI GATEWAY                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  1. Resuelve configuración actual de la tarea       │  │
│  │  2. Selecciona modelo (con A/B test si aplica)      │  │
│  │  3. Carga prompt versionado para ese modelo         │  │
│  │  4. Ejecuta con retry y fallback                    │  │
│  │  5. Valida output con schema                        │  │
│  │  6. Registra métricas (costo, latencia, calidad)    │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                          ↓
              ┌───────────┼───────────┐
              ↓           ↓           ↓
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │ Adapter  │ │ Adapter  │ │ Adapter  │
       │Anthropic │ │  Google  │ │  OpenAI  │
       └──────────┘ └──────────┘ └──────────┘
              ↓           ↓           ↓
        [APIs externas reales]
```

### 3.2 Responsabilidades del Gateway

- **Routing**: decide qué proveedor y modelo usar para cada llamada.
- **Prompt versioning**: carga el prompt correcto para el modelo seleccionado.
- **Validation**: valida outputs contra schemas Zod antes de devolver.
- **Retry policy**: reintenta llamadas fallidas con backoff exponencial.
- **Fallback**: si el primario falla, prueba el secundario automáticamente.
- **Cost tracking**: registra costo de cada llamada por tarea y usuario.
- **A/B testing**: permite enviar % de tráfico a modelos experimentales.
- **Circuit breaking**: si un proveedor falla mucho, rutea temporalmente a otro.

### 3.3 Lo que el Gateway NO hace

- No tiene lógica de negocio (no sabe qué es un "roadmap" o un "Spark").
- No mantiene estado de usuario (es stateless excepto por métricas).
- No genera prompts dinámicamente (los carga de archivos versionados).

---

## 4. Task Registry

### 4.1 Concepto

Toda invocación de IA en el sistema corresponde a una "tarea" registrada
explícitamente. El registry es el inventario completo de la dependencia
de IA del sistema.

### 4.2 Schema del registry

```typescript
interface TaskDefinition {
  id: string;
  description: string;
  category: 'realtime' | 'batch' | 'background';

  // Schemas de input/output
  inputSchema: ZodSchema;
  outputSchema: ZodSchema;

  // Configuración de proveedor
  primary: ModelConfig;
  fallbackChain: ModelConfig[];

  // Experimentación
  experimental?: {
    config: ModelConfig;
    trafficPercentage: number;
    mode: 'shadow' | 'canary';
    startedAt: Date;
  };

  // Restricciones
  maxCostUsd: number;
  timeoutMs: number;
  retryPolicy: { maxAttempts: number; backoffMs: number };

  // Calidad esperada
  qualityThreshold: number; // % de validaciones que deben pasar
}

interface ModelConfig {
  provider: 'anthropic' | 'google' | 'openai' | 'self-hosted';
  model: string;
  promptVersion: string; // referencia a archivo en prompts/
  temperature?: number;
  maxTokens?: number;
}
```

### 4.3 Ejemplo de registry

```typescript
export const taskRegistry: Record<string, TaskDefinition> = {
  generate_initial_roadmap: {
    id: 'generate_initial_roadmap',
    description: 'Genera el roadmap inicial del usuario en onboarding',
    category: 'realtime',
    inputSchema: RoadmapInputSchema,
    outputSchema: RoadmapOutputSchema,
    primary: {
      provider: 'anthropic',
      model: 'claude-haiku-4-5',
      promptVersion: 'v2_claude',
      temperature: 0.3,
      maxTokens: 4000
    },
    fallbackChain: [
      { provider: 'google', model: 'gemini-2.0-flash', promptVersion: 'v1_gemini' },
      { provider: 'openai', model: 'gpt-4o-mini', promptVersion: 'v1_gpt4o_mini' }
    ],
    maxCostUsd: 0.05,
    timeoutMs: 15000,
    retryPolicy: { maxAttempts: 2, backoffMs: 1000 },
    qualityThreshold: 0.95
  },

  nightly_roadmap_update: {
    id: 'nightly_roadmap_update',
    description: 'Actualiza el roadmap del usuario en el job nocturno',
    category: 'batch',
    // ... usa Anthropic Batch API para 50% descuento
    primary: {
      provider: 'anthropic',
      model: 'claude-haiku-4-5-batch',
      promptVersion: 'v1_claude',
    },
    // ...
  },

  generate_morning_message: {
    id: 'generate_morning_message',
    description: 'Genera el mensaje matutino humanizado del usuario',
    category: 'background',
    // ... más simple, modelos más baratos
  },

  evaluate_pronunciation: {
    id: 'evaluate_pronunciation',
    description: 'Evalúa pronunciación de un audio del usuario',
    category: 'realtime',
    // ... primario es modelo propio cuando exista, fallback Azure
    primary: {
      provider: 'self-hosted',
      model: 'pronunciation-v3',
      promptVersion: 'n/a'
    },
    fallbackChain: [
      { provider: 'azure', model: 'pronunciation-assessment', promptVersion: 'n/a' }
    ],
    // ...
  }
};
```

### 4.4 Beneficios del registry

- **Inventario completo** de dependencia de IA visible en un solo archivo.
- **Cambios centralizados**: cambiar de modelo es modificar una línea.
- **Auditabilidad**: histórico de qué modelo se usó para qué tarea cuando.
- **Documentación viva**: el código mismo documenta el sistema.

---

## 5. Versionado de prompts

### 5.1 Estructura de archivos

```
prompts/
├── generate_initial_roadmap/
│   ├── v1_claude.txt         (deprecado, mantenido para auditoría)
│   ├── v2_claude.txt         (actual para Claude)
│   ├── v1_gemini.txt         (actual para Gemini, formato distinto)
│   ├── v1_gpt4o_mini.txt     (actual para GPT-4o-mini)
│   └── README.md             (notas sobre la evolución de prompts)
├── nightly_roadmap_update/
│   └── ...
└── ...
```

### 5.2 Por qué versionar prompts por modelo

Diferentes modelos tienen comportamientos sutilmente diferentes:

- Claude tiende a respetar mejor estructura de XML tags.
- Gemini funciona mejor con instrucciones en formato numerado claro.
- GPT-4o-mini necesita reforzar más las reglas de formato JSON.

Optimizar el prompt para cada modelo extrae el mejor rendimiento posible
de cada uno y permite comparaciones más justas.

### 5.3 Workflow de evolución de prompts

1. Identificar problema (ejemplo: 3% de outputs fallan validación).
2. Crear nueva versión del prompt en archivo versionado nuevo.
3. Correr golden dataset contra ambas versiones.
4. Si la nueva versión mejora, actualizar referencia en task registry.
5. Mantener la versión antigua para auditoría y rollback rápido.

---

## 6. Golden Datasets

### 6.1 Concepto

Para cada tarea, mantener un dataset fijo de casos de prueba que sirve como
verdad de referencia para evaluar cualquier modelo o prompt.

### 6.2 Estructura

```
golden_datasets/
├── generate_initial_roadmap/
│   ├── case_001_b1_tech_no_deadline.json
│   ├── case_002_b2_sales_urgent.json
│   ├── case_003_b1_high_anxiety.json
│   ├── case_004_c1_academic.json
│   ├── ...
│   └── case_100_edge_case_minimal_input.json
├── nightly_roadmap_update/
│   └── ...
└── ...
```

### 6.3 Composición de un caso

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
  "expected_output_sample": "..." // opcional, para inspección manual
}
```

### 6.4 Cómo se construyen

- Casos típicos: derivados de usuarios reales (anonimizados) o generados
  manualmente cubriendo perfiles representativos.
- Edge cases: deadlines extremos, perfiles contradictorios, inputs mínimos,
  inputs ruidosos, valores fuera de rango pero válidos.
- Casos de regresión: cualquier caso que históricamente falló en producción
  se agrega al dataset para garantizar que no vuelva a fallar.

Tamaño objetivo: 30-100 casos por tarea. Más es mejor pero hay rendimientos
decrecientes.

### 6.5 Uso del dataset

```bash
# Evaluar un modelo nuevo contra una tarea
npm run eval -- --task generate_initial_roadmap --model gemini-2.5-flash

# Output:
# ✓ 95/100 cases passed validation
# Average cost: $0.0002 per call
# Average latency: 1.2s
# Comparison vs current (claude-haiku-4-5):
#   Cost: -50%
#   Latency: -15%
#   Validation pass rate: -1%
```

Esto convierte la decisión de migrar de subjetiva a basada en datos.

---

## 7. Estrategia de migración (rollout progresivo)

### 7.1 Fases del rollout

**Fase 0: Evaluación offline (días 1-3)**
- Correr golden dataset contra el modelo candidato.
- Si pasa la threshold de calidad, continuar.
- Ajustar prompt si es necesario para optimizar para el nuevo modelo.

**Fase 1: Shadow mode (días 4-10)**
- 100% del tráfico va al modelo actual y se le devuelve al usuario.
- Mismo input se envía en paralelo al modelo candidato.
- Outputs se loggean pero no se le muestran al usuario.
- Comparación automática de outputs (similitud, costo, latencia).

**Fase 2: Canary 5% (días 11-15)**
- 5% del tráfico real va al modelo candidato.
- Métricas: costo, latencia, validation rate, error rate.
- Idealmente, métrica de producto (tasa de completion del onboarding,
  tiempo en la app) para detectar degradación cualitativa.

**Fase 3: Ramp up (días 16-25)**
- Si las métricas se mantienen, subir a 25%.
- Esperar 2-3 días, evaluar.
- Subir a 50%, esperar, evaluar.
- Subir a 100% si todo está bien.

**Fase 4: Modelo viejo como fallback (días 26+)**
- Modelo viejo queda en la cadena de fallback.
- Después de 30 días estables, removerlo del registry.

### 7.2 Criterios de rollback automático

El sistema debe rollbackear automáticamente si:

- Tasa de validación cae más de 3% respecto a baseline.
- Latencia p95 sube más de 50% respecto a baseline.
- Tasa de errores HTTP del proveedor sube más de 2%.
- Costo por llamada sube inesperadamente más de 20%.

### 7.3 Implementación en el registry

```typescript
generate_initial_roadmap: {
  primary: {
    provider: 'anthropic',
    model: 'claude-haiku-4-5',
    promptVersion: 'v2_claude'
  },
  experimental: {
    config: {
      provider: 'anthropic',
      model: 'claude-haiku-5',  // versión nueva
      promptVersion: 'v3_claude'
    },
    trafficPercentage: 5,
    mode: 'canary',
    startedAt: '2026-04-25',
    rollbackCriteria: {
      maxValidationRateDrop: 0.03,
      maxLatencyP95IncreaseRatio: 1.5,
      maxErrorRateIncrease: 0.02
    }
  }
}
```

---

## 8. Manejo de deprecaciones

### 8.1 Detección automática

Job de CI semanal que verifica el estado de cada modelo en uso:

```bash
# Pseudocódigo
for task in taskRegistry:
    for modelConfig in [task.primary, ...task.fallbackChain]:
        status = checkModelDeprecation(modelConfig)
        if status.deprecated:
            createGitHubIssue({
                title: f"Deprecation: {modelConfig.model}",
                body: f"Sunset date: {status.sunsetDate}, recommended replacement: {status.replacement}",
                labels: ['ai-deprecation', 'priority-medium']
            })
```

### 8.2 Proceso de respuesta

| Tiempo desde anuncio | Acción |
|---------------------:|--------|
| Día 0 | Issue automático creado, equipo notificado |
| Días 1-3 | Identificar modelo de reemplazo |
| Días 4-10 | Correr golden dataset contra reemplazo, ajustar prompts |
| Días 11-30 | Rollout progresivo según protocolo |
| Día 30 | Modelo nuevo sirve 100% del tráfico |
| Días 30-90 | Modelo viejo en fallback, monitoreo |
| Día 90+ | Modelo viejo retirado del registry |

Generalmente los proveedores dan 6-12 meses de notice. El proceso debería
tomar menos de 30 días, dejando amplio margen.

### 8.3 Plan de contingencia para deprecaciones aceleradas

Si un proveedor anuncia deprecación con menos de 30 días de notice (caso
raro pero posible):

1. Activar fallback inmediatamente (rutea a proveedor secundario).
2. Acelerar evaluación del reemplazo (golden dataset en 24-48h).
3. Skip shadow mode si la urgencia lo requiere, ir directo a canary.
4. Migración completa en 2 semanas o menos.

Esto solo es posible si la arquitectura del Gateway está implementada y
el secundario está activo. De ahí la importancia de la inversión inicial.

---

## 9. Optimización proactiva

### 9.1 Job mensual de evaluación

Primer lunes de cada mes, ejecutar automáticamente:

```bash
# Para cada tarea, evaluar todos los modelos disponibles
for task in taskRegistry:
    for model in availableModels:  # incluye nuevos lanzamientos del mes
        results = runGoldenDataset(task, model)
        report.append({
            task,
            model,
            cost,
            latency,
            qualityScore,
            comparedToCurrent
        })

# Generar reporte automático
generateReport(report)
notifyTeam(report)
```

### 9.2 Formato del reporte mensual

```
═══════════════════════════════════════════════════════════
   AI MODEL EVALUATION REPORT - 2026-05
═══════════════════════════════════════════════════════════

TASK: generate_initial_roadmap
Current: claude-haiku-4-5 ($0.0004/call, 96.0% pass rate, 1.4s)

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
  - Similar latency

[... mismo análisis para todas las tareas ...]

ESTIMATED MONTHLY SAVINGS IF ALL MIGRATIONS APPROVED: $1,240
```

### 9.3 Decisión humana, ejecución automatizada

El reporte recomienda, pero la decisión final de migrar es humana. Una
vez aprobada, el rollout sigue el proceso estándar de migración progresiva
sin intervención manual adicional.

---

## 10. Manejo de cambios de precio

### 10.1 Diferencia con deprecaciones

Los cambios de precio son más frecuentes y menos disruptivos, pero
requieren respuesta operativa específica.

### 10.2 Detección

Tracking automático de costos por tarea: alerta si el costo por llamada
sube más de X% respecto a baseline histórica de los últimos 30 días.

### 10.3 Respuesta operativa

**Si el cambio de precio es a la baja**: actualizar baseline, posiblemente
revisar la economía del producto si la mejora es significativa.

**Si el cambio es al alza < 20%**: absorber, no requiere acción inmediata
si los márgenes lo permiten.

**Si el cambio es al alza > 20%**:
1. Evaluar si conviene migrar a otro proveedor (job de evaluación express).
2. Si no, ajustar el costo en Sparks de las operaciones afectadas.
3. NO ajustar el precio mensual al usuario.

### 10.4 Sparks como buffer económico

El sistema de Sparks (ver `docs/architecture/sparks-system.md`) está diseñado
explícitamente para absorber cambios de costo sin afectar al usuario.

Si Claude Haiku sube 30%, una operación que antes costaba 1 Spark ahora
puede costar 1.3 Sparks. El usuario sigue viendo la misma cantidad mensual
de Sparks en su plan.

### 10.5 Provisioned throughput (cuando aplique)

Para tareas críticas y de alto volumen, evaluar contratos de capacidad
reservada con proveedores que lo ofrecen (Anthropic tiene esta opción a
partir de cierto volumen). Esto da:

- Precio fijo por contrato.
- Capacidad garantizada (no rate limits).
- Aislamiento de cambios de precio durante el contrato.

Solo justificable a partir de cierto volumen.

---

## 11. Multi-proveedor activo

### 11.1 Configuración mínima por categoría de tarea

| Categoría | Proveedor primario | Secundario | Terciario |
|-----------|-------------------|------------|-----------|
| Realtime crítico | 95% tráfico | 5% tráfico | Solo fallback de emergencia |
| Batch | 100% tráfico | Solo fallback | Solo fallback |
| Background | 100% tráfico | Solo fallback | — |

### 11.2 Por qué mantener 5% activo en el secundario

- **Fallback verificado**: confirma que la integración funciona, no solo
  que está configurada.
- **Datos comparativos**: 5% del tráfico es suficiente para mantener
  métricas comparativas en tiempo real.
- **Negociación**: poder mostrar al primario que existe migración real
  posible mejora condiciones comerciales.
- **Capacidad de scale-up**: si el primario tiene problema, el secundario
  ya tiene cache caliente y conexiones establecidas.

### 11.3 Costo del seguro

Para una tarea que cuesta $100/mes en el primario, mantener el 5% en el
secundario cuesta aproximadamente $5/mes. Es un seguro extremadamente
barato comparado con el costo de un outage o migración de emergencia.

---

## 12. Modelos propios en el Gateway

### 12.1 Tratamiento uniforme

Los modelos propios (entrenados internamente) son "un proveedor más" en
el Gateway. La capa de abstracción no distingue entre Claude API y un
modelo de pronunciación corriendo en Modal o GPU propia.

### 12.2 Configuración típica

```typescript
evaluate_pronunciation: {
  primary: {
    provider: 'self-hosted',
    model: 'pronunciation-v3',
    endpoint: 'https://ml.internal/pronunciation/v3',
    promptVersion: 'n/a'
  },
  fallbackChain: [
    {
      provider: 'azure',
      model: 'pronunciation-assessment',
      promptVersion: 'n/a'
    }
  ]
}
```

### 12.3 Beneficios del tratamiento uniforme

- Si el modelo propio falla (bug, infra caída), fallback automático a API.
- Si la API externa tiene problemas, modelo propio toma el tráfico.
- Migración gradual de tráfico de API a modelo propio sigue mismo protocolo
  que cualquier otra migración.
- A/B testing entre modelo propio y API es trivial.

---

## 13. Observabilidad

### 13.1 Métricas mínimas por llamada

Cada invocación del Gateway registra:

```typescript
{
  taskId: string;
  modelUsed: string;
  promptVersion: string;
  startedAt: timestamp;
  completedAt: timestamp;
  latencyMs: number;
  inputTokens: number;
  outputTokens: number;
  costUsd: number;
  validationPassed: boolean;
  validationErrors?: string[];
  retryAttempts: number;
  fallbackUsed: boolean;
  userId?: string;  // hasheado para privacy
  experimentVariant?: string;  // si es parte de A/B test
}
```

### 13.2 Dashboard interno

Página interna (admin panel) que muestra:

- **Por tarea**: modelo actual, costo promedio, validation rate, latencia
  p50/p95/p99 en últimas 24h, 7d, 30d.
- **Experimentos activos**: tareas en shadow o canary, métricas comparativas
  en tiempo real.
- **Alertas activas**: deprecaciones anunciadas, picos de costo, fallos
  recientes.
- **Tendencias**: costo total mensual de IA, distribución por tarea, evolución
  semana a semana.

### 13.3 Alertas críticas

Configuradas para notificar inmediatamente (Slack, email, según preferencia):

- Costo del día > 2x promedio histórico.
- Validation rate de cualquier tarea < 90%.
- Latencia p95 > 2x baseline.
- Cualquier usuario individual con costo > $1 en 24h.
- Proveedor primario con error rate > 5%.
- Anuncio de deprecación detectado en endpoint del proveedor.

---

## 14. Herramientas recomendadas

### 14.1 Build vs buy

Construir todo el AI Gateway desde cero es trabajo significativo. Existen
herramientas open source y servicios managed que cubren partes de la
estrategia.

### 14.2 Stack recomendado para empezar solo

| Componente | Herramienta | Justificación |
|------------|-------------|---------------|
| Adapter multi-proveedor | LiteLLM (open source) | Interfaz unificada para múltiples LLMs, gratis |
| Observability + logs | Langfuse o Helicone | Logging y dashboards específicos para LLMs, tier gratuito |
| Validation | Zod (TypeScript) | Type-safe, integración natural |
| Golden datasets | Promptfoo o Braintrust | Frameworks de evaluación, tier gratuito |
| A/B testing | Implementación propia sobre LiteLLM | Lógica simple en el Gateway |

Esta combinación da el 70% de la estrategia con esfuerzo mínimo. El 30%
restante (lógica específica del Gateway, integración con el sistema de
Sparks, dashboards custom) se construye internamente.

### 14.3 Cuándo construir más cosas internamente

A partir de cierta escala (típicamente 50.000+ usuarios activos o equipo
con >5 personas), puede tener sentido construir piezas más sofisticadas:

- Gateway propio en Go o Rust para máximo rendimiento.
- Sistema de evaluación más rico que Promptfoo.
- Provisioned throughput negociado directamente con proveedores.

Antes de eso, las herramientas managed son la decisión correcta.

---

## 15. Plan de implementación incremental

### 15.1 Fase 1: Foundation (mes 1)

- Implementar AI Gateway básico con LiteLLM como adapter.
- Definir Task Registry inicial con las primeras 3-5 tareas.
- Logging básico de cada llamada (costo, latencia, resultado).
- Validation con Zod para cada tarea.
- Un solo proveedor por tarea (sin fallbacks aún).

### 15.2 Fase 2: Resilience (meses 2-3)

- Agregar fallback chain a cada tarea.
- Implementar retry con backoff exponencial.
- Setup de Langfuse o Helicone para observabilidad.
- Primer dashboard interno con métricas básicas.

### 15.3 Fase 3: Quality (meses 3-6)

- Construir golden datasets para tareas críticas (mínimo 30 casos cada una).
- Setup de evaluación automática con Promptfoo.
- Implementar shadow mode en el Gateway.
- Versionado de prompts por modelo.

### 15.4 Fase 4: Optimization (meses 6-9)

- Implementar canary mode con rampa progresiva.
- Job mensual de evaluación de modelos alternativos.
- Sistema de alertas críticas configurado.
- Procesos de respuesta a deprecaciones documentados.

### 15.5 Fase 5: Self-hosted integration (meses 9-12)

- Integrar primeros modelos propios al Gateway.
- Migración progresiva de tráfico de API a modelo propio.
- Métricas comparativas en producción.

---

## 16. Métricas de éxito de la estrategia

Indicadores que demuestran que la estrategia funciona:

- **Tiempo de migración a un modelo nuevo**: menos de 30 días desde decisión
  hasta 100% rollout.
- **Cero outages causados por cambios de proveedor**.
- **Captura de ahorros**: cada trimestre, identificar y capturar al menos
  una optimización de costo significativa.
- **Costo de IA como % del revenue**: tendencia a la baja trimestre a
  trimestre, no por menor uso sino por mejor gestión.
- **Tiempo entre anuncio de deprecación y migración completa**: < 60 días
  consistentemente.

---

## 17. Decisiones abiertas

- [ ] ¿Construir Gateway propio o usar Portkey/OpenRouter como Gateway managed?
- [ ] ¿Promptfoo o Braintrust para evaluación? (probar ambos)
- [ ] ¿Cómo versionar prompts? ¿Archivos en repo o servicio externo (PromptLayer)?
- [ ] Política de retención de logs de llamadas a LLMs (privacy + costo storage).
- [ ] ¿Qué tareas justifican mantener 5% activo en secundario vs solo fallback?

---

## 18. Referencias internas

- `docs/business/plan_de_negocio.docx` — Plan de negocio general.
- `docs/product/ai-roadmap-system.md` — Sistema de roadmap (consumidor del Gateway).
- `docs/architecture/sparks-system.md` — Sistema de tokens (a crear).
- `docs/decisions/ADR-001-ai-gateway.md` — Decisión arquitectónica (a crear).

---

*Documento vivo. Actualizar cuando cambien proveedores, herramientas o cuando
se aprenda de la implementación.*

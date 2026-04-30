# ADR-001: Capa de abstracción AI Gateway propia

**Status:** Accepted
**Date:** 2026-04
**Author:** —
**Audiencia:** agente AI implementador.

---

## Contexto

El producto depende fuertemente de LLMs para curar roadmaps, scoring de
gramática, generación de mensajes humanizados, AI Assistant de soporte
y análisis nocturnos. Los proveedores de LLMs (Anthropic, OpenAI, Google,
Azure) cambian con frecuencia mucho mayor que cualquier otra dependencia:

- Deprecan modelos con 6–12 meses de notice (a veces menos).
- Ajustan precios varias veces al año.
- Lanzan modelos nuevos cada 2–3 meses (frecuentemente más baratos y
  mejores).
- Cambian APIs, rate limits, políticas de uso.

Sin una estrategia explícita, cada uno de esos eventos genera trabajo de
emergencia, riesgo de outage y pérdida de oportunidades de optimización.

El producto también necesita:
- A/B testing entre modelos.
- Validación rigurosa de outputs (los LLMs producen JSON inválido a
  veces).
- Cost tracking por usuario y por tarea.
- Fallback chain para resiliencia.
- Observabilidad uniforme (latencia, costo, validation rate).

## Decisión

Construir una **capa de abstracción única (AI Gateway)** que es el
**único path** desde código de negocio hacia cualquier LLM o servicio
de IA.

### Principios de la decisión

1. **Regla de oro:** ningún archivo de código de negocio menciona
   "anthropic", "openai", "gemini", "azure" o cualquier nombre de
   proveedor. Si lo menciona, es un bug de diseño.
2. **Task Registry:** toda invocación de IA corresponde a una "tarea"
   registrada explícitamente con `taskId`, schemas Zod de input/output,
   primary + fallback chain, prompt versionado.
3. **Multi-proveedor activo:** cada tarea crítica tiene primario +
   secundario activos (5% del tráfico real va al secundario).
4. **Validación de outputs:** todo JSON de LLM se valida con Zod antes
   de devolverlo.
5. **Observability:** cada llamada loggea costo, latencia, validation
   result, model used.

### Build vs Buy

Para MVP usar herramientas managed para 70% del Gateway:
- **LiteLLM** (open source) como adapter multi-proveedor.
- **Langfuse** o **Helicone** para logging y dashboards.
- **Promptfoo** para evaluación con golden datasets.

El 30% restante (lógica específica del Gateway, integración con Sparks,
A/B testing de variantes en Task Registry, dashboards custom) se
construye internamente.

## Alternativas consideradas

### A. Llamadas directas al SDK del proveedor desde código de negocio

**Rechazada.** Acopla el código a un proveedor específico. Cada
deprecación o cambio de precio requiere refactor. Imposible hacer
A/B testing entre modelos. Sin observabilidad uniforme. Es la opción
"obvia" pero fatal a mediano plazo.

### B. Gateway managed (Portkey, OpenRouter)

**Considerada, parcialmente adoptada.** Servicios como Portkey ofrecen
gran parte del Gateway listo. Pros: menos código que mantener. Contras:
otra dependencia externa con su propio SLA, vendor lock-in, costos por
volumen, menor control.

**Decisión:** usar LiteLLM (open source, self-hosted) en lugar de
Portkey/OpenRouter. Conserva control y costos.

### C. Construir todo from scratch

**Rechazada para MVP.** Construir desde cero el adapter multi-proveedor,
logging, dashboards, evaluación es trabajo significativo (3–6 meses) sin
diferenciación real. LiteLLM cubre el 70% gratis.

A escala (>50.000 usuarios o equipo >5 personas), reconsiderar
construcción propia para máximo control y rendimiento. Por ahora, no.

### D. Multi-proveedor pero sin abstracción (if/else en cada call site)

**Rechazada.** Equivalente a A en problemas; solo agrega lógica
condicional duplicada en cada lugar que llame a un LLM.

## Consecuencias

### Positivas

- Migración a un modelo nuevo: < 30 días desde decisión hasta 100%
  rollout, sin tocar código de negocio.
- Cero outages causados por cambios de proveedor (fallback automático).
- A/B testing entre modelos es trivial (config en Task Registry).
- Cost tracking granular por usuario, tarea, modelo.
- Negociación con proveedores: poder mostrar alternativa real activa.

### Negativas

- Capa adicional de indirección: debugging de problemas de IA puede ser
  más complejo (requiere mirar logs del Gateway antes de mirar el
  proveedor).
- Curva de aprendizaje del Task Registry: cada tarea nueva requiere
  registro explícito (intencional, evita acoplamiento).
- LiteLLM es dependencia open source: si se discontinúa, hay que
  reemplazar (evaluación: comunidad activa, riesgo bajo).

### Riesgos a monitorear

- LiteLLM lag detrás de features nuevas de proveedores: monitorear
  releases, contribuir si necesitamos features no soportadas.
- Validación con Zod añade latencia: medir overhead, optimizar si
  significativo.
- Golden datasets requieren mantenimiento: presupuestar tiempo
  trimestral para regenerar casos de regresión.

## Referencias

- [`docs/architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md)
  — diseño detallado del Gateway.
- [`docs/architecture/sparks-system.md`](../architecture/sparks-system.md)
  — Sparks system es el cliente principal del Gateway (cobra antes de
  cada llamada).
- LiteLLM: https://github.com/BerriAI/litellm.
- Langfuse: https://langfuse.com.
- Promptfoo: https://promptfoo.dev.

# EPIC-05: AI Gateway foundation

**Status:** Draft
**Sprint target:** Sprint 0-1 (paralelo a EPIC-01 stories late slices)
**Owner:** —
**Estimación total:** ~52 puntos (17 stories aprox.)

---

## Resumen

Establecer el **AI Gateway** como la capa de abstracción única
sobre los proveedores de LLM y servicios AI (Anthropic, OpenAI,
Google, Azure, ElevenLabs, DALL-E 3, HeyGen). Implementa:

1. Task Registry: cada tarea AI registrada con su config
   (proveedor preferido, fallback chain, budget).
2. Provider adapters: interfaces unificadas sobre 6 proveedores.
3. Multi-provider fallback: si primario falla, automatically usa
   secondary.
4. Cost tracking: por task, por user, por día.
5. Catálogo de **tasks MVP** registradas: las 10+ que el resto del
   producto necesita para funcionar.

Este epic es **bloqueante para varias stories de EPIC-01** (US-017
a US-021) y todo EPIC-02 (Content pipeline) y EPIC-03 (Pedagogical
engine).

---

## Por qué este epic

**Regla de oro:** ningún código de negocio invoca SDKs de LLM
directamente. Todo pasa por el AI Gateway.

Sin esta abstracción:
- Cada feature acopla a un proveedor específico → migración
  costosa cuando precios o calidad cambien.
- No hay control de costos ni budget enforcement.
- No hay observabilidad consistente.
- Imposible A/B testing entre proveedores.
- Catálogo de tasks no auditeable.

Con esta abstracción:
- Cambiar proveedor = update de Task Registry, no refactor de
  código.
- Budget por task / user enforced consistentemente.
- Métricas comparables (latency, cost, quality) cross-task.
- Fallback automático cuando un proveedor está degraded.

---

## Specs autoritativos referenciados

| Spec | Cubre |
|------|-------|
| [`architecture/ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md) | Arquitectura completa, Task Registry, Provider Adapters, fallbacks |
| [`architecture/ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md) §4.2 | Catálogo completo de tasks MVP |
| [`decisions/ADR-001-ai-gateway.md`](../../decisions/ADR-001-ai-gateway.md) | Decisión arquitectónica |
| [`product/atomics-catalog-seed.md`](../../product/atomics-catalog-seed.md) §9 | Pipelines de generación (consumidos por tasks) |
| [`reglas.md`](../../reglas.md) | Regla "AI Gateway es única vía a LLMs" |

---

## Stories del epic

### Slice 1: Foundation (~17 pts, 5 stories)

| US | Título | Pts |
|----|--------|----:|
| US-027 | AI Gateway Worker skeleton + routing | 3 |
| US-028 | Task Registry schema + persistence | 3 |
| US-029 | Provider adapter interface + Anthropic adapter | 5 |
| US-030 | Multi-provider fallback chain | 3 |
| US-031 | Cost tracking + budget enforcement | 3 |

### Slice 2: Tasks críticas para EPIC-01 (~15 pts, 5 stories)

| US | Título | Pts |
|----|--------|----:|
| US-032 | Task `generate_initial_roadmap` | 5 |
| US-033 | Task `score_pronunciation` (Azure) | 3 |
| US-034 | Task `transcribe_user_audio` (Whisper) | 2 |
| US-035 | Task `score_fluency` | 3 |
| US-036 | Task `generate_notification_content` | 2 |

### Slice 3: Tasks de generación de contenido (~11 pts, 4 stories)

| US | Título | Pts |
|----|--------|----:|
| US-037 | Task `generate_audio_tts` (ElevenLabs) | 3 |
| US-038 | Task `generate_image` (DALL-E 3) | 3 |
| US-039 | Task `validate_image_no_text` (OCR) | 2 |
| US-040 | Task `detect_grammar_errors` | 3 |

### Slice 4: Operacional (~8 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-041 | Logging + observability dashboard | 3 |
| US-042 | Prompt versioning + audit trail | 3 |
| US-043 | Admin endpoint para inspeccionar tasks | 2 |

---

## Total estimado

~51 puntos en 17 stories. Asumiendo velocity de 20-25 pts/sprint:
**~3 sprints (semanas 1-6)**, paralelo a EPIC-01.

---

## Dependencies externas

- Cuentas y API keys configuradas (Sprint 0 setup):
  - Anthropic API.
  - OpenAI API.
  - Google AI / Vertex AI.
  - Azure Speech Services (Pronunciation Assessment).
  - ElevenLabs.
- Cloudflare Workers + KV namespace para Task Registry.
- Postgres tablas para cost tracking + audit trail.

---

## Dependencies inter-EPIC

- **EPIC-01 (Auth + Onboarding):**
  - US-017 (Mini-test 1) requiere US-033 (`score_pronunciation`).
  - US-018 (Mini-test 2) requiere US-034 (`transcribe`) +
    US-035 (`score_fluency`).
  - US-019 (Mini-test 3) requiere US-034.
  - US-021 (Initial roadmap) requiere US-032
    (`generate_initial_roadmap`).
- **EPIC-02 (Content pipeline):**
  - Generación de atomics requiere US-037, US-038, US-039.
- **EPIC-03 (Pedagogical engine):**
  - Scoring de ejercicios requiere US-033, US-034, US-035, US-040.

**Bottom line:** Slice 1 + Slice 2 son **bloqueantes**. Slice 3-4
pueden ir en paralelo.

---

## Definition of Done del epic

- [ ] AI Gateway Worker corriendo en dev environment.
- [ ] Task Registry persistido con seed inicial (10+ tasks).
- [ ] Provider adapters funcionales para los 6 proveedores MVP.
- [ ] Multi-provider fallback testeado en al menos 3 tasks.
- [ ] Cost tracking visible en dashboard interno.
- [ ] Las 10+ tasks MVP del catálogo registradas + invocables.
- [ ] Documentation de cómo agregar nueva task.
- [ ] Performance: P50 < proveedor + 200ms overhead.
- [ ] Tests integration que invocan cada task con request real.

---

## Riesgos identificados

| Riesgo | Mitigación |
|--------|------------|
| API keys leak | Secrets en Cloudflare secrets, nunca en repo |
| Provider downtime mid-fallback | Test fallback chain con chaos testing |
| Cost runaway | Budget enforcement hard-stops, alerts en >80% del budget |
| Prompt drift entre versiones | Audit trail con prompt + version cada call |
| Task registry inconsistency | Migrations versionadas, rollback path documented |

---

## A/B tests post-MVP planeados

(Coordinar con `ai-gateway-strategy.md` §10.)

1. Provider primary swap (ej: Anthropic Haiku vs OpenAI gpt-4o-mini
   para `generate_initial_roadmap`).
2. Prompt template variations (ej: 1-shot vs few-shot).
3. Temperature tuning per task.
4. Caching strategies (semantic vs deterministic).

---

*Documento vivo. Actualizar cuando se agreguen tasks nuevas o
cambien proveedores.*

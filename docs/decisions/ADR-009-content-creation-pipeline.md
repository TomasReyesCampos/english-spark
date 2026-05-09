# ADR-009: Pipeline atomic+composite v1.2 con generación AI

**Status:** Accepted
**Date:** 2026-05
**Author:** —
**Audiencia:** agente AI implementador.

---

## Contexto

El producto requiere un volumen significativo de contenido pedagógico
para el MVP:
- **175 bloques** pedagógicos (Job Ready 74 + Travel Confident 39 +
  Daily Conversation 62).
- **~700 composites** (ejercicios) — 4 assets promedio por bloque.
- **~350 atomics** (piezas individuales de media) — con reuse 2x.
- **6 characters recurrentes** con identidad visual + auditiva
  consistente.

Costo si cada composite se genera "from scratch" sin reuse:
- ~$1.50 por composite generation = **~$1.050** total.
- Tiempo generation + revisión: ~30 min/composite = **~350 horas**.

Costo si maximizamos reuse vía modelo atomic+composite:
- ~$24 generation atomics + ~$2.750 human time = **~$2.774** total.
- Aún más caro? **No** — porque el reuse 2x aplica entrega → cada
  atomic se rinde 2x antes de generar uno nuevo.

Decisión inicial estaba documentada en `content-creation-system.md`
v1.1 con un enum monolítico `AssetType` y un solo modelo
`learning_assets`. En 2026-04 se cuestionó:
- El enum `AssetType` no escala (cada nuevo tipo requiere migración).
- No hay separación entre **media reusable** y **composiciones
  pedagógicas**.
- Sin reuse explícito, costos de generación AI escalan linealmente
  con cobertura.

Surgió la pregunta:

> **¿Cómo modelamos contenido para maximizar reuse y minimizar costo
> de generación AI mientras mantenemos calidad?**

## Decisión

**Adoptar modelo atomic+composite v1.2 con sistema de labels en lugar
de enum AssetType monolítico, characters recurrentes con identidad
consistente, y pipeline AI-generation con validación humana
obligatoria.**

### Componentes principales

#### 1. Modelo de 2 capas

| Capa | Tabla Postgres | Qué contiene | Reuse |
|------|----------------|--------------|------:|
| **Atomic** | `media_atomics` | Pieza individual de media (audio, imagen, video) | Alta (target ≥ 1.5x MVP, ≥ 2.5x Fase 2) |
| **Composite** | `learning_assets` | Ejercicio pedagógico que compone atomics | Baja (1-2x típicamente) |

Un atomic puede aparecer en N composites; cada composite tiene 1-N
atomics.

#### 2. Sistema de labels (en lugar de `AssetType` enum)

Los composites se clasifican con dimensiones independientes:

```typescript
interface CompositeAssetLabels {
  primary_media_format: 'text' | 'audio' | 'image' | 'video' | 'multi';
  primary_media_subtype?: string;       // 'talking_head', 'tts', etc.
  interaction_type:                     // pedagógica
    | 'mc' | 'qa' | 'drill' | 'shadow' | 'free_response'
    | 'roleplay_structured' | 'roleplay_free'
    | 'description' | 'fill_blank' | 'translation'
    | 'ordering' | 'dictation';
  pedagogical_focus: string[];          // sub-skill IDs
  cefr_level: string;
  target_variant: 'us' | 'uk' | 'neutral';
  topic: string[];
  context_tags: string[];
  cultural_relevance: string[];
}
```

Validación combinatoria via matrix
`media_format × interaction_type` (12 columnas × 5 filas) — ver
`content-creation-system.md` §3.3.

#### 3. Characters recurrentes con identidad consistente

`characters` table (de `content-creation-system.md` §8.2) define
entidades cross-asset con:
- **Voz fija:** `voice_id` del pool de 12 voces (ver
  `atomics-catalog-seed.md` §3).
- **Apariencia visual fija:** mismo prompt base + `generation_seed`
  determinístico para imágenes.
- **Background pedagógico:** track + context donde aparecen.
- **Personalidad explícita:** key_phrases_signature, avoid list,
  description voice/visual (ver `atomics-catalog-seed.md` §4).

6 characters MVP: `char_alex_friendly_friend`,
`char_grandma_patient`, `char_sarah_witty_coworker`,
`char_mike_chatty_neighbor`, `char_jamie_deep_friend`,
`char_emma_native_fast`.

#### 4. Pipeline AI-generation

Por categoría de atomic:

| Categoría | Proveedor primario | Cost/atomic | Validation |
|-----------|--------------------|-------------|------------|
| Audio TTS | ElevenLabs (`eleven_multilingual_v2`) | $0.0005 | Native review obligatorio |
| Image AI | DALL-E 3 HD vía AI Gateway | $0.04 | Curaduría manual + OCR para `is_text_in_image=false` |
| Video talking-head | HeyGen | $1.00 promedio | Audio sync sample manual |
| Audio ambient | Freesound.org CC0 + edit | $0 license | Manual curation |
| Audio distortion | FFmpeg overlays | $0 | One-time setup |

Detalles de prompts y settings en `atomics-catalog-seed.md` §9.

#### 5. Validación humana obligatoria pre-aprobación

Cada atomic NUEVO pasa por:

1. **Generación AI** (proveedor + prompt template).
2. **Validación automática:**
   - Audio: duration check, normalization, no clipping.
   - Image: dimensions, OCR para detectar texto no deseado, aspect
     ratio.
   - Video: bitrate, audio sync.
3. **Revisión humana TESOL/native speaker** marca `approved = true`.
4. **Beta con 50 users** antes de marcar
   `composite.approved_for_production = true`.

Sin este pipeline: atomic queda en `media_atomics` con
`approved = false` y NO se puede usar en composites en producción.

#### 6. Lifecycle de atomics

| Estado | Significado |
|--------|-------------|
| `approved = false` | Generado pero no validado |
| `approved = true, archived = false` | Listo para uso en composites |
| `archived = true` | Ya no se usa (use_count = 0 después de 90d, o reemplazado por _v2) |

Cron mensual flagea atomics con `use_count = 0` después de 90 días
para revisión + archivado eventual.

#### 7. Versionado con `_v<n>`

Atomic IDs incluyen sufijo de versión: `audio_alex_greeting_v1`. Si
un atomic se mejora significativamente, se crea `_v2` y composites
migran gradualmente. Versiones viejas no se borran (lineage
preservado).

### Schema implications

Schemas autoritativos en `content-creation-system.md` §8:
- §8.1: `media_atomics` (NUEVA en v1.2).
- §8.2: `characters` (NUEVA en v1.2).
- §8.3: `atomic_generation_queue` (NUEVA en v1.2).
- §8.4: `learning_assets` modificada para referenciar atomics.

### Reuse stats target

| Métrica | MVP | Fase 2 |
|---------|----:|-------:|
| Reuse rate (avg uses por atomic) | ≥ 1.5 | ≥ 2.5 |
| % de atomics con `use_count ≥ 3` | 30% | 50% |
| Generación cost per composite | ≤ $1.50 | ≤ $1.00 |

Cuando estos targets no se cumplen: trigger review de naming /
labeling (atomics demasiado específicos no reusan) o de catálogo
(faltan atomics generales).

### Orden de creación recomendado

Daily Conversation **primero**, después Job Ready y Travel
Confident. Razón: DC genera atomics universales (greetings, fillers,
casual phrases) que aparecen en los otros 2 tracks. Crear DC primero
acelera el output marginal de los siguientes tracks.

(Detalle en `track-daily-conversation-blocks.md` §10.)

## Alternativas consideradas

### A. Mantener enum `AssetType` monolítico (status quo v1.1)

**Rechazada.**

- Cada nuevo tipo de ejercicio requiere migración + deploy.
- No escala a las combinaciones reales (audio + listening_mc;
  video + fill_blank; etc.).
- Búsqueda flexible imposible ("todos los videos talking_head US B2"
  requiere filtros complejos).

### B. Modelo de una sola capa (sin atomic + composite)

**Rechazada.**

- Costo de generación lineal con cobertura.
- Inconsistencia visual de characters (cada video genera avatar
  fresh sin seed determinístico).
- Sin reuse, escalar a 700 composites cuesta ~$1.050 vs ~$24 con
  reuse.

### C. Composites generados completamente AI sin validación humana

**Rechazada.**

- Calidad inconsistente.
- Bias detectado en outputs DALL-E (ej: ethnic stereotypes en images
  de "office workers").
- Errors gramaticales/pronunciación TTS pasan a producción sin
  filtro.
- ROI: $50/hora de TESOL review prevé $1.000+ en refunds y churn por
  contenido malo.

### D. Generación 100% manual sin AI

**Rechazada.**

- ~700 composites × 2-4 horas/composite = 1.400-2.800 horas humanas.
- Costo prohibitivo (~$70k-$140k a $50/hora).
- Imposible de escalar para Fase 2-3 (1.000+ composites).

### E. Outsource a third-party content provider

**Considerada, parcialmente.**

- Algunos atomics como ambient audio se outsource (Freesound CC0).
- Pero contenido pedagógico custom (roleplays con characters
  específicos, pronunciation drills target sub-skills) NO puede
  outsource sin perder coherencia con el sistema.

### F. Composite generation completamente automática on-demand

**Rechazada para MVP, reconsiderar Fase 3.**

- Requiere infrastructure compleja (queue, retry, validation
  pipeline streamed).
- Latencia user-facing: ¿user espera 30-60s mientras se genera su
  ejercicio?
- MVP funciona mejor con catálogo pre-generado + curado.
- Fase 3+: explorar para users con perfiles únicos donde catálogo
  estándar no aplique.

## Consecuencias

### Positivas

- **Costo de creación reducido ~50%** vs sin reuse.
- **Coherencia visual de characters** cross-blocks (memoria
  episódica del user).
- **Catálogo extensible:** agregar nuevos tipos de ejercicio NO
  requiere migración (solo nueva combinación de labels).
- **Búsqueda flexible:** queries por dimensión específica
  ("todos los TTS de char_alex en track DC B1+").
- **Validación humana obligatoria** previene contenido malo en
  producción.
- **Versionado preserva lineage** sin breaking composites
  existentes.

### Negativas

- **Complejidad de modelo:** developers deben entender atomic vs
  composite. Aprendizaje curva.
- **2 tablas en lugar de 1:** queries que necesitan ambas
  involucran join.
- **Pipeline de validación humana es bottleneck:** content team
  size limita throughput (~30 atomics/persona/día).
- **Reuse rate requiere disciplina:** atomics demasiado específicos
  no reusan; naming + labeling debe ser cuidadoso.

### Riesgos a monitorear

- **Reuse rate < 1.5 después de 3 meses:** señal de naming malo o
  catálogo demasiado específico. Mitigación: review trimestral del
  catálogo, identificar atomics candidatos a "generalización".
- **Generation cost > target ($1.50/composite MVP):** señal de poco
  reuse o demasiados atomics nuevos por composite. Mitigación:
  prioritize buscar atomic existente antes de generar nuevo.
- **Validation bottleneck:** si content team está saturado, MVP se
  retrasa. Mitigación: contratar 2 TESOL reviewers part-time desde
  Sprint 1.
- **Drift en personalidad de characters:** cuando varios contributors
  generan atomics para el mismo character, personality puede divergir.
  Mitigación: char specs en YAML como single source of truth, review
  obligatoria de char_id antes de approve.

### A/B tests prioritarios (post-MVP)

1. **Reuse rate 1.5x vs 3x:** ¿más reuse correlaciona con menos
   user satisfaction (contenido se siente repetitivo)? Encontrar
   sweet spot.
2. **AI-generated character avatars vs licensed stock:** calidad
   user-percibida.
3. **Validation humana 100% vs spot-check 30%:** ¿podemos relajar
   validación cuando track esté maduro? ROI vs quality.
4. **Composite generation pre-cached vs on-demand para users
   premium:** Premium puede tener "personalized exercises" generados
   on-demand.

## Métricas de éxito

| Métrica | Target MVP | Cómo medir |
|---------|-----------|------------|
| Reuse rate avg | ≥ 1.5 | Cron nightly suma `use_count` / total atomics |
| Generation cost per composite | ≤ $1.50 | Sum `generation_cost_usd` from queue |
| % atomics with `approved = true` post-validation | ≥ 95% | `approved / total` weekly |
| Validation queue latency | < 48h promedio | `reviewed_at - created_at` |
| User-reported content quality issues | < 5 / 1000 users / mes | Tag in customer-support-system |
| Cross-track reuse (atomic en >1 track) | ≥ 30% de atomics universales | Aggregate query monthly |

## Plan de implementación

### Sprint 1: Foundation

- [ ] Migración SQL: `media_atomics` + `characters` +
  `atomic_generation_queue` schemas.
- [ ] Modificar `learning_assets` para referenciar atomics.
- [ ] Voice pool de 12 voces configurado en ElevenLabs.
- [ ] AI Gateway tasks: `generate_audio_tts`, `generate_image`,
  `validate_image_no_text`.

### Sprint 2-3: Characters seed (alex, sarah, grandma)

- [ ] Generar atomics audio + image + video para los primeros 3
  characters (~88 atomics).
- [ ] Pipeline de validation humana operativo.
- [ ] Native review process documented.

### Sprint 4: Characters restantes (mike, jamie, emma)

- [ ] Generar atomics para los 3 characters restantes (~90 atomics).

### Sprint 5: Ambient + scenes

- [ ] 40 ambient audio atomics curados (Freesound CC0).
- [ ] 70 scene image atomics generados (DALL-E 3).

### Sprint 6: Generic phrases + cleanup

- [ ] 60 generic phrase atomics (ElevenLabs).
- [ ] Distortion overlays (FFmpeg).
- [ ] Auditoría naming + use_count tracker funcional.

### Sprint 7+: Composites por track

- [ ] Daily Conversation primero (atomics universales).
- [ ] Job Ready segundo.
- [ ] Travel Confident tercero.

## Referencias

- [`docs/product/content-creation-system.md`](../product/content-creation-system.md)
  §3 — modelo atomic + composite + characters detallado.
- [`docs/product/content-creation-system.md`](../product/content-creation-system.md)
  §8 — schemas Postgres autoritativos.
- [`docs/product/atomics-catalog-seed.md`](../product/atomics-catalog-seed.md)
  — seed inicial de 350 atomics + 6 characters + voice pool.
- [`docs/product/track-daily-conversation-blocks.md`](../product/track-daily-conversation-blocks.md)
  — 62 bloques DC con asset_sequence (estrategia "DC primero").
- [`docs/product/track-job-ready-blocks.md`](../product/track-job-ready-blocks.md)
  — 74 bloques JR con asset_sequence.
- [`docs/product/track-travel-confident-blocks.md`](../product/track-travel-confident-blocks.md)
  — 39 bloques TC con asset_sequence.
- [`docs/architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md)
  — tasks `generate_image`, `generate_audio_tts`, `generate_video`.
- [`docs/explorations/multimedia-assets.md`](../explorations/multimedia-assets.md)
  — exploración original que llevó a v1.2.
- [`docs/decisions/ADR-001-ai-gateway.md`](ADR-001-ai-gateway.md)
  — Gateway donde se enrutan las tasks de generación.

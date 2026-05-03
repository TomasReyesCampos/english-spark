# Exploración: Estrategia de assets multimedia (audio, imagen, video)

> **Status:** **Parcialmente promovido a spec en v1.2 (2026-05).**
> Modelo atomic+composite, sistema de labels, schemas
> `media_atomics`/`characters`/`atomic_generation_queue`, y tasks
> `generate_audio_tts`/`generate_image`/`validate_image_no_text` ya
> están en specs:
> - [`../product/content-creation-system.md`](../product/content-creation-system.md)
>   §3, §8.1-§8.4, §9.2-§9.8
> - [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md)
>   §4.2.9
> - [`../architecture/01-overview.md`](../architecture/01-overview.md) §4.9
> - [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.12
>
> **Sigue en exploración:** specifics de generación de video (HeyGen
> vs Hedra, Runway provider config), pipelines completos de Lottie y
> screen recordings, voice library final del producto.
>
> **Owner:** —
> **Target de implementación:** atomic model en MVP. Generación masiva
> de video se difiere a Fase 2 (mes 4–6).
> **Audiencia:** agente AI implementador + humano revisor.

---

## 1. Resumen ejecutivo

La doc actual de `content-creation-system.md` cubre texto y audio
sistemáticamente, pero **video** está ausente y **imágenes** apenas se
mencionan. Esta exploración propone:

1. **Sistema de labels** que descompone `AssetType` en dimensiones
   independientes (media_format × interaction_type).
2. **Modelo atomic + composite:** separar pieza de media (atomic) del
   ejercicio pedagógico (composite). Permite reuso de un mismo video
   en N ejercicios distintos.
3. **Strategy completa de audio** (100% TTS con voz "host" cloneada +
   pool de voces).
4. **Strategy de imagen** (100% AI generation, DALL-E/Flux).
5. **Strategy de video** (TTS + AI generation: HeyGen/Hedra para
   talking head, Runway para scenes, Lottie para animaciones, manual
   para screen recordings).
6. **Cost model** concreto: ~$2.500 USD para crear MVP de assets, vs
   ~$5.000 si fuéramos atomic-puro sin reuse.

---

## 2. Decisiones del owner (recap)

(De conversación con owner, 2026-05.)

### 2.1 Sobre video

| Decisión | Valor |
|----------|-------|
| Tipos de video | Todos (talking head, scenes, animaciones, screen-recordings, AI-generated) |
| Mecanismo de selección | Sistema de labels según contenido pedagógico |
| Producción | Solo TTS + AI generation (no actores reales) |
| Duración target | 15–30s (corto, foco, engagement) |

### 2.2 Sobre imágenes

| Decisión | Valor |
|----------|-------|
| Usos | Todos (image_description, vocab illustrations, scenes, avatars IA, infografías) |
| Producción | 100% AI generation (no stock, no custom illustrators) |

### 2.3 Sobre audio

| Decisión | Valor |
|----------|-------|
| Recordings native | NO en MVP (foco económico). 100% TTS. |
| Voz "host" cloneada (anchor) | SÍ — una voz consistent del producto |
| Pool de voces para listening | 6–8 voces variadas (US/UK, m/f, mid/young) |
| Voces character-consistent | SÍ para roleplays |

---

## 3. Por qué esto es economic core

### 3.1 Principio rector

> **"Más assets reutilizables = engine más barato y óptimo."** — owner.

Operaciones realtime de IA (conversación 1:1 con LLM) cuestan dinero
por cada uso. Preassets pre-generados cuestan **$0 marginal por uso**
(solo storage + bandwidth, ambos despreciables a escala con R2 que no
cobra egress).

### 3.2 Implicación arquitectónica

```
Realtime AI op    →  $0.01-0.05 por uso  →  Sparks-protected
Preasset access   →  ~$0.0001 por uso    →  Free unlimited
```

Cuanto más solucionemos con preassets, mejor el unit economics.
Confirmado en `sparks-system.md`: `preasset_access` cuesta 0 Sparks.

### 3.3 La economía del reuso (atomic → composite)

Hoy cada asset es monolítico:

```
asset_001: roleplay_structured_interview (incluye su audio inline)
asset_002: free_response_about_interview (audio prompt distinto)
asset_003: listening_mc_about_interview (audio question distinto)
```

→ 3 audios producidos.

Con modelo atomic + composite:

```
ATOMICS                                   COMPOSITES
audio_anita_interview_intro_30s     ←─┬── asset_001: roleplay structured
                                       ├── asset_002: free_response
                                       └── asset_003: listening_mc
```

→ **1 audio producido, usado 3 veces.**

A escala MVP (700 assets), con reuse factor target 2.0 → **350 atomics
realmente creados**. Costo de creación a la mitad.

---

## 4. Sistema de labels unificado

### 4.1 De enum monolítico a dimensiones

**Hoy:** `AssetType` es enum con 14 valores hardcoded
(`pronunciation_drill`, `listening_mc`, etc.). Agregar video implicaría
multiplicar por 5 nuevos tipos.

**Propuesta:** descomponer en labels independientes.

```typescript
interface CompositeAssetLabels {
  // Formato del media principal del ejercicio
  primary_media_format: 'text' | 'audio' | 'image' | 'video' | 'multi';

  // Subtipo del media (cuando aplica)
  primary_media_subtype?:
    | 'talking_head' | 'scene' | 'animation' | 'screen_recording'  // video
    | 'illustration' | 'photo' | 'infographic' | 'avatar'          // image
    | 'tts' | 'cloned_host' | 'native_recording';                  // audio

  // Tipo de interacción pedagógica
  interaction_type:
    | 'mc'              // multiple choice
    | 'qa'              // open Q&A
    | 'drill'           // repeat-after
    | 'shadow'          // simultaneous shadowing
    | 'free_response'   // open speaking
    | 'roleplay_structured'
    | 'roleplay_free'
    | 'description'     // describir image/video
    | 'fill_blank'
    | 'translation'
    | 'ordering'
    | 'dictation';

  // Pedagogía
  pedagogical_focus: string[];     // sub-skill IDs
  duration_seconds?: number;
  cefr_level: string;
  target_variant: 'us' | 'uk' | 'neutral';

  // Contexto
  topic: string[];
  context_tags: string[];
  cultural_relevance: string[];
}
```

### 4.2 Ventajas

- **Escalable:** agregar `video_explainer` no toca un enum, solo
  combinación nueva de labels.
- **Búsqueda flexible:** "todos los videos talking_head de USA business
  B2" es una query.
- **Validación combinatoria:** matrix de qué `media_format` ×
  `interaction_type` son válidos (algunas combinaciones no tienen
  sentido).

### 4.3 Matrix de validez

|  | mc | qa | drill | shadow | free_response | roleplay_s | roleplay_f | description | fill_blank | translation | ordering | dictation |
|--|:--:|:--:|:----:|:------:|:-------------:|:----------:|:----------:|:-----------:|:----------:|:-----------:|:--------:|:---------:|
| text | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ❌ |
| audio | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| image | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| video | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| multi | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |

`multi` = ejercicio que combina más de un media (ej: video + texto +
audio).

---

## 5. Audio strategy

### 5.1 Recomendación final: 100% TTS con jerarquía de voces

**Voz "Anita" (host clonado, anchor del producto):**
- Una sola voz, consistente, cloneada vía ElevenLabs Pro.
- Presente en: track intros, level completions, daily reminder audios,
  weekly summary, transitions.
- Costo de cloning: $0 incluido en plan ElevenLabs Pro.
- ~5–10% del audio total del producto.

**Pool de voces para listening exercises (6–8 voces):**

| Voice | Acento | Género | Edad | Uso |
|-------|--------|:------:|:----:|-----|
| `voice_us_f_25` | US-General | F | 25-35 | Roleplays casual, friend conversations |
| `voice_us_m_30` | US-General | M | 25-35 | Roleplays business, peer interactions |
| `voice_us_f_40` | US-General | F | 35-45 | Authority figures (HR, manager) |
| `voice_us_m_45` | US-General | M | 40-50 | Professional contexts (interviewer) |
| `voice_uk_f_30` | UK-RP | F | 25-40 | Listening exercises UK variant |
| `voice_uk_m_35` | UK-RP | M | 25-45 | Listening exercises UK variant |
| `voice_us_f_60` | US-General | F | 55+ | Older characters (grandmother, mentor) |
| `voice_us_m_22` | US-General | M | 18-25 | Young characters, peer learners |

**Character-consistent para roleplays:**
- Cada character recurrente en un track (ej: "Sarah la HR", "Mike el
  barista") tiene voz fija de las anteriores.
- Reuse cross-asset: misma Sarah aparece en 5 ejercicios distintos del
  track Job Ready.

### 5.2 Pipeline de generación

AI Gateway task `generate_audio_tts`:

```typescript
{
  task_id: 'generate_audio_tts',
  input_schema: z.object({
    text: z.string(),
    voice_id: z.string(),                  // 'voice_us_f_25' o 'voice_anita_host'
    target_variant: z.enum(['us', 'uk']),
    speed: z.number().default(1.0),        // 0.85 = slow, 1.0 = natural
    emotion: z.enum(['neutral', 'friendly', 'serious']).default('neutral'),
    audio_format: z.enum(['mp3_128k', 'mp3_192k', 'pcm_22050']).default('mp3_128k'),
  }),
  output_schema: z.object({
    audio_url: z.string(),                 // R2 signed URL
    duration_ms: z.number(),
    storage_key: z.string(),
  }),
  primary: { provider: 'elevenlabs', model: 'eleven_multilingual_v2', ... },
  fallback_chain: [
    { provider: 'azure', model: 'tts-en-us-neural', ... },
  ],
  // ...
}
```

### 5.3 Costos audio

**Plan ElevenLabs:**
- Pro ($99/mes): 500k chars/month, 192k character voice quality, voice
  cloning. **Insuficiente para MVP** (700 assets × ~50s × ~150
  chars/s = ~5.25M chars).
- Scale ($330/mes): 2M chars/month + commercial license. **Sigue
  insuficiente** para batch inicial.
- Pay-as-you-go: $0.18/1k chars con plan Pro. ~$945 total para MVP
  (one-time).

**Estimación MVP (one-time creación):**

```
700 assets × 30s avg × 150 chars/s = 3.15M chars
3.15M × $0.18/1k = $567 USD
+ Voice cloning (1 host): $0
+ Buffer 30%: +$170
= ~$740 USD para audio MVP completo
```

Con reuse atomic → ~$370 (50% reduction).

### 5.4 Audio para annotations culturales (cross-ref)

Cuando se promueva la exploración de
[`cultural-annotations.md`](cultural-annotations.md), las ~91 frases ×
2-3 voces se generan con mismo pipeline. Costo marginal: ~$5-10.

---

## 6. Image strategy

### 6.1 Recomendación: 100% AI generation con prompts versionados

**Stack:**
- **Primary:** DALL-E 3 (vía OpenAI API). $0.04/img standard, $0.08
  HD. Calidad consistente, follows prompt bien.
- **Secondary:** Flux Pro (vía Replicate o fal.ai). $0.05/img. Mejor
  control de estilo, soporta más artistic styles.
- **Tertiary:** Imagen 3 (Google, vía AI Gateway). $0.04/img.

### 6.2 Tipos de imagen y prompt templates

#### 6.2.1 `image_description` exercises

Imágenes que el user describe verbalmente. Necesitan:
- Contenido visual rico (varias entities, acción).
- Sin texto sobreimpreso (que el user no lea palabras).
- Estilo fotográfico (no cartoon).

**Prompt template:**

```
A photorealistic image of {scene}: {detailed_description}.
Wide angle, natural lighting, vibrant colors.
The scene should clearly show {key_visual_elements}.
NO text or writing visible. NO logos.
{cultural_context_modifier}
```

#### 6.2.2 Vocabulary illustrations (flashcards)

Iconos/ilustraciones simples para palabras nuevas.

**Prompt template:**

```
A clean, minimalist illustration of "{word}" in flat design style.
Solid color background ({brand_color}). Centered subject.
No text. {cultural_context_modifier}
```

#### 6.2.3 Scene backgrounds para roleplays

El user "ve" el lugar donde está hablando.

**Prompt template:**

```
First-person view of {location}, photorealistic.
{environmental_details}.
The viewer's perspective is sitting/standing at typical interaction
point.
NO people in foreground (user es persona).
NO text visible.
```

#### 6.2.4 AI character avatars (post-MVP nice-to-have)

Avatar visual de personajes recurrentes en roleplays. Consistencia
visual cross-asset.

**Prompt template (con character description fija):**

```
Portrait of {character_name}: {fixed_character_description}.
Friendly expression. Professional photo style.
Headshot framing.
```

Character description debe ser **idéntica** en todas las generaciones
del mismo character para preservar consistencia (DALL-E no garantiza
exact match — Flux Pro con seeds es mejor).

#### 6.2.5 Infografías gramaticales (post-MVP)

Diagramas de tiempos verbales, sentence structures, etc.

**Mejor opción:** ilustraciones custom o Figma/Canva templates en
lugar de AI (consistency, accuracy). No AI-generation.

### 6.3 Pipeline de generación

AI Gateway task `generate_image`:

```typescript
{
  task_id: 'generate_image',
  input_schema: z.object({
    prompt: z.string(),
    style: z.enum(['photorealistic', 'flat_illustration', 'minimal',
                   'cartoon']),
    aspect_ratio: z.enum(['1:1', '16:9', '9:16', '4:3']).default('16:9'),
    quality: z.enum(['standard', 'hd']).default('standard'),
    seed?: z.number(),                    // para reproducibilidad
  }),
  output_schema: z.object({
    image_url: z.string(),                // R2 signed URL
    storage_key: z.string(),
    width: z.number(),
    height: z.number(),
  }),
  primary: { provider: 'openai', model: 'dall-e-3', ... },
  fallback_chain: [
    { provider: 'fal', model: 'flux-pro-1.1', ... },
  ],
}
```

### 6.4 Validación post-generación

Validar:
- Resolución mínima (1024×1024 o 1792×1024).
- No texto (OCR check; rechazar si detecta palabras).
- Aspect ratio correcto.
- File size razonable (< 2MB).

Si valida: subir a R2 con storage_key estructurado.

### 6.5 Costos image

**Estimación MVP:**

```
~30% de assets requieren imagen ≈ 210 imágenes para MVP
DALL-E 3 standard: $0.04/img
210 × $0.04 = $8.40 + reroll buffer 50% = ~$13 USD
```

Para Fase 2 (1.500 assets ≈ 450 imgs): $27.

**Imágenes serán siempre re-generables** si decidimos cambiar estilo
visual del producto en el futuro.

### 6.6 Storage paths en R2

```
images/
├── illustrations/      # vocab flashcards
│   └── {word}_{seed}.webp
├── scenes/             # backgrounds para roleplays
│   └── {scene_id}_{version}.webp
├── descriptions/       # image_description exercises
│   └── {asset_id}_{version}.webp
├── avatars/            # character portraits
│   └── {character_id}_{seed}.webp
└── infographics/       # post-MVP
    └── {topic}_{version}.svg
```

Convertir output (PNG) a WebP para reducir bandwidth ~30%.

---

## 7. Video strategy

### 7.1 4 tipos de video con AI generation

#### 7.1.1 Talking head (60-70% de videos del MVP)

**Tool:** **HeyGen** o **Hedra** (avatar + audio TTS).

- HeyGen: $24-89/mes, librería de avatars, lip-sync de calidad.
- Hedra Character-3: avatar custom, $0.30-0.50/min generated.

**Workflow:**
1. Generar audio con ElevenLabs (voz Anita o pool).
2. Pasar audio a HeyGen/Hedra con avatar elegido.
3. Output: video 1080p mp4.

**Use cases:**
- Track intros: "Hi! I'm Sarah, and today we're starting Job Ready..."
- Concept explanations: avatar explica past perfect en 30s.
- Listening prompts: "Listen to this story about..."

#### 7.1.2 Scene videos (15-20% de videos)

**Tool:** **Runway Gen-3** o **Sora** (cuando esté disponible API).

- Runway Gen-3: $0.05/sec generated. 30s video = ~$1.50.
- Calidad escenas variable; consistency mejor con seeds.

**Workflow:**
1. Prompt detallado de escena (ej: "Busy coffee shop interior,
   barista preparing drinks, customers chatting").
2. Generate 30s.
3. Si tiene diálogo: capa de audio TTS encima (lip sync no perfecto
   pero aceptable a 30s).

**Use cases:**
- Listening with visual context: video de café + audio diálogo.
- Cultural scenes: cómo es una entrevista en USA.
- Free response prompts visuales: "describe what's happening in this
  scene".

#### 7.1.3 Animations / explainers (10% de videos)

**Tool:** **Lottie animations** (manual + AI helper).

- Animations vectoriales JSON, livianas, scalables.
- Lottie editor + GPT-4 ayuda a generar animations descriptions.
- Templates reutilizables (verb tense diagrams, sentence structure).

**Workflow:**
1. Diseñar template Lottie (una vez por concepto).
2. Personalizar con texto/colores via params.
3. Reuse cross-assets.

**Use cases:**
- Grammar explanations animadas.
- Sentence structure visualizations.
- Pronunciation mouth diagrams animados.

**Costo:** ~2h por template (humano), reuse infinito. Out-of-pocket
$0.

#### 7.1.4 Screen recordings (5% de videos)

**Tool:** Manual (OBS, screencast on Mac/Windows). No AI.

**Use cases:**
- Demos de cómo navegar un sitio en inglés.
- Email writing tutorials (escribir email en pantalla).
- Live LinkedIn navigation, etc.

**Costo:** producción manual una vez, reuse infinito. ~$0 marginal.

### 7.2 Pipeline de generación

AI Gateway tasks (varias):

```typescript
'generate_video_talking_head': {
  input_schema: z.object({
    audio_url: z.string(),              // pre-generado
    avatar_id: z.string(),              // 'avatar_anita_v1'
    background: z.enum(['neutral', 'office', 'outdoors']),
    aspect_ratio: z.enum(['9:16', '1:1', '16:9']),
  }),
  output_schema: z.object({
    video_url: z.string(),
    duration_ms: z.number(),
    storage_key: z.string(),
  }),
  primary: { provider: 'hedra', model: 'character-3', ... },
  fallback_chain: [
    { provider: 'heygen', model: 'avatar-v3', ... },
  ],
},

'generate_video_scene': {
  input_schema: z.object({
    prompt: z.string(),
    duration_seconds: z.number().min(5).max(30),
    style: z.enum(['photorealistic', 'cinematic']),
    seed?: z.number(),
  }),
  output_schema: z.object({
    video_url: z.string(),
    duration_ms: z.number(),
    storage_key: z.string(),
  }),
  primary: { provider: 'runway', model: 'gen-3-alpha', ... },
},

'compose_video_with_audio_overlay': {
  // FFmpeg pipeline: combina scene video + audio TTS
  input_schema: z.object({
    video_url: z.string(),
    audio_url: z.string(),
    audio_volume: z.number().default(1.0),
    background_music_url?: z.string(),
  }),
  output_schema: z.object({
    composed_video_url: z.string(),
  }),
  primary: { provider: 'self_hosted', model: 'ffmpeg-pipeline', ... },
},
```

### 7.3 Costos video

**Estimación MVP:**

```
~20% de assets tienen video ≈ 140 videos
Distribución:
- 80 talking head × ~$0.40/30s = $32
- 30 scene videos × ~$1.50/30s = $45
- 20 animations (templates reusable) × ~2h dev = labor solamente
- 10 screen recordings × producción manual = labor solamente

Total AI generation: ~$77 USD
Buffer 50% (rerolls, fail validation): +$40
= ~$117 USD para video MVP
```

Para Fase 2 (1.500 assets, 300 videos): ~$300.

### 7.4 Storage paths en R2

```
videos/
├── talking_head/
│   ├── {character_id}/
│   │   └── {asset_id}_{version}.mp4
├── scenes/
│   └── {scene_id}_{seed}.mp4
├── animations/
│   ├── templates/
│   │   └── {template_name}.json     # Lottie
│   └── rendered/
│       └── {asset_id}_{version}.mp4
└── screen_recordings/
    └── {topic}_{version}.mp4
```

Codec: H.264 (compatibilidad universal), 720p (mobile-first), 30fps.
Tamaño target: ~3-5MB por video de 30s.

### 7.5 Bandwidth a escala

A 100k users activos × 5 videos vistos/día × 5MB = 2.5 TB/día = 75
TB/mes.

**Cloudflare R2:** $0/egress. 100k users es manejable.

A 1M users: 750 TB/mes. R2 storage cost: $0.015/GB-mo = ~$11k/mes
solo storage. Reconsiderar CDN strategy ahí.

---

## 8. Schema: atomic + composite

### 8.1 Tabla `media_atomics` (nueva)

```sql
CREATE TABLE media_atomics (
  id              TEXT PRIMARY KEY,             -- 'audio_anita_intro_jr_v1'
  version         TEXT NOT NULL DEFAULT '1.0.0',

  -- Tipo
  media_format    TEXT NOT NULL CHECK (media_format IN (
                    'audio', 'image', 'video'
                  )),
  media_subtype   TEXT NOT NULL,

  -- Storage
  storage_key     TEXT NOT NULL UNIQUE,         -- key en R2
  storage_provider TEXT NOT NULL DEFAULT 'r2',
  duration_ms     INT,                          -- audio/video
  width           INT,                          -- image/video
  height          INT,                          -- image/video
  file_size_bytes INT NOT NULL,
  mime_type       TEXT NOT NULL,

  -- Metadata pedagógica/cultural
  language        TEXT NOT NULL DEFAULT 'en',
  english_variant TEXT NOT NULL DEFAULT 'us',
  cultural_relevance TEXT[] DEFAULT '{}',

  -- Audio specific
  voice_id        TEXT,                         -- 'voice_anita_host', 'voice_us_f_25'
  speed           NUMERIC(3,2),                 -- 0.85, 1.0, 1.15

  -- Video specific
  video_subtype   TEXT,                         -- 'talking_head', 'scene', 'animation'
  has_audio       BOOLEAN DEFAULT true,
  character_id    TEXT,                         -- 'character_sarah_hr' si talking_head

  -- Image specific
  image_subtype   TEXT,                         -- 'illustration', 'photo', 'avatar'

  -- Generation metadata
  generated_by    TEXT NOT NULL,                -- 'ai_dalle3', 'ai_hedra', 'manual', 'lottie_template'
  generation_cost_usd NUMERIC(10,4),
  generation_seed INT,
  generation_prompt TEXT,

  -- Lifecycle
  approved        BOOLEAN NOT NULL DEFAULT false,
  archived        BOOLEAN NOT NULL DEFAULT false,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  reviewed_at     TIMESTAMPTZ,

  -- Reuse tracking
  use_count       INT NOT NULL DEFAULT 0       -- denormalized: # de assets que lo usan
);

CREATE INDEX idx_atomics_format_subtype
  ON media_atomics(media_format, media_subtype)
  WHERE approved = true AND archived = false;
CREATE INDEX idx_atomics_voice ON media_atomics(voice_id)
  WHERE media_format = 'audio';
CREATE INDEX idx_atomics_character ON media_atomics(character_id)
  WHERE character_id IS NOT NULL;
CREATE INDEX idx_atomics_high_reuse ON media_atomics(use_count DESC)
  WHERE approved = true;
```

### 8.2 Modificación a `learning_assets`

Reemplazar el field `content.audio_url` etc. con referencias a atomics:

```typescript
interface LearningAsset {
  // ... campos existentes ...

  // Labels (de §4)
  labels: CompositeAssetLabels;

  // Atomic references
  media_atomics: Array<{
    atomic_id: string;
    role: string;             // 'main_audio', 'background_image', 'avatar', 'audio_question_1', ...
    position?: { start_ms: number; end_ms: number };
  }>;

  // Content específico de la interacción (no media)
  interaction_data: AssetInteractionData;
}

type AssetInteractionData =
  | { type: 'mc'; questions: MultipleChoiceQuestion[] }
  | { type: 'qa'; questions: OpenQuestion[] }
  | { type: 'drill'; phrases: DrillPhrase[] }
  | { type: 'shadow'; reference_atomic_id: string }
  | { type: 'free_response'; prompt: string; expected_topics: string[] }
  | { type: 'roleplay_structured'; flow: RoleplayStep[] }
  // ...etc
```

### 8.3 Tabla `characters` (nueva)

Para consistencia visual + auditiva cross-assets:

```sql
CREATE TABLE characters (
  id              TEXT PRIMARY KEY,             -- 'character_sarah_hr'
  name            TEXT NOT NULL,                -- 'Sarah'
  description     TEXT NOT NULL,                -- prompt fijo para imagen
  voice_id        TEXT NOT NULL,                -- voz de pool
  avatar_atomic_id TEXT REFERENCES media_atomics(id),
  default_video_avatar_id TEXT,                 -- HeyGen avatar id

  -- Persona
  age_range       TEXT,
  occupation      TEXT,
  cultural_background TEXT,                     -- 'us_general', 'us_latino_heritage', etc.

  -- Tracks donde aparece
  appears_in_tracks TEXT[],

  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Cuando un asset roleplay incluye `character_id = 'character_sarah_hr'`,
todas sus interacciones usan automáticamente la voz + apariencia de
Sarah.

### 8.4 Job de actualización de `use_count`

Cron diario:

```sql
UPDATE media_atomics ma
SET use_count = (
  SELECT COUNT(DISTINCT la.id)
  FROM learning_assets la
  WHERE EXISTS (
    SELECT 1 FROM jsonb_array_elements(la.media_atomics) elem
    WHERE elem->>'atomic_id' = ma.id
  )
  AND la.approved = true AND la.archived = false
);
```

Útil para identificar atomics más reutilizados (alta value) y
candidatos a retirar (use_count = 0 después de 90 días).

---

## 9. Pipeline de creación end-to-end

### 9.1 Flow para asset compuesto

```
┌─────────────────────────────────────────────────────────────┐
│ ETAPA 1: PLANIFICACIÓN                                      │
│ - Identificar gap en biblioteca                             │
│ - Definir labels (media_format, interaction_type, etc.)     │
│ - Decidir qué atomics necesita (existentes o nuevos)        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ETAPA 2: ATOMICS                                            │
│ - Reuse de existentes si posible                            │
│ - Generar nuevos atomics:                                   │
│   • Audio: AI Gateway task generate_audio_tts               │
│   • Image: AI Gateway task generate_image                   │
│   • Video: AI Gateway task generate_video_*                 │
│ - Validation post-generation                                │
│ - Persist en media_atomics                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ETAPA 3: COMPOSICIÓN                                        │
│ - Crear learning_asset con references a atomics             │
│ - Definir interaction_data (preguntas, prompts, flow)       │
│ - Validate combinación de labels                            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ETAPA 4: VALIDATION                                         │
│ - Schema Zod                                                │
│ - Pedagogical (CEFR match, sub-skills cubiertas)            │
│ - Atomics existen y son approved                            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ETAPA 5: REVISIÓN HUMANA (sample)                           │
│ - Atomics: 50% review al inicio, 10% en steady state        │
│ - Composites: 30% review al inicio, 10% en steady state     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ ETAPA 6: A/B TEST EN PROD                                   │
│ (Igual que content-creation §4)                             │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 Reuse first

Antes de generar atomic nuevo, **buscar en `media_atomics` existentes**:

```sql
-- Para audio TTS de cierta phrase
SELECT id FROM media_atomics
WHERE media_format = 'audio'
  AND media_subtype = 'tts'
  AND voice_id = $voice_id
  AND generation_prompt = $text
  AND approved = true;
```

Si match: reuse. Si no: generate.

### 9.3 Pre-generation queue

Para batch nights, queue de atomics a generar:

```sql
CREATE TABLE atomic_generation_queue (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  requested_for_asset_id TEXT,
  media_format    TEXT NOT NULL,
  media_subtype   TEXT NOT NULL,
  prompt          TEXT NOT NULL,
  config          JSONB NOT NULL,            -- voice_id, style, etc.
  priority        INT NOT NULL DEFAULT 5,
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending', 'generating', 'completed', 'failed')),
  result_atomic_id TEXT REFERENCES media_atomics(id),
  attempts        INT NOT NULL DEFAULT 0,
  last_error      TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_atomic_queue_pending
  ON atomic_generation_queue(priority, created_at)
  WHERE status = 'pending';
```

Job nocturno consume queue, llama AI Gateway tasks, persist atomics.

---

## 10. AI Gateway tasks nuevas

Resumen de tasks a registrar en `ai-gateway-strategy.md` §4.2:

| Task ID | Provider primary | Costo / op | Categoría |
|---------|------------------|-----------:|-----------|
| `generate_audio_tts` | ElevenLabs | $0.18/1k chars | batch |
| `generate_image` | OpenAI (DALL-E 3) | $0.04/img | batch |
| `generate_video_talking_head` | Hedra | $0.30/30s | batch |
| `generate_video_scene` | Runway Gen-3 | $0.05/sec | batch |
| `compose_video_with_audio_overlay` | Self-hosted FFmpeg | ~$0 | batch |
| `validate_image_no_text` | OpenAI Vision (cheap) | $0.01/img | batch |
| `validate_audio_quality` | Self-hosted (FFmpeg + analysis) | ~$0 | batch |
| `extract_video_thumbnail` | Self-hosted FFmpeg | ~$0 | batch |

Todos son **batch** (no realtime). Generación nocturna o on-demand
admin trigger.

---

## 11. Storage en R2: estrategia consolidada

### 11.1 Bucket structure

```
english-spark-assets/
├── audio/
│   ├── tts/
│   │   ├── voice_anita_host/
│   │   │   └── {atomic_id}.mp3
│   │   ├── voice_us_f_25/
│   │   │   └── {atomic_id}.mp3
│   │   └── ...
│   └── exercises/                         # audio specific to one exercise
│       └── {atomic_id}.mp3
├── images/
│   ├── illustrations/
│   ├── scenes/
│   ├── descriptions/
│   ├── avatars/
│   └── infographics/
├── videos/
│   ├── talking_head/
│   ├── scenes/
│   ├── animations_rendered/
│   └── screen_recordings/
├── lottie_templates/
│   └── {template_name}.json
└── user_uploads/                          # audios del usuario (separado)
    └── users/
        └── {user_id}/
            └── {timestamp}_{random}.webm
```

### 11.2 Signed URLs

Todos los media privados via signed URLs con TTL 5 min.

```typescript
async function getSignedAtomicUrl(atomicId: string): Promise<string> {
  const atomic = await db.media_atomics.findById(atomicId);
  return await r2.createSignedUrl(atomic.storage_key, { expiresIn: 300 });
}
```

Cliente cachea URLs 4 min (margen) y refetcha si expira.

### 11.3 CDN

Cloudflare R2 + Cloudflare CDN nativo. Edge caching automático.

---

## 12. Cost model consolidado

### 12.1 Costos one-time de creación MVP

```
Audio (700 assets × 30s × ~$0.027/asset)        ~$370
  Con reuse atomic 2x:                          ~$185

Imagen (210 imgs × $0.04 + buffer)               ~$13

Video (140 videos × ~$0.55 mix + buffer)        ~$117

Lottie templates (manual, una vez)              ~10h labor (~$200)

Screen recordings (manual)                       ~5h labor (~$100)

Tooling (FFmpeg pipeline, validation scripts)    ~20h dev

----
Total media MVP: ~$615 USD generación + ~$300 labor
                ≈ $1.000 USD all-in
```

### 12.2 Costos one-time Fase 2 (expansión a 1.500 assets)

```
Marginal media generation: ~$1.000 USD
```

### 12.3 Costos recurrentes

```
ElevenLabs Pro plan (si volumen alto):    $99-330/mes
OpenAI/Runway/Hedra:                       Pago por uso (~$0 si no
                                           se generan nuevos)
R2 storage (estimado a 100k users):       ~$50/mes
R2 bandwidth:                              $0 (no egress fees)
```

### 12.4 Comparación: con vs sin reuse atomic

| Escenario | MVP one-time | Notas |
|-----------|-------------:|-------|
| Sin reuse (atomic = composite) | ~$2.000 USD | Cada asset re-genera todo |
| Con reuse 2x | ~$1.000 USD | Mismo media en N assets |
| Con reuse 3x (target Fase 2+) | ~$700 USD | Atomics altamente reusados |

**Reuse de 2x = ahorro $1k inicial; a escala (10k assets) = ahorro
$10k+.**

### 12.5 Comparación: Sparks Plan Pro

User Pro: 200 Sparks/mes (~$5 revenue) → costo de IA realtime
target < $1.50.

Cost of preasset access: ~$0.0001 por delivery. Para 100 deliveries/mes
por user = $0.01. **Despreciable.**

Reuse atomic permite que el catálogo crezca sin que el unit cost por
user pago suba significativamente.

---

## 13. Plan de implementación (cuando se promueva)

### 13.1 Fase pre-MVP (semanas -2 a 0)

- [ ] Crear cuentas: ElevenLabs Pro, OpenAI, Runway, Hedra.
- [ ] Diseñar voz "Anita" (1h con voice cloning ElevenLabs).
- [ ] Definir 8 voices del pool (selección + tests).
- [ ] Crear 3-5 Lottie templates fundamentales.

### 13.2 Sprint 1 (semanas 1-2): Foundation

- [ ] Tabla `media_atomics` migrada.
- [ ] Tabla `characters` migrada.
- [ ] Modificar schema `learning_assets` con labels + atomic refs.
- [ ] AI Gateway task `generate_audio_tts` registrado.
- [ ] Validation post-TTS (audio quality check).
- [ ] Admin panel para review de atomics.

### 13.3 Sprint 2 (semanas 3-4): Image generation

- [ ] AI Gateway task `generate_image` registrado.
- [ ] Validation: OCR check para "no text", aspect ratio.
- [ ] Prompt templates por uso (illustrations, scenes,
  descriptions, avatars).
- [ ] Generación batch primeras ~50 imágenes.

### 13.4 Sprint 3 (semanas 5-6): Video — talking head

- [ ] AI Gateway task `generate_video_talking_head`.
- [ ] Pipeline: TTS → Hedra → R2.
- [ ] Validation video quality.
- [ ] Generación de 30 talking head videos para Job Ready.

### 13.5 Sprint 4 (semanas 7-8): Video — scenes y animations

- [ ] AI Gateway task `generate_video_scene` (Runway).
- [ ] AI Gateway task `compose_video_with_audio_overlay` (FFmpeg).
- [ ] Lottie rendering pipeline.
- [ ] Generación de 20 scene videos + 10 animations.

### 13.6 Sprint 5 (semanas 9-10): Reuse y optimización

- [ ] Cron de update `use_count`.
- [ ] Search de atomics existentes antes de generar.
- [ ] Métricas de reuse rate.
- [ ] Optimización de prompts según learning.

### 13.7 Sprint 6 (semanas 11-12): Volume + polish

- [ ] Generación batch del resto del MVP (~700 assets total).
- [ ] Performance tracking por atomic.
- [ ] A/B test de quality (DALL-E vs Flux para subset).

---

## 14. Decisiones abiertas

- [ ] **¿Voz "Anita" host masculina o femenina?** Probable femenina
  (estadísticamente más bienvenida en educación según research). Pero
  decidir con A/B real.
- [ ] **¿Avatar para "Anita" en HeyGen/Hedra o mantener solo audio?**
  Avatar la vuelve más memorable pero costo +30% por video. Posible
  compromise: avatar solo en track intros y level completions, no en
  cada listening exercise.
- [ ] **Aspect ratios:** ¿9:16 (vertical mobile-first), 1:1 (square),
  16:9 (landscape)? Mobile-first sugiere 9:16. Pero web requiere 16:9.
  Trade-off: generar 9:16 solo (mobile) y crop o letterbox para web.
- [ ] **¿Subtítulos en videos?** Mejora accessibility + comprensión.
  Sí, por default, opt-out por user. Generación automática vía
  Whisper (~$0.006/min).
- [ ] **Política de retention de atomics archivados:** después de 90
  días sin uso ¿se borran físicamente de R2 o se mantienen para audit?
  Recomendación: borrar (R2 cobra storage).
- [ ] **¿Crear pipeline de regeneración masiva si decidimos cambiar
  estilo visual del producto?** Sí, agregar admin endpoint
  `regenerateAtomicsByLabel(filter, new_prompt_template)`. Caro pero
  necesario eventualmente.
- [ ] **Atomic versioning policy:** semver major.minor.patch o flat?
  Recomendación: semver major.minor (sin patch). Major = breaking
  change visual/audio. Minor = mejora.
- [ ] **¿Limitamos cantidad de atomics by character (ej: Sarah)?**
  Demasiados podrían diluir consistencia. Cap target: 20 atomics por
  character. Si se exceden: crear character variant (Sarah_2 hermana?).

---

## 15. Cambios a docs cuando se promueva a spec

### 15.1 `content-creation-system.md`

- §3: agregar sección "Atomic vs Composite assets".
- §3.2: actualizar AssetType para reflejar labels.
- §4: agregar "Etapa 2: Atomics" al pipeline.
- §7: agregar tabla `media_atomics` y `characters`.
- §8: agregar API contracts `getAtomic`, `searchAtomics`, etc.
- §10: agregar costos consolidados de §12 de este doc.

### 15.2 `pedagogical-system.md`

- §5.1: actualizar catálogo de tipos de ejercicios para reflejar
  combinaciones video+interaction (ej: video_listening_qa).
- §5.4: actualizar matriz de pesos para incluir ejercicios con video.

### 15.3 `ai-gateway-strategy.md`

- §4.2: agregar 8 tasks nuevas listadas en §10 de este doc.

### 15.4 `architecture/01-overview.md`

- §4: agregar `media_atomics`, `characters`, `atomic_generation_queue`
  a data ownership de content-creation.
- §5.2: agregar evento `atomic.generated`, `atomic.reused`.

### 15.5 `cross-cutting/data-and-events.md`

- §5: agregar eventos relacionados a atomics y composite assets.

### 15.6 `cross-cutting/security-threat-model.md`

- §3.6: actualizar R2 threats con nuevos paths.

### 15.7 `architecture/sparks-system.md`

- §5: confirmar que `preasset_access` cubre delivery de cualquier
  atomic (audio, image, video). Costo 0 Sparks.

---

## 16. Métricas de éxito

| Métrica | Definición | Target MVP | Target Fase 2 |
|---------|-----------|-----------:|--------------:|
| **Reuse rate** de atomics | avg uses por atomic | ≥ 1.5 | ≥ 2.5 |
| Generación cost per asset | $ / composite asset | ≤ $1.50 | ≤ $1.00 |
| Validation pass rate atomics | % primer try | ≥ 75% | ≥ 90% |
| User completion rate de assets video | % completan vs abandonan | ≥ 70% | ≥ 80% |
| Bandwidth cost / user / mes | R2 + CDN | ≤ $0.01 | ≤ $0.05 |
| Time to generate atomic (avg) | minutes | ≤ 2 | ≤ 1 |
| % de assets con video | engagement metric | 20% | 30% |
| Avatar consistency rating | review humano | ≥ 4/5 | ≥ 4.3/5 |

---

## 17. Riesgos

| Riesgo | Probabilidad | Mitigación |
|--------|:-----------:|-----------|
| AI generation quality inconsistente | Alta | Validation post-gen + sample humano + retry policy |
| ElevenLabs cambia pricing significativamente | Media | Fallback a Azure TTS (más barato pero menor calidad) |
| Bandwidth de video escala mal | Media | R2 sin egress; reconsiderar CDN strategy a 1M users |
| Lip-sync de Hedra/HeyGen falla en español accent | Media | Test con usuarios reales antes de scale; fallback a audio-only |
| Avatar/character consistency cross-asset rota | Alta | Seeds determinísticos en Flux Pro; review humano de batches |
| Costos AI escalan más rápido que reuse | Media | Métricas de reuse rate como guardrail; alerta si baja de 1.5 |
| User reporta video con contenido inapropiado | Baja | OCR check + content moderation API en validation |
| Provider AI deprecated mid-MVP | Baja | Fallback chain en AI Gateway |
| Storage costs en R2 escalan no linealmente | Baja | Compression policy: WebP para imgs, H.264 720p para video |

---

## 18. Referencias internas

Cuando se promueva a spec, vincular con:

- [`../product/content-creation-system.md`](../product/content-creation-system.md) §3.2, §7, §8.
- [`../product/pedagogical-system.md`](../product/pedagogical-system.md) §5.1, §5.4.
- [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) §4.2.
- [`../architecture/sparks-system.md`](../architecture/sparks-system.md) §5 (preasset_access).
- [`../architecture/01-overview.md`](../architecture/01-overview.md) §4 (data ownership).
- [`cultural-annotations.md`](cultural-annotations.md) — usa pipeline
  similar de audio para frases.

---

## 19. Recursos externos

- ElevenLabs (TTS + voice cloning): https://elevenlabs.io/
- HeyGen (talking head avatars): https://www.heygen.com/
- Hedra (Character-3 model): https://www.hedra.com/
- Runway (Gen-3 scene video): https://runwayml.com/
- DALL-E 3 (vía OpenAI API): https://platform.openai.com/docs/guides/images
- Flux Pro (vía Replicate o fal.ai): https://fal.ai/models/fal-ai/flux-pro
- Lottie (vector animations): https://lottiefiles.com/
- FFmpeg (video composition): https://ffmpeg.org/

---

*Documento de exploración. Promover a specs (modificando los docs
referenciados en §15) cuando esté validado el approach. La estrategia
de reuse atomic→composite es CORE para el unit economics — promoverla
incluso si la generación masiva de video se difiere a Fase 2.*

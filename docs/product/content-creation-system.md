# Content Creation and Curation System

> Sistema para crear, mantener y curar la biblioteca de assets y bloques
> que es el activo central del producto. Pipeline IA-asistida con
> revisión humana selectiva.

**Estado:** Diseño v1.2 (modelo atomic+composite promovido desde
exploración)
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 establece **el contenido como activo central**.
- §2 cubre **boundaries**.
- §3 cubre el **modelo atomic + composite** (key economic principle).
- §4 cubre **anatomía de un asset composite**.
- §5 cubre el **proceso de creación** (pipeline).
- §6 cubre **bloques** (composición de assets).
- §7 cubre **cobertura inicial** y roadmap de biblioteca.
- §8 cubre **schemas Postgres**.
- §9 cubre **API contracts**.
- §10 cubre **mantenimiento** (tracking de performance).
- §11 cubre **stack tecnológico**.
- §12 enumera **edge cases**.
- §13 cubre **eventos**.
- §14 cubre **decisiones cerradas**.

> **Cambio en v1.2:** se promovió el modelo atomic+composite desde
> [`docs/explorations/multimedia-assets.md`](../explorations/multimedia-assets.md).
> Detalles específicos de generación de video/imagen siguen en
> exploración. Ver §3 para principio core.

---

## 1. El contenido como activo central

### 1.1 Por qué importa

El contenido (assets en biblioteca + bloques) es:

- **Activo más valioso del producto:** lo que hace que la app realmente
  enseñe vs solo entretenga.
- **Mayor diferenciador:** un track "Job Ready" para hispanohablantes
  con assets curados culturalmente es difícil de replicar.
- **Moat a largo plazo:** cuanto más contenido y mejor calibrado, más
  difícil que un competidor te alcance.
- **Escala costos a la baja:** generar una vez, usar 1.000.000 de
  veces.

### 1.2 El problema sin sistema

Sin proceso definido:
- Calidad inconsistente.
- Gaps en cobertura (sub-skills sin assets).
- Contenido desactualizado.
- Dependencia de quien creó (bus factor).
- Imposibilidad de escalar.

### 1.3 Filosofía

**Calidad sobre cantidad:** mejor 1.000 assets excelentes que 5.000
mediocres.

**Generación asistida por IA, validación humana:** IA hace 80%, humano
calibra el 20% que importa.

**Iteración basada en datos:** assets que funcionan se mantienen, los
que no, se reemplazan.

**Específico para hispanohablantes:** errores típicos, contexto
cultural, vocabulario familiar.

**Reusabilidad máxima:** componibles, etiquetados ricamente. Ver §3
para el modelo atomic+composite que materializa este principio.

**Más preassets reutilizables = engine más barato.** Operaciones
realtime de IA cuestan dinero por uso; preassets cuestan ~$0 marginal
por delivery. Cada atomic reutilizado N veces divide su costo por N.

---

## 2. Boundaries

### 2.1 Es responsable de

- Crear y mantener `media_atomics` (piezas individuales de media).
- Crear y mantener `characters` (consistencia visual+auditiva
  cross-asset).
- Crear y mantener `learning_assets` (composites: ejercicios que
  combinan atomics + interacción pedagógica).
- Crear y mantener `learning_blocks` (secuencias de composites).
- Pipeline de generación con IA + validation humana.
- Versionado de assets y atomics.
- Tracking de performance + reuse stats (rating, completion rate,
  use_count).
- Mantenimiento (retire de assets viejos, mejora de bajo performance).
- Admin panel para review.

### 2.2 NO es responsable de

- **Selección del asset al user:** eso es `ai-roadmap-system`.
- **Scoring de la performance del user en el asset:** eso es
  `pedagogical-system`.
- **Generar contenido en runtime:** "IA cura, no inventa"; runtime
  consume biblioteca.
- **Pricing de assets premium:** los assets en sí no se venden; los
  Sparks se cobran por las operaciones de IA al usarlos.

### 2.3 Tensiones

| Tensión | Resolución |
|---------|-----------|
| Asset deprecated mientras users lo están haciendo | Mantener versión vieja con `archived = true`; nuevos roadmaps usan nueva |
| Bloque depende de asset deprecado | Bloque también se versiona; nuevo bloque usa nuevo asset |
| Calidad vs velocidad de creación | Calidad siempre. Mejor menos assets buenos. |
| Sub-skill nueva sin assets | Bloquear roadmaps que la requieran hasta que existan |

---

## 3. Modelo atomic + composite

### 3.1 Principio rector

> **Más preassets reutilizables = engine más barato.**

Operaciones realtime de IA cuestan dinero por uso (ver
`sparks-system.md` §5). Preassets cuestan ~$0 marginal por delivery.
Cada pieza de media reutilizada N veces divide su costo por N.

Para materializar este principio, los assets se modelan en dos capas:

| Capa | Qué es | Tabla | Ejemplos |
|------|--------|-------|----------|
| **Atomic** | Pieza individual de media | `media_atomics` | `audio_anita_intro_track_jr_v1`, `video_sarah_introduces_30s`, `img_coffee_shop_interior_v1` |
| **Composite** | Ejercicio pedagógico que usa atomics | `learning_assets` | `roleplay_interview_with_sarah`, `listening_mc_about_coffee_scene` |

Un mismo atomic puede ser usado en múltiples composites:

```
ATOMIC                                       COMPOSITE
─────────────────────────────────────        ──────────────────────────────────────
audio_anita_intro_track_jr_v1          ←─── used in: track intro, daily reminder, ...
video_sarah_introduces_herself_30s     ←─┬── composite_001: listening_mc con 3 Qs
                                          ├── composite_002: free_response sobre lo que dijo
                                          └── composite_003: roleplay responder a Sarah

img_coffee_shop_interior_v1            ←─┬── composite_004: image_description
                                          └── composite_005: roleplay "estás en este café"
```

**Implicación económica:** a 700 composites MVP con reuse factor 2.0
target → solo ~350 atomics realmente creados. **Costo de creación a la
mitad.**

### 3.2 Sistema de labels (en lugar de enum AssetType monolítico)

En vez de un enum hardcoded `AssetType`, los assets composites se
clasifican con **dimensiones independientes**:

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
    | 'mc' | 'qa' | 'drill' | 'shadow' | 'free_response'
    | 'roleplay_structured' | 'roleplay_free'
    | 'description' | 'fill_blank' | 'translation' | 'ordering' | 'dictation';

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

**Ventajas:**
- Agregar `video_dialogue_complete` no toca un enum, solo combinación
  nueva de labels.
- Búsqueda flexible: "todos los videos talking_head de US business B2".
- Validación combinatoria via matrix (algunas combinaciones no tienen
  sentido).

### 3.3 Matrix de validez `media_format × interaction_type`

|  | mc | qa | drill | shadow | free_response | roleplay_s | roleplay_f | description | fill_blank | translation | ordering | dictation |
|--|:--:|:--:|:----:|:------:|:-------------:|:----------:|:----------:|:-----------:|:----------:|:-----------:|:--------:|:---------:|
| text | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ❌ |
| audio | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| image | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| video | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| multi | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |

`multi` = ejercicio que combina más de un media (ej: video + texto +
audio).

### 3.4 Characters (consistencia visual + auditiva cross-asset)

Un `character` es una entidad recurrente cross-assets que mantiene
identidad consistent:

- **Voz fija:** ej. "Sarah la HR" siempre con `voice_us_f_40` del pool.
- **Apariencia visual fija** (cuando hay video/avatar): mismo prompt
  base con seeds determinísticos.
- **Background pedagógico:** ej. Sarah aparece en track Job Ready,
  context Business.

Permite que el usuario forme **memoria episódica del personaje**
(reconoce a Sarah cuando aparece en un nuevo ejercicio del track).
Ver §8.3 para schema.

### 3.5 Reuse stats target

| Métrica | MVP | Fase 2 |
|---------|----:|-------:|
| Reuse rate (avg uses por atomic) | ≥ 1.5 | ≥ 2.5 |
| % de atomics con use_count ≥ 3 | 30% | 50% |
| Generación cost per composite | ≤ $1.50 | ≤ $1.00 |

Cuando un atomic tiene `use_count = 0` después de 90 días: candidato a
archivar. Cron mensual flagea.

### 3.6 Detalles específicos en exploración

Los detalles concretos de **generación de video, imagen y audio TTS**
(qué proveedores, prompts templates, costos por dominio) están en
[`docs/explorations/multimedia-assets.md`](../explorations/multimedia-assets.md).

Esta sección establece el **modelo de datos y el principio**. La
implementación específica de cada pipeline de media puede iterarse en
exploración hasta que se promueva.

---

## 4. Anatomía de un asset

### 4.1 Componentes

```typescript
interface LearningAsset {
  // Identificación
  id: string;
  version: string;                  // semver: 'major.minor.patch'

  // Clasificación
  type: AssetType;
  cefr_level: 'A2' | 'B1' | 'B1+' | 'B2' | 'B2+' | 'C1' | 'C2';
  difficulty: number;                // 1-10
  estimated_minutes: number;

  // Pedagogía
  target_subskills: string[];        // de subskills_catalog
  prerequisites: string[];           // assets que deben completarse antes
  learning_objective: string;        // qué se aprende exactamente

  // Contexto
  topic: string[];                   // ['business', 'interview']
  context_tags: string[];            // ['formal', 'remote_work']
  english_variant: 'american' | 'british' | 'neutral';
  cultural_relevance: string[];      // países donde es especialmente relevante

  // Contenido (varía según type)
  content: AssetContent;

  // Metadata
  created_by: 'ai_generated' | 'human' | 'ai_human_curated';
  created_at: string;
  reviewed_by?: string;
  reviewed_at?: string;
  approved_for_production: boolean;

  // Performance tracking
  usage_count: number;
  avg_completion_rate: number;
  avg_user_rating?: number;
  flagged_issues: number;
}
```

### 4.2 Tipos de assets

(14 tipos, alineados con `pedagogical-system.md` §5.1.)

#### Pronunciation Drill

```typescript
interface PronunciationDrillContent {
  target_phonemes: string[];         // ['/θ/', '/ð/']
  phrases: Array<{
    text: string;
    audio_reference_url: string;     // audio del native speaker
    target_words: string[];          // palabras críticas
    expected_phonetic_focus: string;
  }>;
  feedback_templates: {
    common_errors: Record<string, string>;
  };
}
```

#### Roleplay Structured

```typescript
interface RoleplayContent {
  scenario: {
    title: string;
    description: string;
    context: string;
    user_role: string;
    ai_role: string;
  };
  conversation_flow: Array<{
    step: number;
    ai_message: string;
    expected_user_response_topics: string[];
    acceptable_response_examples: string[];
    branching_logic: BranchingRule[];
  }>;
  vocabulary_focus: string[];
  grammar_focus: string[];
  cultural_notes: string;
}
```

#### Listening Exercise

```typescript
interface ListeningContent {
  audio_url: string;
  audio_duration_seconds: number;
  transcript: string;
  speaker_accent: string;            // 'us-general', 'uk-rp', 'us-southern'
  speaker_speed: 'slow' | 'natural' | 'fast';
  questions: Array<{
    type: 'multiple_choice' | 'open' | 'true_false';
    question: string;
    correct_answer: string;
    options?: string[];
    explanation: string;
  }>;
}
```

#### Free Response

```typescript
interface FreeResponseContent {
  prompt: string;
  context: string;
  expected_duration_seconds: number;
  evaluation_criteria: {
    must_include_topics: string[];
    target_grammar_structures: string[];
    target_vocabulary_level: string;
  };
  example_answers: string[];
}
```

(Tipos restantes: shadowing, read_aloud, roleplay_free, listening_qa,
translation, fill_blank, sentence_ordering, image_description,
dictation, vocabulary_in_context — schema análogo.)

### 4.3 Estándares de calidad

#### Audios

- Native speaker o equivalente (TTS premium calibrado).
- Sin ruido de fondo perceptible (SNR > 30dB).
- Volumen normalizado a -16 LUFS.
- Duración apropiada al ejercicio.

#### Texto

- Gramática y ortografía perfectas.
- Lenguaje natural, no robótico.
- Apropiado al nivel CEFR target.
- Sin estereotipos culturales.

#### Roleplays

- Diálogos naturales, no académicos.
- Branching realista.
- Cubre objetivo pedagógico claramente.
- Termina satisfactoriamente.

---

## 5. Proceso de creación

### 5.1 Pipeline (v1.2: 2 capas, atomic + composite)

```
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 1: PLANIFICACIÓN                                     │
│  - Identificar gap en biblioteca                            │
│  - Definir learning objective específico                    │
│  - Especificar labels (media_format, interaction_type, ...) │
│  - Decidir qué atomics necesita (existentes vs nuevos)      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 2: ATOMICS (REUSE FIRST)                             │
│  - searchAtomics(): buscar atomics existentes que matchen   │
│  - Para los que faltan: enqueueAtomicGeneration()           │
│    • Audio: AI Gateway task generate_audio_tts              │
│    • Image: AI Gateway task generate_image                  │
│    • Video: AI Gateway tasks generate_video_*               │
│  - Validation post-generation por atomic                    │
│  - Persist en media_atomics con approved=false              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 3: COMPOSICIÓN                                       │
│  - createAsset() con references a atomics                   │
│  - Definir interaction_data (preguntas, prompts, flow)      │
│  - Validar combinación de labels (matrix §3.3)              │
│  - Validation pedagógica (CEFR, sub-skills cubiertas)       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 4: REVISIÓN HUMANA (sampling)                        │
│  - Atomics: 50% review al inicio, 10% en steady state       │
│  - Composites: 30% review al inicio, 10% en steady state    │
│  - Feedback documentado para mejorar prompts                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 5: A/B TESTING EN PRODUCCIÓN                         │
│  - Composite se libera a 5-10% de usuarios primero          │
│  - Métricas: completion rate, dificultad percibida, ratings │
│  - Si performa bien, expansión a 100%                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 6: MANTENIMIENTO                                     │
│  - Tracking continuo de performance (composite + atomics)   │
│  - Actualización si métricas caen                           │
│  - Retiro de atomics no usados (use_count = 0 por 90 días)  │
│  - Retiro de composites obsoletos                           │
└─────────────────────────────────────────────────────────────┘
```

**Reuse first:** antes de generar atomic nuevo, **siempre buscar en
`media_atomics` existentes**. Match por `media_format + media_subtype +
voice_id + generation_prompt` (audio) o equivalente para image/video.
Si match: reuse. Si no: generate.

### 5.2 Generación con IA

Llamada a AI Gateway tasks:
- `generate_asset_content` (texto del asset).
- `generate_asset_audio` (TTS con ElevenLabs).
- `validate_asset_cefr` (Haiku verifica nivel).

#### Ejemplo de prompt para roleplay

```
Sos un experto en didáctica del inglés especializado en preparación
laboral para hispanohablantes latinoamericanos.

OBJETIVO PEDAGÓGICO:
Generar un roleplay donde el user practica responder preguntas
behavioral en una entrevista, usando el método STAR.

PARÁMETROS:
- Nivel CEFR: B2
- Duración estimada: 8-10 minutos
- Sub-skills target: vocabulary_business, fluency_extended_speech,
  grammar_past_tenses
- Variant inglés: americano
- Contexto cultural: user hispanoamericano aplicando a empresa
  internacional

RESTRICCIONES:
- Vocabulario apropiado a B2 (sin C1 unnecessarily complex)
- Diálogos naturales, no formales en exceso
- Considerar que user puede estar nervioso (primer step warm-up)
- Mínimo 4 ramificaciones para que se sienta real

ESTRUCTURA REQUERIDA:
1. Scenario setup (descripción al user).
2. AI role (interviewer): tipo de empresa, rol que busca.
3. Conversation flow:
   - Step 1: greeting + warm-up question.
   - Step 2: behavioral question 1 (challenge).
   - Step 3: follow-up basado en respuesta.
   - Step 4: behavioral question 2 (teamwork).
   - Step 5: cierre y next steps.
4. Para cada step:
   - AI message (inglés natural).
   - Topics que user debe cubrir.
   - 3 ejemplos de respuestas válidas en B2.
   - Branching logic según calidad.
5. Vocabulary focus: 8-10 palabras/expresiones target.
6. Grammar focus: estructuras a practicar.
7. Cultural notes: tips para hispanohablantes (ej: "evitar modesty
   excesiva", "STAR method valued in US interviews").

DEVOLVÉ JSON exacto siguiendo el schema RoleplayContent.
```

### 5.3 Validación automática

```typescript
async function validateAsset(asset: LearningAsset): Promise<ValidationResult> {
  const checks = [];

  // Estructura
  checks.push(validateSchema(asset));

  // Nivel CEFR del texto
  const measuredCEFR = await measureTextCEFR(extractAllText(asset));
  checks.push({
    name: 'cefr_match',
    passed: matchesCEFR(measuredCEFR, asset.cefr_level),
  });

  // Vocabulario apropiado
  const vocab = analyzeVocabulary(extractAllText(asset));
  checks.push({
    name: 'vocab_distribution',
    passed: vocabAppropriateForLevel(vocab, asset.cefr_level),
  });

  // Sub-skills realmente abordadas
  const detectedSubskills = await detectSubskillsAddressed(asset);
  checks.push({
    name: 'subskills_coverage',
    passed: asset.target_subskills.every(s => detectedSubskills.includes(s)),
  });

  // Audios
  if (asset.content.audio_url) {
    checks.push(await validateAudioQuality(asset.content.audio_url));
  }

  // Duración estimada vs contenido
  checks.push(validateDuration(asset));

  return {
    passed: checks.every(c => c.passed),
    checks,
    needsHumanReview: checks.some(c => !c.passed) || sampleForReview(),
  };
}
```

### 5.4 Revisión humana

#### Quién revisa

**Fase 1 (MVP):** vos (owner) o freelancers nativos hispanohablantes
con C1+ inglés.

**Fase 2 (con tracción):** equipo de 1-2 content reviewers part-time.

**Fase 3 (escala):** equipo dedicado + community contributions
moderadas.

#### Qué se revisa

Checklist:
- [ ] Audio suena natural y claro.
- [ ] Diálogo se siente realista, no robótico.
- [ ] Nivel CEFR es apropiado.
- [ ] No hay errores gramaticales o ortografía.
- [ ] Contenido culturalmente apropiado.
- [ ] No hay estereotipos.
- [ ] Objetivo pedagógico claramente alcanzable.
- [ ] Dificultad estimada razonable.
- [ ] Tiempo estimado preciso.

#### Sample size

- Primeros 200 assets: 100% revisión humana.
- Assets 201-500: 50% revisión.
- Assets 501-2.000: 20% revisión.
- Assets 2.000+: 10% revisión.

A medida que confianza crece, sample baja.

#### Documentar feedback

Cada rejection o ajuste se documenta:

```
Asset rejected: roleplay_interview_b2_011
Reason: AI interviewer asks question too advanced for B2
Fix: ajustar prompt agregando "questions should not require C1
vocabulary"
```

Feedback se incorpora a versiones futuras del prompt.

---

## 6. Bloques (composiciones)

### 6.1 Estructura

Un bloque (`learning_block`) compone múltiples assets:

```typescript
interface LearningBlock {
  id: string;
  version: string;
  title: string;
  description: string;
  cefr_level: string;
  context_tags: string[];
  target_subskills: string[];
  prerequisites: string[];           // otros block IDs
  estimated_minutes: number;

  // Assets que componen el bloque (en orden)
  asset_sequence: Array<{
    asset_id: string;
    order: number;
    optional: boolean;                // false = obligatorio
  }>;

  // Mastery criteria (consumido por pedagogical)
  mastery_criteria: BlockMasteryCriteria;

  archived: boolean;
  approved_for_production: boolean;
  created_at: string;
  updated_at: string;
}
```

### 6.2 Composición típica

3-5 assets de tipos variados que abordan misma sub-skill desde
ángulos diferentes.

Ejemplo `block_past_perfect_storytelling`:
1. `listening_mc_past_perfect_intro` (3 min): identificar uso correcto.
2. `fill_blank_past_perfect_5_phrases` (2 min): completar frases.
3. `free_response_personal_story_past_perfect` (3 min): contar
   experiencia.
4. `roleplay_structured_past_perfect_chat` (5 min): conversación.

Total estimado: ~13 min.

---

## 7. Cobertura inicial y roadmap

### 7.1 MVP target

| Track | Niveles | Bloques/nivel | Total bloques | Assets/bloque | Total assets |
|-------|--------:|--------------:|--------------:|--------------:|-------------:|
| Job Ready | 6 | 12 | 72 | 4 | 288 |
| Travel Confident | 4 | 10 | 40 | 4 | 160 |
| Daily Conversation | 5 | 12 | 60 | 4 | 240 |
| **Total MVP** | | | **172** | | **688** |

**~700 assets de calidad para MVP.** A 30 min de generación + revisión
por asset, son ~350 horas de trabajo.

#### Distribución por tipo

- ~150 pronunciation drills (simples, generación rápida).
- ~200 roleplays (más complejos, más tiempo).
- ~150 listening exercises (audios curados).
- ~100 free response prompts (estructurados simples).
- ~100 vocabulary in context (estructurados).

### 7.2 Roadmap de expansión

| Mes | Nuevos assets | Nuevos tracks |
|-----|--------------:|---------------|
| 0–3 | 700 (MVP) | Job Ready, Travel, Daily |
| 3–6 | +500 | Business English |
| 6–9 | +500 | Tech Professional, Academic |
| 9–12 | +500 | Customer Service, Healthcare |
| 12+ | +200/mes ongoing | Especializados según demanda |

A 24 meses: ~3.500 assets cubriendo 8+ tracks.

### 7.3 Cobertura de sub-skills

Cada sub-skill debe tener mínimo:
- 5 assets en CEFR B1.
- 5 assets en CEFR B2.
- 3 assets en CEFR C1.

50 sub-skills core × ~13 assets promedio = 650 assets solo de
cobertura mínima. Solapa con tracks pero da baseline.

---

## 8. Schemas Postgres

### 8.1 Tabla `media_atomics` (NUEVA en v1.2)

Pieza individual de media (audio, imagen, video) reutilizable
cross-assets.

```sql
CREATE TABLE media_atomics (
  id              TEXT PRIMARY KEY,             -- 'audio_anita_intro_jr_v1'
  version         TEXT NOT NULL DEFAULT '1.0.0',

  -- Tipo
  media_format    TEXT NOT NULL CHECK (media_format IN (
                    'audio', 'image', 'video'
                  )),
  media_subtype   TEXT NOT NULL,
  -- audio: 'tts', 'cloned_host', 'native_recording'
  -- image: 'illustration', 'photo', 'avatar', 'infographic'
  -- video: 'talking_head', 'scene', 'animation', 'screen_recording'

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
  has_audio       BOOLEAN DEFAULT true,
  character_id    TEXT REFERENCES characters(id) DEFERRABLE INITIALLY DEFERRED,

  -- Image specific
  is_text_in_image BOOLEAN DEFAULT false,       -- OCR result; debe ser false para image_description

  -- Generation metadata
  generated_by    TEXT NOT NULL,                -- 'ai_dalle3', 'ai_hedra', 'ai_elevenlabs', 'manual', 'lottie_template'
  generation_cost_usd NUMERIC(10,4),
  generation_seed INT,
  generation_prompt TEXT,

  -- Lifecycle
  approved        BOOLEAN NOT NULL DEFAULT false,
  archived        BOOLEAN NOT NULL DEFAULT false,
  archived_at     TIMESTAMPTZ,
  archive_reason  TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  reviewed_at     TIMESTAMPTZ,
  reviewed_by     TEXT,

  -- Reuse tracking (denormalized counter, recalculado por cron)
  use_count       INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_atomics_format_subtype
  ON media_atomics(media_format, media_subtype)
  WHERE approved = true AND archived = false;
CREATE INDEX idx_atomics_voice ON media_atomics(voice_id)
  WHERE media_format = 'audio' AND voice_id IS NOT NULL;
CREATE INDEX idx_atomics_character ON media_atomics(character_id)
  WHERE character_id IS NOT NULL;
CREATE INDEX idx_atomics_high_reuse ON media_atomics(use_count DESC)
  WHERE approved = true;
CREATE INDEX idx_atomics_unused
  ON media_atomics(created_at)
  WHERE use_count = 0 AND approved = true;
```

### 8.2 Tabla `characters` (NUEVA en v1.2)

Entidad recurrente cross-assets (Sarah la HR, Mike el barista) con
identidad consistent.

```sql
CREATE TABLE characters (
  id              TEXT PRIMARY KEY,             -- 'character_sarah_hr'
  name            TEXT NOT NULL,                -- 'Sarah'
  description     TEXT NOT NULL,                -- prompt fijo para imagen
  voice_id        TEXT NOT NULL,                -- voz de pool del producto
  avatar_atomic_id TEXT REFERENCES media_atomics(id),
  default_video_avatar_id TEXT,                 -- HeyGen/Hedra avatar id

  -- Persona
  age_range       TEXT,                         -- '25-35'
  occupation      TEXT,                         -- 'HR Manager'
  cultural_background TEXT,                     -- 'us_general', 'us_latino_heritage'

  -- Tracks donde aparece
  appears_in_tracks TEXT[] DEFAULT '{}',

  archived        BOOLEAN NOT NULL DEFAULT false,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_characters_active ON characters(id) WHERE archived = false;
```

### 8.3 Tabla `atomic_generation_queue` (NUEVA en v1.2)

Queue para batch generation nocturna de atomics nuevos.

```sql
CREATE TABLE atomic_generation_queue (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  requested_for_asset_id TEXT,                  -- composite que necesita el atomic
  media_format    TEXT NOT NULL,
  media_subtype   TEXT NOT NULL,
  prompt          TEXT NOT NULL,
  config          JSONB NOT NULL,               -- voice_id, style, seed, etc.
  priority        INT NOT NULL DEFAULT 5,       -- 1=critical, 10=low
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending', 'generating', 'completed', 'failed')),
  result_atomic_id TEXT REFERENCES media_atomics(id),
  attempts        INT NOT NULL DEFAULT 0,
  last_error      TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  started_at      TIMESTAMPTZ,
  completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_atomic_queue_pending
  ON atomic_generation_queue(priority, created_at)
  WHERE status = 'pending';
CREATE INDEX idx_atomic_queue_failed
  ON atomic_generation_queue(created_at DESC)
  WHERE status = 'failed';
```

### 8.4 Tabla `learning_assets` (modificada en v1.2)

Composites: ejercicios pedagógicos que referencian atomics.

```sql
CREATE TABLE learning_assets (
  id              TEXT PRIMARY KEY,
  version         TEXT NOT NULL DEFAULT '1.0.0',

  -- Labels (v1.2: en lugar de enum 'type' monolítico)
  primary_media_format TEXT NOT NULL CHECK (primary_media_format IN (
                    'text', 'audio', 'image', 'video', 'multi'
                  )),
  primary_media_subtype TEXT,
  interaction_type TEXT NOT NULL CHECK (interaction_type IN (
                    'mc', 'qa', 'drill', 'shadow', 'free_response',
                    'roleplay_structured', 'roleplay_free',
                    'description', 'fill_blank', 'translation',
                    'ordering', 'dictation'
                  )),

  -- DEPRECATED en v1.2 pero se mantiene para compatibilidad migration
  -- type            TEXT,

  -- Clasificación
  cefr_level      TEXT NOT NULL,
  difficulty      INT NOT NULL CHECK (difficulty BETWEEN 1 AND 10),
  estimated_minutes INT NOT NULL,

  -- Pedagogía
  target_subskills TEXT[] NOT NULL,
  prerequisites   TEXT[] DEFAULT '{}',
  learning_objective TEXT NOT NULL,

  -- Contexto
  topic           TEXT[] NOT NULL,
  context_tags    TEXT[] DEFAULT '{}',
  english_variant TEXT NOT NULL DEFAULT 'neutral'
                  CHECK (english_variant IN ('us', 'uk', 'neutral')),
  cultural_relevance TEXT[] DEFAULT '{}',

  -- Atomics referenced (v1.2: reemplaza media inline en content)
  -- Shape:
  -- [
  --   { "atomic_id": "audio_anita_intro_jr_v1", "role": "main_audio" },
  --   { "atomic_id": "img_coffee_shop_v1",     "role": "background_image" },
  --   { "atomic_id": "video_sarah_intro_30s",  "role": "main_video",
  --     "position": { "start_ms": 0, "end_ms": 30000 } }
  -- ]
  media_atomics   JSONB NOT NULL DEFAULT '[]',

  -- Interaction-specific data (no media; solo preguntas, prompts, flow)
  -- Shape varía por interaction_type:
  -- mc: { questions: [{q, options, correct}] }
  -- qa: { questions: [{q, expected_topics}] }
  -- roleplay_structured: { flow: [...] }
  -- free_response: { prompt, expected_topics }
  -- ...
  interaction_data JSONB NOT NULL DEFAULT '{}',

  -- Metadata
  created_by      TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  reviewed_by     TEXT,
  reviewed_at     TIMESTAMPTZ,
  approved        BOOLEAN NOT NULL DEFAULT false,
  archived        BOOLEAN NOT NULL DEFAULT false,
  archived_at     TIMESTAMPTZ,
  archive_reason  TEXT,

  -- Performance (denormalized counters)
  usage_count     INT NOT NULL DEFAULT 0,
  unique_users    INT NOT NULL DEFAULT 0,
  avg_completion_rate FLOAT,
  avg_score       FLOAT,
  flagged_count   INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_assets_subskills
  ON learning_assets USING gin(target_subskills)
  WHERE approved = true AND archived = false;
CREATE INDEX idx_assets_topic
  ON learning_assets USING gin(topic)
  WHERE approved = true AND archived = false;
CREATE INDEX idx_assets_cefr_active
  ON learning_assets(cefr_level)
  WHERE approved = true AND archived = false;
```

### 8.5 Tabla `learning_blocks`

```sql
CREATE TABLE learning_blocks (
  id              TEXT PRIMARY KEY,
  version         TEXT NOT NULL DEFAULT '1.0.0',
  title           TEXT NOT NULL,
  description     TEXT,
  cefr_level      TEXT NOT NULL,
  context_tags    TEXT[] DEFAULT '{}',
  target_subskills TEXT[] NOT NULL,
  prerequisites   TEXT[] DEFAULT '{}',
  estimated_minutes INT NOT NULL,
  asset_sequence  JSONB NOT NULL,            -- ordered list
  mastery_criteria JSONB NOT NULL,
  archived        BOOLEAN NOT NULL DEFAULT false,
  approved        BOOLEAN NOT NULL DEFAULT false,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_blocks_subskills
  ON learning_blocks USING gin(target_subskills)
  WHERE approved = true AND archived = false;
```

### 8.6 Tabla de versiones

```sql
CREATE TABLE asset_versions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  asset_id        TEXT NOT NULL REFERENCES learning_assets(id),
  version         TEXT NOT NULL,
  content         JSONB NOT NULL,
  changed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  changed_by      TEXT,
  change_reason   TEXT
);

CREATE INDEX idx_asset_versions_asset
  ON asset_versions(asset_id, changed_at DESC);
```

Análogo `block_versions`.

### 8.7 Tabla de feedback

```sql
CREATE TABLE asset_feedback (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  asset_id        TEXT NOT NULL REFERENCES learning_assets(id),
  user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
  rating          INT CHECK (rating BETWEEN 1 AND 5),
  flag_reason     TEXT,                       -- 'inappropriate', 'too_hard', 'broken_audio'
  comment         TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_feedback_asset
  ON asset_feedback(asset_id, created_at DESC);
```

### 8.8 Performance tracking

Job nocturno actualiza counters denormalizados en `learning_assets`
desde `exercise_attempts`:

```sql
UPDATE learning_assets la
SET usage_count = stats.uc,
    unique_users = stats.uu,
    avg_completion_rate = stats.acr,
    avg_score = stats.as
FROM (
  SELECT
    ea.exercise_id as asset_id,
    COUNT(*) as uc,
    COUNT(DISTINCT user_id) as uu,
    AVG(CASE WHEN abandoned = false THEN 1 ELSE 0 END) as acr,
    AVG((scores->>'overall')::numeric) as as
  FROM exercise_attempts ea
  WHERE ea.completed_at > now() - interval '30 days'
  GROUP BY ea.exercise_id
) stats
WHERE la.id = stats.asset_id;
```

---

## 9. API contracts

### 9.1 `getAsset`

**Llamado por:** cliente cuando carga un asset.

```typescript
interface GetAssetRequest {
  asset_id: string;
  user_country?: string;             // para resolver annotations al país (Fase 2)
}

interface GetAssetResponse {
  asset: LearningAsset;
  resolved_atomics: Array<{
    atomic_id: string;
    role: string;
    media_format: 'audio' | 'image' | 'video';
    signed_url: string;              // R2 signed URL, TTL 5 min
    duration_ms?: number;
    width?: number;
    height?: number;
    mime_type: string;
  }>;
}
```

**Reglas v1.2:**
- Solo retorna assets con `approved = true` y `archived = false`
  (excepto si user lo está completando: snapshot version).
- Resuelve cada `media_atomics` reference a signed URL con TTL 5 min.
- Si algún atomic está archived: rechazar el asset (es bug del
  composite).
- Cliente cachea signed URLs ~4 min y refetcha si expira.

### 9.2 `getAtomic` (NUEVA en v1.2)

**Llamado por:** cliente cuando necesita re-fetch un signed URL
expirado, o admin para review individual.

```typescript
interface GetAtomicRequest {
  atomic_id: string;
}

interface GetAtomicResponse {
  atomic: MediaAtomic;
  signed_url: string;                // TTL 5 min
}
```

### 9.3 `searchAtomics` (NUEVA en v1.2, admin only)

Permite buscar atomics existentes antes de generar uno nuevo (reuse
first).

```typescript
interface SearchAtomicsRequest {
  filters: {
    media_format?: 'audio' | 'image' | 'video';
    media_subtype?: string;
    voice_id?: string;
    character_id?: string;
    english_variant?: string;
    generation_prompt_substring?: string;
    approved?: boolean;
    archived?: boolean;
    min_use_count?: number;
  };
  limit?: number;
  offset?: number;
  order_by?: 'use_count_desc' | 'created_at_desc' | 'use_count_asc';
}

interface SearchAtomicsResponse {
  atomics: MediaAtomic[];
  total: number;
}
```

### 9.4 `createAtomicFromGeneration` (NUEVA en v1.2, internal)

**Llamado por:** pipeline de generación al completar atomic batch.

```typescript
interface CreateAtomicRequest {
  draft: Partial<MediaAtomic>;
  storage_key: string;               // ya subido a R2
  generation_metadata: {
    generated_by: string;
    cost_usd: number;
    seed?: number;
    prompt: string;
  };
}

interface CreateAtomicResponse {
  atomic_id: string;
  validation_result: AtomicValidationResult;
  needs_human_review: boolean;
}
```

**Validation incluye:**
- Audio: duration > 1s, file size razonable, sample rate válido.
- Image: resolution mínima, aspect ratio target, OCR check
  (`is_text_in_image`).
- Video: codec H.264, duration en rango (5-60s), bitrate.

### 9.5 `approveAtomic` (NUEVA en v1.2, admin)

```typescript
interface ApproveAtomicRequest {
  atomic_id: string;
  reviewed_by: string;
  notes?: string;
}
```

Marca `approved = true` y `reviewed_at`.

### 9.6 `archiveAtomic` (NUEVA en v1.2)

```typescript
interface ArchiveAtomicRequest {
  atomic_id: string;
  reason: string;
  replaced_by_atomic_id?: string;    // si se reemplaza por nueva versión
}
```

**Reglas:**
- Si `use_count > 0`: warn pero permitir; assets que lo usan deben
  migrar a `replaced_by_atomic_id` o ser archivados también.
- Cron mensual flagea atomics con `use_count = 0` durante 90 días para
  archivado automático.

### 9.7 `enqueueAtomicGeneration` (NUEVA en v1.2, internal)

Para batch generation nocturna:

```typescript
interface EnqueueAtomicGenerationRequest {
  media_format: 'audio' | 'image' | 'video';
  media_subtype: string;
  prompt: string;
  config: Record<string, unknown>;   // voice_id, style, seed, etc.
  priority?: number;                 // 1-10
  requested_for_asset_id?: string;
}
```

Inserta row en `atomic_generation_queue` con `status = 'pending'`.

### 9.8 `getCharacter` (NUEVA en v1.2)

```typescript
interface GetCharacterRequest {
  character_id: string;
}

interface GetCharacterResponse {
  character: Character;
  signed_avatar_url?: string;        // si tiene avatar_atomic_id
}
```

### 9.9 `getBlock`

**Llamado por:** cliente cuando carga un bloque.

```typescript
interface GetBlockRequest {
  block_id: string;
}

interface GetBlockResponse {
  block: LearningBlock;
  asset_summaries: Array<{
    asset_id: string;
    title: string;
    type: string;
    estimated_minutes: number;
  }>;
}
```

### 9.10 `getAssetLegacy`

(Preservado para retro-compat durante migration v1.1 → v1.2.)

```typescript
interface GetAssetLegacyRequest {
  asset_id: string;
}

interface GetAssetLegacyResponse {
  asset: LearningAsset;
  signed_audio_urls?: Record<string, string>;
}
```

**Reglas:**
- Solo retorna assets con `approved = true` y `archived = false`.
- Audios de R2 vía signed URLs con TTL 5 min.

### 9.11 `searchAssets` (admin)

```typescript
interface SearchAssetsRequest {
  filters: {
    cefr_level?: string;
    subskill?: string;
    topic?: string;
    type?: string;
    approved?: boolean;
    archived?: boolean;
  };
  limit?: number;
  offset?: number;
}

interface SearchAssetsResponse {
  assets: LearningAsset[];
  total: number;
}
```

### 9.12 `submitAssetFeedback`

**Llamado por:** cliente si user reporta problema con asset.

```typescript
interface SubmitAssetFeedbackRequest {
  user_id: string;
  asset_id: string;
  rating?: number;                  // 1-5
  flag_reason?: 'inappropriate' | 'too_hard' | 'too_easy' | 'broken_audio' | 'other';
  comment?: string;
}
```

### 9.13 Internal: `createAsset` (admin/pipeline)

```typescript
interface CreateAssetRequest {
  draft: Partial<LearningAsset>;
  generated_by: 'ai_generated' | 'human' | 'ai_human_curated';
}

interface CreateAssetResponse {
  asset_id: string;
  validation_result: ValidationResult;
  needs_human_review: boolean;
}
```

### 9.14 Internal: `approveAsset` (admin)

```typescript
interface ApproveAssetRequest {
  asset_id: string;
  reviewed_by: string;
  notes?: string;
}
```

### 9.15 Internal: `archiveAsset`

```typescript
interface ArchiveAssetRequest {
  asset_id: string;
  reason: string;                   // 'low_performance', 'outdated', 'replaced_by_X'
  replaced_by?: string;
}
```

**Reglas:**
- No borra. Marca `archived = true`.
- Después de 3 meses sin uso, cleanup script puede borrar.
- Bloques que dependen del asset se actualizan a versión nueva o se
  archivan también.

---

## 10. Mantenimiento

### 10.1 Reglas automáticas

| Condición | Acción |
|-----------|--------|
| Asset performing well | Mantener; posible reference para crear similares |
| Completion rate < 50% | Revisar manualmente; demasiado difícil/confuso |
| `avg_score < 30%` del target | Difícil para nivel declarado; re-evaluar nivel o reescribir |
| `retry_rate > 60%` | Muy difícil o muy adictivo; investigar |
| `flagged_count > 5%` de uses | Revisión inmediata; posible problema de calidad |
| `uses_last_30d == 0` | Considerar retirar si no es relevante |

### 10.2 Versionado de assets

Cuando un asset se actualiza:
- Mantiene versión vieja para users que ya lo iniciaron.
- Nuevos users ven versión nueva.
- Trackea performance de ambas versiones.
- Versión vieja se retira cuando ningún user activo la está usando.

### 10.3 Retirement gradual

```
Asset obsoleto detectado
  ↓
Marcar archived = true (no aparece en nuevos roadmaps)
  ↓
Users actuales pueden completar (snapshot)
  ↓
Después de 3 meses sin uso: borrar físicamente
```

### 10.4 A/B testing de assets

Cuando se crea v2 de un asset existente:
1. v2 se libera al 5% de users.
2. Comparar métricas: completion, score, rating.
3. Si v2 mejora: ramp up 25% → 50% → 100%.
4. Si peor: rollback (v1 sigue).

---

## 11. Stack tecnológico

### 11.1 Para generación

| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| LLM para texto | AI Gateway tasks `generate_asset_*` (Claude Opus / GPT-4o) | Roleplays, prompts |
| TTS premium | ElevenLabs (multiple voices) | Audios listening, ejemplos |
| Voice cloning | ElevenLabs voice library | Variedad de acentos |
| Audio editing | FFmpeg pipelines automatizadas | Normalización, trim |
| Validation LLM | AI Gateway task `validate_asset_cefr` (Haiku) | Validación post-generación |

### 11.2 Para curaduría

| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| Admin panel | Next.js + Tailwind | UI para review |
| Storage | Cloudflare R2 | Audios y archivos generados |
| Metadata | Postgres | Toda la metadata |
| Search | Postgres full-text + pgvector | Búsqueda en biblioteca |
| Diff/version control | Custom sobre Postgres | Tracking de cambios |

### 11.3 Para tracking

| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| Analytics | PostHog | Métricas de uso por asset |
| Feedback | Custom + API | Flagging y ratings |
| Dashboard | Internal admin | Vista por asset y agregada |

### 11.4 Costos de creación

**Estimado para 700 assets MVP:**

- LLM generación (Opus): ~$30 por 100 roleplays = $200 total.
- TTS premium (ElevenLabs): ~$0.30 por audio × 600 = $180.
- Validación (Haiku): ~$0.01 × 700 = $7.
- Tiempo humano: ~150h × $20/h freelance = $3.000.
- **Total: ~$3.500 USD para biblioteca MVP.**

Costo marginal por asset adicional: ~$5 USD (dominado por tiempo
humano).

---

## 12. Edge cases (tests obligatorios)

### 12.1 Generación

1. **LLM devuelve roleplay con C1 vocab para B2:** validation falla
   `cefr_match`; retry con prompt más estricto.
2. **TTS genera audio con voz robótica:** validation `audio_quality`
   detecta SNR/naturalness; rechaza, retry con otra voz.
3. **Roleplay generado tiene branching loop infinito:** validation
   detecta ciclo, rechaza.

### 12.2 Versioning

4. **Asset v1 in-use por user, admin sube v2:** user completa v1
   (snapshot). Próximos users ven v2.
5. **Asset v2 reemplaza v1 mientras user en medio del ejercicio:** UI
   advierte "este ejercicio se actualizó" pero permite completar v1
   sin interrupción.

### 12.3 Archivado

6. **Asset archived que sigue en bloques activos:** bloque también se
   marca para review; admin decide remover/reemplazar.
7. **Asset archived y user lo intenta cargar:** retorna versión
   archivada con flag `is_archived = true`. Cliente muestra advertencia
   pero permite (snapshot).

### 12.4 Performance

8. **Asset nuevo con 0 uses tiene NULL en avg_score:** UI no muestra
   "0%" sino "Datos insuficientes". Algoritmos de selección lo tratan
   como neutral.
9. **Asset con 100% completion pero rating bajo:** flag para review
   manual. Posible asset demasiado fácil pero aburrido.

### 12.5 Concurrencia

10. **Dos admins intentan editar mismo asset:** optimistic locking via
    `updated_at` en WHERE clause. Segundo recibe error "asset cambió,
    refrescá".

### 12.6 Feedback

11. **User flag asset 5 veces:** rate limit max 1 flag por user por
    asset por 24h. Repeated flags se ignoran.
12. **Spike de flags en asset (>20 en 1h):** alerta automática para
    review humano inmediato.

---

## 13. Eventos

### 13.1 Emitidos

| Evento | Cuándo |
|--------|--------|
| `asset.created` | Asset draft creado |
| `asset.approved` | Admin aprobó para producción |
| `asset.archived` | Asset retirado |
| `asset.flagged` | User flageó asset |
| `asset.performance_alert` | Métricas caen below threshold |
| `block.created` | Bloque nuevo creado |
| `block.archived` | Bloque retirado |

### 13.2 Consumidos

| Evento | Acción |
|--------|--------|
| `exercise.attempt_completed` (pedagogical) | Update performance counters via job nocturno |

---

## 14. Decisiones cerradas

### 14.1 Crowdsource de assets de la comunidad: **NO en MVP** ✓

**Razón:** calidad variable + moderación intensiva. Reconsiderar año
2 con community manager dedicado.

### 14.2 Licenciar contenido existente de proveedores: **NO en MVP** ✓

**Razón:** dificultad de licensing + control reducido. Crear propio
desde día 1.

### 14.3 Generar assets en runtime para users con perfiles únicos:
**NO** ✓

**Razón:** principio rector "IA cura, no inventa". Costos altos +
calidad variable + imposible validar pedagógicamente. Ese path se
rechaza permanentemente.

### 14.4 Cómo balancear velocidad de creación con calidad: **calidad
siempre** ✓

**Razón:** mejor menos assets buenos. Reputación del producto depende
de calidad consistente. Velocidad es problema solucionable después.

### 14.5 Versionado: **semver con releases major.minor only** ✓

**Razón:** patches no agregan valor para assets (no son código).
Major: breaking change pedagógico. Minor: mejora notable. Cualquier
cambio bumpea version.

---

## 15. Plan de implementación

### 15.1 Fase 0: Pre-MVP (mes -1)

- Schema completo en Postgres.
- Admin panel básico para review.
- Pipeline de generación con prompts iniciales.
- Pipeline de validation automática.
- Storage en R2.
- Templates por tipo.

### 15.2 Fase 1: Creación inicial (meses 0–3)

- Mes 0-1: 250 assets (Job Ready completo).
- Mes 1-2: 250 assets (Travel + Daily).
- Mes 2-3: 200 assets de refinamiento + cobertura sub-skills.

### 15.3 Fase 2: Iteración (meses 3–6)

- Tracking de performance.
- Re-trabajar 10-20% peores performers.
- Crear assets nuevos en gaps.
- Expandir biblioteca a 1.200 assets.

### 15.4 Fase 3: Escala (meses 6–12)

- 2-3 nuevos tracks.
- 2.500 assets.
- Documentar patrones que funcionan.

### 15.5 Fase 4: Sofisticación (año 2+)

- Generación procedural avanzada (assets parametrizados).
- Community-contributed con moderación.
- Personalización: assets generados específicamente cuando catálogo no
  encaja perfectamente.

---

## 16. Métricas

### 16.1 Calidad de la biblioteca

| Métrica | Definición | Target MVP | Target 12m |
|---------|-----------|-----------:|-----------:|
| Assets totales | | 700 | 2.500 |
| Coverage de sub-skills | % con cobertura mínima | 80% | 100% |
| Avg user rating | | n/a | > 4.3 |
| Pass rate validación | % que pasan sin re-trabajo | 70% | 95% |
| Cost per asset | USD | $5 | $3 |

### 16.2 Eficiencia del proceso

- Time to publish: tiempo desde concepción hasta producción.
- Pass rate validación.
- Human review time per asset.
- Cost per asset.

### 16.3 Alertas

- > 30% de users struggling en mismo bloque: bloque mal calibrado.
- Sub-skill nunca mastered por nadie: criterios irreales.
- Spike en flags: review inmediato.

---

## 17. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`pedagogical-system.md`](pedagogical-system.md) | Define qué se mide en cada asset; consume `learning_blocks`. |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) | Variant de inglés por país. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) | Selecciona assets/blocks de la biblioteca. |
| [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) | Tasks `generate_asset_*`, `validate_asset_cefr`. |
| [`../architecture/sparks-system.md`](../architecture/sparks-system.md) | Operaciones de IA al usar assets cobran Sparks. |
| [`../cross-cutting/i18n.md`](../cross-cutting/i18n.md) | Variant de inglés y contenido cultural por país. |

---

*Documento vivo. Actualizar cuando se redefinan procesos, agreguen
tipos de assets, o se cambien proveedores de generación.*

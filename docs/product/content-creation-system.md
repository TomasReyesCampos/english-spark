# Content Creation and Curation System

> Sistema para crear, mantener y curar la biblioteca de assets y bloques
> que es el activo central del producto. Pipeline IA-asistida con
> revisión humana selectiva.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 establece **el contenido como activo central**.
- §2 cubre **boundaries**.
- §3 cubre **anatomía de un asset**.
- §4 cubre el **proceso de creación** (pipeline).
- §5 cubre **bloques** (composición de assets).
- §6 cubre **cobertura inicial** y roadmap de biblioteca.
- §7 cubre **schemas Postgres**.
- §8 cubre **API contracts**.
- §9 cubre **mantenimiento** (tracking de performance).
- §10 cubre **stack tecnológico**.
- §11 enumera **edge cases**.
- §12 cubre **eventos**.
- §13 cubre **decisiones cerradas**.

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

**Reusabilidad máxima:** componibles, etiquetados ricamente.

---

## 2. Boundaries

### 2.1 Es responsable de

- Crear y mantener `learning_assets` (entries individuales).
- Crear y mantener `learning_blocks` (composiciones de assets).
- Pipeline de generación con IA + validation humana.
- Versionado de assets.
- Tracking de performance (rating, completion rate).
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

## 3. Anatomía de un asset

### 3.1 Componentes

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

### 3.2 Tipos de assets

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

### 3.3 Estándares de calidad

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

## 4. Proceso de creación

### 4.1 Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 1: PLANIFICACIÓN                                     │
│  - Identificar gap en biblioteca                            │
│  - Definir learning objective específico                    │
│  - Especificar tipo, CEFR, sub-skills target                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 2: GENERACIÓN ASISTIDA POR IA                        │
│  - Prompt detallado a LLM avanzado (Claude/GPT-4)           │
│  - Generación de contenido base                             │
│  - Generación de audios con TTS premium                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 3: VALIDACIÓN AUTOMÁTICA                             │
│  - Checks técnicos (formato, estructura, completitud)       │
│  - Verificación de nivel CEFR                               │
│  - Verificación de pronunciación de TTS                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 4: REVISIÓN HUMANA (sampling)                        │
│  - 10-20% de assets revisados manualmente                   │
│  - Más alto al inicio (50% en MVP), baja con confianza      │
│  - Feedback documentado para mejorar prompts                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 5: A/B TESTING EN PRODUCCIÓN                         │
│  - Asset se libera a 5-10% de usuarios primero              │
│  - Métricas: completion rate, dificultad percibida, ratings │
│  - Si performa bien, expansión a 100%                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  ETAPA 6: MANTENIMIENTO                                     │
│  - Tracking continuo de performance                         │
│  - Actualización si métricas caen                           │
│  - Retiro si se vuelve obsoleto                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Generación con IA

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

### 4.3 Validación automática

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

### 4.4 Revisión humana

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

## 5. Bloques (composiciones)

### 5.1 Estructura

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

### 5.2 Composición típica

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

## 6. Cobertura inicial y roadmap

### 6.1 MVP target

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

### 6.2 Roadmap de expansión

| Mes | Nuevos assets | Nuevos tracks |
|-----|--------------:|---------------|
| 0–3 | 700 (MVP) | Job Ready, Travel, Daily |
| 3–6 | +500 | Business English |
| 6–9 | +500 | Tech Professional, Academic |
| 9–12 | +500 | Customer Service, Healthcare |
| 12+ | +200/mes ongoing | Especializados según demanda |

A 24 meses: ~3.500 assets cubriendo 8+ tracks.

### 6.3 Cobertura de sub-skills

Cada sub-skill debe tener mínimo:
- 5 assets en CEFR B1.
- 5 assets en CEFR B2.
- 3 assets en CEFR C1.

50 sub-skills core × ~13 assets promedio = 650 assets solo de
cobertura mínima. Solapa con tracks pero da baseline.

---

## 7. Schemas Postgres

### 7.1 Tabla `learning_assets`

```sql
CREATE TABLE learning_assets (
  id              TEXT PRIMARY KEY,
  version         TEXT NOT NULL DEFAULT '1.0.0',

  -- Clasificación
  type            TEXT NOT NULL,
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
  english_variant TEXT NOT NULL DEFAULT 'neutral',
  cultural_relevance TEXT[] DEFAULT '{}',

  -- Contenido (JSONB porque varía por type)
  content         JSONB NOT NULL,

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

### 7.2 Tabla `learning_blocks`

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

### 7.3 Tabla de versiones

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

### 7.4 Tabla de feedback

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

### 7.5 Performance tracking

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

## 8. API contracts

### 8.1 `getAsset`

**Llamado por:** cliente cuando carga un asset.

```typescript
interface GetAssetRequest {
  asset_id: string;
}

interface GetAssetResponse {
  asset: LearningAsset;
  signed_audio_urls?: Record<string, string>;  // signed URLs para audios
}
```

**Reglas:**
- Solo retorna assets con `approved = true` y `archived = false`
  (excepto si user lo está completando: snapshot version).
- Audios de R2 vía signed URLs con TTL 5 min.

### 8.2 `getBlock`

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

### 8.3 `searchAssets` (admin)

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

### 8.4 `submitAssetFeedback`

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

### 8.5 Internal: `createAsset` (admin/pipeline)

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

### 8.6 Internal: `approveAsset` (admin)

```typescript
interface ApproveAssetRequest {
  asset_id: string;
  reviewed_by: string;
  notes?: string;
}
```

### 8.7 Internal: `archiveAsset`

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

## 9. Mantenimiento

### 9.1 Reglas automáticas

| Condición | Acción |
|-----------|--------|
| Asset performing well | Mantener; posible reference para crear similares |
| Completion rate < 50% | Revisar manualmente; demasiado difícil/confuso |
| `avg_score < 30%` del target | Difícil para nivel declarado; re-evaluar nivel o reescribir |
| `retry_rate > 60%` | Muy difícil o muy adictivo; investigar |
| `flagged_count > 5%` de uses | Revisión inmediata; posible problema de calidad |
| `uses_last_30d == 0` | Considerar retirar si no es relevante |

### 9.2 Versionado de assets

Cuando un asset se actualiza:
- Mantiene versión vieja para users que ya lo iniciaron.
- Nuevos users ven versión nueva.
- Trackea performance de ambas versiones.
- Versión vieja se retira cuando ningún user activo la está usando.

### 9.3 Retirement gradual

```
Asset obsoleto detectado
  ↓
Marcar archived = true (no aparece en nuevos roadmaps)
  ↓
Users actuales pueden completar (snapshot)
  ↓
Después de 3 meses sin uso: borrar físicamente
```

### 9.4 A/B testing de assets

Cuando se crea v2 de un asset existente:
1. v2 se libera al 5% de users.
2. Comparar métricas: completion, score, rating.
3. Si v2 mejora: ramp up 25% → 50% → 100%.
4. Si peor: rollback (v1 sigue).

---

## 10. Stack tecnológico

### 10.1 Para generación

| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| LLM para texto | AI Gateway tasks `generate_asset_*` (Claude Opus / GPT-4o) | Roleplays, prompts |
| TTS premium | ElevenLabs (multiple voices) | Audios listening, ejemplos |
| Voice cloning | ElevenLabs voice library | Variedad de acentos |
| Audio editing | FFmpeg pipelines automatizadas | Normalización, trim |
| Validation LLM | AI Gateway task `validate_asset_cefr` (Haiku) | Validación post-generación |

### 10.2 Para curaduría

| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| Admin panel | Next.js + Tailwind | UI para review |
| Storage | Cloudflare R2 | Audios y archivos generados |
| Metadata | Postgres | Toda la metadata |
| Search | Postgres full-text + pgvector | Búsqueda en biblioteca |
| Diff/version control | Custom sobre Postgres | Tracking de cambios |

### 10.3 Para tracking

| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| Analytics | PostHog | Métricas de uso por asset |
| Feedback | Custom + API | Flagging y ratings |
| Dashboard | Internal admin | Vista por asset y agregada |

### 10.4 Costos de creación

**Estimado para 700 assets MVP:**

- LLM generación (Opus): ~$30 por 100 roleplays = $200 total.
- TTS premium (ElevenLabs): ~$0.30 por audio × 600 = $180.
- Validación (Haiku): ~$0.01 × 700 = $7.
- Tiempo humano: ~150h × $20/h freelance = $3.000.
- **Total: ~$3.500 USD para biblioteca MVP.**

Costo marginal por asset adicional: ~$5 USD (dominado por tiempo
humano).

---

## 11. Edge cases (tests obligatorios)

### 11.1 Generación

1. **LLM devuelve roleplay con C1 vocab para B2:** validation falla
   `cefr_match`; retry con prompt más estricto.
2. **TTS genera audio con voz robótica:** validation `audio_quality`
   detecta SNR/naturalness; rechaza, retry con otra voz.
3. **Roleplay generado tiene branching loop infinito:** validation
   detecta ciclo, rechaza.

### 11.2 Versioning

4. **Asset v1 in-use por user, admin sube v2:** user completa v1
   (snapshot). Próximos users ven v2.
5. **Asset v2 reemplaza v1 mientras user en medio del ejercicio:** UI
   advierte "este ejercicio se actualizó" pero permite completar v1
   sin interrupción.

### 11.3 Archivado

6. **Asset archived que sigue en bloques activos:** bloque también se
   marca para review; admin decide remover/reemplazar.
7. **Asset archived y user lo intenta cargar:** retorna versión
   archivada con flag `is_archived = true`. Cliente muestra advertencia
   pero permite (snapshot).

### 11.4 Performance

8. **Asset nuevo con 0 uses tiene NULL en avg_score:** UI no muestra
   "0%" sino "Datos insuficientes". Algoritmos de selección lo tratan
   como neutral.
9. **Asset con 100% completion pero rating bajo:** flag para review
   manual. Posible asset demasiado fácil pero aburrido.

### 11.5 Concurrencia

10. **Dos admins intentan editar mismo asset:** optimistic locking via
    `updated_at` en WHERE clause. Segundo recibe error "asset cambió,
    refrescá".

### 11.6 Feedback

11. **User flag asset 5 veces:** rate limit max 1 flag por user por
    asset por 24h. Repeated flags se ignoran.
12. **Spike de flags en asset (>20 en 1h):** alerta automática para
    review humano inmediato.

---

## 12. Eventos

### 12.1 Emitidos

| Evento | Cuándo |
|--------|--------|
| `asset.created` | Asset draft creado |
| `asset.approved` | Admin aprobó para producción |
| `asset.archived` | Asset retirado |
| `asset.flagged` | User flageó asset |
| `asset.performance_alert` | Métricas caen below threshold |
| `block.created` | Bloque nuevo creado |
| `block.archived` | Bloque retirado |

### 12.2 Consumidos

| Evento | Acción |
|--------|--------|
| `exercise.attempt_completed` (pedagogical) | Update performance counters via job nocturno |

---

## 13. Decisiones cerradas

### 13.1 Crowdsource de assets de la comunidad: **NO en MVP** ✓

**Razón:** calidad variable + moderación intensiva. Reconsiderar año
2 con community manager dedicado.

### 13.2 Licenciar contenido existente de proveedores: **NO en MVP** ✓

**Razón:** dificultad de licensing + control reducido. Crear propio
desde día 1.

### 13.3 Generar assets en runtime para users con perfiles únicos:
**NO** ✓

**Razón:** principio rector "IA cura, no inventa". Costos altos +
calidad variable + imposible validar pedagógicamente. Ese path se
rechaza permanentemente.

### 13.4 Cómo balancear velocidad de creación con calidad: **calidad
siempre** ✓

**Razón:** mejor menos assets buenos. Reputación del producto depende
de calidad consistente. Velocidad es problema solucionable después.

### 13.5 Versionado: **semver con releases major.minor only** ✓

**Razón:** patches no agregan valor para assets (no son código).
Major: breaking change pedagógico. Minor: mejora notable. Cualquier
cambio bumpea version.

---

## 14. Plan de implementación

### 14.1 Fase 0: Pre-MVP (mes -1)

- Schema completo en Postgres.
- Admin panel básico para review.
- Pipeline de generación con prompts iniciales.
- Pipeline de validation automática.
- Storage en R2.
- Templates por tipo.

### 14.2 Fase 1: Creación inicial (meses 0–3)

- Mes 0-1: 250 assets (Job Ready completo).
- Mes 1-2: 250 assets (Travel + Daily).
- Mes 2-3: 200 assets de refinamiento + cobertura sub-skills.

### 14.3 Fase 2: Iteración (meses 3–6)

- Tracking de performance.
- Re-trabajar 10-20% peores performers.
- Crear assets nuevos en gaps.
- Expandir biblioteca a 1.200 assets.

### 14.4 Fase 3: Escala (meses 6–12)

- 2-3 nuevos tracks.
- 2.500 assets.
- Documentar patrones que funcionan.

### 14.5 Fase 4: Sofisticación (año 2+)

- Generación procedural avanzada (assets parametrizados).
- Community-contributed con moderación.
- Personalización: assets generados específicamente cuando catálogo no
  encaja perfectamente.

---

## 15. Métricas

### 15.1 Calidad de la biblioteca

| Métrica | Definición | Target MVP | Target 12m |
|---------|-----------|-----------:|-----------:|
| Assets totales | | 700 | 2.500 |
| Coverage de sub-skills | % con cobertura mínima | 80% | 100% |
| Avg user rating | | n/a | > 4.3 |
| Pass rate validación | % que pasan sin re-trabajo | 70% | 95% |
| Cost per asset | USD | $5 | $3 |

### 15.2 Eficiencia del proceso

- Time to publish: tiempo desde concepción hasta producción.
- Pass rate validación.
- Human review time per asset.
- Cost per asset.

### 15.3 Alertas

- > 30% de users struggling en mismo bloque: bloque mal calibrado.
- Sub-skill nunca mastered por nadie: criterios irreales.
- Spike en flags: review inmediato.

---

## 16. Referencias internas

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

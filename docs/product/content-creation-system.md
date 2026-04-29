# Content Creation and Curation System

> Sistema para crear, mantener y curar la biblioteca de preassets que es
> el activo central del producto. Define cómo se construye la biblioteca
> inicial de miles de bloques y cómo evoluciona en el tiempo.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Alcance:** Sistema completo

---

## 1. El contenido como activo central

### 1.1 Por qué importa

El contenido (preassets en la biblioteca) es:

- **El activo más valioso del producto:** lo que hace que la app realmente
  enseñe vs solo entretenga.
- **El mayor diferenciador frente a competidores:** un track "Job Ready"
  para hispanohablantes con assets curados culturalmente es difícil de
  replicar.
- **El moat a largo plazo:** cuanto más contenido y mejor calibrado, más
  difícil que un competidor te alcance.
- **Lo que escala los costos a la baja:** generar una vez, usar 1.000.000
  de veces.

### 1.2 El problema sin un sistema

Sin proceso definido de creación de contenido:
- Calidad inconsistente entre bloques.
- Gaps en cobertura (sub-skills sin assets).
- Contenido desactualizado o irrelevante.
- Dependencia de quien creó (bus factor).
- Imposibilidad de escalar.

### 1.3 Filosofía de creación

**Calidad sobre cantidad:** mejor 1.000 assets excelentes que 5.000 mediocres.

**Generación asistida por IA, validación humana:** la IA hace el 80% del
trabajo, el humano calibra el 20% que importa.

**Iteración basada en datos:** los assets que funcionan se mantienen, los
que no, se reemplazan.

**Específico para hispanohablantes:** cada asset considera errores
típicos, contexto cultural, vocabulario familiar.

**Reusabilidad máxima:** assets componibles, etiquetados ricamente, con
metadata para múltiples casos de uso.

---

## 2. Anatomía de un asset

### 2.1 Qué compone un asset

```typescript
interface LearningAsset {
  // Identificación
  id: string;
  version: string;            // semver para tracking de cambios

  // Clasificación
  type: AssetType;
  cefr_level: 'A2' | 'B1' | 'B1+' | 'B2' | 'B2+' | 'C1' | 'C2';
  difficulty: number;         // 1-10
  estimated_minutes: number;

  // Pedagogía
  target_subskills: string[];   // de subskills_catalog
  prerequisites: string[];      // assets que deben completarse antes
  learning_objective: string;   // qué se aprende exactamente

  // Contexto
  topic: string[];              // ['business', 'interview', 'technical']
  context_tags: string[];       // ['formal', 'remote_work', 'tech_industry']
  english_variant: 'american' | 'british' | 'neutral';
  cultural_relevance: string[]; // países donde es especialmente relevante

  // Contenido
  content: AssetContent;        // varía según type

  // Metadata de creación
  created_by: string;           // 'ai_generated' | 'human' | 'ai_human_curated'
  created_at: Date;
  reviewed_by?: string;
  reviewed_at?: Date;
  approved_for_production: boolean;

  // Performance tracking
  usage_count: number;
  avg_completion_rate: number;
  avg_user_rating?: number;
  flagged_issues: number;
}
```

### 2.2 Tipos de assets y su contenido

#### Pronunciation Drill

```typescript
interface PronunciationDrillContent {
  target_phonemes: string[];    // ['/θ/', '/ð/']
  phrases: Array<{
    text: string;
    audio_reference_url: string; // audio del native speaker
    target_words: string[];      // palabras críticas
    expected_phonetic_focus: string;
  }>;
  feedback_templates: {
    common_errors: Record<string, string>; // error → corrección
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
  speaker_accent: string;       // 'us-general', 'uk-rp', 'us-southern'
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

### 2.3 Calidad de cada componente

Para cada tipo de asset, criterios de calidad mínimos:

**Audios:**
- Native speaker o equivalente (TTS premium calibrado).
- Sin ruido de fondo perceptible.
- Volumen normalizado.
- Duración apropiada al ejercicio.

**Texto:**
- Gramática y ortografía perfectas.
- Lenguaje natural, no robótico.
- Apropiado al nivel CEFR target.
- Sin estereotipos culturales.

**Roleplays:**
- Diálogos naturales, no académicos.
- Branching realista (cómo respondería un humano).
- Cubre objetivo pedagógico claramente.
- Termina satisfactoriamente.

---

## 3. Proceso de creación

### 3.1 Pipeline de creación de contenido

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
│  - Verificación de nivel CEFR con análisis automático       │
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

### 3.2 Generación con IA: prompts efectivos

Para crear un roleplay de "Job Interview - Behavioral Questions" en B2:

```
Sos un experto en didáctica del inglés especializado en preparación
laboral para hispanohablantes latinoamericanos.

OBJETIVO PEDAGÓGICO:
Generar un roleplay donde el usuario practica responder preguntas
behavioral en una entrevista laboral, usando el método STAR.

PARÁMETROS:
- Nivel CEFR: B2
- Duración estimada: 8-10 minutos de práctica
- Sub-skills target: vocabulary_business, fluency_extended_speech, grammar_past_tenses
- Variante de inglés: americano (mercado USA es el más relevante)
- Contexto cultural: usuario hispanoamericano aplicando a empresa internacional

RESTRICCIONES:
- Vocabulario apropiado a B2 (sin C1 unnecessarily complex words)
- Diálogos naturales, no formales en exceso
- Considerar que el usuario puede estar nervioso (primer scenario step
  debe ser warm-up)
- Mínimo 4 ramificaciones para que se sienta real

ESTRUCTURA REQUERIDA:
1. Scenario setup (descripción del contexto al usuario).
2. AI role (interviewer): qué tipo de empresa, qué rol busca el candidato.
3. Conversation flow:
   - Step 1: greeting + warm-up question.
   - Step 2: behavioral question 1 (challenge overcome).
   - Step 3: follow-up basado en respuesta.
   - Step 4: behavioral question 2 (teamwork).
   - Step 5: cierre y next steps.
4. Para cada step:
   - AI message (en inglés natural).
   - Topics que el usuario debe cubrir en respuesta.
   - 3 ejemplos de respuestas válidas en B2.
   - Branching logic según calidad de respuesta.
5. Vocabulary focus: 8-10 palabras/expresiones target.
6. Grammar focus: estructuras a practicar.
7. Cultural notes: tips específicos para hispanohablantes (ej: "evitar
   modesty excesiva", "STAR method valued in US interviews").

DEVOLVÉ JSON exacto siguiendo el schema RoleplayContent que conocés.
```

### 3.3 Validación automática post-generación

Checks automáticos sobre el output del LLM:

```typescript
async function validateAsset(asset: LearningAsset): Promise<ValidationResult> {
  const checks = [];

  // Estructura
  checks.push(validateSchema(asset));

  // Nivel CEFR del texto
  const measuredCEFR = await measureTextCEFR(extractAllText(asset));
  checks.push({
    name: 'cefr_match',
    passed: matchesCEFR(measuredCEFR, asset.cefr_level)
  });

  // Vocabulario apropiado
  const vocab = analyzeVocabulary(extractAllText(asset));
  checks.push({
    name: 'vocab_distribution',
    passed: vocabAppropriateForLevel(vocab, asset.cefr_level)
  });

  // Sub-skills realmente abordadas
  const detectedSubskills = await detectSubskillsAddressed(asset);
  checks.push({
    name: 'subskills_coverage',
    passed: asset.target_subskills.every(s => detectedSubskills.includes(s))
  });

  // Audios (si aplica)
  if (asset.content.audio_url) {
    checks.push(await validateAudioQuality(asset.content.audio_url));
  }

  // Duración estimada vs contenido
  checks.push(validateDuration(asset));

  return {
    passed: checks.every(c => c.passed),
    checks,
    needsHumanReview: checks.some(c => !c.passed) || sampleForReview()
  };
}
```

### 3.4 Revisión humana

#### Quién revisa

**Fase 1 (MVP):** vos mismo o contratación de freelancers nativos
hispanohablantes con buen inglés (C1+).

**Fase 2 (con tracción):** equipo de 1-2 content reviewers part-time.

**Fase 3 (escala):** equipo dedicado + community contributions
moderadas.

#### Qué se revisa

Checklist de revisión humana:

- [ ] El audio suena natural y claro.
- [ ] El diálogo se siente realista, no robótico.
- [ ] El nivel CEFR es apropiado.
- [ ] No hay errores gramaticales o de ortografía.
- [ ] El contenido es culturalmente apropiado.
- [ ] No hay estereotipos.
- [ ] El objetivo pedagógico es claramente alcanzable.
- [ ] La dificultad estimada es razonable.
- [ ] El tiempo estimado es preciso.

#### Sample size

- Primeros 200 assets: 100% revisión humana.
- Assets 201-500: 50% revisión.
- Assets 501-2.000: 20% revisión.
- Assets 2.000+: 10% revisión.

A medida que la confianza en los prompts crece, el sample baja.

#### Documentar feedback

Cada rejection o ajuste se documenta para mejorar prompts:

```
Asset rejected: roleplay_interview_b2_011
Reason: AI interviewer asks question too advanced for B2 level
Fix: ajustar prompt agregando "questions should not require C1 vocabulary"
```

Este feedback se incorpora a versiones futuras del prompt.

---

## 4. Cobertura inicial de la biblioteca

### 4.1 Cantidad target para MVP

Para lanzar MVP funcional, biblioteca mínima:

| Track | Niveles | Bloques por nivel | Total bloques | Assets por bloque | Total assets |
|-------|---------|-------------------|---------------|-------------------|--------------|
| Job Ready | 6 | 12 | 72 | 4 | 288 |
| Travel Confident | 4 | 10 | 40 | 4 | 160 |
| Daily Conversation | 5 | 12 | 60 | 4 | 240 |
| **Total MVP** | | | **172** | | **688** |

Aproximadamente **700 assets de calidad** para MVP. A 30 minutos de
generación + revisión por asset, son ~350 horas de trabajo. Distribuido
entre tipos:

- ~150 pronunciation drills (simples, generación rápida)
- ~200 roleplays (más complejos, más tiempo)
- ~150 listening exercises (requieren audios curados)
- ~100 free response prompts (relativamente simples)
- ~100 vocabulary in context (estructurados)

### 4.2 Roadmap de expansión post-MVP

| Mes | Nuevos assets | Nuevos tracks |
|-----|---------------|---------------|
| 0-3 | 700 (MVP) | Job Ready, Travel, Daily |
| 3-6 | +500 | Business English |
| 6-9 | +500 | Tech Professional, Academic |
| 9-12 | +500 | Customer Service, Healthcare |
| 12+ | +200/mes ongoing | Especializados según demanda |

A 24 meses: ~3.500 assets cubriendo 8+ tracks.

### 4.3 Cobertura de sub-skills

Cada sub-skill debe tener mínimo:
- 5 assets en CEFR B1
- 5 assets en CEFR B2
- 3 assets en CEFR C1

Para 50 sub-skills core × 13 assets promedio = 650 assets solo de
cobertura mínima de sub-skills.

Este número se solapa parcialmente con assets de tracks (un asset puede
abordar múltiples sub-skills), pero da una baseline.

---

## 5. Mantenimiento y evolución de la biblioteca

### 5.1 Tracking de performance por asset

Para cada asset, métricas continuas:

```typescript
interface AssetPerformance {
  asset_id: string;

  // Volumen
  total_uses: number;
  unique_users: number;
  uses_last_30d: number;

  // Calidad pedagógica
  completion_rate: number;          // % que completan vs abandonan
  avg_score: number;                // score promedio obtenido
  retry_rate: number;               // % que repiten (puede indicar dificultad)

  // Feedback explícito
  user_ratings: number[];
  flagged_count: number;            // usuarios que reportaron problema

  // Performance temporal
  trend: 'improving' | 'stable' | 'declining';
}
```

### 5.2 Reglas de mantenimiento automático

**Asset "performing well":** mantener sin cambios. Posiblemente promocionar
como reference para crear similares.

**Asset con completion < 50%:** revisar manualmente, posiblemente
demasiado difícil o confuso.

**Asset con avg_score < 30%:** muy difícil para su nivel CEFR declarado.
Re-evaluar nivel o reescribir.

**Asset con retry_rate > 60%:** o muy difícil o muy adictivo. Investigar.

**Asset con flagged_count > 5%:** revisión inmediata, posible problema
de calidad o ofensivo.

**Asset con uses_last_30d == 0:** considerar retirar si no es relevante.

### 5.3 Versionado de assets

Cuando un asset se actualiza:
- Se mantiene versión antigua para usuarios que ya lo iniciaron.
- Nuevos usuarios ven versión nueva.
- Se trackea performance de ambas versiones para validar mejora.
- Versión antigua se retira cuando ningún usuario activo la está usando.

### 5.4 Retirement gradual

Assets obsoletos se retiran:
- Marcados como `archived = true` (no aparecen en nuevos roadmaps).
- Usuarios actuales pueden completar.
- Después de 3 meses sin uso, se borran físicamente.

---

## 6. Stack tecnológico de creación

### 6.1 Para generación

| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| LLM para texto | Claude Opus / GPT-4o | Generación de roleplays, prompts |
| TTS premium | ElevenLabs (multiple voices) | Audios para listening, ejemplos |
| Voice cloning | ElevenLabs voice library | Variedad de acentos |
| Audio editing | FFmpeg automated pipelines | Normalización, trim |
| Validation LLM | Claude Haiku / Gemini Flash | Validación automática post-generación |

### 6.2 Para curaduría

| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| Admin panel | Next.js + Tailwind | UI para review de assets |
| Storage | Cloudflare R2 | Audios y archivos generados |
| Metadata | Postgres | Toda la metadata de assets |
| Search | Postgres full-text + pgvector | Búsqueda en biblioteca |
| Diff/version control | Custom sobre Postgres | Tracking de cambios |

### 6.3 Para tracking

| Componente | Tecnología | Uso |
|-----------|-----------|-----|
| Analytics | PostHog | Métricas de uso por asset |
| Feedback | Custom tabla + API | Flagging y ratings |
| Performance dashboard | Internal admin | Vista por asset y agregada |

### 6.4 Costos de creación

**Estimado para 700 assets MVP:**

- LLM generación (Claude Opus): ~$30 por 100 roleplays = $200 total
- TTS premium (ElevenLabs): ~$0.30 por audio × 600 audios = $180
- Validación automática (Haiku): ~$0.01 × 700 = $7
- Tiempo humano: ~150 horas × $20/hora freelance = $3.000
- **Total: ~$3.500 USD para biblioteca MVP**

Costo marginal por asset adicional post-MVP: ~$5 USD (dominado por
tiempo humano de revisión).

---

## 7. Schema y código

### 7.1 Schema en Postgres

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
  created_at      TIMESTAMPTZ DEFAULT now(),
  reviewed_by     TEXT,
  reviewed_at     TIMESTAMPTZ,
  approved        BOOLEAN DEFAULT false,
  archived        BOOLEAN DEFAULT false,

  -- Performance (denormalized counters, updated periodically)
  usage_count     INT DEFAULT 0,
  unique_users    INT DEFAULT 0,
  avg_completion_rate FLOAT,
  avg_score       FLOAT,
  flagged_count   INT DEFAULT 0
);

CREATE INDEX idx_assets_subskills ON learning_assets USING gin(target_subskills);
CREATE INDEX idx_assets_topic ON learning_assets USING gin(topic);
CREATE INDEX idx_assets_active ON learning_assets(approved, archived)
  WHERE approved = true AND archived = false;

CREATE TABLE asset_versions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  asset_id        TEXT NOT NULL REFERENCES learning_assets(id),
  version         TEXT NOT NULL,
  content         JSONB NOT NULL,
  changed_at      TIMESTAMPTZ DEFAULT now(),
  changed_by      TEXT,
  change_reason   TEXT
);

CREATE TABLE asset_feedback (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  asset_id        TEXT NOT NULL REFERENCES learning_assets(id),
  user_id         UUID NOT NULL REFERENCES users(id),
  rating          INT CHECK (rating BETWEEN 1 AND 5),
  flag_reason     TEXT,
  comment         TEXT,
  created_at      TIMESTAMPTZ DEFAULT now()
);
```

---

## 8. Plan de implementación

### 8.1 Fase 0: Pre-MVP (mes -1)

**Objetivos:** preparar el sistema antes de empezar a crear contenido a
escala.

- Schema completo en Postgres.
- Admin panel básico para review.
- Pipeline de generación con prompts iniciales.
- Pipeline de validación automática.
- Storage configurado en R2.
- Templates definidos para cada tipo de asset.

### 8.2 Fase 1: Creación inicial (meses 0-3)

**Objetivos:** crear los 700 assets de MVP.

- Mes 0-1: 250 assets (priorizando Job Ready completo).
- Mes 1-2: 250 assets (Travel + Daily Conversation).
- Mes 2-3: 200 assets de refinamiento + cobertura de sub-skills.

### 8.3 Fase 2: Iteración basada en datos (meses 3-6)

**Objetivos:** mejorar assets existentes con datos reales y crear nuevo
contenido según gaps.

- Tracking de performance de cada asset.
- Re-trabajar 10-20% peores performers.
- Crear assets nuevos en sub-skills donde hay gap.
- Expandir biblioteca a 1.200 assets.

### 8.4 Fase 3: Escala (meses 6-12)

**Objetivos:** crecer biblioteca y diversificar.

- Lanzar 2-3 nuevos tracks.
- Llegar a 2.500 assets.
- Empezar a documentar patrones que funcionan para futuros assets.

### 8.5 Fase 4: Sofisticación (año 2+)

- Generación procedural avanzada (assets dinámicos parametrizados).
- Community-contributed assets con moderación.
- Personalización extrema: assets generados específicamente para usuarios
  cuando los del catálogo no encajan perfectamente.

---

## 9. Métricas de éxito

### 9.1 Calidad de la biblioteca

- **Coverage:** % de sub-skills con cobertura mínima por nivel CEFR.
- **Quality score:** rating promedio de assets activos.
- **Reusability:** uso promedio de un asset (más alto = mejor curado).
- **Freshness:** % de assets actualizados en últimos 6 meses.

### 9.2 Eficiencia del proceso

- **Time to publish:** tiempo desde concepción hasta producción de un asset.
- **Pass rate:** % de assets generados que pasan validación sin re-trabajo.
- **Human review time:** tiempo promedio de revisión humana por asset.
- **Cost per asset:** costo total dividido por assets producidos.

### 9.3 Targets

| Métrica | MVP | 6 meses | 12 meses |
|---------|-----|---------|----------|
| Assets totales | 700 | 1.500 | 2.500 |
| Coverage de sub-skills | 80% | 95% | 100% |
| Avg user rating | n/a | >4.0 | >4.3 |
| Pass rate validación | 70% | 85% | 95% |
| Cost per asset | $5 | $4 | $3 |

---

## 10. Decisiones abiertas

- [ ] ¿Crowdsource de assets de la comunidad? Pros: escala. Contras:
  calidad variable, moderación.
- [ ] ¿Licenciar contenido existente de proveedores educativos? Cuidado
  con derechos.
- [ ] ¿Generar assets en tiempo real para usuarios con perfiles únicos?
  Posible pero costoso.
- [ ] ¿Cómo balancear velocidad de creación con calidad? Mi recomendación:
  no comprometer calidad nunca, mejor menos assets buenos.
- [ ] ¿Versionado semántico (semver) o flat para assets? Semver es overkill
  posiblemente.

---

## 11. Referencias internas

- `docs/product/pedagogical-system.md` — Define qué se mide en cada asset.
- `docs/product/student-profile-and-assessment.md` — Variantes por país.
- `docs/product/ai-roadmap-system.md` — Roadmap consume estos assets.
- `docs/architecture/ai-gateway-strategy.md` — Generación pasa por Gateway.

---

*Documento vivo. Actualizar cuando se redefinan procesos, agreguen nuevos
tipos de assets, o se cambien proveedores de generación.*

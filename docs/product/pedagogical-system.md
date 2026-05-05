# Pedagogical System and Mastery Evaluation

> Sistema pedagógico que define qué es "saber" en cada bloque, cómo se
> evalúa el dominio, qué criterios determinan progresión, y cómo se
> garantiza que el usuario realmente está mejorando (no solo completando).

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador. Toda ambigüedad debe
resolverse leyendo este documento entero y sus referencias internas
(§16). Si la duda persiste, escalar a humano antes de implementar. No
inventar.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

El documento está estructurado para que **podás implementarlo sin
ambigüedades**:

- **Schemas Postgres** en §9 son autoritativos. Si necesitás un campo
  nuevo, primero actualizá este documento.
- **Constantes numéricas** (§3, §4) son los **valores por defecto** de
  configuración, no constantes hardcoded del código. Deben vivir en una
  tabla `pedagogical_config` o en variables de entorno con default a
  estos valores.
- **API contracts** (§10) son los puntos de entrada/salida del sistema.
  Otros sistemas (`ai-roadmap-system`, `motivation-and-achievements`,
  `student-profile-and-assessment`) consumen solo esto, no acceden
  directamente a las tablas.
- **Edge cases** (§11) deben tener tests unitarios o de integración
  explícitos.
- **Eventos** (§12) deben emitirse al bus de eventos con los nombres y
  shapes exactos especificados.

Decisiones cerradas en §15. No quedan decisiones abiertas en este
documento — si encontrás una, es un bug del diseño que debe escalarse.

---

## 1. Por qué este sistema importa

### 1.1 El problema central

Las apps de aprendizaje suelen confundir **completitud** con **dominio**.
Un usuario que completó 100 ejercicios no necesariamente sabe más que
uno que completó 50 con calidad.

Sin un sistema riguroso de evaluación de mastery:
- Los Sparks otorgados son arbitrarios.
- Los certificados pierden valor.
- El roadmap recomienda contenido que el usuario ya domina o no debería
  estar haciendo.
- El usuario percibe progreso falso, lo que daña la confianza a largo plazo.

### 1.2 Filosofía pedagógica

**Mastery sobre completion:** mejor hacer 5 ejercicios bien que 20 mal.

**Evaluación contextual:** un mismo "error" puede ser aceptable en B1 e
inaceptable en C1. Las métricas se calibran por nivel.

**Progreso medible y honesto:** si un usuario no está mejorando, el
sistema lo reconoce y ajusta, en lugar de premiar esfuerzo sin resultado.

**Multi-dimensional:** el dominio del inglés tiene varias dimensiones
(pronunciación, fluidez, gramática, vocabulario, comprensión). Una sola
métrica es engañosa.

### 1.3 Responsabilidades de este sistema (boundary)

**Es responsable de:**
- Calcular scores por dimensión a partir de un intento de ejercicio.
- Mantener el estado de mastery por sub-skill por usuario.
- Mantener el estado de los bloques por usuario (locked/available/...).
- Detectar `struggling` y aplicar el decay temporal.
- Calcular `next_review_at` con spaced repetition.
- Recalcular mastery global y CEFR del usuario.
- Emitir eventos de cambios de estado para que otros sistemas los
  consuman.

**NO es responsable de:**
- Generar el contenido de los bloques (eso es `content-creation-system`).
- Decidir qué bloque mostrar al usuario en su próxima sesión (eso lo
  decide `ai-roadmap-system` consumiendo este sistema).
- Otorgar logros o Sparks bonus (eso es `motivation-and-achievements`
  consumiendo eventos de este sistema).
- Cobrar Sparks por la operación de scoring (eso lo hace `sparks-system`
  vía AI Gateway antes de invocar el scoring).

---

## 2. Modelo de evaluación

### 2.1 Las 5 dimensiones evaluadas

```typescript
interface MasteryDimensions {
  pronunciation: number;     // 0-100, claridad fonética
  fluency: number;           // 0-100, flujo, velocidad, coherencia
  grammar: number;           // 0-100, corrección sintáctica
  vocabulary: number;        // 0-100, riqueza y precisión léxica
  listening: number;         // 0-100, comprensión auditiva
}
```

Cada dimensión se mide independientemente. Un usuario puede tener
pronunciación B2 y gramática B1, lo cual es información valiosa para
personalización.

### 2.2 Sub-skills por dimensión (resumen)

Cada dimensión se descompone en sub-skills concretas evaluables. El
catálogo completo está en §2.4. Resumen:

- **Pronunciation:** fonemas individuales, vocales tensas vs laxas,
  stress, entonación, linking, schwa.
- **Fluency:** velocidad (WPM), pausas, muletillas, auto-correcciones,
  conectores, turnos largos.
- **Grammar:** tiempos verbales, artículos, preposiciones, word order,
  concordancia, phrasal verbs.
- **Vocabulary:** vocabulario activo, precisión léxica, variedad,
  dominio específico, idioms, collocations.
- **Listening:** velocidad nativa, ruido de fondo, acentos, detalles vs
  idea general, inferencia.

### 2.3 Niveles de mastery por sub-skill

| Nivel | Score | Descripción | Acción del sistema |
|-------|-------|-------------|---------------------|
| Unfamiliar | 0–30 | No domina la sub-skill | Introducir gradualmente |
| Learning | 31–60 | Practicando, errores frecuentes | Práctica intensiva, feedback corrector |
| Developing | 61–80 | Domina mayormente, errores ocasionales | Refuerzo, contextos variados |
| Proficient | 81–95 | Dominio sólido, errores raros | Mantenimiento ocasional |
| Mastered | 96–100 | Dominio completo y consistente | Test out, no requiere más práctica |

Transición de nivel se calcula al final de cada `exercise_attempt` que
toca la sub-skill. Ver §2.5 para reglas de agregación.

### 2.4 Catálogo inicial de sub-skills core (50)

Este catálogo es el **seed** de la tabla `subskills_catalog`. Se carga
en migración inicial. Expansión a 200+ en Fase 2 según gaps observados.

#### Pronunciation (14)

| ID | Nombre | CEFR intro | Prerequisites | Ejercicios sugeridos |
|----|--------|-----------|---------------|----------------------|
| `pron_th_voiceless` | /θ/ (think, three) | A2 | — | drill, shadowing |
| `pron_th_voiced` | /ð/ (this, mother) | A2 | `pron_th_voiceless` | drill, shadowing |
| `pron_v_vs_b` | /v/ vs /b/ contrast | A2 | — | drill, minimal pairs |
| `pron_zh` | /ʒ/ (measure, vision) | B1 | `pron_sh` | drill |
| `pron_sh` | /ʃ/ (she, sure) | A2 | — | drill, shadowing |
| `pron_short_long_i` | /ɪ/ vs /iː/ (ship vs sheep) | A2 | — | minimal pairs |
| `pron_short_long_u` | /ʊ/ vs /uː/ (full vs fool) | B1 | `pron_short_long_i` | minimal pairs |
| `pron_ae` | /æ/ (cat, hand) | A2 | — | drill |
| `pron_schwa` | schwa /ə/ en sílabas átonas | B1 | `pron_word_stress` | shadowing |
| `pron_final_consonants` | -ed, -s, -t endings | A2 | — | drill, read aloud |
| `pron_word_stress` | stress en multi-sílabas | B1 | — | drill, marcar acento |
| `pron_sentence_stress` | content vs function words | B2 | `pron_word_stress` | shadowing |
| `pron_intonation_questions` | rising vs falling | B1 | — | drill, roleplay |
| `pron_linking` | connected speech | B2 | `pron_schwa` | shadowing, read aloud |

#### Fluency (8)

| ID | Nombre | CEFR intro | Prerequisites | Ejercicios sugeridos |
|----|--------|-----------|---------------|----------------------|
| `flu_speaking_pace` | velocidad apropiada | A2 | — | free response, roleplay |
| `flu_pause_management` | pausas funcionales | B1 | `flu_speaking_pace` | shadowing, roleplay |
| `flu_filler_reduction` | reducir muletillas | B1 | `flu_speaking_pace` | free response |
| `flu_self_correction_control` | no sobre-corregir | B2 | `flu_filler_reduction` | free response, roleplay |
| `flu_discourse_connectors` | however, therefore, etc. | B1 | — | free response |
| `flu_extended_speech` | mantener turnos largos | B2 | `flu_discourse_connectors` | free response, roleplay |
| `flu_turn_taking` | tomar y ceder turno | B1 | — | roleplay |
| `flu_thinking_in_english` | sin traducir mentalmente | B2 | `flu_extended_speech` | roleplay free, image description |

#### Grammar (14)

| ID | Nombre | CEFR intro | Prerequisites |
|----|--------|-----------|---------------|
| `gram_present_simple_continuous` | present simple/continuous | A2 | — |
| `gram_past_simple_continuous` | past simple/continuous | A2 | `gram_present_simple_continuous` |
| `gram_present_perfect` | present perfect uso | B1 | `gram_past_simple_continuous` |
| `gram_past_perfect` | past perfect uso | B2 | `gram_present_perfect` |
| `gram_future_forms` | will/going to/present cont | B1 | `gram_present_simple_continuous` |
| `gram_conditionals_zero_first` | zero/first conditional | B1 | `gram_future_forms` |
| `gram_conditionals_second_third` | second/third conditional | B2 | `gram_conditionals_zero_first` |
| `gram_articles` | a/an/the (alta dificultad ES) | B1 | — |
| `gram_prepositions_time` | in/on/at de tiempo | A2 | — |
| `gram_prepositions_place` | in/on/at de lugar | A2 | — |
| `gram_phrasal_verbs_basic` | get up, look for, etc. | B1 | — |
| `gram_relative_clauses` | who/which/that | B2 | — |
| `gram_reported_speech` | direct → indirect | B2 | `gram_past_simple_continuous` |
| `gram_passive_voice` | pasiva en presente y pasado | B1 | `gram_past_simple_continuous` |

#### Vocabulary (8)

| ID | Nombre | CEFR intro | Prerequisites |
|----|--------|-----------|---------------|
| `vocab_high_frequency_b1` | top 2.000 palabras B1 | B1 | — |
| `vocab_collocations_general` | run a meeting, take a break | B1 | `vocab_high_frequency_b1` |
| `vocab_idioms_common` | piece of cake, hit the road | B2 | `vocab_collocations_general` |
| `vocab_business_general` | meeting, deadline, pitch | B1 | — |
| `vocab_tech_industry` | sprint, deploy, ship | B2 | `vocab_business_general` |
| `vocab_travel_practical` | check-in, boarding, layover | A2 | — |
| `vocab_emotion_register` | formal vs casual emotion | B2 | — |
| `vocab_academic_discourse` | hence, thereby, albeit | C1 | `vocab_idioms_common` |

#### Listening (6)

| ID | Nombre | CEFR intro | Prerequisites |
|----|--------|-----------|---------------|
| `list_detail_capture` | captar números, nombres | B1 | — |
| `list_native_speed_general` | velocidad nativa, idea general | B2 | `list_detail_capture` |
| `list_distinguishing_accents` | US/UK/AU básicos | B2 | `list_native_speed_general` |
| `list_inference` | lo no dicho explícitamente | B2 | `list_native_speed_general` |
| `list_phone_distorted_audio` | audio con ruido | B2 | `list_native_speed_general` |
| `list_overlapping_speech` | múltiples hablantes simultáneos | C1 | `list_distinguishing_accents` |

**Total: 50 sub-skills core.** Distribución por nivel CEFR de
introducción:

| Nivel | Sub-skills introducidas |
|-------|-------------------------|
| A2 | 14 |
| B1 | 18 |
| B2 | 16 |
| C1 | 2 |

### 2.5 Reglas de agregación

Score se agrega en cuatro niveles. Reglas exactas:

**Nivel 1 — Attempt → dimensión.** Cada `exercise_attempt` produce un
`scores: Record<Dimension, number>`. Solo dimensiones que el ejercicio
mide (ver matriz §5.4) tienen valor; las demás quedan `null`.

**Nivel 2 — Attempt → sub-skill.** Cada attempt impacta una o varias
sub-skills (campo `target_subskills` del bloque). El delta a aplicar
sobre `current_score` de cada sub-skill:

```
delta = (attempt_score - current_score) * learning_rate
where:
  learning_rate = 0.30 if subskill.total_attempts < 5    (early learning)
                = 0.15 if subskill.total_attempts < 20   (consolidation)
                = 0.08 if subskill.total_attempts >= 20  (refinement)
  attempt_score = score de la dimensión principal de la sub-skill
                  en este attempt (ver matriz §5.4)
new_score = clamp(current_score + delta, 0, 100)
```

Esto evita que un mal día baje a un usuario de `mastered` a `learning`
y que un usuario novato salte a `proficient` con un golpe de suerte.

**Nivel 3 — Sub-skill → dimensión.** Score de dimensión es el promedio
ponderado de sus sub-skills, donde el peso es la frecuencia de práctica
relativa (sub-skills más practicadas pesan más en la dimensión):

```
dimension_score = Σ (subskill_score × subskill_practice_weight)
                  / Σ subskill_practice_weight
where:
  subskill_practice_weight = log(1 + total_attempts_subskill)
```

**Nivel 4 — Dimensión → CEFR.** El CEFR overall del usuario se mapea
desde el vector de scores por dimensión usando la función calibrada
(§7.1):

```python
def map_dimensions_to_cefr(scores: MasteryDimensions) -> str:
    # Speaking-weighted (producto está enfocado en speaking)
    weighted = (
        scores.pronunciation * 0.25 +
        scores.fluency       * 0.25 +
        scores.grammar       * 0.20 +
        scores.vocabulary    * 0.15 +
        scores.listening     * 0.15
    )
    if weighted < 35:  return 'A2'
    if weighted < 55:  return 'B1'
    if weighted < 70:  return 'B1+'
    if weighted < 82:  return 'B2'
    if weighted < 90:  return 'B2+'
    return 'C1'
```

Estos thresholds **deben recalibrarse** después del paso de calibración
inicial (§7.1) y persistirse en `pedagogical_config`.

### 2.6 Source weight: sporadic vs assessment vs exercise (v1.2)

> Promovido desde [`docs/explorations/sporadic-questions.md`](../explorations/sporadic-questions.md).

Los attempts que llegan al sistema de scoring tienen un campo `source`
que determina su peso en la actualización de mastery:

| `source` | Weight | Origen |
|----------|-------:|--------|
| `exercise` | 1.0 | Ejercicio normal del roadmap (default) |
| `assessment` | 1.0 | Ejercicio del assessment formal Day 7 |
| `mini_test` | 1.0 | Mini-test del onboarding Day 0 |
| `sporadic` | 0.5 | Sporadic question audio capture (pre-assessment) |
| `test_out` | 1.0 | Test-out voluntario (§4.5) |
| `correction` | 0.0 | Sólo log, no actualiza mastery |

**Implementación:**

```python
def apply_attempt_to_subskill_mastery(attempt, subskill_id):
    base_delta = (attempt.score - current_score) * learning_rate
    weighted_delta = base_delta * SOURCE_WEIGHTS[attempt.source]
    new_score = clamp(current_score + weighted_delta, 0, 100)
    return new_score
```

**Razón del 0.5x para sporadic:**
- Audio captures cortas (5-15s) tienen menos data que ejercicios
  completos.
- Anti-noise: si user responde fake (no detectado), el impacto es
  reducido.
- Permite calibración pre-assessment sin overcommitting.

**Anti-fake:** attempts con `flagged_likely_fake = true` tienen weight
**0.0** (descartados de calibración pero logged).

---

## 3. Métodos de medición

### 3.1 Pronunciation scoring

#### Método: análisis acústico de fonemas

Para cada audio del usuario:

1. Forced alignment con el texto esperado (alinea audio con palabras).
2. Extracción de cada fonema individualmente.
3. Comparación con modelos de referencia de hablantes nativos.
4. Score por fonema en escala 0–100.
5. Agregación a nivel palabra y oración.

#### Implementación

**Fase 1 (MVP):** Azure Pronunciation Assessment API.
- Pros: producción-ready, calibrado, soporte multilingüe.
- Costos: $0.01–0.03 USD por minuto evaluado.
- Cons: API externa con costo escalando con uso.

**Fase 2 (post-validación):** modelo propio fine-tuneado.
- Base: Whisper o wav2vec2 open source.
- Fine-tuning con audios de hispanohablantes etiquetados.
- Inferencia self-hosted en Modal o GPU propia.
- Costo marginal mucho menor a escala.

**Trigger de migración (decisión cerrada §15):** evaluar migración cuando
spend mensual de Azure supere **$500 USD/mes**. Iniciar migración cuando
supere **$2.000 USD/mes**, con dataset mínimo de 5.000 audios etiquetados
de hispanohablantes ya recolectado.

#### Constantes de scoring

```typescript
const PRONUNCIATION_CONFIG = {
  phoneme_score_floor: 30,        // Azure < 30 se trata como outlier
  word_score_aggregation: 'mean', // agregación de fonemas en palabra
  sentence_score_aggregation: 'weighted_mean', // por longitud de palabra
  min_audio_duration_ms: 1000,    // descartar attempts < 1s útil
  silence_threshold_db: -50,      // qué se considera silencio
  noise_floor_warning_db: -25,    // alertar si SNR es muy bajo
};
```

#### Calibración para hispanohablantes

Errores típicos a detectar específicamente (priorizar en feedback):

| Error | Patrón | Sub-skill afectada |
|-------|--------|--------------------|
| /θ/ → /s/ o /t/ | think → sink | `pron_th_voiceless` |
| /ð/ → /d/ | this → dis | `pron_th_voiced` |
| /v/ → /b/ | very → bery | `pron_v_vs_b` |
| /ʒ/ → /j/ o /sh/ | measure → mesha | `pron_zh` |
| /ɪ/ → /iː/ | ship → sheep | `pron_short_long_i` |
| /æ/ → /a/ ESP | cat con vocal española | `pron_ae` |
| Schwa → vocal completa | about → "a-bout" | `pron_schwa` |
| Final consonants omitidas | cold → col | `pron_final_consonants` |

Esta tabla se persiste en `pronunciation_error_patterns` (§9) y se usa
para mapear errores detectados → sub-skill afectada.

### 3.2 Fluency scoring

#### Método: análisis temporal de la producción

Métricas calculadas a partir de la transcripción + alineamiento temporal:

```typescript
interface FluencyMetrics {
  words_per_minute: number;
  pause_frequency: number;          // pausas por minuto
  pause_avg_duration_ms: number;
  pause_long_count: number;         // pausas > 2s
  filler_word_count: number;        // uh, uhm, like, you know
  filler_per_minute: number;
  self_correction_count: number;
  speech_rate_consistency: number;  // 1 - (stddev_wpm / mean_wpm)
  utterance_length_avg: number;     // palabras por turno
  total_speaking_time_ms: number;
  total_silence_time_ms: number;
}
```

#### Detección de muletillas

Lista cerrada de fillers a detectar (configurable):

```typescript
const FILLERS = ['uh', 'uhm', 'um', 'er', 'ah', 'like', 'you know',
                 'i mean', 'so', 'well', 'right', 'okay'];
```

Se cuentan ocurrencias en posiciones no-funcionales. Heurística: `like`
cuenta como filler si está rodeado de pausas cortas; cuenta como conector
genuino si introduce ejemplo.

#### Cálculo del score

```python
def calculate_fluency_score(metrics: FluencyMetrics, target_cefr: str) -> float:
    targets = FLUENCY_TARGETS[target_cefr]

    # WPM score: 100 si está en rango target, decae fuera
    wpm_score = score_in_range(
        metrics.words_per_minute,
        ideal=targets['wpm_target_mean'],
        acceptable_lower=targets['wpm_min'],
        acceptable_upper=targets['wpm_max'],
        decay_per_unit=2.0,  # -2 puntos por WPM fuera de rango
    )

    # Pause score: penalizar pausas frecuentes y largas
    pause_score = 100
    pause_score -= max(0, metrics.pause_frequency - targets['pause_freq_max']) * 4
    pause_score -= metrics.pause_long_count * 3
    pause_score = clamp(pause_score, 0, 100)

    # Filler score: cada filler/min sobre umbral resta
    filler_score = 100 - max(0, metrics.filler_per_minute - targets['filler_per_min_max']) * 5
    filler_score = clamp(filler_score, 0, 100)

    # Consistency score: directo desde la métrica
    consistency_score = metrics.speech_rate_consistency * 100

    return weighted_average([
        (wpm_score, 0.30),
        (pause_score, 0.30),
        (filler_score, 0.25),
        (consistency_score, 0.15),
    ])
```

#### Calibración por nivel CEFR

```typescript
const FLUENCY_TARGETS = {
  'A2':  { wpm_min: 60,  wpm_target_mean: 80,  wpm_max: 100, pause_freq_max: 12, filler_per_min_max: 8 },
  'B1':  { wpm_min: 80,  wpm_target_mean: 90,  wpm_max: 110, pause_freq_max: 8,  filler_per_min_max: 6 },
  'B1+': { wpm_min: 90,  wpm_target_mean: 105, wpm_max: 125, pause_freq_max: 6,  filler_per_min_max: 5 },
  'B2':  { wpm_min: 110, wpm_target_mean: 120, wpm_max: 140, pause_freq_max: 5,  filler_per_min_max: 4 },
  'B2+': { wpm_min: 120, wpm_target_mean: 130, wpm_max: 150, pause_freq_max: 4,  filler_per_min_max: 3 },
  'C1':  { wpm_min: 130, wpm_target_mean: 145, wpm_max: 170, pause_freq_max: 3,  filler_per_min_max: 2 },
};
```

### 3.3 Grammar scoring

#### Pipeline

1. STT del audio del usuario (vía AI Gateway, task `transcribe_user_audio`).
2. Análisis de la transcripción contra el ejercicio:
   - Para ejercicios estructurados (fill-in-blank, sentence-ordering):
     comparación directa con la respuesta esperada.
   - Para producción libre: detección de errores con LLM via AI Gateway,
     task `detect_grammar_errors`.
3. Categorización de errores por tipo y severidad.
4. Score basado en frecuencia y gravedad.

#### Severidad de errores

```typescript
interface GrammarError {
  type: string;          // 'past_perfect_misuse', 'article_omission', etc.
  severity: 'minor' | 'moderate' | 'major';
  position: number;      // char offset en la transcripción
  excerpt: string;       // contexto local
  suggestion: string;    // corrección propuesta
  affected_subskill: string;  // ID de la sub-skill afectada
}
```

| Severidad | Definición | Penalización (puntos) |
|-----------|-----------|-----------------------|
| Minor | No afecta comprensión, sería natural en native speaker en speech informal | -1 |
| Moderate | Notable pero comprensible | -3 |
| Major | Causa confusión, malentendido o quiebre comunicativo | -10 |

#### Cálculo del score

```python
def calculate_grammar_score(transcription: str, errors: list[GrammarError],
                            target_cefr: str, exercise_context: str) -> float:
    score = 100

    # Penalizaciones por error
    for err in errors:
        penalty = {'minor': 1, 'moderate': 3, 'major': 10}[err.severity]
        # Atenuar penalización para errores en speech espontáneo
        penalty *= GRAMMAR_CONTEXT_TOLERANCE[exercise_context]
        score -= penalty

    # Bonus por complejidad apropiada al nivel
    complexity_bonus = measure_syntactic_complexity(transcription, target_cefr)
    score += complexity_bonus  # típicamente -5 a +10

    return clamp(score, 0, 100)
```

#### Calibración por contexto

```typescript
const GRAMMAR_CONTEXT_TOLERANCE = {
  'free_response': 0.6,        // multiplicador de penalty (más tolerante)
  'roleplay_free': 0.6,
  'roleplay_structured': 0.8,
  'image_description': 0.7,
  'translation': 1.0,          // intolerante (es un test directo)
  'fill_blank': 1.0,
  'sentence_ordering': 1.0,
  'read_aloud': 1.0,
};
```

### 3.4 Vocabulary scoring

#### Métricas calculadas

```typescript
interface VocabularyMetrics {
  total_unique_words: number;       // tipos únicos
  total_words: number;              // tokens
  type_token_ratio: number;         // riqueza léxica
  cefr_distribution: {              // % por nivel CEFR
    A1: number; A2: number; B1: number;
    B2: number; C1: number; C2: number;
  };
  domain_words_used: number;        // palabras del dominio target
  domain_words_expected: number;    // del ejercicio
  collocations_correct: number;
  collocations_incorrect: number;
}
```

#### Source de datos para CEFR de palabras

Listas open source CEFR-J (vocabulario etiquetado por nivel) +
extensiones propias para vocabulario específico (tech, business). Se
persisten en tabla `vocabulary_cefr_dictionary` (§9) y se actualizan
cuatrimestralmente.

#### Score combinado

```python
def calculate_vocabulary_score(metrics: VocabularyMetrics, target_cefr: str,
                               domain: str | None) -> float:
    # 1. Riqueza léxica (TTR ajustado por longitud)
    ttr_score = adjust_ttr_for_length(metrics.type_token_ratio,
                                       metrics.total_words) * 100

    # 2. Distribución CEFR apropiada
    target_dist = TARGET_CEFR_DIST[target_cefr]
    cefr_match_score = compare_distributions(metrics.cefr_distribution,
                                              target_dist) * 100

    # 3. Vocabulario de dominio (si aplica)
    if domain and metrics.domain_words_expected > 0:
        domain_score = (metrics.domain_words_used /
                        metrics.domain_words_expected) * 100
        domain_score = min(domain_score, 110)  # bonus si excede
    else:
        domain_score = None

    # 4. Collocations
    total_coll = metrics.collocations_correct + metrics.collocations_incorrect
    coll_score = (metrics.collocations_correct / total_coll * 100
                  if total_coll > 0 else 80)

    # Pesos
    if domain_score is not None:
        return weighted_average([
            (ttr_score, 0.30),
            (cefr_match_score, 0.30),
            (domain_score, 0.20),
            (coll_score, 0.20),
        ])
    else:
        return weighted_average([
            (ttr_score, 0.40),
            (cefr_match_score, 0.40),
            (coll_score, 0.20),
        ])
```

### 3.5 Listening scoring

#### Método: ejercicios estructurados con respuestas verificables

| Tipo | Score |
|------|-------|
| Multiple choice | % correctas (binario por pregunta) |
| Comprehension Q&A oral | LLM evalúa respuesta vs key points (0–100) |
| Dictation | 1 - (Levenshtein normalizada al largo del target) |
| Inference | LLM evalúa correctness (0/100, binario) + parcial (50) |

Score del ejercicio = promedio simple de los items.

### 3.6 Manejo de dialectos (decisión cerrada §15)

Cada usuario tiene `target_english_variant ∈ {american, british, neutral}`
en su `student_profile`. El sistema responde así:

| Componente | Comportamiento por variant |
|------------|----------------------------|
| Pronunciation reference model | Switching del modelo acústico de Azure (`en-US` vs `en-GB`); modelo propio Fase 2 entrena ambos. `neutral` usa `en-US` como default. |
| Vocabulary acceptable lists | `truck`/`lorry`, `apartment`/`flat`, `elevator`/`lift` aceptados según variant. La opuesta no penaliza pero genera nota de feedback. |
| Listening exposure | 80% audio en variant target, 20% otra (para exposición). `neutral` → 50/50 US/UK. |
| Spelling (si hay ejercicio escrito) | `color`/`colour` aceptado según variant. |
| Grammar diferencias | `have got` (UK) vs `have` (US): ambos aceptados sin penalty cuando es decisión válida. |

**Implementación:** función `getDialectConfig(variant)` retorna config
unificada que cada scorer consume.

```typescript
interface DialectConfig {
  acoustic_model: 'en-US' | 'en-GB';
  vocabulary_dictionary_id: string;
  listening_exposure_ratio: { primary: number; secondary: number };
  spelling_variant: 'us' | 'uk';
  accepted_grammar_alternatives: string[];  // ej: 'have_got_uk'
}
```

---

## 4. Mastery por bloque

### 4.1 Definición de mastery por bloque

```typescript
interface BlockMasteryCriteria {
  block_id: string;

  // Mínimos para considerar bloque "completed"
  min_attempts: number;            // típico: 1
  min_accuracy: number;            // típico: 0.6

  // Mínimos para considerar bloque "mastered"
  mastered_min_attempts: number;   // típico: 2
  mastered_min_accuracy: number;   // típico: 0.85
  mastered_min_consistency: number;// típico: 0.8 (últimos 3 intentos)

  // Sub-skills target del bloque
  target_subskills: Array<{
    subskill_id: string;
    min_score: number;             // típico: 60
    weight: number;                // 0.0–1.0, suma a 1.0
  }>;
}
```

### 4.2 Estados de un bloque para un usuario

```
LOCKED → AVAILABLE → IN_PROGRESS → COMPLETED → MASTERED
                                 ↓
                             STRUGGLING → AVAILABLE (con ajuste)
                                        → COMPLETED (si remonta)
```

| Estado | Definición | Transición a... |
|--------|-----------|-----------------|
| `locked` | Prerequisites no cumplidos | `available` cuando todos los prereqs están `completed`/`mastered`/`tested_out` |
| `available` | Desbloqueado, no iniciado | `in_progress` al primer attempt |
| `in_progress` | 1+ intentos, no cumple completed | `completed`/`struggling` |
| `completed` | Cumple criterios mínimos para avanzar | `mastered` con más práctica |
| `mastered` | Excede criterios | (terminal en este flujo) |
| `struggling` | Múltiples intentos fallidos | `available` con simplificación, o `completed` si remonta |
| `tested_out` | Saltado vía test-out (§4.5) | (terminal) |

### 4.3 Detección de "struggling"

Un bloque transiciona a `struggling` cuando se cumple **al menos 2** de:

1. `attempts_count >= 5` AND `current_mastery_score < min_accuracy * 100`
2. Score promedio de los **últimos 3 intentos** < 40% del target.
3. Tiempo por intento aumenta monotónicamente en los últimos 3
   (sin mejora de score paralela).
4. Usuario abandonó (`abandoned = true`) en >= 2 de los últimos 3
   intentos.

Cuando un bloque entra en `struggling`:
- Se emite evento `block.struggling_detected` (§12).
- `ai-roadmap-system` evalúa: sugerir prerequisito adicional, ofrecer
  versión simplificada, o pausar el bloque.
- Notification con tono empático: "Este bloque es desafiante. Vamos a
  probarlo de otra manera." (no usar palabras "fallaste",
  "incorrecto").

### 4.4 Progresión adaptativa post-completed

| Mastery score | Acción del roadmap |
|---------------|--------------------|
| `≥ 90%` | Skip bloques similares en sub-skill, avanzar más rápido |
| `70%–90%` | Siguiente bloque normal del roadmap |
| `50%–70%` | Insertar 1 bloque de refuerzo (similar sub-skill, diferente contexto) antes de avanzar |
| `< 50%` | Insertar 2 bloques de refuerzo + revisar prerequisites, considerar regresión |

Esta lógica vive en `ai-roadmap-system` consumiendo eventos de este
sistema.

### 4.5 Test-out voluntario (decisión cerrada §15)

#### Mecánica

Usuario puede solicitar "test out" desde la pantalla de detalle de una
sub-skill o de un bloque. Genera un mini-test de 5 ejercicios curados
para probar dominio rápido.

#### Configuración

```typescript
const TEST_OUT_CONFIG = {
  num_exercises: 5,
  pass_threshold: 4,            // 4/5 correctos para aprobar
  time_limit_minutes: 10,
  cooldown_per_subskill_days: 7,
  cost_in_sparks: 2,            // costo simbólico anti-abuse
  exercises_must_cover: 'all_target_subskills',
};
```

#### Reglas

1. **Cooldown:** un usuario no puede hacer test-out de la misma sub-skill
   hasta 7 días después del intento anterior.
2. **Cost:** 2 Sparks por test-out. Se devuelven si el usuario aprueba.
3. **Pass:** 4/5 correctos. La sub-skill se marca como `mastered` con
   `total_attempts = max(current, 5)` para que no la vuelvan a evaluar
   prematuramente.
4. **Fail:** se reporta fail al usuario sin marcarla como struggling.
   Solo bloquea otro test-out por 7 días.
5. **Bloque:** test-out a nivel bloque requiere aprobar test-out en
   **todas** las sub-skills target del bloque.

#### Por qué cuesta Sparks

Anti-abuse: previene que un usuario haga test-out repetido en sub-skills
para "saltar" todo. El costo simbólico (2 Sparks ≈ $1 MXN) es bajo para
usuarios genuinos pero suficiente para fricción.

### 4.6 Concurrencia y conflictos

#### Casos posibles

1. **Usuario abre el mismo bloque en dos devices simultáneamente.**
   - Ambos clients ven `in_progress` cuando empiezan.
   - El primer `submitExerciseAttempt` que llega gana. Los siguientes
     attempts se siguen aceptando (cada uno es independiente).
   - El cálculo de `current_mastery_score` es eventual consistency: se
     ejecuta en transacción al final de cada attempt.

2. **Dos attempts del mismo ejercicio llegan en <1s.**
   - Cada uno se procesa independientemente (tienen UUIDs distintos).
   - `subskill_changes` se aplica una sola vez (idempotency key =
     `exercise_id + completed_at`).

3. **Usuario cambia su `target_cefr` mientras tiene un bloque
   `in_progress`.**
   - El bloque mantiene su `target_cefr` original (frozen al empezar,
     persistido en `user_block_status`).
   - Próximos bloques se calibran con el nuevo target.

4. **Bloque se actualiza en biblioteca mientras un usuario lo está
   haciendo.**
   - El usuario completa la versión que empezó (snapshot en
     `user_block_status.block_version`).
   - Próximos usuarios reciben la nueva versión.

#### Implementación

Toda actualización del estado de un bloque o sub-skill se hace dentro de
una transacción Postgres con `SELECT ... FOR UPDATE` sobre la fila
relevante.

---

## 5. Tipos de ejercicios

### 5.1 Catálogo de tipos

| Tipo | Descripción | Mide principalmente | STT | LLM |
|------|-------------|---------------------|:-:|:-:|
| `pronunciation_drill` | Repetir frase específica | Pronunciation | Sí | No |
| `shadowing` | Repetir simultáneamente con audio | Pronunciation, Fluency | Sí | No |
| `read_aloud` | Leer texto en voz alta | Pronunciation, Fluency | Sí | No |
| `free_response` | Responder pregunta abierta | All | Sí | Sí |
| `roleplay_structured` | Conversación guiada con IA | All | Sí | Sí |
| `roleplay_free` | Conversación abierta con IA | All | Sí | Sí |
| `listening_mc` | Audio + multiple choice | Listening | No | No |
| `listening_qa` | Audio + preguntas abiertas | Listening, Vocabulary | Sí | Sí |
| `translation` | Traducir frase ES → EN | Grammar, Vocabulary | Opcional | Sí |
| `fill_blank` | Completar frase | Grammar, Vocabulary | No | No |
| `sentence_ordering` | Ordenar palabras | Grammar | No | No |
| `image_description` | Describir imagen en voz alta | Vocabulary, Fluency | Sí | Sí |
| `dictation` | Transcribir audio | Listening | No | No |
| `vocabulary_in_context` | Elegir palabra correcta | Vocabulary | No | No |

### 5.2 Composición de un bloque

Un bloque típico no es un ejercicio único, sino una **secuencia de
3–5 ejercicios** de tipos variados que abordan la misma sub-skill desde
ángulos diferentes.

Ejemplo: bloque `block_past_perfect_storytelling`
1. `listening_mc`: identificar uso correcto de past perfect en audio.
2. `fill_blank`: 5 frases para completar con past perfect.
3. `free_response`: contar una experiencia usando past perfect.
4. `roleplay_structured`: conversación que requiere past perfect.

### 5.3 Algoritmo de espaciado (spaced repetition)

Inspirado en SuperMemo SM-2 y Anki, simplificado:

```typescript
function calculateNextReview(
  blockId: string,
  userId: string,
  lastScore: number,
  reviewCount: number
): Date {
  const intervalsByPerformance = {
    perfect: [1, 3, 7, 14, 30, 60, 120],   // días, score >= 90
    good:    [1, 2, 5, 10, 20, 40, 80],    // 75–89
    fair:    [1, 1, 3, 7, 14, 28, 56],     // 60–74
    poor:    [1, 1, 1, 3, 7, 14, 28],      // < 60
  };

  const performance = scoreToPerformance(lastScore);
  const intervalDays =
    intervalsByPerformance[performance][Math.min(reviewCount, 6)];
  return addDays(new Date(), intervalDays);
}

function scoreToPerformance(score: number): 'perfect' | 'good' | 'fair' | 'poor' {
  if (score >= 90) return 'perfect';
  if (score >= 75) return 'good';
  if (score >= 60) return 'fair';
  return 'poor';
}
```

El job nocturno (en `ai-roadmap-system`) lee
`user_subskill_mastery.next_review_at` y agrega bloques con review
vencida al plan del día.

### 5.4 Matriz ejercicio × dimensión (pesos de contribución)

Cada tipo de ejercicio contribuye con peso distinto al score de cada
dimensión. La matriz exacta:

| Tipo | Pron | Flu | Gram | Vocab | List |
|------|:----:|:---:|:----:|:-----:|:----:|
| `pronunciation_drill` | 1.0 | 0.0 | 0.0 | 0.0 | 0.0 |
| `shadowing` | 0.7 | 0.3 | 0.0 | 0.0 | 0.0 |
| `read_aloud` | 0.6 | 0.4 | 0.0 | 0.0 | 0.0 |
| `free_response` | 0.2 | 0.3 | 0.2 | 0.2 | 0.1 |
| `roleplay_structured` | 0.2 | 0.25 | 0.2 | 0.2 | 0.15 |
| `roleplay_free` | 0.15 | 0.3 | 0.2 | 0.2 | 0.15 |
| `listening_mc` | 0.0 | 0.0 | 0.0 | 0.0 | 1.0 |
| `listening_qa` | 0.0 | 0.1 | 0.1 | 0.2 | 0.6 |
| `translation` | 0.0 | 0.0 | 0.5 | 0.5 | 0.0 |
| `fill_blank` | 0.0 | 0.0 | 0.6 | 0.4 | 0.0 |
| `sentence_ordering` | 0.0 | 0.0 | 1.0 | 0.0 | 0.0 |
| `image_description` | 0.1 | 0.3 | 0.1 | 0.5 | 0.0 |
| `dictation` | 0.0 | 0.0 | 0.0 | 0.1 | 0.9 |
| `vocabulary_in_context` | 0.0 | 0.0 | 0.0 | 1.0 | 0.0 |

(Cada fila suma 1.0.)

Esta matriz se persiste en tabla `exercise_dimension_weights` (§9) y es
configurable. Se usa para distribuir el score del attempt entre las
dimensiones del usuario.

---

## 6. Re-evaluación y mantenimiento

### 6.1 Decay de skills sin uso

```python
def calculate_current_skill_level(
    baseline: float,
    days_since_last_practice: int,
    total_practice_hours: float
) -> float:
    decay_rate_base = 0.005  # 0.5% por día sin práctica
    consolidation_factor = min(total_practice_hours / 100, 1.0)
    # Habilidades muy practicadas decaen menos
    adjusted_decay = decay_rate_base * (1 - consolidation_factor * 0.7)
    decayed = baseline * math.exp(-adjusted_decay * days_since_last_practice)
    # Floor: no decae por debajo del 50% del baseline
    return max(decayed, baseline * 0.5)
```

#### Constantes

```typescript
const DECAY_CONFIG = {
  decay_rate_base: 0.005,
  consolidation_hours_to_max: 100,
  consolidation_max_reduction: 0.7,
  recheck_after_days: 60,           // forzar verificación
  decay_floor_ratio: 0.5,           // mínimo 50% del baseline
};
```

### 6.2 Re-evaluación periódica

Sub-skills marcadas como `mastered` se re-verifican:

- **Cada 30 días** se incluye 1 ejercicio de la sub-skill en el plan
  diario. Job nocturno (`ai-roadmap-system`) detecta y agenda.
- **Si el usuario falla** (score < 75), vuelve a estado `developing` y
  se re-aborda. Evento `subskill.regressed` emitido (§12).
- **Si el usuario pasa**, se actualiza `last_practiced_at` y
  `next_review_at = now() + 30 days`.

### 6.3 Mastery global del usuario

```typescript
interface UserMasteryProfile {
  user_id: string;
  overall_cefr: string;
  dimension_scores: MasteryDimensions;
  subskill_mastery: Record<string, number>;  // subskill_id → score
  last_calculated: Date;

  // Histórico (snapshots semanales)
  cefr_history: Array<{ date: string; level: string }>;
  dimension_history: Array<{ date: string; scores: MasteryDimensions }>;
}
```

Recalculado por job nocturno semanal (domingo 02:00 UTC). Histórico vive
en `weekly_mastery_snapshots` (§9).

---

## 7. Calibración del sistema

### 7.1 Calibración inicial

1. **Recolectar dataset de calibración:** 50–100 usuarios voluntarios
   con nivel CEFR conocido (validado por examen oficial o evaluación
   docente).
2. **Composición target:**
   - 15+ A2, 25+ B1, 25+ B1+, 20+ B2, 10+ B2+, 5+ C1.
   - Distribución geográfica: México 40%, Argentina 25%, Colombia 20%,
     Chile/Perú/otros 15%.
   - Mix de género y rangos etarios.
3. **Hacer que cada uno haga el assessment completo** y registrar todos
   los scores producidos.
4. **Ajustar thresholds** en `mapDimensionsToCEFR` (§2.5) para minimizar
   error. Métrica objetivo: < 1 nivel CEFR de error en el 90% de casos.
5. **Validar:** holdout de 30 voluntarios adicionales.

Persistir thresholds finales en `pedagogical_config` con versión.

### 7.2 Calibración continua

- Usuarios que reportan exámenes oficiales y comparten resultados.
- Comparación con tests externos (Duolingo English Test, IELTS practice).
- A/B testing de ajustes en algoritmos de scoring (variantes en
  `pedagogical_config`).
- Feedback explícito ("este score me parece muy alto/bajo").

### 7.3 Validación pedagógica

Contratación puntual de un experto en lingüística aplicada (consultor
4–8 horas) para revisar:
- Si los criterios de mastery son razonables.
- Si la progresión entre niveles es apropiada.
- Si las sub-skills están bien definidas.
- Si hay sesgos contra ciertos perfiles.

---

## 8. Configuración persistida

Toda constante numérica de las §3, §4, §6 vive en tabla
`pedagogical_config`:

```sql
CREATE TABLE pedagogical_config (
  key             TEXT PRIMARY KEY,
  value           JSONB NOT NULL,
  version         INT NOT NULL DEFAULT 1,
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_by      TEXT,
  notes           TEXT
);

-- Seeds iniciales (ejecutados en migración)
INSERT INTO pedagogical_config (key, value) VALUES
  ('learning_rate.early',           '0.30'),
  ('learning_rate.consolidation',   '0.15'),
  ('learning_rate.refinement',      '0.08'),
  ('cefr_thresholds',
   '{"a2_max":35,"b1_max":55,"b1plus_max":70,"b2_max":82,"b2plus_max":90}'),
  ('dimension_weights_for_cefr',
   '{"pron":0.25,"flu":0.25,"gram":0.20,"vocab":0.15,"list":0.15}'),
  ('decay.rate_base',               '0.005'),
  ('decay.floor_ratio',             '0.5'),
  ('test_out.cost_sparks',          '2'),
  ('test_out.cooldown_days',        '7'),
  ('test_out.pass_threshold',       '4');
```

Cambios a `pedagogical_config` en producción **requieren A/B test**
previo (rollout 5% → 25% → 100%).

---

## 9. Schema en Postgres

### 9.1 Tablas core

```sql
-- Sub-skills catalog
CREATE TABLE subskills_catalog (
  id              TEXT PRIMARY KEY,
  dimension       TEXT NOT NULL CHECK (dimension IN (
    'pronunciation', 'fluency', 'grammar', 'vocabulary', 'listening'
  )),
  name            TEXT NOT NULL,
  description     TEXT,
  cefr_introduction TEXT NOT NULL CHECK (cefr_introduction IN (
    'A1','A2','B1','B1+','B2','B2+','C1','C2'
  )),
  difficulty      INT NOT NULL CHECK (difficulty BETWEEN 1 AND 10),
  prerequisites   TEXT[] NOT NULL DEFAULT '{}',
  is_active       BOOLEAN NOT NULL DEFAULT true,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subskills_dimension ON subskills_catalog(dimension)
  WHERE is_active = true;
CREATE INDEX idx_subskills_cefr ON subskills_catalog(cefr_introduction)
  WHERE is_active = true;

-- Mastery del usuario por sub-skill
CREATE TABLE user_subskill_mastery (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  subskill_id     TEXT NOT NULL REFERENCES subskills_catalog(id),
  current_score   FLOAT NOT NULL CHECK (current_score BETWEEN 0 AND 100),
  level           TEXT NOT NULL CHECK (level IN (
    'unfamiliar', 'learning', 'developing', 'proficient', 'mastered'
  )),
  total_attempts  INT NOT NULL DEFAULT 0,
  successful_attempts INT NOT NULL DEFAULT 0,
  last_practiced_at TIMESTAMPTZ,
  last_evaluated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  next_review_at  TIMESTAMPTZ,
  PRIMARY KEY (user_id, subskill_id)
);

CREATE INDEX idx_subskill_mastery_review
  ON user_subskill_mastery(next_review_at)
  WHERE next_review_at IS NOT NULL;
CREATE INDEX idx_subskill_mastery_user_level
  ON user_subskill_mastery(user_id, level);

-- Resultados de cada intento de ejercicio
CREATE TABLE exercise_attempts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  block_id        TEXT NOT NULL REFERENCES learning_blocks(id),
  block_version   TEXT NOT NULL,        -- snapshot del bloque
  exercise_id     TEXT NOT NULL,
  exercise_type   TEXT NOT NULL,
  started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at    TIMESTAMPTZ,
  abandoned       BOOLEAN NOT NULL DEFAULT false,
  scores          JSONB NOT NULL DEFAULT '{}',  -- por dimensión
  detailed_results JSONB,                       -- errores, audio refs
  audio_storage_key TEXT,                       -- ref a R2
  duration_seconds INT,
  device          TEXT                          -- 'ios','android','web'
);

CREATE INDEX idx_attempts_user_block
  ON exercise_attempts(user_id, block_id);
CREATE INDEX idx_attempts_user_date
  ON exercise_attempts(user_id, completed_at DESC NULLS LAST);
CREATE INDEX idx_attempts_block_id_date
  ON exercise_attempts(block_id, completed_at DESC NULLS LAST);

-- Estado del bloque para cada usuario
CREATE TABLE user_block_status (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  block_id        TEXT NOT NULL REFERENCES learning_blocks(id),
  block_version   TEXT NOT NULL,
  target_cefr     TEXT NOT NULL,        -- frozen al empezar
  status          TEXT NOT NULL CHECK (status IN (
    'locked', 'available', 'in_progress', 'completed',
    'mastered', 'struggling', 'tested_out'
  )),
  attempts_count  INT NOT NULL DEFAULT 0,
  current_mastery_score FLOAT NOT NULL DEFAULT 0,
  first_attempt_at TIMESTAMPTZ,
  completed_at    TIMESTAMPTZ,
  mastered_at     TIMESTAMPTZ,
  struggling_since TIMESTAMPTZ,
  last_attempt_at TIMESTAMPTZ,
  next_review_at  TIMESTAMPTZ,
  PRIMARY KEY (user_id, block_id)
);

CREATE INDEX idx_block_status_user_status
  ON user_block_status(user_id, status);
CREATE INDEX idx_block_status_review
  ON user_block_status(next_review_at)
  WHERE next_review_at IS NOT NULL AND status = 'mastered';
```

### 9.2 Tablas auxiliares

```sql
-- Pesos por tipo de ejercicio (§5.4)
CREATE TABLE exercise_dimension_weights (
  exercise_type   TEXT PRIMARY KEY,
  pronunciation   FLOAT NOT NULL DEFAULT 0,
  fluency         FLOAT NOT NULL DEFAULT 0,
  grammar         FLOAT NOT NULL DEFAULT 0,
  vocabulary      FLOAT NOT NULL DEFAULT 0,
  listening       FLOAT NOT NULL DEFAULT 0,
  CHECK (
    pronunciation + fluency + grammar + vocabulary + listening
    BETWEEN 0.99 AND 1.01
  )
);

-- Patrones de error de pronunciación (§3.1)
CREATE TABLE pronunciation_error_patterns (
  id              SERIAL PRIMARY KEY,
  pattern_name    TEXT UNIQUE NOT NULL,
  expected_phoneme TEXT NOT NULL,
  produced_phoneme TEXT NOT NULL,
  example_word    TEXT,
  affected_subskill TEXT NOT NULL REFERENCES subskills_catalog(id),
  feedback_template TEXT NOT NULL    -- mensaje al usuario
);

-- Diccionario CEFR de vocabulario (§3.4)
CREATE TABLE vocabulary_cefr_dictionary (
  word            TEXT PRIMARY KEY,
  cefr_level      TEXT NOT NULL CHECK (cefr_level IN (
    'A1','A2','B1','B2','C1','C2'
  )),
  pos             TEXT,            -- noun, verb, etc.
  variant         TEXT             -- 'us'|'uk'|'both'
);

CREATE INDEX idx_vocab_cefr ON vocabulary_cefr_dictionary(cefr_level);

-- Snapshots semanales para visualización
CREATE TABLE weekly_mastery_snapshots (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  snapshot_date   DATE NOT NULL,
  overall_cefr    TEXT NOT NULL,
  pronunciation   FLOAT NOT NULL,
  fluency         FLOAT NOT NULL,
  grammar         FLOAT NOT NULL,
  vocabulary      FLOAT NOT NULL,
  listening       FLOAT NOT NULL,
  PRIMARY KEY (user_id, snapshot_date)
);

-- Test-out attempts (§4.5)
CREATE TABLE test_out_attempts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  subskill_id     TEXT REFERENCES subskills_catalog(id),
  block_id        TEXT REFERENCES learning_blocks(id),
  scope           TEXT NOT NULL CHECK (scope IN ('subskill', 'block')),
  started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at    TIMESTAMPTZ,
  passed          BOOLEAN,
  exercises_correct INT,
  exercises_total INT,
  sparks_charged  INT NOT NULL DEFAULT 0,
  sparks_refunded INT NOT NULL DEFAULT 0,
  CHECK (
    (scope = 'subskill' AND subskill_id IS NOT NULL AND block_id IS NULL) OR
    (scope = 'block' AND block_id IS NOT NULL AND subskill_id IS NULL)
  )
);

CREATE INDEX idx_test_out_user_recent
  ON test_out_attempts(user_id, started_at DESC);
```

### 9.3 Reglas de cascade

- `users` → cascade delete propaga a `user_subskill_mastery`,
  `exercise_attempts`, `user_block_status`, `weekly_mastery_snapshots`,
  `test_out_attempts`.
- `subskills_catalog` → **NO** cascade. Sub-skill `is_active = false` se
  retira gradualmente.
- `learning_blocks` → **NO** cascade. Bloques deprecados se mantienen
  para usuarios que ya los iniciaron.

---

## 10. API contracts

Estos son los puntos de entrada/salida del sistema. Otros sistemas
**solo** consumen estos endpoints, no acceden directamente a las tablas.

### 10.1 `submitExerciseAttempt`

**Llamado por:** cliente móvil/web cuando el usuario completa un
ejercicio.

**Request:**

```typescript
interface SubmitExerciseAttemptRequest {
  user_id: string;
  block_id: string;
  exercise_id: string;
  exercise_type: ExerciseType;
  started_at: string;        // ISO timestamp
  completed_at: string;
  abandoned: boolean;
  payload: ExercisePayload;  // shape varía por exercise_type
  device: 'ios' | 'android' | 'web';
}

type ExercisePayload =
  | { type: 'audio'; audio_storage_key: string; expected_text?: string }
  | { type: 'text_choice'; selected_option: string }
  | { type: 'text_input'; text: string }
  | { type: 'ordering'; ordered_items: string[] };
```

**Response:**

```typescript
interface SubmitExerciseAttemptResponse {
  attempt_id: string;
  scores: Partial<MasteryDimensions>;
  feedback: {
    summary: string;
    errors_detected: GrammarError[] | PronunciationError[];
    encouragement: string;
  };
  block_status_after: {
    status: BlockStatus;
    mastery_score: number;
    progress_pct: number;
  };
  subskill_changes: Array<{
    subskill_id: string;
    previous_score: number;
    new_score: number;
    new_level: MasteryLevel;
  }>;
  events_emitted: string[];   // para debugging
}
```

**Errores:**

| Código | Causa |
|--------|-------|
| `400 INVALID_PAYLOAD` | `payload` no matchea `exercise_type` |
| `402 INSUFFICIENT_SPARKS` | scoring requiere Sparks y balance es 0 |
| `404 BLOCK_NOT_FOUND` | `block_id` no existe |
| `409 BLOCK_LOCKED` | usuario no tiene este bloque desbloqueado |
| `422 AUDIO_UNUSABLE` | audio < 1s útil o ruido excesivo |
| `503 SCORING_UNAVAILABLE` | downstream caído, fallback no disponible |

### 10.2 `getMasteryProfile`

**Llamado por:** UI de perfil del usuario, `motivation-and-achievements`,
`ai-roadmap-system`.

**Request:**

```typescript
interface GetMasteryProfileRequest {
  user_id: string;
  include_history?: boolean;  // default false
  history_weeks?: number;     // default 12
}
```

**Response:**

```typescript
interface GetMasteryProfileResponse {
  user_id: string;
  overall_cefr: string;
  dimension_scores: MasteryDimensions;
  subskill_mastery: Record<string, {
    score: number;
    level: MasteryLevel;
    last_practiced_at: string | null;
    next_review_at: string | null;
  }>;
  last_calculated: string;
  history?: {
    cefr: Array<{ date: string; level: string }>;
    dimensions: Array<{ date: string; scores: MasteryDimensions }>;
  };
}
```

### 10.3 `requestTestOut`

**Llamado por:** UI de detalle de sub-skill o bloque.

**Request:**

```typescript
interface RequestTestOutRequest {
  user_id: string;
  scope: 'subskill' | 'block';
  target_id: string;          // subskill_id o block_id
}
```

**Response:**

```typescript
interface RequestTestOutResponse {
  test_out_id: string;
  exercises: Array<{
    exercise_id: string;
    exercise_type: ExerciseType;
    payload: any;             // del bloque
  }>;
  expires_at: string;          // started_at + 10 min
  cost_sparks: number;
}
```

**Errores:**

| Código | Causa |
|--------|-------|
| `409 COOLDOWN_ACTIVE` | < 7 días desde último test-out |
| `409 ALREADY_MASTERED` | sub-skill ya está mastered |
| `402 INSUFFICIENT_SPARKS` | < 2 Sparks |
| `404 NOT_FOUND` | sub-skill o bloque no existe |

### 10.4 `getNextBlocksForUser`

**Llamado por:** `ai-roadmap-system` cuando arma el plan del día.

**Request:**

```typescript
interface GetNextBlocksRequest {
  user_id: string;
  max_blocks?: number;     // default 5
  include_reviews: boolean;
}
```

**Response:**

```typescript
interface GetNextBlocksResponse {
  blocks: Array<{
    block_id: string;
    reason: 'next_in_roadmap' | 'spaced_review' |
            'reinforcement' | 'struggling_recovery';
    estimated_minutes: number;
    target_subskills: string[];
  }>;
}
```

### 10.5 `recalculateMasteryGlobal`

**Llamado por:** job nocturno semanal (Inngest).

```typescript
async function recalculateMasteryGlobal(userId: string): Promise<void>
```

Recalcula `overall_cefr` y `dimension_scores`, persiste snapshot
semanal, emite evento `user.mastery_recalculated`.

---

## 11. Edge cases (tests obligatorios)

Cada uno debe tener test de integración explícito.

### 11.1 Audio

1. **Audio silencioso** (< -50dB durante > 80% de la grabación):
   `submitExerciseAttempt` retorna `422 AUDIO_UNUSABLE`. No se cobra
   Sparks. No se crea `exercise_attempts` row.
2. **Audio ruidoso pero usable** (SNR bajo pero hay habla detectable):
   se procesa con warning en `feedback.summary` ("audio con ruido,
   intentá en lugar más silencioso").
3. **Audio cortado mid-recording** (cliente perdió red): el cliente debe
   reintentar el upload. El server acepta upload tardío hasta 60s
   después de `completed_at`.
4. **STT devuelve texto vacío:** se trata como abandono, `abandoned =
   true`, no se otorga score, sub-skills no se actualizan.

### 11.2 Concurrencia

5. **Doble submit del mismo `exercise_id` en <1s:** ambos se procesan
   pero `subskill_changes` se aplica una sola vez (idempotency key =
   `exercise_id + completed_at`).
6. **Submit mientras un job nocturno corre sobre el mismo usuario:** el
   submit toma `FOR UPDATE` lock; el job nocturno espera o reintenta.

### 11.3 Validación

7. **Payload no matchea exercise_type:** `400 INVALID_PAYLOAD`. Ej:
   exercise_type `listening_mc` con payload de tipo audio.
8. **`block_version` enviado por cliente está obsoleto:** el server
   acepta el attempt pero usa el `block_version` que el cliente declaró
   (frozen). Si la versión actual difiere, marca el attempt con
   `version_drift = true` para análisis.

### 11.4 Estados

9. **Submit a un bloque `locked`:** `409 BLOCK_LOCKED`.
10. **Test-out durante un bloque `in_progress`:** permitido. Si el
    test-out aprueba, el bloque salta a `tested_out` y los attempts
    in-progress se descartan (no contribuyen a sub-skills).
11. **Bloque pasa a `mastered` durante un attempt:** el attempt se
    sigue procesando normalmente; sus scores actualizan sub-skills (no
    afectan estado del bloque que ya está terminal).

### 11.5 Decay y review

12. **Sub-skill no practicada por > 60 días pero usuario completa
    re-evaluación con score alto:** `current_score` se restaura al
    valor pre-decay (no al baseline original).
13. **Re-evaluación falla con score muy bajo (< 50):** sub-skill
    regresa a `learning` (no `unfamiliar`, evitar loss aversion brusca).

### 11.6 Test-out

14. **Test-out fail:** sub-skill no cambia. `cooldown_until` se setea a
    `now + 7 days`. No se devuelven Sparks.
15. **Test-out abandonado (timeout 10min):** se cierra como fail, mismo
    cooldown. Sparks NO se devuelven (anti-abuse).
16. **Test-out de bloque pero solo algunas sub-skills aprueban:** bloque
    NO pasa a `tested_out`. Las sub-skills aprobadas SÍ se marcan como
    mastered.

---

## 12. Telemetry / eventos emitidos

Eventos al bus de eventos (Inngest events). Otros sistemas se suscriben
para reaccionar.

### 12.1 Eventos de attempt

| Event | Properties | Sinks |
|-------|-----------|-------|
| `exercise.attempt_started` | `user_id, block_id, exercise_id, exercise_type, started_at` | PostHog |
| `exercise.attempt_completed` | `attempt_id, user_id, block_id, exercise_type, scores, duration_seconds, device` | PostHog, internal log |
| `exercise.attempt_abandoned` | `attempt_id, user_id, block_id, partial_duration_seconds` | PostHog |

### 12.2 Eventos de bloque

| Event | Properties | Sinks |
|-------|-----------|-------|
| `block.started` | `user_id, block_id, block_version` | PostHog |
| `block.completed` | `user_id, block_id, mastery_score, attempts_count` | PostHog, ai-roadmap, motivation |
| `block.mastered` | `user_id, block_id, mastery_score` | PostHog, ai-roadmap, motivation |
| `block.struggling_detected` | `user_id, block_id, signals: string[]` | PostHog, ai-roadmap |
| `block.tested_out` | `user_id, block_id, test_out_id` | PostHog, motivation |

### 12.3 Eventos de sub-skill

| Event | Properties | Sinks |
|-------|-----------|-------|
| `subskill.level_up` | `user_id, subskill_id, previous_level, new_level, score` | PostHog, motivation |
| `subskill.regressed` | `user_id, subskill_id, previous_level, new_level, reason` | PostHog, internal log |
| `subskill.review_due` | `user_id, subskill_id, days_since_practice` | ai-roadmap |

### 12.4 Eventos de mastery

| Event | Properties | Sinks |
|-------|-----------|-------|
| `user.dimension_changed` | `user_id, dimension, previous_score, new_score` | PostHog |
| `user.cefr_changed` | `user_id, previous_cefr, new_cefr` | PostHog, motivation, notifications |
| `user.mastery_recalculated` | `user_id, snapshot: MasteryDimensions` | internal log |

### 12.5 Convenciones

- Naming: `<entity>.<action_past_tense>`.
- `user_id` siempre hasheado (SHA-256) antes de enviar a PostHog.
- Properties nunca incluyen audios crudos, transcripciones completas, ni
  PII directa.
- Eventos críticos (CEFR change, struggling) también van a internal log
  (`event_log` table) para auditoría.

---

## 13. Métricas del sistema

### 13.1 Métricas de calidad pedagógica

| Métrica | Definición | Target |
|---------|-----------|--------|
| Mastery accuracy | Re-test de sub-skills mastered: ¿score sigue ≥ 85? | > 90% |
| CEFR alignment | vs CEFR oficiales conocidos | < 1 nivel error en 90% |
| Progress reality | Mejora del sistema vs benchmark externo (DET, IELTS practice) | correlación > 0.7 |
| Struggling detection lead time | Días desde primera señal hasta intervención | < 2 días |
| Calibration drift | Desviación del CEFR del sistema vs usuarios validados externamente | < 5% mensual |

### 13.2 Métricas de salud

| Métrica | Definición | Alerta si... |
|---------|-----------|--------------|
| Distribución de mastery levels | Por sub-skill | bimodal extremo (>40% en `unfamiliar` Y >40% en `mastered`) |
| Tasa de bloques `struggling` | % de bloques activos | > 15% global, > 30% en un mismo bloque |
| Bloques completed pero no mastered | % de transiciones que se quedan | > 50% (criterios irreales) |
| Velocidad de progresión | Días promedio para subir de CEFR | divergencia > 50% del estimado |
| Test-out usage | % de sub-skills marcadas vía test-out | > 30% (posible gaming) |

### 13.3 Alertas críticas

- Bloque con `struggling_count / attempts_count > 0.4` durante 7 días:
  bloque mal calibrado, escalar.
- Sub-skill que nunca se mastered por nadie en 30 días: criterios
  irreales.
- CEFR del sistema diverge consistentemente > 1 nivel del CEFR oficial
  conocido en 5+ usuarios: re-calibrar.
- Audio rejection rate (`422 AUDIO_UNUSABLE`) > 10%: problema de UX o
  bug del cliente.

---

## 14. Plan de implementación

### 14.1 Fase 1: MVP (meses 0–3)

**Crítico (Sprint 1–4):**
- Schemas §9 completos en migración inicial.
- Seed `subskills_catalog` con 50 entries de §2.4.
- Seed `exercise_dimension_weights` con matriz §5.4.
- Seed `pronunciation_error_patterns` con tabla §3.1.
- Pronunciation scoring vía Azure API (task `score_pronunciation` en
  AI Gateway).
- Fluency scoring básico (WPM, pausas, fillers).
- Grammar scoring vía LLM (task `detect_grammar_errors`).
- API contracts §10 implementados (5 endpoints).
- Estados de bloque §4.2 con transiciones.
- Cálculo de mastery por dimensión §2.5.
- Detección de `struggling` §4.3.
- Eventos §12 emitidos.

**Importante (Sprint 5–6):**
- Algoritmo de espaciado §5.3.
- Decay de skills §6.1.
- Re-evaluación periódica §6.2.
- Test-out §4.5.
- Edge cases §11 con tests de integración.
- Calibración inicial §7.1.

### 14.2 Fase 2: Sofisticación (meses 3–9)

- Modelo propio de pronunciation scoring (cuando spend Azure > $2.000/mes).
- Catálogo expandido de sub-skills (50 → 200+).
- Calibración continua con datos reales.
- Validación pedagógica con experto externo.
- A/B testing de algoritmos vía `pedagogical_config` versions.
- Manejo robusto de dialectos (Fase 1: solo American/Neutral; Fase 2:
  British completo).

### 14.3 Fase 3: Excelencia (año 2+)

- Modelos de ML propios para todas las dimensiones.
- Personalización de criterios por perfil de usuario.
- Predicción de tiempo a mastery por sub-skill.
- Generación dinámica de ejercicios para llenar gaps específicos.

---

## 15. Decisiones cerradas

Decisiones que estaban abiertas en v1.0 y se cierran ahora con
justificación:

### 15.1 Cantidad inicial de sub-skills: **50** ✓

**Razón:** cubre ~80% del curriculum B1–B2 (audiencia primaria), permite
calibración rápida, ratio de ejercicios por sub-skill (700 / 50 = 14)
es saludable. Expansión a 200+ en Fase 2 según gaps observados en
producción.

### 15.2 Test-out voluntario: **Sí, con restricciones** ✓

**Razón:** respeta autonomía (Self-Determination Theory), evita tedio
para usuarios sobre el nivel, genera datos rápidos de assessment.
Restricciones (cooldown 7 días, costo 2 Sparks, 4/5 para aprobar)
previenen gaming. Ver §4.5 para detalle.

### 15.3 Mostrar scores por dimensión al usuario: **Sí, con explicación** ✓

**Razón:** transparencia genera confianza. Una sola métrica oculta
información valiosa para el usuario. Explicación contextual (qué
significa cada dimensión) evita confusión.

### 15.4 Migración de Azure Pronunciation Assessment: **Cuando spend > $2.000/mes** ✓

**Razón:** trigger numérico explícito. Hasta entonces, costo de Azure
es manejable y enfoque va al producto, no a infra de ML. Pre-requisito:
dataset de 5.000+ audios etiquetados ya recolectado.

### 15.5 Manejo de dialectos: **`target_english_variant` por usuario** ✓

**Razón:** dialecto es preferencia personal con implicaciones múltiples
(scoring, vocabulary, listening). Ver §3.6 para tabla completa.
Default por país documentado en
`student-profile-and-assessment.md` §8.

---

## 16. Referencias internas

Documentos de los que este sistema **consume** o **emite hacia**:

| Documento | Relación |
|-----------|----------|
| [`product/student-profile-and-assessment.md`](student-profile-and-assessment.md) | Provee `target_english_variant`, `target_cefr`, perfil del usuario. |
| [`product/ai-roadmap-system.md`](ai-roadmap-system.md) | Consume `getMasteryProfile`, `getNextBlocksForUser`. Reacciona a eventos `block.*`, `subskill.*`. |
| [`product/motivation-and-achievements.md`](motivation-and-achievements.md) | Consume eventos para detectar logros y otorgar Sparks bonus. |
| [`product/content-creation-system.md`](content-creation-system.md) | Define `learning_blocks`, `target_subskills` por bloque. |
| [`architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) | Encapsula tasks: `transcribe_user_audio`, `score_pronunciation`, `detect_grammar_errors`, `evaluate_listening_response`. |
| [`architecture/sparks-system.md`](../architecture/sparks-system.md) | Cobra Sparks por scoring (vía AI Gateway) y por test-out. |
| [`architecture/notifications-system.md`](../architecture/notifications-system.md) | Consume `user.cefr_changed` y `block.struggling_detected` para mensajes. |

---

*Documento vivo. Actualizar cuando se calibren métricas, cambien
algoritmos o se descubran problemas pedagógicos en producción.
Cualquier cambio a §9 (schemas) requiere migración Postgres
correspondiente y actualización sincronizada de §10 (API contracts).*

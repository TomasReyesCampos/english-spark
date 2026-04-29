# Pedagogical System and Mastery Evaluation

> Sistema pedagógico que define qué es "saber" en cada bloque, cómo se
> evalúa el dominio, qué criterios determinan progresión, y cómo se
> garantiza que el usuario realmente está mejorando (no solo completando).

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Alcance:** Sistema completo

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

Cada dimensión se mide independientemente. Un usuario puede tener pronunciación
B2 y gramática B1, lo cual es información valiosa para personalización.

### 2.2 Sub-skills por dimensión

Cada dimensión se descompone en sub-skills concretas evaluables:

#### Pronunciation
- Fonemas individuales: /θ/, /ð/, /v/, /ʒ/, /ʃ/, etc.
- Vocales tensas vs laxas: /iː/ vs /ɪ/, /uː/ vs /ʊ/
- Patrones de stress (énfasis silábico).
- Entonación de preguntas vs afirmaciones.
- Linking entre palabras (connected speech).
- Reducción de vocales (schwa).

#### Fluency
- Velocidad (palabras por minuto, target 120-150 para B2).
- Pausas (frecuencia, duración).
- Muletillas (uh, uhm, like, you know).
- Auto-correcciones excesivas.
- Coherencia discursiva (uso de conectores).
- Capacidad de mantener turnos largos.

#### Grammar
- Tiempos verbales: present perfect vs past simple, conditionals.
- Artículos: a/an/the (especialmente difícil para hispanohablantes).
- Preposiciones: in/on/at, dependencies.
- Word order: questions, relative clauses.
- Concordancia subject-verb.
- Phrasal verbs.

#### Vocabulary
- Tamaño del vocabulario activo.
- Precisión léxica (la palabra correcta para el contexto).
- Variedad (no repetir las mismas palabras).
- Vocabulario específico del dominio (tech, business, casual).
- Idioms y expresiones naturales.
- Collocations (run a meeting, not "do a meeting").

#### Listening
- Comprensión a velocidad nativa.
- Comprensión con ruido de fondo.
- Comprensión de acentos variados.
- Captura de detalles vs idea general.
- Inferencia (lo no dicho explícitamente).

### 2.3 Niveles de mastery por sub-skill

| Nivel | Score | Descripción | Acción del sistema |
|-------|-------|-------------|---------------------|
| Unfamiliar | 0-30 | No domina la sub-skill | Introducir gradualmente |
| Learning | 31-60 | Practicando, errores frecuentes | Práctica intensiva, feedback corrector |
| Developing | 61-80 | Domina mayormente, errores ocasionales | Refuerzo, contextos variados |
| Proficient | 81-95 | Dominio sólido, errores raros | Mantenimiento ocasional |
| Mastered | 96-100 | Dominio completo y consistente | Test out, no requiere más práctica |

---

## 3. Métodos de medición

### 3.1 Pronunciation scoring

#### Método: análisis acústico de fonemas

Para cada audio del usuario:

1. Forced alignment con el texto esperado (alinea audio con palabras).
2. Extracción de cada fonema individualmente.
3. Comparación con modelos de referencia de hablantes nativos.
4. Score por fonema en escala 0-100.
5. Agregación a nivel palabra y oración.

#### Implementación

**Fase 1 (MVP):** Azure Pronunciation Assessment API.
- Pros: producción-ready, calibrado, soporte multilingüe.
- Costos: ~$0.01-0.03 USD por minuto evaluado.
- Cons: API externa con costo escalando con uso.

**Fase 2 (post-validación):** modelo propio fine-tuneado.
- Base: Whisper o wav2vec2 open source.
- Fine-tuning con audios de hispanohablantes etiquetados.
- Inferencia self-hosted en Modal o GPU propia.
- Costo marginal mucho menor a escala.

#### Calibración para hispanohablantes

Errores típicos a detectar específicamente:
- /θ/ pronunciado como /s/ o /t/ (think → sink)
- /ð/ pronunciado como /d/ (this → dis)
- /v/ confundido con /b/ (very → bery)
- /ʒ/ confundido con /j/ o /sh/ (measure → mesha)
- Vocal /ɪ/ pronunciada como /iː/ (ship → sheep)
- /æ/ pronunciada como /a/ (cat → cat con vocal española)
- Schwa pronunciada como vocal completa (about → "a-bout" en lugar de
  "ə-bout")
- Final consonants omitidas (cold → col, walked → walk)

### 3.2 Fluency scoring

#### Método: análisis temporal de la producción

Métricas calculadas:

```typescript
interface FluencyMetrics {
  words_per_minute: number;
  pause_frequency: number;          // pausas por minuto
  pause_avg_duration_ms: number;
  filler_word_count: number;        // uh, uhm, like
  self_correction_count: number;
  speech_rate_consistency: number;  // varianza en WPM
  utterance_length_avg: number;     // palabras por turno
}
```

#### Cálculo del score

Algoritmo conceptual:

```python
def calculate_fluency_score(metrics, target_cefr_level):
    targets = FLUENCY_TARGETS[target_cefr_level]

    wpm_score = score_against_target(
        metrics.words_per_minute,
        targets.wpm_target,
        targets.wpm_acceptable_range
    )

    pause_score = penalize_pauses(
        metrics.pause_frequency,
        metrics.pause_avg_duration_ms
    )

    filler_score = penalize_fillers(metrics.filler_word_count)

    consistency_score = reward_consistency(metrics.speech_rate_consistency)

    return weighted_average([
        (wpm_score, 0.30),
        (pause_score, 0.30),
        (filler_score, 0.25),
        (consistency_score, 0.15)
    ])
```

#### Calibración por nivel CEFR

| CEFR | WPM target | Pauses/min target | Fillers/min target |
|------|-----------|-------------------|---------------------|
| B1 | 80-100 | < 8 | < 6 |
| B2 | 110-130 | < 5 | < 4 |
| C1 | 130-160 | < 3 | < 2 |

Un B1 con 95 WPM tiene buen score en su nivel. Un C1 con 95 WPM tiene
score bajo (no es nivel apropiado).

### 3.3 Grammar scoring

#### Método: análisis automático de errores

Pipeline:

1. STT del audio del usuario.
2. Análisis sintáctico de la transcripción.
3. Detección de errores con LLM especializado o tools (LanguageTool, Grammarly API).
4. Categorización de errores por tipo y severidad.
5. Score basado en frecuencia y gravedad.

#### Severidad de errores

```typescript
interface GrammarError {
  type: string;          // 'past_perfect_misuse', 'article_omission', etc.
  severity: 'minor' | 'moderate' | 'major';
  position: number;
  suggestion: string;
}
```

| Severidad | Definición | Penalización |
|-----------|-----------|--------------|
| Minor | No afecta comprensión, sería natural en native speaker | -1 punto |
| Moderate | Notable pero comprensible | -3 puntos |
| Major | Causa confusión o malentendido | -10 puntos |

#### Calibración por contexto

Errores aceptables en speech rápido espontáneo no son aceptables en
escritura formal. Sistema calibra según tipo de ejercicio.

### 3.4 Vocabulary scoring

#### Métricas calculadas

```typescript
interface VocabularyMetrics {
  total_unique_words: number;       // tipos únicos
  total_words: number;              // tokens
  type_token_ratio: number;         // riqueza léxica
  cefr_distribution: {              // % por nivel CEFR de cada palabra
    A1: number;
    A2: number;
    B1: number;
    B2: number;
    C1: number;
    C2: number;
  };
  domain_words_used: number;        // palabras del dominio target
  collocations_correct: number;
  collocations_incorrect: number;
}
```

#### Score combinado

- Riqueza léxica (TTR): 30%
- Distribución CEFR apropiada al nivel: 30%
- Vocabulario de dominio (si aplica): 20%
- Collocations correctas: 20%

### 3.5 Listening scoring

#### Método: ejercicios estructurados con respuestas verificables

Los ejercicios de listening tienen formato controlado:

**Multiple choice:**
- Audio + 4 opciones.
- Score: simple, % correcto.

**Comprehension Q&A:**
- Audio + preguntas abiertas.
- User responde verbalmente o por texto.
- Sistema evalúa si la respuesta refleja comprensión.

**Dictation:**
- Audio + transcripción del usuario.
- Score: similitud con transcripción esperada (Levenshtein normalizada).

**Inference questions:**
- Audio + pregunta sobre algo no explícito.
- Sistema evalúa con LLM si la inferencia es correcta.

---

## 4. Mastery por bloque

### 4.1 Definición de mastery por bloque

Cada `learning_block` tiene criterios específicos de mastery:

```typescript
interface BlockMasteryCriteria {
  block_id: string;

  // Mínimos para considerar bloque "completed"
  min_attempts: number;
  min_accuracy: number;            // % correctas

  // Mínimos para considerar bloque "mastered"
  mastered_min_attempts: number;
  mastered_min_accuracy: number;
  mastered_min_consistency: number; // accuracy en últimos N intentos

  // Sub-skills target del bloque
  target_subskills: Array<{
    subskill_id: string;
    min_score: number;
  }>;
}
```

### 4.2 Estados de un bloque para un usuario

```
LOCKED → AVAILABLE → IN_PROGRESS → COMPLETED → MASTERED
                                ↓
                            STRUGGLING → AVAILABLE (con ajuste)
```

| Estado | Definición |
|--------|-----------|
| Locked | Prerequisites no cumplidos |
| Available | Desbloqueado, no iniciado |
| In Progress | 1+ intentos hechos, no cumple completed |
| Completed | Cumple criterios mínimos para avanzar |
| Mastered | Excede criterios, no requiere más práctica |
| Struggling | Múltiples intentos fallidos, necesita intervención |

### 4.3 Detección de "struggling"

Si un usuario:
- Hace 5+ intentos en un bloque sin alcanzar `min_accuracy`.
- Score promedio es < 40% del target.
- Tiempo por intento aumenta sin mejora.

El bloque se marca como `STRUGGLING` y el sistema:
- Sugiere bloques pre-requisitos que tal vez faltó dominar.
- Ofrece versión simplificada del contenido.
- Notifica al usuario sin culpa: "este bloque es desafiante, probemos
  algo distinto primero".

### 4.4 Progresión adaptativa

Cuando un usuario completa un bloque, el sistema decide qué viene:

**Si mastery alto (> 90%):** salta bloques similares, avanza más rápido.

**Si mastery medio (70-90%):** sigue al siguiente bloque normal.

**Si mastery bajo (50-70%):** agrega bloques de refuerzo antes de avanzar.

**Si mastery muy bajo (< 50%):** repite con contexto distinto, busca raíz
del problema.

---

## 5. Tipos de ejercicios

### 5.1 Catálogo de tipos

| Tipo | Descripción | Mide principalmente |
|------|-------------|---------------------|
| Pronunciation drill | Repetir frase específica | Pronunciation |
| Shadowing | Repetir simultáneamente con audio | Pronunciation, Fluency |
| Read aloud | Leer texto en voz alta | Pronunciation, Fluency |
| Free response | Responder pregunta abierta | All |
| Roleplay structured | Conversación guiada con IA | All |
| Roleplay free | Conversación abierta con IA | All |
| Listening MC | Audio + multiple choice | Listening |
| Listening Q&A | Audio + preguntas abiertas | Listening, Vocabulary |
| Translation | Traducir frase español → inglés | Grammar, Vocabulary |
| Fill in the blank | Completar frase con palabra correcta | Grammar, Vocabulary |
| Sentence ordering | Ordenar palabras correctamente | Grammar |
| Image description | Describir imagen en voz alta | Vocabulary, Fluency |
| Dictation | Transcribir audio escuchado | Listening |
| Vocabulary in context | Elegir palabra correcta para oración | Vocabulary |

### 5.2 Composición de un bloque

Un bloque típico no es un ejercicio único, es una **secuencia de ejercicios
de tipos variados** que abordan la misma sub-skill desde ángulos diferentes.

Ejemplo: bloque "Past Perfect in storytelling"

1. Listening MC: identificar uso correcto de past perfect en audio.
2. Fill in the blank: 5 frases para completar con past perfect.
3. Free response: contar una experiencia usando past perfect.
4. Roleplay structured: conversación que requiere past perfect naturalmente.

Esta variedad asegura que el usuario realmente domina la sub-skill, no
solo memorizó un patrón.

### 5.3 Algoritmo de espaciado (spaced repetition)

Inspirado en SuperMemo y Anki, pero adaptado:

```typescript
function calculateNextReview(
  block_id: string,
  user_id: string,
  last_score: number
): Date {
  const intervals = {
    perfect: [1, 3, 7, 14, 30, 60, 120], // días
    good:    [1, 2, 5, 10, 20, 40, 80],
    fair:    [1, 1, 3, 7, 14, 28, 56],
    poor:    [1, 1, 1, 3, 7, 14, 28]
  };

  const performance = scoreToPerformance(last_score);
  const reviewCount = getReviewCount(user_id, block_id);
  const intervalDays = intervals[performance][Math.min(reviewCount, 6)];

  return addDays(now(), intervalDays);
}
```

El job nocturno usa esto para identificar bloques que necesitan revisión
y agregarlos al plan del día.

---

## 6. Re-evaluación y mantenimiento

### 6.1 Decay de skills sin uso

Las habilidades se olvidan sin práctica. Sistema modela esto:

```typescript
function calculateCurrentSkillLevel(
  baseline: number,
  daysSinceLastPractice: number,
  totalPracticeHours: number
): number {
  const decayRate = 0.005;  // 0.5% por día sin práctica
  const consolidationFactor = Math.min(totalPracticeHours / 100, 1);

  // Habilidades muy practicadas decaen menos
  const adjustedDecay = decayRate * (1 - consolidationFactor * 0.7);

  return baseline * Math.exp(-adjustedDecay * daysSinceLastPractice);
}
```

Si un usuario domina una sub-skill pero no la practica por 60 días, el
sistema asume que puede haber declined y la introduce ocasionalmente
para verificar.

### 6.2 Re-evaluación periódica

Sub-skills marcadas como `mastered` se re-verifican:
- Cada 30 días se incluye 1 ejercicio de la sub-skill en el plan diario.
- Si el usuario falla, vuelve a estado `developing` y se re-aborda.
- Si el usuario lo hace bien, se actualiza la fecha de último uso.

### 6.3 Mastery global del usuario

```typescript
interface UserMasteryProfile {
  overall_cefr: string;
  dimension_scores: MasteryDimensions;
  subskill_mastery: Record<string, number>;
  last_calculated: Date;

  // Historico para visualización
  cefr_history: Array<{ date: Date, level: string }>;
  dimension_history: Array<{ date: Date, scores: MasteryDimensions }>;
}
```

Recalculado semanalmente como parte del job nocturno.

---

## 7. Calibración del sistema

### 7.1 Calibración inicial

Antes de lanzar, el sistema necesita calibrarse contra niveles CEFR
reales:

1. **Recolectar dataset de calibración:** 50-100 usuarios voluntarios
   con nivel CEFR conocido (validado por examen oficial o evaluación
   docente).
2. **Hacer que cada uno haga el assessment completo:** registrar todos
   los scores producidos por el sistema.
3. **Mapear scores → CEFR:** función que relaciona los scores del
   sistema con CEFR oficial.
4. **Validar:** que el sistema clasifique correctamente nuevos usuarios.

### 7.2 Calibración continua

Después del lanzamiento, calibración mejora con:
- Usuarios que reportan exámenes oficiales y comparten resultados.
- Comparación con tests externos (Duolingo English Test, IELTS practice).
- A/B testing de ajustes en algoritmos de scoring.
- Feedback explícito ("este score me parece muy alto/bajo").

### 7.3 Validación pedagógica

Idealmente, contratación puntual de un experto en linguística aplicada
(consultor 4-8 horas) para revisar:
- Si los criterios de mastery son razonables.
- Si la progresión entre niveles es apropiada.
- Si las sub-skills están bien definidas.
- Si hay sesgos contra ciertos perfiles.

Esto valida que el sistema no es solo técnicamente correcto sino también
pedagógicamente sólido.

---

## 8. Schema en Postgres

```sql
-- Sub-skills catalog
CREATE TABLE subskills_catalog (
  id              TEXT PRIMARY KEY,
  dimension       TEXT NOT NULL CHECK (dimension IN (
    'pronunciation', 'fluency', 'grammar', 'vocabulary', 'listening'
  )),
  name            TEXT NOT NULL,
  description     TEXT,
  cefr_introduction TEXT,    -- en qué nivel se introduce
  difficulty      INT,        -- 1-10
  prerequisites   TEXT[] DEFAULT '{}',
  is_active       BOOLEAN DEFAULT true
);

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

CREATE INDEX idx_subskill_mastery_review ON user_subskill_mastery(next_review_at)
  WHERE next_review_at IS NOT NULL;

-- Resultados de cada intento de ejercicio
CREATE TABLE exercise_attempts (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  block_id        TEXT NOT NULL REFERENCES learning_blocks(id),
  exercise_id     TEXT NOT NULL,
  started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at    TIMESTAMPTZ,
  abandoned       BOOLEAN DEFAULT false,
  scores          JSONB NOT NULL DEFAULT '{}', -- por dimensión
  detailed_results JSONB,                       -- errores específicos, audio refs
  duration_seconds INT
);

CREATE INDEX idx_attempts_user_block ON exercise_attempts(user_id, block_id);
CREATE INDEX idx_attempts_user_date ON exercise_attempts(user_id, completed_at DESC);

-- Estado del bloque para cada usuario
CREATE TABLE user_block_status (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  block_id        TEXT NOT NULL REFERENCES learning_blocks(id),
  status          TEXT NOT NULL CHECK (status IN (
    'locked', 'available', 'in_progress', 'completed', 'mastered', 'struggling'
  )),
  attempts_count  INT NOT NULL DEFAULT 0,
  current_mastery_score FLOAT DEFAULT 0,
  first_attempt_at TIMESTAMPTZ,
  completed_at    TIMESTAMPTZ,
  mastered_at     TIMESTAMPTZ,
  PRIMARY KEY (user_id, block_id)
);
```

---

## 9. Métricas del sistema

### 9.1 Métricas de calidad pedagógica

- **Mastery accuracy:** ¿usuarios marcados como "mastered" en una
  sub-skill realmente la dominan? Medido con re-test.
- **CEFR alignment:** ¿el CEFR del sistema coincide con CEFR oficiales
  cuando se conocen?
- **Progress reality:** ¿usuarios que el sistema dice que mejoraron
  realmente mejoraron? (medido vs benchmarks externos).
- **Struggling detection:** ¿el sistema detecta a tiempo a usuarios que
  no están avanzando?

### 9.2 Métricas de salud

- Distribución de mastery levels (no debería ser bimodal extremo).
- Tasa de bloques marcados como `struggling`.
- Tasa de bloques completados pero no mastered.
- Velocidad promedio de progresión por nivel CEFR.

### 9.3 Alertas

- Si > 30% de usuarios están struggling en un mismo bloque: el bloque
  está mal calibrado.
- Si un sub-skill nunca se mastered por nadie: criterios irreales.
- Si CEFR del sistema diverge consistentemente de CEFR oficial: re-calibrar.

---

## 10. Plan de implementación

### 10.1 Fase 1: MVP (meses 0-3)

**Crítico:**
- Schema completo en Postgres.
- Pronunciation scoring vía Azure API.
- Fluency scoring básico (WPM, pausas).
- Grammar scoring vía LLM.
- Estados de bloque y mastery por sub-skill.
- Calibración inicial con 30-50 sub-skills core.

**Importante:**
- Algoritmo básico de espaciado.
- Detección de "struggling".
- Decay de skills.

### 10.2 Fase 2: Sofisticación (meses 3-9)

**Crítico:**
- Modelo propio de pronunciation scoring (reduce costos).
- Catálogo expandido de 200+ sub-skills.
- Re-evaluación periódica automática.
- Calibración continua con datos reales.

**Importante:**
- Validación pedagógica con experto.
- A/B testing de algoritmos de scoring.

### 10.3 Fase 3: Excelencia (año 2+)

- Modelos de ML propios para todas las dimensiones.
- Personalización de criterios por perfil de usuario.
- Predicción de tiempo a mastery por sub-skill.
- Generación dinámica de ejercicios para llenar gaps específicos.

---

## 11. Decisiones abiertas

- [ ] ¿Cuántas sub-skills inicialmente? (Mi recomendación: 50 core, expandir
  con datos)
- [ ] ¿Permitir que el usuario "test out" voluntariamente de una sub-skill
  con un mini-test?
- [ ] ¿Mostrar al usuario sus scores en cada dimensión o solo CEFR global?
  (Decision: mostrar con explicación, transparencia genera confianza)
- [ ] ¿Usar Azure Pronunciation Assessment indefinidamente o migrar a
  modelo propio? (Decision: migrar cuando volumen lo justifique)
- [ ] ¿Cómo manejar dialectos del inglés (British vs American) en scoring?

---

## 12. Referencias internas

- `docs/product/student-profile-and-assessment.md` — Perfil y assessment.
- `docs/product/ai-roadmap-system.md` — Roadmap consume mastery scores.
- `docs/product/motivation-and-achievements.md` — Logros basados en mastery.
- `docs/architecture/ai-gateway-strategy.md` — APIs de scoring pasan por Gateway.

---

*Documento vivo. Actualizar cuando se calibren métricas, cambien algoritmos
o se descubran problemas pedagógicos en producción.*

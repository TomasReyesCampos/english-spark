# Student Profile and Assessment System

> Sistema integral de perfilado del estudiante, free trial de 7 días
> con observación de comportamiento, y assessment profundo que genera
> el roadmap definitivo.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 establece **filosofía** (3 capas: declarado, observado, medido).
- §2 cubre **boundaries**.
- §3 cubre el **modelo de perfil**.
- §4 cubre **onboarding** (Day 0).
- §5 cubre el **free trial** (Days 1–7).
- §6 cubre el **assessment** (Day 7).
- §7 cubre **conversion al pago**.
- §8 cubre **re-evaluación periódica**.
- §9 cubre **personalización por país**.
- §10 cubre **API contracts**.
- §11 enumera **edge cases**.
- §12 cubre **eventos**.
- §13 cubre **decisiones cerradas**.

---

## 1. Filosofía

### 1.1 Tres capas de conocimiento del estudiante

1. **Lo declarado:** lo que dice de sí mismo en onboarding inicial.
2. **Lo observado:** cómo se comporta realmente durante 7 días.
3. **Lo medido:** resultado del assessment estructurado al Day 7.

La combinación es mucho más rica que cualquiera de ellas por separado.
Lo que el usuario dice, lo que hace, y lo que las pruebas revelan,
frecuentemente difieren.

### 1.2 El free trial como inversión

Los 7 días gratuitos son fase deliberada de:
- **Recolección de datos:** el sistema aprende cómo aprende este user.
- **Construcción de hábito:** primeros 7 días son críticos para
  retención.
- **Demostración de valor:** experimenta el producto real, no demos.
- **Reducción de fricción:** sin paywall inicial, más users entran al
  funnel.

### 1.3 Assessment opcional pero fuertemente incentivado

Forzarlo como gate genera fricción y churn. Hacerlo opcional + atractivo
permite:
- Users listos lo hacen y obtienen plan superior.
- Users no listos siguen con versión limitada.
- Sistema mide cuántos lo hacen y por qué.

---

## 2. Boundaries

### 2.1 Es responsable de

- Persistir y actualizar el perfil del estudiante.
- Coordinar onboarding (5 preguntas + mini-test 3 min).
- Tracking del trial de 7 días.
- Calcular `observed_behavior` desde events recientes.
- Coordinar el assessment de Day 7 (estructura, ejercicios, completitud).
- Disparar generación de roadmap inicial (Day 0) y definitivo (Day 7).
- Manejar re-evaluaciones periódicas.

### 2.2 NO es responsable de

- **Generar el roadmap:** delega a `ai-roadmap-system` via
  `generateInitialRoadmap`/`generateDefinitiveRoadmap`.
- **Scoring de los ejercicios del assessment:** delega a
  `pedagogical-system` via `submitExerciseAttempt`.
- **Pricing y conversion:** UI lo presenta, este sistema solo provee
  data.
- **Notificaciones del trial:** delega a `notifications-system` con
  events.

### 2.3 Tensiones

| Tensión | Resolución |
|---------|-----------|
| User cambia objetivo significativamente mid-trial | `profile.updated` con `significant_change = true`; AI Roadmap regenera |
| User completa assessment con respuestas obviamente al azar | Detectar (audio silencioso, respuestas random) → recovery prompt |
| User no completa assessment al Day 7 | Permitir hacerlo cuando esté listo; banner persistente |
| User quiere re-take onboarding | Permitido pero solo en Settings, con confirmación |

---

## 3. Modelo de perfil

### 3.1 Campos del perfil

```typescript
interface StudentProfile {
  user_id: string;

  // Demographics
  country: string;                     // ISO 3166-1 alpha-2
  region?: string;
  city?: string;
  timezone: string;                    // 'America/Mexico_City'
  native_language: string;             // 'es-MX', 'es-AR', 'pt-BR'
  spanish_variant?: 'mexicano' | 'rioplatense' | 'andino' | 'caribeño';
  age_range?: '18-24' | '25-34' | '35-44' | '45-54' | '55+';
  current_occupation?: string;

  // Learning context
  primary_goals: Array<
    'job_interview' | 'remote_work' | 'business_communication' |
    'travel' | 'studies' | 'personal_growth' | 'exam_prep' |
    'communication_with_family' | 'entertainment'
  >;
  target_english_variant: 'american' | 'british' | 'neutral';
  has_deadline: boolean;
  deadline_date?: string;
  deadline_context?: string;
  professional_field: 'tech' | 'health' | 'finance' | 'sales' |
                      'marketing' | 'education' | 'legal' | 'creative' |
                      'service' | 'student' | 'other';
  exam_target?: 'TOEFL' | 'IELTS' | 'Duolingo_English_Test' | 'Cambridge';
  exam_target_score?: number;

  // Self assessment
  self_perceived_level: 'A2' | 'B1' | 'B1+' | 'B2' | 'B2+' | 'C1';
  speaking_confidence: number;         // 1-5
  language_anxiety: number;            // 1-5
  self_identified_weaknesses: string[];
  daily_minutes_available: 5 | 10 | 15 | 20 | 30 | 45 | 60;
  preferred_practice_hour: number;     // 0-23
  practice_days_per_week: number;      // 1-7

  // Observed behavior (calculado periódicamente)
  observed_behavior: ObservedBehavior;
  observed_last_calculated: string;

  // Initial test results (Day 0)
  initial_test_results: InitialTestResults;

  // Assessment results (Day 7+)
  assessment_completed_at?: string;
  assessment_results?: AssessmentResults;

  // Trial status
  trial_started_at: string;
  trial_ends_at: string;
  trial_sparks_initial: number;
  trial_status: 'active' | 'expired' | 'converted' | 'abandoned';

  // Consents
  consent_audio_processing: boolean;
  consent_anonymized_research: boolean;
  consent_training_models: boolean;

  created_at: string;
  updated_at: string;
}
```

### 3.2 Schema Postgres

```sql
CREATE TABLE student_profiles (
  user_id                  UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,

  -- Demographics
  country                  TEXT NOT NULL,
  region                   TEXT,
  city                     TEXT,
  timezone                 TEXT NOT NULL,
  native_language          TEXT NOT NULL DEFAULT 'es',
  spanish_variant          TEXT,
  age_range                TEXT,
  current_occupation       TEXT,

  -- Learning context
  primary_goals            TEXT[] NOT NULL DEFAULT '{}',
  target_english_variant   TEXT NOT NULL DEFAULT 'neutral'
                           CHECK (target_english_variant IN ('american', 'british', 'neutral')),
  has_deadline             BOOLEAN NOT NULL DEFAULT false,
  deadline_date            DATE,
  deadline_context         TEXT,
  professional_field       TEXT,
  exam_target              TEXT,
  exam_target_score        INT,

  -- Self assessment
  self_perceived_level     TEXT,
  speaking_confidence      INT CHECK (speaking_confidence BETWEEN 1 AND 5),
  language_anxiety         INT CHECK (language_anxiety BETWEEN 1 AND 5),
  self_identified_weaknesses TEXT[] DEFAULT '{}',
  daily_minutes_available  INT,
  preferred_practice_hour  INT CHECK (preferred_practice_hour BETWEEN 0 AND 23),
  practice_days_per_week   INT CHECK (practice_days_per_week BETWEEN 1 AND 7),

  -- Observed behavior (JSONB porque shape evoluciona)
  observed_behavior        JSONB NOT NULL DEFAULT '{}',
  observed_last_calculated TIMESTAMPTZ,

  -- Initial test results
  initial_test_results     JSONB,

  -- Assessment results
  assessment_completed_at  TIMESTAMPTZ,
  assessment_results       JSONB,

  -- Trial
  trial_started_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  trial_ends_at            TIMESTAMPTZ NOT NULL DEFAULT (now() + interval '7 days'),
  trial_sparks_initial     INT NOT NULL DEFAULT 50,
  trial_status             TEXT NOT NULL DEFAULT 'active'
                           CHECK (trial_status IN ('active', 'expired', 'converted', 'abandoned')),

  -- Consents
  consent_audio_processing BOOLEAN NOT NULL DEFAULT true,
  consent_anonymized_research BOOLEAN NOT NULL DEFAULT false,
  consent_training_models  BOOLEAN NOT NULL DEFAULT true,

  created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at               TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_profiles_country ON student_profiles(country);
CREATE INDEX idx_profiles_trial_active ON student_profiles(trial_ends_at)
  WHERE trial_status = 'active';
CREATE INDEX idx_profiles_pending_assessment ON student_profiles(trial_ends_at)
  WHERE trial_status = 'active' AND assessment_completed_at IS NULL;
```

### 3.3 Observed behavior shape

```typescript
interface ObservedBehavior {
  // Patrones de uso
  actual_avg_minutes_per_day: number;
  actual_practice_days_last_week: number;
  preferred_session_length_minutes: number;
  most_active_hour: number;
  most_active_day_of_week: number;

  // Patrones de aprendizaje
  exercises_completed_total: number;
  exercises_abandoned_rate: number;
  average_attempts_per_exercise: number;
  preferred_exercise_types: string[];   // ranking

  // Performance observada
  pronunciation_avg_score: number;
  fluency_wpm_average: number;
  filler_word_rate: number;
  detected_error_patterns: string[];

  // Engagement
  current_streak_days: number;
  longest_streak_days: number;
  total_active_days: number;

  // Calculado periódicamente
  last_calculated_at: string;
}
```

Recalculado por job nocturno (cron diario) consumiendo events de
pedagogical y motivation.

### 3.4 Initial test results shape

```typescript
interface InitialTestResults {
  cefr_estimate: 'A2' | 'B1' | 'B1+' | 'B2' | 'B2+' | 'C1';
  fluency_score: number;             // 0-10
  pronunciation_score: number;        // 0-10
  detected_error_patterns: string[];
  recorded_at: string;
}
```

### 3.5 Assessment results shape

```typescript
interface AssessmentResults {
  completed_at: string;
  assessment_version: string;        // 'v1.0' del assessment template
  duration_seconds: number;

  measured_cefr_level: string;

  scores: {
    pronunciation: number;            // 0-100
    fluency: number;
    grammar: number;
    vocabulary: number;
    listening: number;
    overall_speaking: number;
  };

  phonetic_error_patterns: string[];
  grammar_error_patterns: string[];

  strongest_areas: string[];
  weakest_areas: string[];

  peer_percentile?: number;
}
```

---

## 4. Onboarding inicial (Day 0)

### 4.1 Objetivos

- Capturar perfil mínimo viable.
- Generar roadmap inicial básico (provisional).
- Demostrar valor rápido: user practicando en < 5 min.
- Crear expectativa positiva sobre el assessment del Day 7.

### 4.2 Flujo (5–7 minutos)

#### Paso 1: Bienvenida (30s)

Pantalla de bienvenida. Tono español neutro. Promesa: "vas a hablar
inglés con confianza".

#### Paso 2: Ubicación (15s)

```
¿Dónde estás?

[🇲🇽 México] [🇦🇷 Argentina] [🇨🇴 Colombia]
[🇨🇱 Chile] [🇵🇪 Perú] [🌎 Otro país de Latam]
```

Detección automática vía IP (Cloudflare `cf.country`); user confirma.

#### Paso 3: Objetivo principal (20s)

Multi-select de `primary_goals`.

#### Paso 4: Deadline (10s)

```
[ ] Sí, en menos de 1 mes
[ ] En 1 a 3 meses
[ ] En 3 a 6 meses
[ ] No tengo apuro
```

#### Paso 5: Contexto profesional (15s)

Single-select de `professional_field`.

#### Paso 6: Autoevaluación (30s)

```
¿Cómo te sentís hablando inglés?
😰 Me bloqueo completamente
😟 Muy nervioso, evito hablar
😐 Nervioso pero hablo
😊 Hablo con errores pero fluyo
😎 Hablo bien pero quiero pulir

¿Cuánto tiempo podés dedicar al día?
[5 min] [10 min] [15 min] [20 min] [30 min] [+30 min]
```

#### Paso 7: Mini-test rápido (3 minutos)

3 ejercicios:
1. Repetir frase estándar (pronunciation base).
2. Responder "How are you doing today?" en 30s (fluidez espontánea).
3. Identificar 3 imágenes nombrándolas (vocabulario).

Cada ejercicio:
- Cliente sube audio a R2.
- Llama `pedagogical.submitExerciseAttempt` con `block_id =
  'onboarding_minitest'`.
- Pedagogical scoring → integra en `initial_test_results`.

#### Paso 8: Análisis y bienvenida (30s)

Mientras procesa, animación de "armando tu plan".

Llama `ai-roadmap.generateInitialRoadmap`. Resultado:

```
"Listo, María. Detectamos que tu nivel está alrededor de B1 y tu mayor
desafío es la fluidez. Armamos un plan inicial de 4 semanas.

⚡ Tienes 50 Sparks gratuitos por 7 días para probar todo.
🎯 El día 7 vas a hacer un assessment más profundo que ajustará tu plan."

[Empezar mi primer ejercicio]
```

### 4.3 Roadmap inicial: provisional explícito

User ve: "Tu plan inicial – se actualizará el día 7".

Características (de `ai-roadmap-system.md` §5):
- 4 semanas estimadas.
- 30–40 bloques.
- Variant inglés según ubicación + objetivo.
- Foco general (no hiper-específico).

---

## 5. Free trial (Days 1–7)

### 5.1 Características

- Duración: 7 días desde registro.
- Sparks: 50 gratuitos (cap absoluto, ver `sparks-system.md` §4.3).
- Acceso: completo a features de Plan Pro:
  - Conversación 1 a 1 con IA.
  - Roleplays personalizados.
  - Pronunciation assessment con feedback IA.
  - Análisis nocturno (versión completa).
  - Notificaciones humanizadas.
- Limitaciones: ninguna funcional, solo cap de Sparks.

### 5.2 Por qué 50 Sparks específicamente

Cálculo:
- 1 conversación 10 min = 10 Sparks.
- 3-5 conversaciones durante trial = 30-50 Sparks.
- + algunas operaciones menores: ~50.
- User intensivo: 50 duran ~5 días → oportunidad de "early conversion".
- User casual: 50 sobran para 7 días.

Costo aproximado para nosotros: $1.50–2.50 USD si user los usa todos.

### 5.3 Sistema de observación

Durante 7 días, sistema captura:

```typescript
interface TrialObservation {
  day_1_completed: boolean;
  day_2_completed: boolean;
  // ... hasta day_7

  // Agregados
  total_minutes_practiced: number;
  total_sessions: number;
  total_sparks_used: number;
  preferred_session_length: number;
  preferred_practice_hour: number;
  exercise_types_tried: string[];
  exercise_types_preferred: string[];
  abandoned_exercises_count: number;
  conversation_sessions_completed: number;

  // Trends
  pronunciation_score_trend: 'improving' | 'stable' | 'declining';
  fluency_score_trend: 'improving' | 'stable' | 'declining';
  detected_strengths: string[];
  detected_weaknesses: string[];

  // Engagement
  notification_open_rate: number;
  app_opens_count: number;
  feature_exploration_score: number;
}
```

Calculado incrementalmente cada noche del trial. Se persiste en
`student_profiles.observed_behavior`.

### 5.4 Comunicación durante el trial

| Día | Mensaje | Canal | Objetivo |
|-----|---------|-------|----------|
| 1 (24h post-registro) | "¡Buen primer día! ¿Listo para el segundo?" | Push | Reforzar hábito |
| 3 | "Llevás 3 días. Tu pronunciación de [X] mejoró." | Push | Mostrar progreso |
| 5 | "Faltan 2 días para tu assessment. Va a desbloquear tu plan completo." | Push | Crear expectativa |
| 6 | "Mañana es el gran día. Toma 20 minutos y va a transformar tu plan." | Push + In-app | Preparar |
| 7 | "Llegó el día. Hacé tu assessment ahora." | Push prominente | Trigger |

### 5.5 Edge: Sparks runout pre-Day 7

User entusiasta consume 50 Sparks al Day 4-5.

**Comportamiento:**
- Mensaje claro: "Usaste todos tus Sparks gratuitos. Tu plan completo te
  espera al hacer el assessment."
- Acceso continuo a preassets ilimitados.
- Opción de "hacer assessment ahora" si user está listo (después del
  Day 5 mínimo).
- Si user elige hacerlo antes del Day 7: assessment se desbloquea.

Convierte momento de fricción en oportunidad de conversión adelantada.

### 5.6 Edge: Day 8 sin assessment

User casual no completó al Day 7.

**Comportamiento:**
- App sigue funcionando con preassets.
- Sin más Sparks gratuitos (los 50 expiraron).
- Sin conversación 1 a 1.
- Banner persistente: "Hacé tu assessment para desbloquear tu plan".
- Notificaciones suaves recordándolo (días 8, 10, 14, 21, 30).
- Después de 30 días sin assessment: `trial_status = abandoned` pero
  permite seguir usando preassets gratis indefinidamente.

**Filosofía:** nunca cerrar la puerta. User que abandonó hoy puede
volver mañana.

---

## 6. El Assessment (Day 7)

### 6.1 Estructura

Duración total: 20–25 minutos. Diseñado para ser desafiante pero no
agotador. 4 partes.

#### Parte 1: Confirmación y profundización del perfil (3 min)

Re-confirmar y profundizar lo declarado:
- ¿Cambió tu objetivo desde que empezaste?
- ¿En qué situaciones específicas necesitás hablar inglés?
- ¿Qué tan importante es sonar "natural" vs "correcto"?
- Si laboral: ¿cómo te imaginás usándolo? (entrevistas, reuniones,
  escritura).
- Para deadline: ¿qué exactamente esperan haber logrado?

#### Parte 2: Test de habilidades activas (12 min)

5 ejercicios estructurados:

**Ej 1: Lectura comprensiva con resumen oral (3 min)**
- Leer artículo de 200 palabras.
- Resumir en voz alta sin verlo (60s).
- Mide: comprensión lectora, vocabulario receptivo, fluidez productiva.

**Ej 2: Roleplay situacional (3 min)**
- Situación específica al objetivo.
- Job interview: "Te están entrevistando. Contame sobre un desafío que
  superaste."
- Travel: "Estás en hotel y la habitación tiene problemas. ¿Qué decís?"
- Mide: vocabulario activo en contexto, fluidez espontánea, gramática.

**Ej 3: Pronunciación específica (2 min)**
- 10 frases con sonidos críticos para hispanohablantes.
- /θ/, /ð/, /v/, /ʒ/, vocales tensas vs laxas, finales consonánticos.
- Mide: pronunciación detallada por fonema.

**Ej 4: Listening con respuesta (2 min)**
- Audio de 90s a velocidad nativa con acento del target_english_variant.
- 3 preguntas comprensivas con respuesta oral.
- Mide: comprensión auditiva real.

**Ej 5: Producción libre (2 min)**
- Pregunta abierta relacionada al objetivo.
- 2 minutos para responder libremente.
- Mide: fluidez sostenida, organización de ideas, vocabulario amplio.

#### Parte 3: Test de habilidades receptivas (3 min)

**Vocabulary in context:** 10 frases con palabras subrayadas. Elegir
sinónimo correcto. Adaptativo.

**Grammar recognition:** 8 frases con errores sutiles. Identificar.

#### Parte 4: Aspiraciones y motivación (2 min)

- ¿Qué te frustra más cuando hablás inglés?
- ¿En qué situación te gustaría sentirte cómodo en 3 meses?
- ¿Tipo de feedback preferís? (corrección inmediata, al final, suave,
  directo).
- ¿Sesiones cortas frecuentes o largas espaciadas?

### 6.2 Análisis del assessment

Combina:

**Análisis técnico (pedagogical-system):**
- STT de todos los audios.
- Pronunciation scoring por fonema.
- Análisis sintáctico de producciones.
- Detección de errores gramaticales.
- Métricas: WPM, pausas, muletillas.

**Análisis con IA (LLM):**
- Evaluación holística de fluidez y coherencia.
- Detección de patrones de error específicos.
- Identificación de strengths y weaknesses ranqueadas.
- CEFR estimado con justificación.

**Combinación con observed_behavior:**
- ¿Autoevaluación coincide con lo medido?
- ¿Comportamiento durante trial confirma o contradice preferencias
  declaradas?
- ¿Qué ejercicios funcionaron mejor?

### 6.3 Generación del roadmap definitivo

(Detalle en `ai-roadmap-system.md` §6.)

Llamada a `ai-roadmap.generateDefinitiveRoadmap` con
`assessment_results` + `observed_behavior`.

### 6.4 Presentación de resultados

Inmediatamente después, pantalla emocional:

```
🎉 Tu Assessment está listo, María

📊 Tu nivel actual: B1+ (mejor de lo que pensabas)
💪 Tu fortaleza: comprensión lectora
🎯 Tu mayor oportunidad: fluidez en producción libre

[Ver detalle completo →]

✨ Tu Plan Personalizado:
14 semanas para llegar a tu objetivo de entrevistas en inglés.
68 lecciones específicas para vos.
Foco principal: confianza al hablar y fluidez.

[Empezar mi plan →]
```

Después, página detallada con:
- Gráfico de scores por dimensión.
- Comparación con percentiles.
- Top 3 áreas de mejora con explicación.
- Top 3 fortalezas.
- Camino visualizado.

### 6.5 Costo del assessment

- Análisis técnico (Whisper, pronunciation): ~$0.05.
- LLM análisis holístico (Haiku batch): ~$0.03.
- Generación de roadmap definitivo: ~$0.02.
- **Total: ~$0.10 por assessment.**

Para 1.000 users: $100. Para 10.000: $1.000. Operación única por user.

### 6.6 Detección de assessment "rotos"

Si user obviamente no toma en serio:
- Audio silencioso > 80% del ejercicio.
- Respuestas de texto random ("aaa", "asdf").
- Multiple choice todo "A".
- Tiempo total < 5 min (vs 20 esperado).

**Comportamiento:**
- Detectar en validación post-completion.
- Mostrar prompt: "Notamos que el assessment fue muy rápido.
  ¿Querés rehacerlo para que sea preciso?"
- Si user confirma: re-iniciar assessment, no cobrar Sparks
  adicionales.
- Si user dice "está bien así": persistir results pero flag
  `low_confidence = true`. AI Roadmap genera con caveat.

---

## 7. Conversion al pago

### 7.1 Momento ideal

Day 7, después del assessment. User:
- Acaba de invertir 20 min.
- Vio resultados detallados.
- Tiene plan personalizado.
- Está en pico de motivación y compromiso.

### 7.2 Presentación

Después del plan personalizado:

```
Tu plan está listo. ¿Cómo querés practicarlo?

┌─────────────────────────────────────┐
│  Plan Básico - $30/mes              │
│  • Acceso a tu plan completo        │
│  • 30 Sparks/mes                    │
│  • Análisis semanal                 │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  Plan Pro - $100/mes  ⭐ RECOMENDADO │
│  • Todo lo del Básico               │
│  • 200 Sparks/mes                   │
│  • Insights personalizados con IA   │
│  • Re-evaluación cada 4 semanas     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  Plan Premium - $250/mes            │
│  • Todo lo del Pro                  │
│  • 600 Sparks/mes                   │
│  • Roleplays personalizados         │
│  • Prioridad en procesamiento       │
└─────────────────────────────────────┘

[Empezar con Pro] [Ver todos los detalles]
```

### 7.3 Optimizaciones

- **Anchoring:** Pro recomendado hace que Básico parezca poco y Premium
  mucho. Pro captura mayoría.
- **Scarcity transparente:** "tu plan personalizado está listo".
- **Risk reversal:** "cancelá cuando quieras, los Sparks comprados no
  expiran por 6 meses".
- **Social proof:** "más de X usuarios en [país] eligieron Pro este mes".

### 7.4 Si user no convierte

No tácticas agresivas. Estrategia:
- Acceso continuo a preassets gratis.
- Plan personalizado visible pero con bloqueos en bloques que requieren
  Sparks.
- Ofertas suaves periódicas (descuento primer mes).
- Re-engagement campaigns mostrando valor del plan.
- Permitir compra de packs de Sparks únicos para users que no quieren
  suscripción.

---

## 7.1 Sporadic data collection (pre-assessment, v1.2)

> **Promovido desde** [`docs/explorations/sporadic-questions.md`](../explorations/sporadic-questions.md)
> en 2026-05. Esta sección establece el principio y schema. Detalles
> finos (UX, pool de items, throttle) viven en la exploración.

### 7.1.1 Concepto

Entre Day 1 y assessment_completion, el sistema presenta
**micro-preguntas esporádicas** (5-30s, max 1-2 por sesión) que
calibran contenido en tiempo real y pre-arman data para reducir
duración del assessment formal.

**Lifespan:** Day 1 hasta `student_profiles.assessment_completed_at IS NOT NULL`.
Una vez completado el assessment formal, sporadic termina.

### 7.1.2 5 categorías del pool

1. Self-assessment dimensional (~7 items): listening, reading,
   speaking, vocab, grammar, pronunciación, frustración.
2. Micro audio captures (~10 items): frases targeting phonemes
   específicos.
3. Vocab quick checks (~20): MC ~5s.
4. Grammar quick checks (~20): correct/wrong.
5. Goal validation (~1): detección de cambio de objetivo.

Total MVP: **~58 items**.

### 7.1.3 Throttle (resumen)

- Max 1-2 por sesión.
- Max 3 por día.
- Max 8-12 totales pre-assessment.
- 3 skips consecutivos → pausa 24h.
- Setting para desactivar manualmente.

### 7.1.4 Schema

```sql
CREATE TABLE sporadic_questions (
  id              TEXT PRIMARY KEY,
  category        TEXT NOT NULL,                -- 'self_assessment', 'micro_audio', etc.
  cefr_level      TEXT,
  goal_relevance  TEXT[] DEFAULT '{}',
  target_dimension TEXT,
  content         JSONB NOT NULL,
  estimated_seconds INT NOT NULL,
  weight_in_calibration NUMERIC(3,2) DEFAULT 0.5,
  approved        BOOLEAN NOT NULL DEFAULT false,
  archived        BOOLEAN NOT NULL DEFAULT false,
  use_count       INT NOT NULL DEFAULT 0,
  skip_count      INT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sporadic_responses (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  question_id     TEXT NOT NULL REFERENCES sporadic_questions(id),
  asked_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  responded_at    TIMESTAMPTZ,
  skipped         BOOLEAN NOT NULL DEFAULT false,
  response_data   JSONB,
  audio_atomic_id TEXT REFERENCES media_atomics(id),
  scoring_result  JSONB,
  session_id      TEXT,
  trigger_context TEXT,
  flagged_likely_fake BOOLEAN NOT NULL DEFAULT false,
  flag_reason     TEXT
);

CREATE INDEX idx_sr_user_date ON sporadic_responses(user_id, asked_at DESC);
CREATE INDEX idx_sr_user_question ON sporadic_responses(user_id, question_id);

-- Log para entrenar ML de detección de respuestas fake (decisión §10.6)
CREATE TABLE sporadic_responses_ml_training (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  response_id     UUID NOT NULL REFERENCES sporadic_responses(id),
  flag_signals    JSONB NOT NULL,
  user_state_snapshot JSONB,
  logged_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Modificación a student_profiles
ALTER TABLE student_profiles
  ADD COLUMN self_reported_dimensions JSONB DEFAULT '{}',
  ADD COLUMN sporadic_mismatch_flags TEXT[] DEFAULT '{}',
  ADD COLUMN sporadic_paused_until TIMESTAMPTZ;
```

### 7.1.5 Detección de respuestas fake

Heurísticas at-time-of-response:
- Multiple choice: misma opción en 3+ self-assessment consecutivas.
- Self-assessment: tiempo de respuesta < 2s con 5 opciones.
- Audio capture: silencio total > 70% del audio.
- Goal validation: cambio de objetivo cada vez que se pregunta.

Si flag `likely_fake = true`:
- NO entra a calibración (no afecta `self_reported_dimensions`).
- SÍ se persiste con flag.
- Log a `sporadic_responses_ml_training` para entrenar detector futuro
  (decisión cerrada §10.6 de la exploración).
- NO notificar al user del flag.

### 7.1.6 Mismatch detection

Si self-perception ≥ 4 pero measured score ≤ 60 (o inverso):
- Flag en `sporadic_mismatch_flags`.
- Cuando assessment formal corra, considerar la discrepancia para
  ajustar tono del results screen
  (`post-assessment-flow.md` §3.5).

### 7.1.7 Effect en roadmap provisional

Las respuestas (no-fake) reordenan blocks del roadmap provisional Day
1-2 (NO regeneran). Detalle en `ai-roadmap-system.md` §7.4.

### 7.1.8 Pre-data para assessment formal

Cuando assessment se construye, sporadic responses no-fake permiten:
- Skip de preguntas redundantes en Parte 1 del assessment formal.
- Audio captures contribuyen a `user_subskill_mastery` con peso
  reducido (0.5x vs 1.0x del assessment formal).

Reduce duración del assessment de 20 min a 15-17 min sin perder
calidad de data.

---

## 8. Re-evaluación periódica

### 8.1 Cadencia recomendada

(Coordinado con `ai-roadmap-system.md` §14.5.)

| Plan | Frecuencia | Profundidad |
|------|-----------|-------------|
| Básico | Cada 8 semanas | Mini-assessment 5 min |
| Pro | Cada 4 semanas | Assessment intermedio 10 min |
| Premium | Cada 4 semanas | Assessment completo 20 min |

### 8.2 Comparación temporal

Pieza más poderosa: mostrar **evolución**.

```
Tu progreso desde el último assessment:

Pronunciation:  72 → 84  (+12) 📈
Fluency:        65 → 71  (+6)  📈
Grammar:        78 → 80  (+2)  →
Vocabulary:     70 → 79  (+9)  📈

🎉 Subiste de B1 a B1+
🎯 Tu siguiente objetivo: alcanzar B2 en 6 semanas
```

Convierte progreso "invisible" en algo concreto y motivador.

### 8.3 Roadmap dinámico post-re-evaluación

- Bloques completados: ya no aparecen.
- Bloques que ya domina (verificado): `tested_out`, se saltan.
- Bloques nuevos: agregados según patrones detectados.
- `target_cefr` puede ajustarse si va más rápido/lento.

---

## 9. Personalización por país y región

### 9.1 Variantes de contenido

Adaptaciones según `country` y `spanish_variant`:

**México:**
- Variant inglés default: americano.
- Roleplays con nombres de empresas/productos del mercado mexicano.
- Vocabulario: español neutro con cierta inclinación mexicana.
- Eventos locales: Día del Estudiante (23 mayo), vacaciones verano.

**Argentina:**
- Variant inglés default: neutral con cierta inclinación británica.
- Roleplays adaptados al contexto rioplatense.
- Voseo en explicaciones.
- Eventos: Día del Estudiante (21 sept), vacaciones invierno.

**Colombia:**
- Variant inglés default: americano.
- Énfasis en empresas multinacionales (BPO, tech).

**Otros:**
- Variant default: neutral.
- Contenido más generalizado.
- Sistema aprende preferencias.

### 9.2 Pricing localizado

(Detalle en `cross-cutting/i18n.md` §4.)

Plan en MXN como referencia; UI muestra en moneda local.

### 9.3 Métodos de pago localizados

| País | Métodos prioritarios |
|------|---------------------|
| México | Tarjeta, OXXO, MercadoPago, Apple/Google Pay |
| Argentina | MercadoPago, tarjeta, transferencia |
| Colombia | Nequi, PSE, tarjeta, Bold |
| Chile | Webpay, tarjeta, MACH |
| Perú | Yape, Plin, tarjeta |

Stack inicial: tarjeta + Apple/Google Pay + 1-2 locales por país clave.

---

## 10. API contracts

### 10.1 `startOnboarding`

**Request:**

```typescript
interface StartOnboardingRequest {
  user_id: string;
  detected_country?: string;          // de cf.country
  detected_timezone?: string;         // de Intl
}

interface StartOnboardingResponse {
  onboarding_session_id: string;
  prefilled_country: string;
  prefilled_timezone: string;
  questions: OnboardingQuestion[];
}
```

### 10.2 `submitOnboardingAnswers`

**Request:**

```typescript
interface SubmitOnboardingAnswersRequest {
  onboarding_session_id: string;
  answers: {
    country: string;
    primary_goals: string[];
    deadline_choice: string;
    professional_field: string;
    speaking_confidence: number;
    daily_minutes_available: number;
  };
}
```

**Response:**

```typescript
interface SubmitOnboardingAnswersResponse {
  profile_id: string;
  next_step: 'mini_test';
  mini_test_exercises: ExerciseConfig[];
}
```

### 10.3 `submitMiniTestExercise`

**Request:**

```typescript
interface SubmitMiniTestExerciseRequest {
  onboarding_session_id: string;
  exercise_index: number;
  payload: ExercisePayload;          // mismo shape que pedagogical
}
```

**Response:**

```typescript
interface SubmitMiniTestExerciseResponse {
  scores: Partial<MasteryDimensions>;
  is_last_exercise: boolean;
  next_step?: 'submit_exercise' | 'completing_onboarding';
}
```

### 10.4 `completeOnboarding`

Trigger interno cuando se completa mini-test:

```typescript
interface CompleteOnboardingRequest {
  onboarding_session_id: string;
}

interface CompleteOnboardingResponse {
  profile_id: string;
  initial_test_results: InitialTestResults;
  initial_roadmap: {
    roadmap_id: string;
    ai_summary: string;
    first_block_id: string;
  };
  trial_sparks_credited: number;
}
```

**Reglas:**
- Persiste profile completo.
- Llama `ai-roadmap.generateInitialRoadmap`.
- Llama `sparks.awardBonus(50, expiry: cycle_end)`.
- Emite `onboarding.completed`.

### 10.5 `getProfile`

```typescript
interface GetProfileRequest { user_id: string; }

interface GetProfileResponse {
  profile: StudentProfile;
  trial_status_detail: {
    days_remaining: number;
    sparks_remaining_in_trial: number;
    can_request_assessment_now: boolean;
  };
}
```

### 10.6 `updateProfile`

**Request:**

```typescript
interface UpdateProfileRequest {
  user_id: string;
  changes: Partial<StudentProfile>;
}
```

**Response:**

```typescript
interface UpdateProfileResponse {
  profile: StudentProfile;
  significant_change: boolean;       // si trigger regeneración del roadmap
}
```

**Reglas:**
- Detectar `significant_change`: cambio de `primary_goals` major,
  cambio de `deadline_date` > 50%, cambio de `target_english_variant`.
- Si significant: emitir `profile.updated` con flag.

### 10.7 `requestAssessment`

**Request:**

```typescript
interface RequestAssessmentRequest {
  user_id: string;
  reason?: 'trial_day_7' | 'sparks_runout_early' | 'user_initiated' | 'periodic_reassessment';
}
```

**Response:**

```typescript
interface RequestAssessmentResponse {
  assessment_session_id: string;
  parts: AssessmentPart[];           // las 4 partes detalladas
  estimated_duration_minutes: 20;
}
```

**Reglas:**
- Antes del Day 5: rechazado salvo `sparks_runout_early`.
- Después del Day 7: permitido siempre.
- Re-evaluation periódica: gated by plan + cadencia.

### 10.8 `submitAssessmentExercise`

```typescript
interface SubmitAssessmentExerciseRequest {
  assessment_session_id: string;
  part: 1 | 2 | 3 | 4;
  exercise_index: number;
  payload: ExercisePayload;
}
```

### 10.9 `completeAssessment`

```typescript
interface CompleteAssessmentRequest {
  assessment_session_id: string;
}

interface CompleteAssessmentResponse {
  assessment_results: AssessmentResults;
  definitive_roadmap: {
    roadmap_id: string;
    ai_summary: string;
    estimated_completion_weeks: number;
  };
  trial_status_after: 'converted' | 'completed_assessment' | 'expired';
  conversion_offer: PaywallOffer;
}
```

### 10.10 `cancelAssessment`

User puede cancel mid-assessment. Persiste responses parciales como
`assessment_attempts` (separate table) para análisis. No genera
definitive roadmap.

---

## 11. Edge cases (tests obligatorios)

### 11.1 Onboarding

1. **User abandona después de respuesta 3:** session se mantiene 24h
   con respuestas parciales. Si vuelve: continúa donde quedó.
2. **User cierra app durante mini-test:** mismo. Audios uploaded
   persistidos.
3. **Detección de país falla:** default a `'OT'` (otro), user elige
   manual.
4. **Mini-test audio fail (< 1s útil):** request retry, no fail
   onboarding entero.

### 11.2 Trial

5. **User practica todos los 7 días intensamente:** Sparks runout
   probable Day 4-5; ofrecer assessment temprano.
6. **User no abre la app entre Day 1-7:** notificaciones progresivas;
   Day 7 con assessment trigger; si no abre, marcar `inactive` pero
   permitir comeback.
7. **User cambia de TZ mid-trial:** ajustar `trial_ends_at` si necesario
   (no acortar, posible extender).

### 11.3 Assessment

8. **User completa assessment con todas respuestas idénticas:** detección
   de "broken assessment" → recovery prompt.
9. **Assessment session expira (1h sin actividad):** session se cierra,
   user puede iniciar nueva.
10. **User tiene insufficient Sparks para assessment (raro, assessment
    es free):** assessment NO cobra Sparks. Si bug detecta cobro, refund.
11. **Multiple sessions abiertas (web + móvil):** primera completed gana,
    segunda se cierra automático.

### 11.4 Profile updates

12. **User cambia `primary_goals` de ['travel'] a ['job_interview']:**
    `significant_change = true` → regeneration prompt.
13. **User cambia `daily_minutes_available` de 30 a 5:** ajustar
    `estimated_completion_weeks` del roadmap; no regenerar.
14. **User cambia `target_english_variant` de american a british:**
    significant; afecta scoring; regenerar.

### 11.5 Re-evaluation

15. **User en plan Pro pasa 4 semanas y sistema ofrece re-evaluation:**
    push notification + banner; opcional pero gentle.
16. **User Premium completa 4 weeks pero está struggling:** ofrecer
    re-evaluation con tono empático; puede ayudar a recalibrar.

### 11.6 Trial conversion

17. **User compra plan durante trial Day 3:** trial Sparks restantes se
    mantienen como bonus (no expiran prematuramente). Plan Sparks se
    acreditan al inicio del primer ciclo facturado.
18. **User compra y cancela en mismo día:** Stripe permite refund
    completo si <14d. Plan permanece hasta fin del primer ciclo.

---

## 12. Eventos

### 12.1 Emitidos

(Detalle en `cross-cutting/data-and-events.md` §5.2.)

| Evento | Cuándo |
|--------|--------|
| `onboarding.started` | User inicia onboarding |
| `onboarding.completed` | User completa onboarding + mini-test |
| `assessment.started` | User inicia assessment |
| `assessment.exercise_completed` | Ejercicio individual completed |
| `assessment.completed` | Assessment full completed |
| `assessment.abandoned` | User canceled/timeout |
| `profile.updated` | Cualquier cambio al perfil |
| `trial.expired` | Day 7 alcanzado sin conversión |
| `trial.converted` | User pagó (de payment.succeeded) |

### 12.2 Consumidos

| Evento | Acción |
|--------|--------|
| `user.signed_up` (de auth) | Crear `student_profile` con valores default |
| `block.completed` (de pedagogical) | Update `observed_behavior` (job nocturno) |
| `payment.succeeded` (de sparks) | Update `trial_status = 'converted'` |
| `user.deleted` (de auth) | Cascade delete del profile |

---

## 13. Decisiones cerradas

### 13.1 Timing del assessment: **3 niveles según día del trial** ✓ (actualizado 2026-05)

| Día del trial | Status | Comportamiento |
|---------------|--------|----------------|
| **1-2** | Locked | No disponible. Necesitamos data observada del trial mínima. Solo se desbloquea si Sparks runout (caso de power user). |
| **3-4** | Optional | User puede solicitar desde Settings o banner sutil. Si no lo hace, app funciona normal. |
| **5-6** | "Obligatorio" suave | Modal prominente al abrir app: **"Empezar ahora"** (CTA primario) / **"Postergar 1 día"** (secundario, max 2 postpones). Premium features siguen disponibles. |
| **7+** | Obligatorio firme | Modal no-dismissible hasta empezar. Features premium (conversation 1:1, roleplays personalizados) bloqueadas hasta completar. **Preassets gratis siempre disponibles** (regla del producto). |

**Excepción transversal:** Sparks runout antes del Day 3 → assessment
se ofrece adelantado independiente del día (oportunidad de conversion
temprana).

**Razón del cambio (2026-05):**
- Day 3 con observed_behavior básico ya es suficiente para roadmap
  útil (no perfecto, pero útil).
- Día 5 obligatorio suave alinea con behavior real: la mayoría de
  power users ha gastado Sparks para entonces y querría pasar al plan.
- Premium features bloqueadas Day 7+ es la "graduation": el trial
  cumplió su rol de exploración; ahora se necesita el plan para
  premium.
- Filosofía intacta: nunca se cierra la puerta. Preassets gratis
  siempre.

**Implementación:**
- Cliente checa `student_profile.trial_status` + days since
  `trial_started_at` en cada app open.
- Si Day 5-6: muestra modal con `postponeCount`.
- Si Day 7+: bloquea features premium en backend (pedagogical-system
  rechaza tasks que requieran Sparks para premium operations si
  assessment no completed).
- Postpone counter persistido en `student_profiles.assessment_postpone_count`.

### 13.2 CEFR para sampling del assessment: **Hybrid self-perceived + measured** ✓ (cerrado 2026-05)

```typescript
function determineCefrForAssessment(profile: StudentProfile) {
  const selfPerceived = profile.self_perceived_level;
  const measured = profile.initial_test_results?.cefr_estimate;

  // Caso 1: matchean (~50% casos) → usar ese, sin fricción
  if (cefrDistance(selfPerceived, measured) === 0) {
    return { cefr: selfPerceived, show_choice: false };
  }

  // Caso 2: difieren por 1 nivel (~40% casos) → preguntar al user
  // Pantalla: "Detectamos que estás cerca de B1+, tú sientes B2.
  //           ¿Cuál prefieres evaluar?
  //           [B1+ (más cómodo)] [B2 (más desafiante)]"
  if (cefrDistance(selfPerceived, measured) === 1) {
    return { cefr: 'user_choice_required', show_choice: true,
             options: [measured, selfPerceived] };
  }

  // Caso 3: difieren por 2+ niveles (~10% casos) → tomar el HIGHER,
  //         con disclaimer
  // "Vamos a evaluarte en [higher]. Si es muy difícil podés repetir
  //  más adelante en otro nivel."
  if (cefrDistance(selfPerceived, measured) >= 2) {
    return { cefr: max(selfPerceived, measured), show_choice: false,
             show_explanation: true };
  }
}
```

**Razón:**
- Self-perceived solo es riesgoso (Dunning-Kruger).
- Measured solo ignora agency del user.
- Hybrid balancea: respeta intuición del user cuando hay alineación,
  le da choice cuando difieren marginalmente, toma decisión informed
  cuando difieren mucho.

### 13.3 Re-aplicar assessment al cambiar objetivo: **Sí, si significant_change** ✓

**Razón:** un objetivo nuevo cambia qué medir. Mantener assessment
viejo daría scoring desalineado.

**Implementación:** `profile.updated` con `significant_change = true`
→ prompt al user "¿quieres rehacer el assessment para tu nuevo
objetivo?". Si rechaza, mantener actual con flag `objective_misalignment`.

### 13.3 Manejo de "broken assessments": **detectar + recovery prompt** ✓

(Detalle en §6.6.)

### 13.4 Mostrar roadmap inicial completo o solo primeros niveles:
**solo primeros 2 niveles + preview borrosa** ✓

**Razón:**
- Mantiene intriga sobre qué desbloquea el assessment.
- Reduce overwhelm cognitivo en Day 0.
- User ve suficiente para sentir dirección.

### 13.5 PDF descargable del resultado del assessment: **Post-MVP** ✓

**Razón:**
- Útil para usuarios que quieren mostrar a profesores/empleadores.
- Pero requiere diseño + generación. No crítico para MVP.
- Reconsiderar si > 10% de usuarios lo solicitan via support.

---

## 14. Plan de implementación

### 14.1 Fase 1: MVP (meses 0–3)

**Crítico:**
- Schema completo `student_profiles`.
- Onboarding con 5 preguntas.
- Detección de país + variant inglés.
- Trial 7 días con cap 50 Sparks.
- Mini-test inicial.
- Roadmap inicial via AI.

**Importante:**
- Sistema de observación durante trial.
- Notificaciones progresivas.
- Pre-aviso del assessment Day 5-6.

### 14.2 Fase 2: Assessment (meses 3–6)

**Crítico:**
- Assessment 20 min completo (4 partes).
- Análisis combinado: técnico + LLM + observation.
- Roadmap definitivo.
- Presentación emocional.
- Conversion flow al pago.

**Importante:**
- Manejo de broken assessments.
- Re-engagement campaigns para no-assessors.

### 14.3 Fase 3: Sofisticación (meses 6–12)

- Re-evaluaciones periódicas.
- Comparación temporal visual.
- Pricing localizado real.
- Métodos de pago locales por país.

### 14.4 Fase 4: Optimización (año 2+)

- A/B testing del assessment.
- Personalización del onboarding por ubicación/device.
- Segmentación dinámica.
- PDF del resultado.

---

## 15. Métricas

### 15.1 Embudo de conversión

| Etapa | Métrica | Target |
|-------|---------|--------|
| Registro | Completed onboarding | > 85% |
| Trial | Volvió Day 2 | > 60% |
| Trial | Volvió Day 7 | > 40% |
| Trial | Completó assessment | > 30% |
| Post-assessment | Compró suscripción | > 40% |
| Conversion total | Registro → pago | > 12% |

### 15.2 Calidad del perfil

- % users con perfil completo.
- Coincidencia entre nivel autopercibido y nivel medido (gap analysis).
- Predicción de observed_behavior vs reality.

### 15.3 Ajustes basados en datos

- Conversion onboarding < 70%: simplificar preguntas.
- Return Day 2 < 50%: revisar primer ejercicio (debe ser épico).
- Return Day 7 < 30%: revisar comunicación durante trial.
- Completion assessment < 20%: assessment muy largo o no bien
  comunicado su valor.

---

## 16. Privacy y datos sensibles

### 16.1 Datos sensibles

Del perfil, los más sensibles:
- Audios capturados durante práctica.
- Resultados del assessment (nivel real).
- Comportamiento detallado de uso.
- Ubicación geográfica precisa.

### 16.2 Retención

(De `cross-cutting/data-and-events.md` §7.1.)

### 16.3 Consentimiento granular

En onboarding, opt-ins separados:
- ☑ Procesar mis audios para mejorar mi pronunciación (necesario).
- ☑ Usar mis datos anonimizados para mejorar el producto.
- ☐ Compartir datos anonimizados con investigación lingüística.

Último es **opt-in explícito**, no por defecto.

### 16.4 Acceso del usuario a sus datos

Settings → "Mis datos":
- Ver todo el perfil.
- Descargar export en JSON.
- Editar datos declarativos.
- Borrar datos específicos (audios viejos).
- Borrar cuenta completa.

---

## 17. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`ai-roadmap-system.md`](ai-roadmap-system.md) | Genera roadmap inicial y definitivo. |
| [`pedagogical-system.md`](pedagogical-system.md) | Score del mini-test y assessment. |
| [`motivation-and-achievements.md`](motivation-and-achievements.md) | Consume `observed_behavior` para perfil motivacional. |
| [`../architecture/sparks-system.md`](../architecture/sparks-system.md) | 50 trial Sparks; conversion al pago. |
| [`../architecture/authentication-system.md`](../architecture/authentication-system.md) | `user.signed_up` triggers profile creation. |
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) | Notificaciones progresivas del trial. |
| [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) | Tasks de scoring durante assessment. |
| [`../decisions/ADR-006-trial-assessment.md`](../decisions/ADR-006-trial-assessment.md) | Decisión arquitectónica. |
| [`../cross-cutting/i18n.md`](../cross-cutting/i18n.md) | Localización por país. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.2 | Eventos. |

---

*Documento vivo. Actualizar cuando cambien tipos de assessment, flujo
de trial, o se observen patrones de comportamiento que sugieran ajustes.*

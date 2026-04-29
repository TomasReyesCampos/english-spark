# Student Profile and Assessment System

> Sistema integral de perfilado del estudiante, free trial de 7 días con
> observación de comportamiento, y assessment profundo que genera el
> roadmap definitivo de aprendizaje.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Alcance:** Sistema completo (MVP y evolución)

---

## 1. Filosofía del sistema

### 1.1 Principio central

El conocimiento sobre el estudiante se construye en tres capas progresivas:

1. **Lo declarado** (lo que dice de sí mismo en el onboarding inicial).
2. **Lo observado** (cómo se comporta realmente durante 7 días de uso).
3. **Lo medido** (resultado de un assessment estructurado al día 7).

La combinación de estas tres fuentes genera un perfil mucho más rico que
cualquiera de ellas por separado. Lo que el usuario dice que es, lo que
hace en realidad, y lo que las pruebas revelan, frecuentemente difieren.
El sistema trabaja con los tres.

### 1.2 El free trial como inversión, no como prueba

Los 7 días gratuitos no son solo una prueba comercial. Son una fase
deliberada de:

- **Recolección de datos:** el sistema aprende cómo aprende este usuario.
- **Construcción de hábito:** los primeros 7 días son críticos para retención.
- **Demostración de valor:** el usuario experimenta el producto real, no demos.
- **Reducción de fricción:** sin paywall inicial, más usuarios entran al funnel.

El assessment al día 7 convierte esa inversión de tiempo del usuario en un
plan personalizado que genuinamente le aporta valor.

### 1.3 Por qué el assessment es opcional pero fuertemente incentivado

Forzar el assessment como gate obligatorio genera fricción y churn. Hacerlo
opcional pero atractivo permite:

- Usuarios que están listos lo hacen y obtienen un plan superior.
- Usuarios no listos siguen usando la versión limitada (con incentivo
  permanente a hacerlo cuando estén listos).
- El sistema mide cuántos lo hacen y por qué los demás no, y itera.

---

## 2. Perfil del estudiante

### 2.1 Datos del perfil

El perfil del estudiante combina datos de tres fuentes y se mantiene
actualizado durante toda la vida del usuario.

#### 2.1.1 Datos demográficos y geográficos

```typescript
interface StudentDemographics {
  // Geografía
  country: string;                    // ISO 3166-1 alpha-2 ("MX", "AR", "CO")
  region?: string;                    // estado/provincia
  city?: string;
  timezone: string;                   // "America/Mexico_City"

  // Idioma nativo
  native_language: string;            // típicamente "es-MX", "es-AR", "pt-BR"
  spanish_variant?: string;           // mexicano, rioplatense, andino, caribeño

  // Edad y ocupación (opcionales, mejoran personalización)
  age_range?: '18-24' | '25-34' | '35-44' | '45-54' | '55+';
  current_occupation?: string;
}
```

#### 2.1.2 Objetivos y contexto de aprendizaje

```typescript
interface LearningContext {
  // Objetivos primarios (multi-select)
  primary_goals: Array<
    'job_interview' | 'remote_work' | 'business_communication' |
    'travel' | 'studies' | 'personal_growth' | 'exam_prep' |
    'communication_with_family' | 'entertainment'
  >;

  // Variante de inglés objetivo (deriva de objetivos + ubicación)
  target_english_variant: 'american' | 'british' | 'neutral';

  // Deadline si existe
  has_deadline: boolean;
  deadline_date?: Date;
  deadline_context?: string;          // "entrevista en TechCorp"

  // Contexto profesional
  professional_field: 'tech' | 'health' | 'finance' | 'sales' |
                      'marketing' | 'education' | 'legal' | 'creative' |
                      'service' | 'student' | 'other';

  // Para preparación de exámenes
  exam_target?: 'TOEFL' | 'IELTS' | 'Duolingo_English_Test' | 'Cambridge';
  exam_target_score?: number;
}
```

#### 2.1.3 Autoevaluación y preferencias

```typescript
interface SelfAssessment {
  // Nivel autopercibido (CEFR)
  self_perceived_level: 'A2' | 'B1' | 'B1+' | 'B2' | 'B2+' | 'C1';

  // Confianza al hablar (1-5)
  speaking_confidence: number;

  // Ansiedad lingüística (1-5)
  language_anxiety: number;

  // Áreas autopercibidas como débiles
  self_identified_weaknesses: Array<
    'pronunciation' | 'fluency' | 'vocabulary' | 'grammar' |
    'listening' | 'confidence'
  >;

  // Tiempo disponible
  daily_minutes_available: 5 | 10 | 15 | 20 | 30 | 45 | 60;
  preferred_practice_hour: number;    // 0-23
  practice_days_per_week: number;     // 1-7
}
```

#### 2.1.4 Datos observados (calculados por el sistema)

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
  exercises_abandoned_rate: number;   // % que abandona sin completar
  average_attempts_per_exercise: number;
  preferred_exercise_types: string[]; // ranking de tipos preferidos

  // Performance observada
  pronunciation_avg_score: number;
  fluency_wpm_average: number;        // palabras por minuto
  filler_word_rate: number;           // muletillas por minuto
  detected_error_patterns: string[];  // errores recurrentes

  // Indicadores de engagement
  current_streak_days: number;
  longest_streak_days: number;
  total_active_days: number;

  // Calculado periódicamente, no en tiempo real
  last_calculated_at: Date;
}
```

#### 2.1.5 Resultados del assessment

```typescript
interface AssessmentResults {
  completed_at: Date;
  assessment_version: string;         // versión del assessment usado

  // CEFR estimado por el assessment
  measured_cefr_level: 'A2' | 'B1' | 'B1+' | 'B2' | 'B2+' | 'C1';

  // Sub-scores detallados
  scores: {
    pronunciation: number;            // 0-100
    fluency: number;
    grammar: number;
    vocabulary: number;
    listening: number;
    overall_speaking: number;
  };

  // Errores fonéticos específicos detectados
  phonetic_error_patterns: string[];

  // Errores gramaticales específicos detectados
  grammar_error_patterns: string[];

  // Áreas fuertes y débiles ranqueadas
  strongest_areas: string[];
  weakest_areas: string[];

  // Comparación con peers
  peer_percentile?: number;           // vs otros usuarios del mismo país/edad
}
```

### 2.2 Schema en Postgres

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
  target_english_variant   TEXT NOT NULL DEFAULT 'neutral',
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

  -- Observed behavior (calculado periódicamente)
  observed_behavior        JSONB DEFAULT '{}',
  observed_last_calculated TIMESTAMPTZ,

  -- Assessment results
  assessment_completed_at  TIMESTAMPTZ,
  assessment_results       JSONB,

  -- Trial status
  trial_started_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  trial_ends_at            TIMESTAMPTZ NOT NULL DEFAULT (now() + interval '7 days'),
  trial_sparks_remaining   INT NOT NULL DEFAULT 50,
  trial_status             TEXT NOT NULL DEFAULT 'active'
                           CHECK (trial_status IN ('active', 'expired', 'converted', 'abandoned')),

  created_at               TIMESTAMPTZ DEFAULT now(),
  updated_at               TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_profiles_country ON student_profiles(country);
CREATE INDEX idx_profiles_trial_active ON student_profiles(trial_ends_at)
  WHERE trial_status = 'active';
```

### 2.3 Por qué cada campo importa

**Ubicación geográfica** habilita:
- Variante de inglés a priorizar (americano vs británico vs neutral).
- Contexto cultural en roleplays y ejemplos.
- Pricing localizado y métodos de pago apropiados.
- Eventos y campañas localizadas (Día del Estudiante, vacaciones).
- Comunidad regional cuando se implemente.
- Contenido específico para errores comunes según variante de español.

**Variante de español** habilita:
- Detección de "false friends" específicos (un mexicano vs argentino tiene
  diferentes interferencias del español al inglés).
- Pronunciación: rioplatenses tienen menos problema con /s/ final, andinos
  con /b/ vs /v/.
- Vocabulario familiar para explicaciones.

**Profesional field** habilita:
- Roleplays específicos (entrevista técnica vs entrevista de ventas).
- Vocabulario relevante al área.
- Ejemplos contextualizados.

**Variantes de inglés y por qué importan**

| Si el usuario está en | Y su objetivo es | Variante recomendada |
|----------------------|------------------|----------------------|
| Cualquier Latam | Trabajo remoto USA | Americano |
| México | Cualquier objetivo | Americano (proximidad) |
| Cualquier Latam | Estudios en UK | Británico |
| Cualquier Latam | Trabajo en empresa europea | Británico o neutral |
| Cualquier Latam | Sin objetivo específico | Neutral |
| Cualquier Latam | Viajes internacionales | Neutral |

---

## 3. Onboarding inicial (Día 0)

### 3.1 Objetivos del onboarding

- Capturar perfil mínimo viable para empezar a usar la app.
- Generar un roadmap inicial básico (será refinado en el assessment).
- Demostrar valor rápidamente: usuario practicando en menos de 5 minutos.
- Crear expectativa positiva sobre el assessment del día 7.

### 3.2 Flujo del onboarding (5-7 minutos)

**Paso 1: Bienvenida y motivación (30 segundos)**

Pantalla de bienvenida con:
- Mensaje cálido en español neutro.
- Promesa clara: "vas a hablar inglés con confianza".
- "Vamos a conocernos en 5 minutos".

**Paso 2: Ubicación (15 segundos)**

```
¿Dónde estás?

[🇲🇽 México] [🇦🇷 Argentina] [🇨🇴 Colombia]
[🇨🇱 Chile] [🇵🇪 Perú] [🌎 Otro país de Latam]
```

Detección automática inicial vía IP, confirmación por el usuario. Esto
es crítico y debe estar al inicio para personalizar todo lo siguiente.

**Paso 3: Objetivo principal (20 segundos)**

```
¿Por qué querés mejorar tu inglés?
(Podés elegir más de uno)

☐ Conseguir trabajo en empresa internacional
☐ Trabajo remoto desde Latam
☐ Comunicarme mejor en mi trabajo actual
☐ Viajar
☐ Estudios o intercambio
☐ Examen (TOEFL, IELTS, etc.)
☐ Crecimiento personal
```

**Paso 4: Deadline (10 segundos)**

```
¿Tenés alguna fecha en mente?

[ ] Sí, en menos de 1 mes
[ ] En 1 a 3 meses
[ ] En 3 a 6 meses
[ ] No tengo apuro
```

**Paso 5: Contexto profesional (15 segundos)**

```
¿En qué área trabajás o estudiás?

[Tecnología] [Salud] [Finanzas] [Ventas]
[Marketing] [Educación] [Servicios] [Otro]
```

**Paso 6: Autoevaluación (30 segundos)**

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

**Paso 7: Mini-test rápido (3 minutos)**

Versión corta del test que se profundiza el día 7. Tres ejercicios:

1. Repetir una frase estándar para captar pronunciación base.
2. Responder "How are you doing today?" en 30 segundos para captar
   fluidez espontánea.
3. Identificar 3 imágenes nombrándolas en inglés.

**Paso 8: Análisis y bienvenida (30 segundos)**

Mientras el sistema procesa los datos, animación de "armando tu plan".
Resultado: roadmap inicial básico, mensaje personalizado.

```
"Listo, María. Detectamos que tu nivel está alrededor de B1 y tu mayor
desafío es la fluidez. Armamos un plan inicial de 4 semanas.

⚡ Tienes 50 Sparks gratuitos por 7 días para probar todo.
🎯 El día 7 vas a hacer un assessment más profundo que ajustará tu plan."

[Empezar mi primer ejercicio]
```

### 3.3 Roadmap inicial (Día 0)

El roadmap generado en este punto es **explícitamente provisional**. Se
comunica como tal al usuario para crear expectativa positiva sobre el
assessment del día 7.

Características del roadmap inicial:
- 4 semanas de duración estimada.
- 30-40 bloques pre-seleccionados.
- Variante de inglés según ubicación + objetivo.
- Foco general (no hiper-específico) según autoevaluación.

El usuario ve: "Tu plan inicial - se actualizará el día 7".

---

## 4. Free Trial (Días 1-7)

### 4.1 Características del trial

- **Duración:** 7 días desde el registro.
- **Sparks incluidos:** 50 gratuitos (cap absoluto, no rolling).
- **Acceso:** completo a todas las features del Plan Pro.
  - Conversación 1 a 1 con IA.
  - Roleplays personalizados.
  - Pronunciation assessment con feedback IA.
  - Análisis nocturno (versión completa).
  - Notificaciones humanizadas.
- **Limitaciones:** ninguna funcional, solo el cap de Sparks.

### 4.2 Por qué 50 Sparks específicamente

Cálculo:
- 1 conversación de 10 min = 10 Sparks.
- Si el usuario hace 3-5 conversaciones durante el trial, son 30-50 Sparks.
- Más algunas operaciones menores (pronunciación, análisis), llega a ~50.
- Si el usuario es muy intensivo, los 50 Sparks duran 5 días aprox.
- Si el usuario es casual, los 50 Sparks le sobran para los 7 días.

50 Sparks permite probar la magia del producto sin costo descontrolado.
Costo aproximado por usuario en trial: $1.50-2.50 USD si los usa todos.

### 4.3 Sistema de observación durante el trial

Durante los 7 días, el sistema captura sistemáticamente:

```typescript
interface TrialObservation {
  // Día por día
  day_1_completed: boolean;
  day_2_completed: boolean;
  // ... hasta day_7

  // Patrones agregados al día 7
  total_minutes_practiced: number;
  total_sessions: number;
  total_sparks_used: number;
  preferred_session_length: number;
  preferred_practice_hour: number;
  exercise_types_tried: string[];
  exercise_types_preferred: string[];
  abandoned_exercises_count: number;
  conversation_sessions_completed: number;

  // Performance trends
  pronunciation_score_trend: 'improving' | 'stable' | 'declining';
  fluency_score_trend: 'improving' | 'stable' | 'declining';
  detected_strengths: string[];
  detected_weaknesses: string[];

  // Engagement signals
  notification_open_rate: number;
  app_opens_count: number;
  feature_exploration_score: number;  // qué tan diversificado fue el uso
}
```

Esta data se calcula incrementalmente cada noche del trial y se persiste
en `student_profiles.observed_behavior`.

### 4.4 Comunicación durante el trial

Notificaciones progresivas que mantienen al usuario engaged y construyen
expectativa hacia el assessment:

| Día | Mensaje | Canal | Objetivo |
|-----|---------|-------|----------|
| 1 (24h post-registro) | "¡Buen primer día! ¿Listo para el segundo?" | Push | Reforzar hábito |
| 3 | "Llevás 3 días. Tu pronunciación de [X] mejoró." | Push | Mostrar progreso |
| 5 | "Faltan 2 días para tu assessment personalizado. Va a desbloquear tu plan completo." | Push | Crear expectativa |
| 6 | "Mañana es el gran día. El assessment toma 20 minutos y va a transformar tu plan." | Push + In-app | Preparar |
| 7 | "Llegó el día. Hacé tu assessment ahora y desbloqueá tu plan personalizado." | Push prominente | Trigger |

### 4.5 ¿Qué pasa si los Sparks se acaban antes del día 7?

Escenario probable: usuario muy entusiasta consume sus 50 Sparks en día 4-5.

Comportamiento del sistema:
- Mensaje claro: "Usaste todos tus Sparks gratuitos. Tu plan completo te
  espera al hacer el assessment del día 7."
- Acceso continuo a preassets ilimitados (no consumen Sparks).
- Opción de "hacer el assessment ahora" si el usuario está listo.
- Si elige hacerlo antes del día 7, el assessment se desbloquea.

Esto convierte un potencial momento de frustración en una oportunidad de
conversión adelantada.

### 4.6 ¿Qué pasa el día 8 si no hizo el assessment?

Escenario: usuario casual no completó el assessment al día 7.

Comportamiento:
- App sigue funcionando con acceso a preassets.
- Sin más Sparks gratuitos (los 50 se acabaron o expiraron).
- Sin conversación 1 a 1.
- Banner persistente: "Hacé tu assessment para desbloquear tu plan".
- Notificaciones suaves recordándolo (días 8, 10, 14, 21, 30).
- Si pasan 30 días sin assessment, marca como "trial_status: abandoned"
  pero permite seguir usando preassets gratis indefinidamente.

Filosofía: nunca cerrar la puerta. El usuario que abandonó hoy puede
volver mañana.

---

## 5. El Assessment (Día 7)

### 5.1 Estructura del assessment

Duración total: 20-25 minutos. Diseñado para ser desafiante pero no
agotador. Dividido en 4 partes claras.

#### 5.1.1 Parte 1: Confirmación y profundización del perfil (3 minutos)

Re-confirmar y profundizar lo declarado en el onboarding inicial. Algunas
preguntas nuevas:

- ¿Cambió tu objetivo desde que empezaste?
- ¿En qué situaciones específicas necesitás hablar inglés?
- ¿Qué tan importante es para vos sonar "natural" vs "correcto"?
- Si tu objetivo es laboral: ¿cómo te imaginás usándolo? (entrevistas,
  reuniones, escritura, llamadas).
- Para los que tienen deadline: profundizar en qué exactamente esperan
  haber logrado.

#### 5.1.2 Parte 2: Test de habilidades activas (12 minutos)

Cinco ejercicios estructurados:

**Ejercicio 1: Lectura comprensiva con resumen oral (3 min)**
- Leer un artículo de 200 palabras.
- Resumir en voz alta sin verlo (60 segundos).
- Mide: comprensión lectora, vocabulario receptivo, fluidez productiva.

**Ejercicio 2: Roleplay situacional (3 min)**
- Situación específica al objetivo del usuario.
- Para "job_interview": "Te están entrevistando para un puesto. Contame
  sobre un desafío que superaste en tu trabajo".
- Para "travel": "Estás en un hotel y la habitación tiene problemas. ¿Qué decís?".
- Mide: vocabulario activo en contexto, fluidez espontánea, gramática.

**Ejercicio 3: Pronunciación específica (2 min)**
- 10 frases con sonidos críticos para hispanohablantes.
- Cubren /θ/, /ð/, /v/, /ʒ/, vocales tensas vs laxas, finales consonánticos.
- Mide: pronunciación detallada por fonema.

**Ejercicio 4: Listening con respuesta (2 min)**
- Audio de 90 segundos a velocidad nativa con acento del target_english_variant.
- 3 preguntas comprensivas con respuesta oral.
- Mide: comprensión auditiva a velocidad real, capacidad de procesar.

**Ejercicio 5: Producción libre (2 min)**
- Pregunta abierta relacionada al objetivo.
- 2 minutos para responder libremente.
- Mide: fluidez sostenida, capacidad de organizar ideas, vocabulario amplio.

#### 5.1.3 Parte 3: Test de habilidades receptivas (3 minutos)

**Vocabulary in context:** 10 frases con palabras subrayadas. Elegir
sinónimo correcto. Adaptativo: si responde bien, sube dificultad; si
falla, baja.

**Grammar recognition:** 8 frases con errores sutiles. Identificar cuál
está bien y cuál mal.

#### 5.1.4 Parte 4: Aspiraciones y motivación (2 minutos)

- ¿Qué te frustra más cuando hablás inglés?
- ¿En qué situación te gustaría sentirte cómodo en 3 meses?
- ¿Qué tipo de feedback preferís? (corrección inmediata, al final, suave, directo).
- ¿Preferís sesiones cortas frecuentes o largas espaciadas?

### 5.2 Análisis del assessment

Después del assessment, el sistema procesa todos los inputs:

**Análisis técnico (automatizado):**
- STT de todos los audios con timestamps.
- Pronunciation scoring por fonema (modelo propio o Azure).
- Análisis sintáctico de producciones.
- Detección de errores gramaticales.
- Cálculo de métricas: WPM, pausas, muletillas.

**Análisis con IA (LLM):**
- Evaluación holística de fluidez y coherencia.
- Detección de patrones de error específicos.
- Identificación de strengths y weaknesses ranqueadas.
- Generación de CEFR estimado con justificación.

**Combinación con datos observados de los 7 días:**
- ¿La autoevaluación coincide con lo medido?
- ¿El comportamiento durante el trial confirma o contradice las preferencias declaradas?
- ¿Qué tipo de ejercicios funcionaron mejor para este usuario?

### 5.3 Generación del plan personalizado definitivo

Con toda esta data, el sistema genera el roadmap definitivo. Diferencias
con el roadmap inicial:

| Aspecto | Roadmap inicial (Día 0) | Roadmap definitivo (Día 7+) |
|---------|------------------------|----------------------------|
| Basado en | 5 preguntas + mini-test | Perfil completo + 7 días observación + assessment de 20 min |
| Duración | 4 semanas estimadas | 8-16 semanas precisas según meta |
| Bloques | 30-40 generales | 50-80 específicos al perfil |
| Foco | General por objetivo | Específico a errores detectados |
| Variante inglés | Por defecto del país | Confirmada por preferencia |
| Niveles | 4 estándar | 4-6 personalizados con nombres motivadores |
| Mensaje al usuario | "Plan inicial provisional" | "Tu plan definitivo, hecho para vos" |

### 5.4 Presentación de resultados al usuario

Inmediatamente después del assessment, pantalla de resultados que es
emocional y memorable:

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
- Gráfico de scores en cada dimensión.
- Comparación con percentiles de usuarios similares.
- Top 3 áreas de mejora con explicación.
- Top 3 fortalezas para sentir orgullo.
- Tu camino visualizado con los niveles del roadmap.

### 5.5 Costo del assessment

Por usuario que completa el assessment:
- Análisis técnico (Whisper, pronunciation scoring): ~$0.05
- LLM análisis holístico (Claude Haiku batch): ~$0.03
- Generación de roadmap definitivo: ~$0.02
- **Total: ~$0.10 por assessment**

Para 1.000 usuarios que lo completan: $100.
Para 10.000: $1.000.
Manejable considerando que es una operación única por usuario que
desbloquea valor significativo.

---

## 6. Conversion al pago

### 6.1 Momento de la conversión

El día 7, después del assessment, es el momento ideal para presentar la
oferta de pago. El usuario:

- Acaba de invertir 20 minutos en un assessment.
- Vio resultados detallados de su nivel real.
- Tiene un plan personalizado creado para él/ella.
- Está en pico de motivación y compromiso.

### 6.2 Presentación de planes

Después de mostrar el plan personalizado:

```
Tu plan está listo. ¿Cómo querés practicarlo?

┌─────────────────────────────────────┐
│  Plan Básico - $30/mes              │
│  • Acceso a tu plan completo        │
│  • 30 Sparks/mes para conversación  │
│  • Análisis de progreso semanal     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  Plan Pro - $100/mes  ⭐ RECOMENDADO │
│  • Todo lo del Básico               │
│  • 200 Sparks/mes (sesiones diarias)│
│  • Insights personalizados con IA   │
│  • Re-evaluación cada 4 semanas     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│  Plan Premium - $250/mes            │
│  • Todo lo del Pro                  │
│  • 600 Sparks/mes (uso intensivo)   │
│  • Roleplays personalizados         │
│  • Prioridad en procesamiento       │
└─────────────────────────────────────┘

[Empezar con Pro] [Ver todos los detalles]
```

### 6.3 Optimizaciones de conversión

**Anchoring:** mostrar Pro como recomendado hace que Básico parezca poco
y Premium parezca mucho. Pro captura la mayoría.

**Scarcity transparente:** "tu plan personalizado está listo y esperándote".

**Risk reversal:** "cancelá cuando quieras, los Sparks comprados no expiran
por 6 meses".

**Social proof:** "más de X usuarios en [país del usuario] eligieron Pro
este mes".

**Comparación clara:** mostrar qué obtiene el usuario por el precio,
incluyendo equivalencias (ej: "el costo de un café al día").

### 6.4 Si el usuario no convierte

No usar tácticas agresivas. Estrategia:
- Acceso continuo a preassets ilimitados gratis.
- Plan personalizado visible pero con bloqueos en bloques que requieren Sparks.
- Ofertas suaves periódicas (descuento primer mes, etc.).
- Re-engagement campaigns que mostren valor del plan.
- Permitir compra por packs de Sparks únicos para usuarios que no quieren
  suscripción.

---

## 7. Re-evaluación periódica

### 7.1 ¿Por qué re-evaluar?

El nivel del estudiante evoluciona. Un usuario que entró en B1 puede
estar en B2 después de 3 meses. Sin re-evaluación:
- El roadmap queda obsoleto.
- Se pierde la oportunidad de celebrar progreso.
- El usuario no ve evidencia objetiva de mejora.

### 7.2 Cadencia recomendada

| Plan | Frecuencia | Profundidad |
|------|-----------|-------------|
| Básico | Cada 8 semanas | Mini-assessment (5 min) |
| Pro | Cada 4 semanas | Assessment intermedio (10 min) |
| Premium | Cada 4 semanas | Assessment completo (20 min) |

Las re-evaluaciones del Pro y Premium son una feature diferenciadora del
plan, no una obligación. Premium recibe assessment completo regularmente.

### 7.3 Comparación temporal

La pieza más poderosa de las re-evaluaciones es **mostrar la evolución**:

```
Tu progreso desde el último assessment:

Pronunciation:  72 → 84  (+12) 📈
Fluency:        65 → 71  (+6)  📈
Grammar:        78 → 80  (+2)  →
Vocabulary:     70 → 79  (+9)  📈

🎉 Subiste de B1 a B1+
🎯 Tu siguiente objetivo: alcanzar B2 en 6 semanas
```

Esto convierte el progreso "invisible" en algo concreto y motivador.

### 7.4 Roadmap dinámico

Cada re-evaluación ajusta el roadmap:
- Bloques completados ya no aparecen.
- Bloques que ya domina (verificado en re-assessment) se marcan como
  "tested out" y se saltan.
- Bloques nuevos se agregan según patrones detectados.
- El target_cefr puede ajustarse si va más rápido/lento de lo esperado.

---

## 8. Personalización por país y región

### 8.1 Variantes de contenido por región

Adaptaciones específicas según `country` y `spanish_variant`:

**Para usuarios de México:**
- Variante de inglés default: americano.
- Roleplays con nombres de empresas y productos del mercado mexicano.
- Vocabulario en explicaciones: español neutro con cierta inclinación mexicana.
- Eventos locales: Día del Estudiante (23 mayo), vacaciones de verano (julio-agosto).

**Para usuarios de Argentina:**
- Variante de inglés default: neutral con cierta inclinación británica.
- Roleplays adaptados al contexto rioplatense.
- Vocabulario: español neutro pero reconociendo voseo en explicaciones.
- Eventos locales: Día del Estudiante (21 septiembre), vacaciones de invierno (julio).

**Para usuarios de Colombia:**
- Variante de inglés default: americano.
- Contexto profesional con énfasis en empresas multinacionales presentes
  (sector outsourcing, BPO, tech).

**Para usuarios de otros países:**
- Variante por defecto: neutral.
- Contenido más generalizado.
- Sistema aprende preferencias con uso.

### 8.2 Pricing localizado

Aunque el plan en MXN es la referencia, el sistema muestra precios en la
moneda local del usuario:

| País | Plan Básico | Plan Pro | Plan Premium |
|------|-------------|----------|--------------|
| México | $30 MXN | $100 MXN | $250 MXN |
| Argentina | Dinámico (ajustado por inflación) | | |
| Colombia | $7.000 COP | $22.000 COP | $55.000 COP |
| Chile | $1.500 CLP | $5.000 CLP | $12.500 CLP |
| Perú | $7 PEN | $22 PEN | $55 PEN |

Argentina requiere ajuste especial por inflación: revisar mensualmente o
implementar pricing en USD con conversión al momento del cobro.

### 8.3 Métodos de pago localizados

| País | Métodos prioritarios |
|------|---------------------|
| México | Tarjeta, OXXO, MercadoPago, Apple/Google Pay |
| Argentina | MercadoPago, tarjeta, transferencia |
| Colombia | Nequi, PSE, tarjeta, Bold |
| Chile | Webpay, tarjeta, MACH |
| Perú | Yape, Plin, tarjeta |
| Brasil (futuro) | PIX, tarjeta, boleto |

Implementar todos requiere stack robusto (dLocal, EBANX, o gateways
locales). Empezar con tarjeta + Apple/Google Pay + 1-2 locales por país
clave.

---

## 9. Privacy y manejo de datos sensibles

### 9.1 Qué datos son sensibles

Del perfil del estudiante, los más sensibles:
- Audios capturados durante práctica.
- Resultados del assessment (nivel real).
- Comportamiento detallado de uso.
- Ubicación geográfica precisa.

### 9.2 Política de retención

| Tipo de dato | Retención por defecto |
|-------------|----------------------|
| Audios crudos | 30 días desde grabación |
| Transcripciones de audio | Indefinido (anonimizadas después de 90 días) |
| Eventos de comportamiento | 12 meses para análisis |
| Datos del perfil | Mientras la cuenta esté activa |
| Resultados de assessment | Indefinidos (parte del progreso) |

Después de account deletion, todo se borra siguiendo el flujo del documento
de authentication.

### 9.3 Consentimiento granular

En el onboarding, opt-ins separados para:
- ☑ Procesar mis audios para mejorar mi pronunciación (necesario).
- ☑ Usar mis datos anonimizados para mejorar el producto.
- ☐ Compartir datos anonimizados con investigación lingüística.

El último es opt-in explícito, no por defecto.

### 9.4 Acceso del usuario a sus datos

Settings → "Mis datos":
- Ver todo el perfil.
- Descargar export en JSON.
- Editar datos declarativos.
- Borrar datos específicos (audios viejos, etc.).
- Borrar cuenta completa.

---

## 10. Plan de implementación por fases

### 10.1 Fase 1: MVP (meses 0-3)

**Crítico - implementar primero:**
- Schema de `student_profiles` completo.
- Onboarding inicial con todas las preguntas.
- Detección de ubicación + variante de inglés.
- Trial de 7 días con cap de 50 Sparks.
- Mini-test inicial (5 min).
- Roadmap inicial generado por IA.

**Importante - segunda mitad del trimestre:**
- Sistema de observación durante el trial.
- Notificaciones progresivas del trial.
- Pre-aviso del assessment día 5-6.

### 10.2 Fase 2: Assessment completo (meses 3-6)

**Crítico:**
- Assessment de 20 minutos completo (las 4 partes).
- Análisis combinado: técnico + LLM + observación.
- Generación de roadmap definitivo.
- Presentación emocional de resultados.
- Conversion flow al pago.

**Importante:**
- Sistema de "qué pasa si no hace assessment" con bloqueos suaves.
- Re-engagement campaigns para no-assessors.

### 10.3 Fase 3: Sofisticación (meses 6-12)

**Importante:**
- Re-evaluaciones periódicas según plan.
- Comparación temporal visual.
- Pricing localizado real (no solo MXN).
- Métodos de pago locales por país.

**Aspiracional:**
- Variantes de contenido específicas por región.
- Modelo propio de pronunciation scoring entrenado con data acumulada.
- Predicción de churn temprano basada en observación del trial.

### 10.4 Fase 4: Optimización (año 2+)

- A/B testing del assessment (variantes para optimizar conversion).
- Personalización del onboarding según ubicación y device.
- Segmentación dinámica para campañas y comunicación.
- Re-evaluaciones gamificadas con elementos sociales.

---

## 11. Métricas críticas

### 11.1 Embudo de conversión

| Etapa | Métrica | Target |
|-------|---------|--------|
| Registro | Completed onboarding | >85% |
| Trial | Volvió día 2 | >60% |
| Trial | Volvió día 7 | >40% |
| Trial | Completó assessment | >30% |
| Post-assessment | Compró suscripción | >40% |
| Conversion total | Registro → pago | >12% |

### 11.2 Calidad del perfil

- % de usuarios con perfil "completo" (todos los campos críticos).
- Coincidencia entre nivel autopercibido y nivel medido (gap analysis).
- Predicción del observed_behavior vs reality.

### 11.3 Ajustes basados en datos

- Si conversion del onboarding < 70%: simplificar preguntas.
- Si return día 2 < 50%: revisar primer ejercicio (debe ser épico).
- Si return día 7 < 30%: revisar comunicación durante trial.
- Si completion del assessment < 20%: asssessment es muy largo o no se
  comunicó bien su valor.

---

## 12. Decisiones abiertas

- [ ] ¿Permitir hacer el assessment antes del día 7 si el usuario lo
  pide explícitamente?
- [ ] ¿Re-aplicar el assessment cuando el usuario cambia significativamente
  su objetivo (ej: pasa de "travel" a "job interview")?
- [ ] ¿Cómo manejar usuarios que claramente "rompen" el assessment
  (responden al azar, no graban audio, etc.)? Detectar y reintentar.
- [ ] ¿Mostrar el roadmap inicial completo o solo los primeros niveles
  para mantener intriga sobre el assessment?
- [ ] ¿Permitir descargar un PDF del resultado del assessment?
  (Usable como prueba para empleadores, profesores, etc.)

---

## 13. Referencias internas

- `docs/business/plan_de_negocio.docx` — Plan de negocio.
- `docs/product/ai-roadmap-system.md` — Sistema de roadmap (consume el perfil).
- `docs/product/motivation-and-achievements.md` — Sistema de logros (a crear).
- `docs/architecture/sparks-system.md` — Sistema de Sparks (gobierna el trial).
- `docs/architecture/authentication-system.md` — Auth (anonymous → trial → permanent).
- `docs/architecture/ai-gateway-strategy.md` — AI Gateway (procesa assessments).
- `docs/architecture/notifications-system.md` — Notificaciones del trial.
- `docs/decisions/ADR-006-trial-assessment.md` — ADR formal (a crear).

---

*Documento vivo. Actualizar cuando cambien tipos de assessment, fluyo de
trial, o se observen patrones de comportamiento que sugieran ajustes.*

# Assessment Content Bank

> Pool de items concretos para el assessment del Day 7 (definido
> estructuralmente en `student-profile-and-assessment.md` §6). Cada
> assessment session se compone seleccionando items del pool según el
> goal y CEFR estimado del user.

**Estado:** Diseño v1.0
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador (al crear contenido) +
content reviewer humano (validación pedagógica).
**Alcance:** Items para assessment del Day 7 + re-evaluaciones
periódicas.

---

## 0. Cómo leer este documento

- §1 establece **el modelo de pool** y sampling logic.
- §2 cubre **estructura del assessment** (recap de
  `student-profile-and-assessment.md` §6).
- §3-§7 contienen **items concretos por parte** (1, 2, 3, 4 + variants).
- §8 cubre **scoring y validation**.
- §9 cubre **mantenimiento del pool**.
- §10 cubre **decisiones cerradas**.

---

## 1. Modelo de pool y sampling

### 1.1 Concepto

Un **item del pool** es una unidad reutilizable del assessment
(passage, roleplay scenario, audio, vocab question, grammar question,
etc.). Items se almacenan con metadata que permite sampling
inteligente.

Ventajas vs assessment fijo:
- Cada user obtiene un assessment **único** (anti-cheating: no se
  comparte entre users).
- Re-evaluaciones cada 4 semanas no repiten items.
- Pool crece con el tiempo sin tocar el código del assessment.

### 1.2 Sampling logic

Al iniciar `requestAssessment`:

```typescript
async function buildAssessmentSession(
  userProfile: StudentProfile,
  observedBehavior: ObservedBehavior,
): Promise<AssessmentSession | { user_choice_required: true; options: string[] }> {
  const goal = userProfile.primary_goals[0];
  const targetVariant = userProfile.target_english_variant;
  const previousAssessmentItems = await getPreviousAssessmentItemIds(userProfile.user_id);

  // CEFR hybrid: combina self-perceived + measured (mini-test)
  // Ver student-profile-and-assessment.md §13.2
  const cefrDecision = determineCefrForAssessment(userProfile);

  // v1.2: Skip preguntas redundantes si ya tenemos respuestas válidas
  // de sporadic questions (no flagged como fake).
  // Ver student-profile-and-assessment.md §7.1.8
  const sporadicResponses = await getSporadicResponsesByUser(userProfile.user_id, {
    only_non_fake: true,
  });
  const skipQuestions = inferRedundantQuestions(sporadicResponses);

  if (cefrDecision.show_choice) {
    // Cliente muestra UI: "Detectamos X, declaraste Y. ¿Cuál prefieres?"
    return { user_choice_required: true, options: cefrDecision.options };
  }

  const cefrLevel = cefrDecision.cefr;

  return {
    parte_1_questions: sampleProfileQuestions(userProfile),
    parte_2_exercises: [
      sampleReadingPassage({ goal, cefr: cefrLevel, exclude: previousAssessmentItems }),
      sampleRoleplayScenario({ goal, cefr: cefrLevel, exclude: ... }),
      samplePronunciationSet({ targetVariant, exclude: ... }),
      sampleListeningExercise({ goal, cefr: cefrLevel, targetVariant, exclude: ... }),
      sampleFreeResponsePrompt({ goal, exclude: ... }),
    ],
    parte_3_exercises: [
      sampleVocabularyQuestions({ cefr: cefrLevel, count: 10, exclude: ... }),
      sampleGrammarQuestions({ cefr: cefrLevel, count: 8, exclude: ... }),
    ],
    parte_4_questions: sampleAspirationsQuestions(userProfile),
  };
}

// Lógica hybrid (ver student-profile-and-assessment.md §13.2)
function determineCefrForAssessment(profile: StudentProfile): {
  cefr: string;
  show_choice: boolean;
  options?: [string, string];
  show_explanation?: boolean;
} {
  const selfPerceived = profile.self_perceived_level;
  const measured = profile.initial_test_results?.cefr_estimate;

  if (!measured) return { cefr: selfPerceived, show_choice: false };
  if (!selfPerceived) return { cefr: measured, show_choice: false };

  const distance = cefrLevelDistance(selfPerceived, measured);

  if (distance === 0) {
    return { cefr: selfPerceived, show_choice: false };
  }
  if (distance === 1) {
    return {
      cefr: 'user_choice_required',
      show_choice: true,
      options: [measured, selfPerceived],
    };
  }
  // distance >= 2: usar el más alto (benefit of doubt al user)
  return {
    cefr: cefrLevelMax(selfPerceived, measured),
    show_choice: false,
    show_explanation: true,
  };
}
```

#### UX cuando hay user_choice

```
┌─────────────────────────────────────┐
│  Estamos por evaluarte              │
│                                     │
│  Detectamos en tu primera prueba    │
│  que estás cerca de B1+.            │
│                                     │
│  Tú declaraste sentirte en B2.      │
│                                     │
│  ¿En qué nivel prefieres evaluarte? │
│                                     │
│  [ B1+ (más cómodo)            ]    │
│  [ B2 (más desafiante)         ]    │
│                                     │
│  💡 Si te queda muy difícil/fácil   │
│     podemos recalibrar después.     │
└─────────────────────────────────────┘
```

### 1.3 Tabla `assessment_items` (schema)

```sql
CREATE TABLE assessment_items (
  id              TEXT PRIMARY KEY,             -- 'reading_passage_remote_work_b2_v1'
  version         TEXT NOT NULL DEFAULT '1.0.0',

  item_type       TEXT NOT NULL CHECK (item_type IN (
                    'profile_question',
                    'reading_passage',
                    'roleplay_scenario',
                    'pronunciation_set',
                    'listening_exercise',
                    'free_response_prompt',
                    'vocabulary_question',
                    'grammar_question',
                    'aspirations_question'
                  )),

  -- Targeting
  cefr_level      TEXT NOT NULL,                -- 'B1', 'B1+', 'B2', 'B2+', 'C1'
  goal_relevance  TEXT[] DEFAULT '{}',          -- ['job_interview', 'travel', 'all']
  target_variant  TEXT NOT NULL DEFAULT 'us'
                  CHECK (target_variant IN ('us', 'uk', 'neutral')),

  -- Contenido
  content         JSONB NOT NULL,               -- shape varía por item_type

  -- Atomics referenciados (para items que usan media)
  media_atomics   JSONB DEFAULT '[]',

  -- Scoring config
  scoring_rubric  JSONB,                        -- cómo se evalúa

  -- Lifecycle
  approved        BOOLEAN NOT NULL DEFAULT false,
  archived        BOOLEAN NOT NULL DEFAULT false,
  use_count       INT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_items_type_cefr ON assessment_items(item_type, cefr_level)
  WHERE approved = true AND archived = false;
CREATE INDEX idx_items_goal ON assessment_items USING gin(goal_relevance)
  WHERE approved = true;
```

### 1.4 Pool size target

| Item type | MVP | Fase 2 |
|-----------|----:|-------:|
| Reading passages (por CEFR × goal) | 5 | 15 |
| Roleplay scenarios (por goal) | 4 | 10 |
| Pronunciation sets | 3 | 5 |
| Listening exercises (por CEFR × goal) | 5 | 15 |
| Free response prompts (por goal) | 4 | 10 |
| Vocabulary questions (por CEFR) | 50 | 200 |
| Grammar questions (por CEFR) | 40 | 150 |
| Profile questions (estáticos, no sample) | 8 | 8 |
| Aspirations questions (estáticos, no sample) | 4 | 4 |

A 4 CEFR levels MVP × 3 goals + variantes:
- ~60-100 items totales en MVP.
- Suficiente para 3-4 re-evaluations sin repetir.

---

## 2. Estructura recap

(De `student-profile-and-assessment.md` §6.)

```
PARTE 1: Confirmación y profundización del perfil (3 min)
  ├── 5 preguntas estáticas
  └── 3 preguntas adaptativas según goal

PARTE 2: Test de habilidades activas (12 min)
  ├── Ejercicio 1: Lectura comprensiva con resumen oral (3 min)
  ├── Ejercicio 2: Roleplay situacional (3 min)
  ├── Ejercicio 3: Pronunciación específica (2 min)
  ├── Ejercicio 4: Listening con respuesta (2 min)
  └── Ejercicio 5: Producción libre (2 min)

PARTE 3: Test de habilidades receptivas (3 min)
  ├── Vocabulary in context (10 questions, 90s)
  └── Grammar recognition (8 questions, 90s)

PARTE 4: Aspiraciones y motivación (2 min)
  └── 4 preguntas
```

Total: 20-25 min, 12 ejercicios principales.

---

## 3. Parte 1: Profile questions (concretas)

### 3.1 Preguntas estáticas (siempre se hacen)

#### Q1: Cambio de objetivo

```
¿Cambió tu objetivo desde que empezaste a usar la app?

[ ] No, sigue igual
[ ] Sí, ahora es: [select del catálogo de goals]
[ ] No estoy seguro/a
```

#### Q2: Situaciones específicas (multi-select)

```
¿En qué situaciones específicas necesitás hablar inglés?
(Marcá todas las que apliquen)

☐ Reuniones de trabajo
☐ Entrevistas laborales
☐ Presentaciones
☐ Llamadas con clientes/proveedores
☐ Email y escritura
☐ Viajes
☐ Comunicación informal con amigos/familia
☐ Estudios o academia
☐ Otra: [input]
```

#### Q3: Natural vs correcto

```
¿Qué tan importante es para vos sonar "natural" vs "correcto"?

[1: Solo correcto importa] ─── [3: Ambos por igual] ─── [5: Solo natural importa]
```

### 3.2 Preguntas adaptativas (depende de goal)

#### Si goal incluye `job_interview` o `remote_work`:

```
Q4a: ¿Cómo te imaginás usando inglés en tu trabajo?
(Marcá las situaciones más probables)

☐ Entrevistas (1:1)
☐ Reuniones de equipo
☐ Presentaciones a clientes
☐ Email y mensajes (Slack, Teams)
☐ Llamadas de teléfono
☐ Workshops o trainings
☐ Networking events
```

#### Si goal incluye `travel`:

```
Q4b: ¿A qué tipo de país viajarías?

[ ] Estados Unidos
[ ] Reino Unido / Europa
[ ] Asia (donde inglés es común)
[ ] Cualquier país de habla inglesa
```

#### Si tiene deadline:

```
Q5: ¿Qué exactamente esperás haber logrado para esa fecha?

[Open text, max 200 chars]

Ejemplo: "Pasar la entrevista final en TechCorp y poder mantener
reuniones de 1h sin esfuerzo."
```

---

## 4. Parte 2: Test de habilidades activas

### 4.1 Ejercicio 1: Lectura comprensiva con resumen oral

**Estructura:**
1. User ve passage en pantalla (200 palabras, ~60s para leer).
2. Tap "Continuar" oculta el texto.
3. User tiene 60s para resumir oralmente.
4. Audio se procesa (STT + análisis).

**Sampling:** 1 passage de pool, matchando `goal_relevance` y
`cefr_level`.

**Mide:** comprensión lectora, vocabulario receptivo, fluidez
productiva, capacidad de paraphrase.

#### Item: `reading_passage_remote_work_b2_v1`

**Goal relevance:** `job_interview`, `remote_work`.
**CEFR:** B2.

**Passage:**

```
Remote work has fundamentally changed the way teams collaborate.
Before the pandemic, most professionals commuted daily to physical
offices, where they relied on face-to-face meetings and impromptu
hallway conversations. Today, distributed teams across multiple time
zones use asynchronous tools like Slack, Notion, and shared video
recordings to keep everyone aligned.

While this flexibility has improved work-life balance for many, it has
also introduced new challenges: decision-making can slow down when
teammates wait days for responses, and remote workers sometimes report
feeling disconnected from company culture. Companies that thrive in
this new environment have invested in clear written communication,
frequent video check-ins, and intentional in-person gatherings a few
times a year.
```

**Word count:** 137 (target ~150-200, slightly under for B2 entry).

**Prompt al user:** "Now, in your own words, summarize what you just
read. You have 60 seconds. Try to mention the main idea and at least
two specific points."

**Scoring rubric:**
```yaml
key_points_to_mention:
  - "remote work changed how teams work"
  - "challenges: slow decisions OR disconnection from culture"
  - "solutions: written communication OR video OR in-person meetings"
weight_by_dimension:
  fluency: 0.30
  vocabulary: 0.25
  grammar: 0.20
  pronunciation: 0.15
  comprehension: 0.10
```

#### Item: `reading_passage_travel_chile_b2_v1`

**Goal relevance:** `travel`.
**CEFR:** B2.

**Passage:**

```
Patagonia, the southernmost region of South America, attracts thousands
of travelers each year despite its remote location and challenging
weather. Visitors come from all over the world to hike through the
dramatic landscapes of Torres del Paine National Park, where granite
peaks rise above turquoise lakes and ancient glaciers slowly carve
through the mountains.

What makes the region particularly special is its unpredictability.
Weather can change from sunshine to fierce winds within minutes, and
experienced hikers always pack layers regardless of the forecast. Local
guides recommend at least five days to fully experience the park,
though many travelers wish they had stayed longer once they see the
scenery firsthand.
```

**Prompt:** Same as before.

#### Item: `reading_passage_daily_friendship_b2_v1`

**Goal relevance:** `daily_conversation`, `personal_growth`.
**CEFR:** B2.

**Passage:**

```
Maintaining friendships in adulthood can be surprisingly difficult.
Unlike school or college, where you naturally see the same people
every day, adult life pulls everyone in different directions: career,
family, relocations, and changing priorities. Many people report
feeling lonelier in their thirties than they did at any other point
in their lives.

The good news is that strong friendships can survive these challenges
when both people put in consistent, low-pressure effort. A monthly
phone call, an occasional voice message, or a quick "thinking of you"
text can keep a connection alive across years of physical distance.
The key is not the frequency, but the genuine quality of the
interactions when they do happen.
```

#### Pool inicial MVP

5 passages × 4 CEFR levels × 3 goals = 60 max, pero con goal_relevance =
`['all']` para algunos genéricos podemos cubrir con ~15 passages.

### 4.2 Ejercicio 2: Roleplay situacional

**Estructura:**
1. AI introduces scenario (texto + audio).
2. AI hace primer message.
3. User responde (90s max).
4. AI responde adaptativamente (1-2 follow-ups).
5. Cierre.

**Sampling:** 1 scenario de pool, matchando `goal_relevance`.

**Mide:** vocabulario activo en contexto, fluidez espontánea, gramática,
listening, pragmática.

#### Item: `roleplay_job_interview_behavioral_b2_v1`

**Goal relevance:** `job_interview`.
**CEFR:** B2.

**Setup mostrado al user:**

```
🎭 Scenario: Job Interview

You are interviewing for a Senior Marketing Manager position at a
US tech company. The interviewer (Sarah) will ask you behavioral
questions. Answer naturally, as you would in a real interview.

Tip: It's OK to take a moment to think before answering. Aim for
clear, structured responses (around 60-90 seconds each).

Ready? Tap below to start.
```

**Conversation flow:**

```yaml
step_1:
  ai_message: "Hi, thanks for joining today. Before we dive in, could you tell me a little about yourself and what brought you to apply for this role?"
  expected_topics: ["background", "career_progression", "motivation"]
  duration_target_seconds: 60

step_2_followup_strong:
  trigger: "if user mentioned specific achievements"
  ai_message: "That's interesting. Could you tell me about a specific challenge you faced in your previous role and how you handled it?"

step_2_followup_weak:
  trigger: "if user response was very short or vague"
  ai_message: "Let's get more specific. Could you tell me about a project you led that didn't go as planned, and what you learned from it?"

step_3:
  ai_message: "Last question: where do you see yourself in three years, and how does this role fit into that vision?"
  expected_topics: ["future_goals", "growth_mindset", "alignment_with_role"]

step_4:
  ai_message: "Great. Thanks for sharing that. We'll be in touch with next steps soon."
```

**Scoring rubric:**
```yaml
weights:
  fluency: 0.25
  vocabulary: 0.25 (business + STAR-relevant)
  grammar: 0.20
  pronunciation: 0.15
  pragmatics: 0.15 (appropriate register, structure)
target_subskills:
  - flu_extended_speech
  - vocab_business_general
  - gram_past_simple_continuous
  - gram_present_perfect
  - gram_conditionals_second_third
```

#### Item: `roleplay_travel_hotel_problem_b2_v1`

**Goal relevance:** `travel`.
**CEFR:** B2.

**Setup:**

```
🎭 Scenario: Hotel Issue

You just checked into a hotel in San Francisco. As you enter your
room, you notice the air conditioning isn't working and there's a
strong smell. You go down to the front desk to talk to the receptionist.

Goal: Get the issue resolved (room change, repair, or compensation).
```

**Flow:**

```yaml
step_1:
  ai_message: "Hi! Welcome back. How can I help you?"
  expected_topics: ["explain_problem", "request_solution"]

step_2_followup_apologetic:
  trigger: "if user requested room change clearly"
  ai_message: "Oh, I'm so sorry about that. Let me check our availability. Unfortunately, we're fully booked tonight, but I can offer you a maintenance team to fix it within an hour, or a complimentary breakfast tomorrow. What would you prefer?"

step_3:
  ai_message: "Got it. I'll arrange that right away. Is there anything else I can help with?"
```

#### Item: `roleplay_business_email_explain_b2_v1`

**Goal relevance:** `business_communication`, `remote_work`.
**CEFR:** B2.

**Setup:**

```
🎭 Scenario: Project Update Call

Your manager (Mike) has called you to ask about a project that's
behind schedule. Be honest, professional, and propose solutions.
```

**Flow:** análogo al pattern.

#### Item: `roleplay_daily_meeting_friend_b1_v1`

Para B1 entry users con goal `daily_conversation`.

**Setup:** "You haven't seen your friend in 6 months. Catch up over
coffee."

### 4.3 Ejercicio 3: Pronunciación específica

**Estructura:**
1. 10 frases en pantalla, una a la vez.
2. User lee cada una (10s max por frase).
3. Audio se analiza por phoneme.

**Sampling:** 1 set de pool, matchando `target_variant`.

**Mide:** pronunciation por fonema, stress, connected speech.

#### Item: `pronunciation_set_us_general_v1`

**Target variant:** `us`.

**10 frases (cada una con phoneme target específico):**

```
1. "I think these things are difficult to remember."
   Phonemes target: /θ/, /ð/

2. "Very well, very busy, very brave."
   Phonemes target: /v/ vs /b/

3. "She measures the vision with great precision."
   Phoneme target: /ʒ/

4. "The ship and the sheep look quite similar from far away."
   Phoneme target: /ɪ/ vs /iː/

5. "I have a black cat and a small bag in my hand."
   Phoneme target: /æ/

6. "I walked home, talked to him, and worked late yesterday."
   Phoneme target: -ed final consonants

7. "Photographs about photography teach us about light."
   Phoneme target: schwa /ə/, word stress

8. "Are you ready? Yes, I am ready to go."
   Phoneme target: question intonation rising

9. "Could you tell me where you live and what you do?"
   Phoneme target: connected speech, linking

10. "She asked if he had finished the report by then."
    Phoneme target: past perfect, reported speech intonation
```

**Scoring rubric:**

```yaml
per_phrase_score: 0-100
aggregated_dimensions:
  pron_th_voiceless: avg(phrase 1)
  pron_th_voiced: avg(phrase 1)
  pron_v_vs_b: avg(phrase 2)
  pron_zh: avg(phrase 3)
  pron_short_long_i: avg(phrase 4)
  pron_ae: avg(phrase 5)
  pron_final_consonants: avg(phrase 6)
  pron_schwa: avg(phrase 7)
  pron_word_stress: avg(phrase 7)
  pron_intonation_questions: avg(phrase 8)
  pron_linking: avg(phrase 9)
overall_pronunciation: weighted_avg
```

#### Item: `pronunciation_set_uk_rp_v1`

Análogo pero con frases adaptadas para British RP.

#### Item: `pronunciation_set_us_general_v2`

Variante alternativa para re-evaluations (mismas sub-skills, frases
distintas).

### 4.4 Ejercicio 4: Listening con respuesta

**Estructura:**
1. Audio plays (90s, no replay).
2. 3 preguntas appear post-audio.
3. User responde (mix MC, open, inference).

**Sampling:** 1 audio de pool, matchando `goal_relevance`,
`cefr_level`, `target_variant`.

**Mide:** listening comprehension a velocidad nativa, detail capture,
inference.

#### Item: `listening_team_meeting_b2_v1`

**Goal relevance:** `job_interview`, `remote_work`,
`business_communication`.
**CEFR:** B2.
**Target variant:** `us`.

**Audio script (TTS via voice_us_f_40 - "Sarah Manager"):**

```
"Hi everyone, thanks for joining today's planning meeting. As you
know, we've been working on the new product launch for about three
months, and we're now in the final stretch.

Before we dive into the details, I want to remind everyone that the
deadline has been pushed up by two weeks because of competitor
activity. This means we need to prioritize ruthlessly. The marketing
team has been brilliant in preparing materials, and engineering is
tracking well on the technical side, but we need to coordinate the
launch announcement carefully.

Sarah, could you walk us through the timeline you proposed last week?
And Mike, I'd like you to highlight any blockers from your side.
Let's keep this focused so we can wrap up in 30 minutes."
```

**Duration:** ~75s a velocidad natural (~140 WPM).

**Questions:**

```yaml
q1:
  type: multiple_choice
  question: "Why has the deadline been pushed up?"
  options:
    a: "Customer requests"
    b: "Competitor activity"
    c: "Internal delays"
    d: "Marketing not ready"
  correct: b

q2:
  type: open_short
  question: "Who is asked to present the timeline?"
  expected_answer: "Sarah"
  acceptable_variations: ["Sarah", "the person named Sarah"]

q3:
  type: inference
  question: "Based on the speaker's tone, what's their attitude towards the team's progress?"
  expected_topics: ["positive but pressed for time", "proud", "worried about deadline"]
  scoring_rubric: |
    Full credit (100): captures both positive aspect AND time pressure.
    Partial (60): captures only one of the two.
    Low (30): misses the dual nature.
```

#### Item: `listening_airport_announcement_b1_v1`

Para users B1 con goal `travel`.

**Audio script:**

```
"Attention please. This is the final boarding call for American
Airlines flight 245 to Dallas-Fort Worth, departing from Gate B12.
All passengers should now be on board. If you are still in the
terminal, please proceed immediately to Gate B12 with your boarding
pass. The aircraft doors will close in five minutes. Please have
your passport and boarding pass ready for inspection. Thank you."
```

**Questions:** flight number, gate, time to closure, what to have
ready.

#### Item: `listening_friend_phone_b2_v1`

Para users con goal `daily_conversation`.

**Audio:** llamada informal entre amigos sobre planes de fin de semana.

### 4.5 Ejercicio 5: Producción libre

**Estructura:**
1. User ve prompt en pantalla.
2. 30s de prep time.
3. 2 min de speaking continuo.

**Sampling:** 1 prompt de pool, matchando `goal_relevance`.

**Mide:** fluencia sostenida, organización de ideas, vocabulario
amplio, capacidad de mantener turnos largos.

#### Item: `free_response_meeting_prep_b2_v1`

**Goal relevance:** `job_interview`, `remote_work`,
`business_communication`.
**CEFR:** B2.

**Prompt:**

```
🎤 Speaking task

Imagine you have an important meeting next month with potential
clients. They will be evaluating whether to work with your company
on a major project.

Talk for 2 minutes about how you would prepare for this meeting.
Mention specific actions, what information you would gather, and how
you would practice.

You have 30 seconds to think. The recording will start automatically.
```

**Scoring:**

```yaml
target_dimensions:
  fluency: 0.35 (sustained speech, target 110+ WPM at B2)
  vocabulary: 0.25 (business vocab, specific terms)
  grammar: 0.20 (conditionals, future forms)
  pronunciation: 0.15
  organization: 0.05 (logical flow)

target_subskills:
  - flu_extended_speech
  - flu_discourse_connectors
  - flu_thinking_in_english
  - vocab_business_general
  - gram_conditionals_zero_first
  - gram_future_forms
```

#### Item: `free_response_memorable_trip_b2_v1`

**Goal relevance:** `travel`, `daily_conversation`.

**Prompt:**

```
🎤 Speaking task

Tell me about a memorable trip you've taken or one you'd like to take.
Talk for 2 minutes about why it's significant to you, what you did
or would do there, and what you learned or expect to learn from it.

You have 30 seconds to think. The recording will start automatically.
```

#### Item: `free_response_describe_yourself_pro_b2_v1`

**Goal relevance:** `job_interview`.

**Prompt:**

```
🎤 Speaking task

Describe yourself professionally to a recruiter who's never met you.
Talk for 2 minutes about your strengths, experience, and what makes
you unique as a candidate.

This is similar to "Tell me about yourself" in interviews. You have
30 seconds to think.
```

#### Item: `free_response_change_career_b1_v1`

Para users B1 con goal genérico.

**Prompt:** "If you could change your career or area of study, what
would you choose and why?"

---

## 5. Parte 3: Test de habilidades receptivas

### 5.1 Vocabulary in context

**Estructura:**
1. 10 preguntas, 90s total.
2. Frase con palabra clave subrayada.
3. User elige sinónimo correcto (4 opciones).
4. **Adaptive:** si user responde bien, sube dificultad; si falla,
   baja.

**Sampling:** 10 questions de pool, distribuidas:
- 2 fáciles (CEFR -1)
- 6 al nivel del user (CEFR =)
- 2 difíciles (CEFR +1)

#### Items para B2 (10+ ejemplos)

```yaml
vocab_q_b2_001:
  sentence: "Despite the challenges, the team managed to **deliver** the project on time."
  options: ["refuse", "complete", "postpone", "ignore"]
  correct: "complete"

vocab_q_b2_002:
  sentence: "Her presentation was particularly **insightful**, offering a fresh perspective."
  options: ["brief", "confusing", "revealing", "standard"]
  correct: "revealing"

vocab_q_b2_003:
  sentence: "We need to **mitigate** the risks before launch."
  options: ["increase", "reduce", "ignore", "study"]
  correct: "reduce"

vocab_q_b2_004:
  sentence: "The proposal was **unanimously** approved by the board."
  options: ["by everyone", "reluctantly", "hastily", "by majority"]
  correct: "by everyone"

vocab_q_b2_005:
  sentence: "He's **reluctant** to share his opinion in meetings."
  options: ["eager", "unwilling", "trained", "prepared"]
  correct: "unwilling"

vocab_q_b2_006:
  sentence: "She tends to **delegate** tasks to junior team members."
  options: ["assign", "remove", "complete", "hide"]
  correct: "assign"

vocab_q_b2_007:
  sentence: "The merger had **far-reaching** consequences for the industry."
  options: ["minor", "extensive", "predictable", "temporary"]
  correct: "extensive"

vocab_q_b2_008:
  sentence: "We've been working on this **relentlessly** for weeks."
  options: ["occasionally", "without stopping", "carefully", "lazily"]
  correct: "without stopping"

vocab_q_b2_009:
  sentence: "His approach was **pragmatic** rather than idealistic."
  options: ["theoretical", "practical", "emotional", "dishonest"]
  correct: "practical"

vocab_q_b2_010:
  sentence: "The new policy was **rolled out** gradually."
  options: ["canceled", "introduced", "criticized", "translated"]
  correct: "introduced"
```

#### Items para B1 (más simples)

```yaml
vocab_q_b1_001:
  sentence: "She always **arrives** early to meetings."
  options: ["leaves", "comes", "calls", "writes"]
  correct: "comes"

vocab_q_b1_002:
  sentence: "Could you **explain** that again, please?"
  options: ["repeat the meaning", "leave", "sing", "translate"]
  correct: "repeat the meaning"
```

#### Items para C1 (más complejos)

```yaml
vocab_q_c1_001:
  sentence: "His arguments were **compelling**, even to skeptics."
  options: ["weak", "persuasive", "boring", "lengthy"]
  correct: "persuasive"

vocab_q_c1_002:
  sentence: "The CEO took an **unequivocal** stance on the issue."
  options: ["confused", "clear and firm", "quiet", "diplomatic"]
  correct: "clear and firm"
```

**Pool MVP:** 50 items por CEFR × 4 levels = 200 items mínimos.

### 5.2 Grammar recognition

**Estructura:**
1. 8 frases en orden.
2. User decide si cada una es **correct** o **wrong**.
3. Si wrong, tiene 10s opcional para indicar el error.

**Sampling:** 8 questions, distribuidas:
- 2 fáciles (CEFR -1)
- 4 al nivel del user
- 2 difíciles (CEFR +1)
- Mix 50/50 correct/wrong (no patrón obvio).

#### Items para B2

```yaml
grammar_q_b2_001:
  sentence: "By the time we arrived, they had already left."
  is_correct: true
  topic: "past perfect with by the time"

grammar_q_b2_002:
  sentence: "If I would have known, I would have called."
  is_correct: false
  error_explanation: "Should be 'If I had known, I would have called.' (third conditional)"
  topic: "third conditional"

grammar_q_b2_003:
  sentence: "She suggested that he goes home early."
  is_correct: false
  error_explanation: "Subjunctive: should be 'go' not 'goes'."
  topic: "subjunctive after suggest"

grammar_q_b2_004:
  sentence: "The book that I read it was excellent."
  is_correct: false
  error_explanation: "Extra 'it' — relative clause already has the object."
  topic: "relative clauses"

grammar_q_b2_005:
  sentence: "He's been working here since 2020."
  is_correct: true
  topic: "present perfect continuous with since"

grammar_q_b2_006:
  sentence: "I look forward to meet you."
  is_correct: false
  error_explanation: "Should be 'meeting' (look forward to + gerund)."
  topic: "verb patterns"

grammar_q_b2_007:
  sentence: "She's the woman whose son is a doctor."
  is_correct: true
  topic: "relative pronouns whose"

grammar_q_b2_008:
  sentence: "The information are very useful."
  is_correct: false
  error_explanation: "'Information' is uncountable, should be 'is'."
  topic: "uncountable nouns"
```

#### Items para B1

```yaml
grammar_q_b1_001:
  sentence: "I have lived in Mexico for 5 years."
  is_correct: true
  topic: "present perfect with for"

grammar_q_b1_002:
  sentence: "She don't like coffee."
  is_correct: false
  error_explanation: "Third person singular: 'doesn't' not 'don't'."
  topic: "subject-verb agreement"

grammar_q_b1_003:
  sentence: "Yesterday I go to the store."
  is_correct: false
  error_explanation: "Past tense: 'went' not 'go'."
  topic: "past simple"
```

**Pool MVP:** 40 items por CEFR × 4 levels = 160 items mínimos.

---

## 6. Parte 4: Aspirations questions

### 6.1 Q1: Frustración principal

```
¿Qué te frustra más cuando hablás inglés?
(Single select)

[ ] No encuentro las palabras que quiero usar
[ ] Me bloqueo con la gramática
[ ] No me entienden por mi pronunciación
[ ] No entiendo lo que me dicen
[ ] Mi acento me da inseguridad
[ ] Mezclo tiempos verbales
[ ] Otra: [input]
```

### 6.2 Q2: Visión a 3 meses

```
¿En qué situación te gustaría sentirte cómodo en 3 meses?

[Open text, max 200 chars]

Ejemplo: "Poder dar una presentación de 15 min sobre mi trabajo sin
preparar palabra por palabra."
```

### 6.3 Q3: Tipo de feedback preferido

```
¿Qué tipo de feedback preferís recibir?

[ ] Corrección inmediata (interrupción cuando hay error)
[ ] Corrección al final de la sesión (resumen)
[ ] Solo cuando el error es grave
[ ] Feedback suave y motivador
[ ] Feedback directo y honesto
[ ] Mix: directo en gramática, suave en pronunciación
```

### 6.4 Q4: Estilo de práctica

```
¿Sesiones cortas frecuentes o largas espaciadas?

[ ] 5-10 min diarios (consistencia)
[ ] 20-30 min cada 2-3 días
[ ] 60+ min los fines de semana
[ ] Variable según la semana
```

---

## 7. Variantes por goal (resumen)

| Goal del user | Reading passages priorizan | Roleplay scenarios | Listening priorizan | Free response priorizan |
|---------------|--------------------------|---------------------|---------------------|------------------------|
| `job_interview` | Work, careers, leadership | Behavioral interview | Team meetings, business calls | Meeting prep, self-intro pro |
| `remote_work` | Tech, distributed teams | Slack/email scenarios, project updates | Async vs sync communication | Project description |
| `travel` | Culture, geography, travel stories | Hotel, airport, restaurant | Airport announcements, tour guides | Memorable trip |
| `business_communication` | Business news, leadership | Client calls, negotiation | Conference calls, presentations | Pitch, proposal |
| `daily_conversation` | Lifestyle, friendships, hobbies | Friend catch-up, party invitation | Phone calls, casual chats | Hobby, personal story |
| `studies` | Academic topics, research | Office hours, study group | Lectures, podcasts | Research interest |
| `personal_growth` | Self-development, psychology | Therapist, coach | TED talks, podcasts | Goal description |

---

## 8. Scoring y validation

### 8.1 Scoring por ejercicio

(Detalle en `pedagogical-system.md` §3.)

Cada ejercicio del assessment llama al pipeline normal de scoring:
1. STT del audio (`transcribe_user_audio`).
2. Pronunciation scoring (`score_pronunciation`).
3. Grammar errors detection (`detect_grammar_errors`).
4. Fluency analysis (computed).
5. Vocabulary analysis (computed).

### 8.2 Score agregado del assessment

```python
def compute_assessment_score(exercise_results, parte_3_results):
    # Parte 2 (5 ejercicios) → 5 dimension scores agregados
    pronunciation = avg([r.pronunciation for r in parte_2 if r.pronunciation])
    fluency = avg([r.fluency for r in parte_2 if r.fluency])
    grammar = avg([r.grammar for r in parte_2 if r.grammar])
    vocabulary = avg([r.vocabulary for r in parte_2 if r.vocabulary])
    listening = parte_2[3].listening_score  # del ejercicio 4

    # Parte 3 (vocabulary + grammar) → bumps
    vocabulary_bump = compute_vocab_bump(parte_3.vocab_score)
    grammar_bump = compute_grammar_bump(parte_3.grammar_score)
    vocabulary += vocabulary_bump
    grammar += grammar_bump

    return {
        'pronunciation': clamp(pronunciation, 0, 100),
        'fluency': clamp(fluency, 0, 100),
        'grammar': clamp(grammar, 0, 100),
        'vocabulary': clamp(vocabulary, 0, 100),
        'listening': clamp(listening, 0, 100),
    }
```

### 8.3 Mapeo a CEFR

(Función `mapDimensionsToCEFR` en `pedagogical-system.md` §2.5.)

### 8.4 Detección de "broken assessment"

Flags si:
- Audio silencioso > 80% en cualquier ejercicio de Parte 2.
- Tiempo total < 5 min (vs 20 esperado).
- Vocabulary q3-Q10 todas = primera opción (patrón "all A").
- Grammar q1-Q8 todas = "correct" o todas = "wrong".

Si flag: prompt al user "El assessment fue muy rápido, ¿querés
rehacerlo?" (ver `student-profile-and-assessment.md` §6.6).

---

## 9. Mantenimiento del pool

### 9.1 Adición de items

Items nuevos pasan por:
1. Generación AI o creación humana.
2. Validation Zod del schema.
3. Validation pedagógica:
   - CEFR level matchea complexity del item.
   - No bias cultural / sesgo geográfico.
   - No errores de redacción.
4. Sample test con 5 users beta para verificar dificultad.
5. Approve.

### 9.2 Retirement

Items que:
- Tienen `use_count = 0` en 90 días: candidate to archive.
- Tienen rate de discrepancia con CEFR esperado > 30%: review (puede
  estar mal calibrado).
- User feedback negativo > 5%: review.

### 9.3 Anti-leak

- Items NO se exponen a través de API pública.
- Solo backend genera assessments.
- Si un item se filtra (ej: alguien lo postea online): retirar de pool y
  reemplazar.

### 9.4 Refresh para re-evaluations

User que ya hizo assessment trae lista `previousAssessmentItemIds`.
Sampling excluye esos items. Pool target ≥ 4x item types necesarios
para soportar 4 re-evaluations sin repetir.

---

## 10. Decisiones cerradas

### 10.1 Pool vs assessment fijo: **Pool con sampling** ✓

**Razón:** evita anti-cheating, permite re-evaluations sin repetir,
escalable.

### 10.2 Adaptive difficulty en Parte 3: **Sí** ✓

**Razón:** ajustar dificultad según response → estimación más precisa
de CEFR.

### 10.3 Scoring inmediato vs diferido: **Diferido (post-completion)** ✓

**Razón:** el user ve **resultados al final**, no después de cada
ejercicio. Reduce ansiedad y permite scoring holístico.

### 10.4 Items con audio compartido vs unique per user: **Audio
compartido (atomic reuse)** ✓

**Razón:** alineado con modelo atomic+composite. Misma audio puede
servir a 100 users diferentes en el assessment listening.

### 10.5 Scoring rubric exposed al user: **NO durante assessment, SÍ
post-completion** ✓

**Razón:** durante el assessment, mostrar rubric da pistas. Post,
explica por qué scores son lo que son.

---

## 11. Plan de implementación

### 11.1 Fase 1: MVP Pool inicial (mes 0–2)

- [ ] Schema `assessment_items` migrado.
- [ ] Sampling logic implementado.
- [ ] Pool inicial:
  - 8 reading passages (B1, B1+, B2, B2+ × goals).
  - 6 roleplay scenarios.
  - 2 pronunciation sets.
  - 8 listening exercises.
  - 6 free response prompts.
  - 50 vocabulary questions B1, 50 B2.
  - 40 grammar questions B1, 40 B2.
  - 8 profile questions (estáticas).
  - 4 aspirations questions (estáticas).
- [ ] Total ~220 items MVP.

### 11.2 Fase 2: Expansión (meses 2–6)

- Pool 4x para re-evaluations.
- A2 y C1 items para users en extremos.

### 11.3 Fase 3: Sofisticación (mes 6+)

- Adaptive item generation (LLM crea variantes en runtime de items
  base).
- Personalization deeper (items adaptados a profession específica).

---

## 12. Métricas

- **Cobertura del pool:** ratio items / item_types target.
- **Re-evaluation freshness:** % de items repetidos en re-eval (target
  < 10%).
- **CEFR calibration accuracy:** correlación entre dificultad declarada
  del item y respuesta promedio (target > 0.7).
- **User completion rate del assessment:** % que completan sin
  abandonar (target > 80%).
- **Time per exercise:** dentro del target (no flag de "muy rápido").

---

## 13. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §6 | Estructura del assessment (4 partes, 12 ejercicios). |
| [`pedagogical-system.md`](pedagogical-system.md) §3 | Métodos de scoring por dimensión. |
| [`pedagogical-system.md`](pedagogical-system.md) §2.5 | Mapping dimensiones → CEFR. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) §6 | Roadmap definitivo se genera con assessment_results. |
| [`content-creation-system.md`](content-creation-system.md) §3 | Modelo atomic+composite. Items audio se almacenan como atomics. |
| [`curriculum-by-cefr.md`](curriculum-by-cefr.md) | Plan de estudios al que el assessment apunta. |

---

*Documento vivo. Actualizar cuando se agreguen items nuevos al pool,
se descubran items mal calibrados, o se agreguen goals nuevos.*

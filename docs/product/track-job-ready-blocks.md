# Track Job Ready — Bloques B1 → B2+

> Desglose bloque-por-bloque del **Job Ready track** (track principal
> del MVP). Define los 74 bloques que componen el track desde B1 hasta
> B2+, con sub-skills target, prerequisites, duración estimada,
> composición de assets y criterios de mastery por bloque. Esta es la
> fuente de verdad para crear contenido del track más importante del
> producto.

**Estado:** Diseño v1.0 (listo para implementación)
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador (al crear contenido) +
humano (validación pedagógica).
**Alcance:** Job Ready track, niveles MVP (B1, B1+, B2, B2+).

---

## 0. Cómo leer este documento

- §1 establece **filosofía** del track Job Ready.
- §2 cubre **arquitectura** (módulos por CEFR).
- §3 muestra la **tabla resumen** de los 74 bloques.
- §4 detalla **B1: Foundation profesional** (22 bloques).
- §5 detalla **B1+: Consolidación profesional** (12 bloques).
- §6 detalla **B2: Job Ready completo** (28 bloques).
- §7 detalla **B2+: Senior / leadership transition** (12 bloques).
- §8 cubre **prerequisites graph** y orden recomendado.
- §9 cubre **mastery criteria por bloque y por nivel**.
- §10 cubre **assets compartidos cross-block** (reuse).
- §11 cubre **decisiones cerradas**.
- §12 cubre **plan de implementación**.

---

## 1. Filosofía del track Job Ready

### 1.1 Audiencia target

Profesional latinoamericano, B1+ self-perceived, buscando:
- **Job interview prep** (entrevistas en USA, BPO internacional, remote
  work).
- **Daily work in English** (reuniones, emails, Slack, status reports).
- **Career advancement** (mover de individual contributor a senior).

### 1.2 Diferenciador vs Daily Conversation track

| Dimensión | Daily Conversation | Job Ready |
|-----------|--------------------|-----------|
| Vocabulario | High-frequency genérico | Business + tech específico |
| Roleplays | Cotidianos (café, tienda) | Reuniones, entrevistas, emails |
| Register | Casual neutro | Formal y semi-formal profesional |
| Cultural | LatAm-USA general | USA business culture específica |
| Output target | Conversación social | Comunicación efectiva al trabajo |

### 1.3 Outcome al completar el track

User capaz de:
- Pasar entrevistas técnicas y behavioral en USA / remote.
- Liderar reuniones de status y proyectos.
- Escribir emails y mensajes de Slack profesionales.
- Negociar salario y scope con confianza.
- Dar y recibir feedback profesional.
- (B2+) Transitar a roles senior con presencia ejecutiva.

### 1.4 Lo que NO cubre este track

- **English specialized for medicine, law, finance**: requieren tracks
  especializados (post-MVP, ver `curriculum-by-cefr.md` §11.2).
- **TOEFL/IELTS exam prep**: tracks dedicados (post-MVP).
- **Academic writing**: track Academic (post-MVP).
- **Customer service scripts**: track Customer Service (post-MVP).

---

## 2. Arquitectura del track

### 2.1 Módulos por CEFR

```
┌──────────────────────────────────────────────────────────────┐
│  Job Ready Track (74 bloques MVP)                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  B1 (22) — Foundation profesional                            │
│    ├ Module 1: Self & background (5)                         │
│    ├ Module 2: Daily work routines (5)                       │
│    ├ Module 3: Communication basics (6)                      │
│    └ Module 4: Career conversations intro (6)                │
│                                                              │
│  B1+ (12) — Consolidación profesional                        │
│    ├ Module 5: Written communication (4)                     │
│    └ Module 6: Spoken communication consolidación (8)        │
│                                                              │
│  B2 (28) — Job Ready completo                                │
│    ├ Module 7: Interview mastery (10)                        │
│    ├ Module 8: Technical meetings (8)                        │
│    ├ Module 9: Professional written register (4)             │
│    └ Module 10: Workplace dynamics (6)                       │
│                                                              │
│  B2+ (12) — Senior / leadership transition                   │
│    ├ Module 11: Senior interviews (4)                        │
│    └ Module 12: Leadership communication (8)                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Nomenclatura de bloques

`jr_<cefr>_<short_slug>` donde:
- `jr` = Job Ready prefix.
- `<cefr>` = `b1`, `b1plus`, `b2`, `b2plus`.
- `<short_slug>` = snake_case descriptivo (max 30 chars).

Ejemplos:
- `jr_b1_intro_self_professional`
- `jr_b2_interview_star_method`
- `jr_b2plus_strategic_communication`

### 2.3 Tamaño estándar del bloque

- 4 assets por bloque (estándar; algunos 3 o 5).
- Duración estimada: 12-18 minutos.
- 1 listening + 1 fill_blank/vocab + 1 free_response + 1 roleplay
  (composición típica de §6.2 de `content-creation-system.md`).

---

## 3. Tabla resumen de los 74 bloques

### 3.1 Por CEFR y módulo

| # | Block ID | Título | CEFR | Min |
|--:|----------|--------|------|----:|
| **B1: Foundation profesional (22)** ||||
| 1 | `jr_b1_intro_self_professional` | Presentarte profesionalmente | B1 | 14 |
| 2 | `jr_b1_describe_role_company` | Describir tu rol y empresa | B1 | 13 |
| 3 | `jr_b1_education_background` | Hablar de tu formación | B1 | 12 |
| 4 | `jr_b1_work_experience_simple` | Experiencia laboral básica | B1 | 15 |
| 5 | `jr_b1_career_path_so_far` | Tu trayectoria hasta ahora | B1 | 16 |
| 6 | `jr_b1_daily_work_routine` | Rutina diaria de trabajo | B1 | 12 |
| 7 | `jr_b1_team_structure` | Estructura de tu equipo | B1 | 13 |
| 8 | `jr_b1_tools_you_use` | Herramientas que usas | B1 | 12 |
| 9 | `jr_b1_typical_workday_story` | Contar un día típico | B1 | 15 |
| 10 | `jr_b1_industry_vocab_intro` | Vocabulario de tu industria | B1 | 14 |
| 11 | `jr_b1_polite_requests_work` | Pedidos corteses en el trabajo | B1 | 13 |
| 12 | `jr_b1_asking_clarification` | Pedir aclaración | B1 | 12 |
| 13 | `jr_b1_status_update_short` | Status update breve | B1 | 14 |
| 14 | `jr_b1_disagreeing_politely` | Estar en desacuerdo cortésmente | B1 | 13 |
| 15 | `jr_b1_suggesting_ideas` | Proponer ideas | B1 | 13 |
| 16 | `jr_b1_following_instructions` | Seguir y dar instrucciones | B1 | 14 |
| 17 | `jr_b1_career_goals_simple` | Objetivos de carrera básicos | B1 | 14 |
| 18 | `jr_b1_strengths_intro` | Hablar de tus fortalezas | B1 | 13 |
| 19 | `jr_b1_what_you_learned` | Lo que aprendiste de un trabajo | B1 | 15 |
| 20 | `jr_b1_future_plans_work` | Planes laborales futuros | B1 | 14 |
| 21 | `jr_b1_work_small_talk` | Small talk en el trabajo | B1 | 13 |
| 22 | `jr_b1_short_self_pitch` | Pitch personal de 30s | B1 | 16 |
| **B1+: Consolidación profesional (12)** ||||
| 23 | `jr_b1plus_email_basics` | Emails profesionales básicos | B1+ | 15 |
| 24 | `jr_b1plus_email_request` | Email de pedido / solicitud | B1+ | 14 |
| 25 | `jr_b1plus_email_followup` | Email de seguimiento | B1+ | 14 |
| 26 | `jr_b1plus_slack_register` | Mensajes de Slack/Teams | B1+ | 13 |
| 27 | `jr_b1plus_meeting_participation` | Participar en reuniones | B1+ | 16 |
| 28 | `jr_b1plus_disagreeing_proposal` | Disagreement con propuestas | B1+ | 15 |
| 29 | `jr_b1plus_suggesting_alternative` | Sugerir alternativas | B1+ | 14 |
| 30 | `jr_b1plus_past_project_describe` | Describir proyecto pasado | B1+ | 16 |
| 31 | `jr_b1plus_reporting_others` | Reportar lo que dijo otro | B1+ | 14 |
| 32 | `jr_b1plus_expressing_concerns` | Expresar preocupaciones | B1+ | 14 |
| 33 | `jr_b1plus_giving_feedback_simple` | Dar feedback simple | B1+ | 15 |
| 34 | `jr_b1plus_recap_summarize` | Resumir y hacer recap | B1+ | 13 |
| **B2: Job Ready completo (28)** ||||
| 35 | `jr_b2_interview_tell_me_about` | "Tell me about yourself" extendido | B2 | 18 |
| 36 | `jr_b2_interview_star_method` | STAR method para behavioral | B2 | 18 |
| 37 | `jr_b2_interview_behavioral_q` | Preguntas behavioral comunes | B2 | 17 |
| 38 | `jr_b2_interview_strengths_weaknesses` | Strengths / weaknesses refinado | B2 | 16 |
| 39 | `jr_b2_interview_why_this_company` | Por qué esta empresa | B2 | 15 |
| 40 | `jr_b2_interview_difficult_situations` | Situaciones difíciles superadas | B2 | 17 |
| 41 | `jr_b2_interview_leadership_story` | Historia de leadership | B2 | 17 |
| 42 | `jr_b2_interview_technical_q_response` | Respuestas a preguntas técnicas | B2 | 18 |
| 43 | `jr_b2_interview_questions_to_ask` | Preguntas para hacer al final | B2 | 14 |
| 44 | `jr_b2_salary_negotiation_basic` | Negociación salarial básica | B2 | 17 |
| 45 | `jr_b2_meeting_present_status` | Presentar status en reunión | B2 | 16 |
| 46 | `jr_b2_meeting_explain_problem` | Explicar un problema técnico | B2 | 17 |
| 47 | `jr_b2_meeting_propose_solution` | Proponer solución | B2 | 17 |
| 48 | `jr_b2_meeting_diplomatic_pushback` | Pushback diplomático | B2 | 16 |
| 49 | `jr_b2_meeting_running_meeting` | Liderar una reunión | B2 | 18 |
| 50 | `jr_b2_meeting_qa_handling` | Manejar Q&A | B2 | 16 |
| 51 | `jr_b2_cross_team_communication` | Comunicación cross-team | B2 | 15 |
| 52 | `jr_b2_handling_unexpected` | Manejar imprevistos en reunión | B2 | 16 |
| 53 | `jr_b2_email_formal_register` | Email register formal | B2 | 14 |
| 54 | `jr_b2_email_difficult_topic` | Email sobre tema difícil | B2 | 16 |
| 55 | `jr_b2_written_proposal` | Propuesta escrita corta | B2 | 17 |
| 56 | `jr_b2_status_report_written` | Status report escrito | B2 | 14 |
| 57 | `jr_b2_conflict_resolution` | Resolución de conflicto | B2 | 17 |
| 58 | `jr_b2_performance_review_receiving` | Recibir performance review | B2 | 16 |
| 59 | `jr_b2_giving_critical_feedback` | Dar feedback crítico | B2 | 17 |
| 60 | `jr_b2_remote_work_communication` | Comunicación remota | B2 | 14 |
| 61 | `jr_b2_business_idioms_us` | Idioms de business USA | B2 | 13 |
| 62 | `jr_b2_directness_vs_diplomacy` | Directness vs diplomacia | B2 | 15 |
| **B2+: Senior / leadership transition (12)** ||||
| 63 | `jr_b2plus_senior_interview_vision` | Senior interview: visión | B2+ | 18 |
| 64 | `jr_b2plus_senior_interview_leadership` | Senior interview: leadership | B2+ | 18 |
| 65 | `jr_b2plus_senior_interview_strategy` | Senior interview: estrategia | B2+ | 18 |
| 66 | `jr_b2plus_senior_interview_failure` | Senior interview: fracaso aprendido | B2+ | 17 |
| 67 | `jr_b2plus_stakeholder_management` | Stakeholder management | B2+ | 17 |
| 68 | `jr_b2plus_influencing_without_authority` | Influir sin autoridad | B2+ | 17 |
| 69 | `jr_b2plus_hedging_diplomatic` | Hedging diplomático | B2+ | 15 |
| 70 | `jr_b2plus_persuasion_advanced` | Persuasión avanzada | B2+ | 17 |
| 71 | `jr_b2plus_difficult_conversations` | Conversaciones difíciles | B2+ | 18 |
| 72 | `jr_b2plus_giving_performance_review` | Dar performance review | B2+ | 17 |
| 73 | `jr_b2plus_mentoring_conversations` | Conversaciones de mentoring | B2+ | 16 |
| 74 | `jr_b2plus_executive_presence` | Presencia ejecutiva | B2+ | 18 |

**Totales:**
- B1: 22 bloques, ~310 minutos.
- B1+: 12 bloques, ~173 minutos.
- B2: 28 bloques, ~456 minutos.
- B2+: 12 bloques, ~206 minutos.
- **Total Job Ready MVP: 74 bloques, ~1.145 minutos** (~19 horas de
  contenido pedagógico).

---

## 4. B1: Foundation profesional (22 bloques)

### 4.1 Module 1: Self & background (5 bloques)

#### `jr_b1_intro_self_professional`

- **Título:** Presentarte profesionalmente
- **Descripción:** Introducción profesional estándar (60s) con nombre,
  rol, empresa, experiencia.
- **Sub-skills:** `vocab_business_general`, `flu_speaking_pace`,
  `pron_th_voiceless`, `gram_present_simple_continuous`.
- **Prerequisites:** ninguno (entry point del track B1).
- **Estimated min:** 14
- **Asset sequence:**
  1. `listening_mc_professional_intros` (3 min): escuchar 4 ejemplos
     de auto-presentación profesional, identificar elementos.
  2. `vocab_drill_intro_phrases` (2 min): drill de frases comunes
     ("I work as a...", "I've been in [field] for...").
  3. `free_response_intro_yourself` (3 min): grabar tu propia
     intro de 60 segundos.
  4. `roleplay_networking_event` (5 min): roleplay AI en evento de
     networking, presentarte y responder follow-ups.
- **Mastery criteria:**
  - Score promedio ≥ 70 en pronunciation y fluency.
  - Capacidad de auto-presentarte en < 60s sin pausas largas.
  - Uso correcto de present simple para rol actual y present perfect
    para experiencia.

#### `jr_b1_describe_role_company`

- **Título:** Describir tu rol y empresa
- **Descripción:** Explicar qué hace tu empresa y cuál es tu rol
  específico, sin vocabulario técnico complejo.
- **Sub-skills:** `vocab_business_general`,
  `flu_discourse_connectors`, `gram_articles`.
- **Prerequisites:** `jr_b1_intro_self_professional`.
- **Estimated min:** 13
- **Asset sequence:**
  1. `listening_mc_company_descriptions` (3 min): identificar tipos
     de empresa y roles.
  2. `fill_blank_role_descriptions` (2 min): completar frases con
     job titles y descripción.
  3. `free_response_describe_your_company` (3 min): describir tu
     empresa en 90s.
  4. `roleplay_curious_friend_about_work` (5 min): amigo pregunta
     sobre tu empresa.

#### `jr_b1_education_background`

- **Título:** Hablar de tu formación
- **Descripción:** Hablar de educación formal, certificaciones,
  cursos relevantes para el rol.
- **Sub-skills:** `gram_present_perfect`, `gram_past_simple_continuous`,
  `vocab_business_general`.
- **Prerequisites:** `jr_b1_intro_self_professional`.
- **Estimated min:** 12
- **Asset sequence:**
  1. `listening_mc_education_paths` (3 min).
  2. `fill_blank_education_phrases` (2 min): "I graduated from...",
     "I have a degree in...".
  3. `free_response_your_education` (3 min).
  4. `roleplay_recruiter_about_education` (4 min).

#### `jr_b1_work_experience_simple`

- **Título:** Experiencia laboral básica
- **Descripción:** Contar trabajos previos usando past simple y
  present perfect.
- **Sub-skills:** `gram_present_perfect`,
  `gram_past_simple_continuous`, `flu_pause_management`.
- **Prerequisites:** `jr_b1_describe_role_company`.
- **Estimated min:** 15
- **Asset sequence:**
  1. `listening_mc_work_experience_examples` (3 min).
  2. `gram_drill_past_vs_present_perfect` (3 min).
  3. `free_response_your_work_history` (4 min).
  4. `roleplay_short_job_chat` (5 min).

#### `jr_b1_career_path_so_far`

- **Título:** Tu trayectoria hasta ahora
- **Descripción:** Hilar tu carrera hasta el presente con conectores
  temporales (then, after, before that, currently).
- **Sub-skills:** `flu_discourse_connectors`, `gram_present_perfect`,
  `flu_extended_speech`.
- **Prerequisites:** `jr_b1_work_experience_simple`,
  `jr_b1_education_background`.
- **Estimated min:** 16
- **Asset sequence:**
  1. `listening_mc_career_stories` (4 min).
  2. `vocab_drill_temporal_connectors` (2 min).
  3. `free_response_career_journey` (4 min): contar tu trayectoria
     en 2 minutos.
  4. `roleplay_networking_career_chat` (6 min).

### 4.2 Module 2: Daily work routines (5 bloques)

#### `jr_b1_daily_work_routine`

- **Título:** Rutina diaria de trabajo
- **Descripción:** Describir actividades diarias usando present
  simple, adverbios de frecuencia y vocabulario de oficina.
- **Sub-skills:** `gram_present_simple_continuous`,
  `vocab_business_general`, `pron_word_stress`.
- **Prerequisites:** `jr_b1_describe_role_company`.
- **Estimated min:** 12
- **Asset sequence:** listening MC + vocab drill + free response +
  roleplay (estructura estándar).

#### `jr_b1_team_structure`

- **Título:** Estructura de tu equipo
- **Descripción:** Describir tu equipo (team lead, peers, reports),
  reporting lines, y cómo encaja en la org.
- **Sub-skills:** `vocab_business_general`, `gram_articles`,
  `flu_pause_management`.
- **Prerequisites:** `jr_b1_describe_role_company`.
- **Estimated min:** 13

#### `jr_b1_tools_you_use`

- **Título:** Herramientas que usas
- **Descripción:** Vocabulario de software/herramientas comunes
  (Slack, Jira, Google Docs, etc.).
- **Sub-skills:** `vocab_business_general`, `vocab_tech_industry`
  (intro), `pron_final_consonants`.
- **Prerequisites:** `jr_b1_daily_work_routine`.
- **Estimated min:** 12

#### `jr_b1_typical_workday_story`

- **Título:** Contar un día típico
- **Descripción:** Narrar un día completo desde mañana hasta tarde,
  con conectores temporales.
- **Sub-skills:** `flu_discourse_connectors`, `flu_extended_speech`,
  `gram_present_simple_continuous`.
- **Prerequisites:** `jr_b1_daily_work_routine`,
  `jr_b1_tools_you_use`.
- **Estimated min:** 15

#### `jr_b1_industry_vocab_intro`

- **Título:** Vocabulario de tu industria
- **Descripción:** Vocabulario base de tu industria (selecciona 1 de:
  tech, finance, healthcare, education, sales, marketing).
- **Sub-skills:** `vocab_business_general`, `vocab_tech_industry`
  (intro), `pron_word_stress`.
- **Prerequisites:** `jr_b1_describe_role_company`.
- **Estimated min:** 14
- **Variantes:** este bloque tiene **5 variantes** (`_tech`,
  `_finance`, `_healthcare`, `_education`, `_sales_marketing`). Roadmap
  asigna basado en `student_profile.professional_field`.

### 4.3 Module 3: Communication basics (6 bloques)

#### `jr_b1_polite_requests_work`

- **Título:** Pedidos corteses en el trabajo
- **Descripción:** "Could you...", "Would you mind...", "Sorry to
  bother you, but..." en contexto laboral.
- **Sub-skills:** `gram_phrasal_verbs_basic`,
  `pron_intonation_questions`, `flu_turn_taking`.
- **Prerequisites:** `jr_b1_daily_work_routine`.
- **Estimated min:** 13
- **Asset sequence:**
  1. `listening_mc_polite_requests` (3 min).
  2. `vocab_drill_polite_phrases` (2 min).
  3. `free_response_request_practice` (3 min): grabar 5 pedidos en
     contexto distinto.
  4. `roleplay_office_requests` (5 min): pedir cosas a colega AI.

#### `jr_b1_asking_clarification`

- **Título:** Pedir aclaración
- **Descripción:** "Could you repeat that?", "What do you mean by...",
  "Sorry, I missed that".
- **Sub-skills:** `flu_turn_taking`, `pron_intonation_questions`,
  `list_detail_capture`.
- **Prerequisites:** `jr_b1_polite_requests_work`.
- **Estimated min:** 12

#### `jr_b1_status_update_short`

- **Título:** Status update breve
- **Descripción:** "Last week I... This week I'm... Next week I'll..."
  patrón estándar de status update.
- **Sub-skills:** `gram_future_forms`, `gram_present_perfect`,
  `flu_discourse_connectors`.
- **Prerequisites:** `jr_b1_describe_role_company`.
- **Estimated min:** 14

#### `jr_b1_disagreeing_politely`

- **Título:** Estar en desacuerdo cortésmente
- **Descripción:** "I see your point, but...", "I'm not sure I agree
  because...", evitar "No, you're wrong".
- **Sub-skills:** `flu_discourse_connectors`,
  `pron_intonation_questions`, `vocab_emotion_register` (intro).
- **Prerequisites:** `jr_b1_polite_requests_work`.
- **Estimated min:** 13

#### `jr_b1_suggesting_ideas`

- **Título:** Proponer ideas
- **Descripción:** "What if we...", "Maybe we could...", "Have you
  considered...".
- **Sub-skills:** `gram_conditionals_zero_first`,
  `flu_turn_taking`, `flu_discourse_connectors`.
- **Prerequisites:** `jr_b1_disagreeing_politely`.
- **Estimated min:** 13

#### `jr_b1_following_instructions`

- **Título:** Seguir y dar instrucciones
- **Descripción:** Pedir / dar instrucciones paso a paso (first,
  then, after that, finally).
- **Sub-skills:** `flu_discourse_connectors`,
  `gram_phrasal_verbs_basic`, `list_detail_capture`.
- **Prerequisites:** `jr_b1_asking_clarification`.
- **Estimated min:** 14

### 4.4 Module 4: Career conversations intro (6 bloques)

#### `jr_b1_career_goals_simple`

- **Título:** Objetivos de carrera básicos
- **Descripción:** "I'd like to...", "My goal is to...", "In the next
  X years I want to...".
- **Sub-skills:** `gram_future_forms`, `flu_extended_speech`,
  `vocab_business_general`.
- **Prerequisites:** `jr_b1_career_path_so_far`.
- **Estimated min:** 14

#### `jr_b1_strengths_intro`

- **Título:** Hablar de tus fortalezas
- **Descripción:** Identificar y articular 3 fortalezas profesionales
  con ejemplos.
- **Sub-skills:** `vocab_business_general`, `flu_extended_speech`,
  `gram_present_perfect`.
- **Prerequisites:** `jr_b1_career_path_so_far`.
- **Estimated min:** 13

#### `jr_b1_what_you_learned`

- **Título:** Lo que aprendiste de un trabajo
- **Descripción:** Articular lecciones aprendidas en past simple +
  present perfect.
- **Sub-skills:** `gram_present_perfect`,
  `gram_past_simple_continuous`, `flu_extended_speech`.
- **Prerequisites:** `jr_b1_work_experience_simple`,
  `jr_b1_strengths_intro`.
- **Estimated min:** 15

#### `jr_b1_future_plans_work`

- **Título:** Planes laborales futuros
- **Descripción:** "I'm planning to...", "I'll probably...", "I'm
  hoping to..." con will/going to/present continuous.
- **Sub-skills:** `gram_future_forms`,
  `gram_conditionals_zero_first`, `flu_extended_speech`.
- **Prerequisites:** `jr_b1_career_goals_simple`.
- **Estimated min:** 14

#### `jr_b1_work_small_talk`

- **Título:** Small talk en el trabajo
- **Descripción:** Weekend chat, "How was your weekend?", weather,
  sports, sin ir muy personal.
- **Sub-skills:** `flu_turn_taking`, `flu_speaking_pace`,
  `pron_intonation_questions`.
- **Prerequisites:** `jr_b1_polite_requests_work`.
- **Estimated min:** 13

#### `jr_b1_short_self_pitch`

- **Título:** Pitch personal de 30s
- **Descripción:** Capstone del módulo B1: armar elevator pitch
  profesional de 30 segundos integrando todo lo aprendido.
- **Sub-skills:** consolida todas las sub-skills B1 cubiertas.
- **Prerequisites:** `jr_b1_intro_self_professional`,
  `jr_b1_strengths_intro`, `jr_b1_career_goals_simple`,
  `jr_b1_describe_role_company`.
- **Estimated min:** 16
- **Asset sequence:**
  1. `listening_mc_elevator_pitches` (4 min): escuchar 6 ejemplos.
  2. `vocab_drill_pitch_phrases` (2 min).
  3. `free_response_your_pitch_v1` (4 min): primera grabación.
  4. `free_response_your_pitch_v2` (3 min): segunda grabación con
     feedback de la primera.
  5. `roleplay_pitch_to_recruiter` (3 min): roleplay con AI recruiter.
- **Mastery criteria del bloque:**
  - Pitch en 30s ± 5s.
  - Score promedio ≥ 75 (mayor exigencia, capstone block).
  - Estructura clara: rol → experiencia → strength → goal.

### 4.5 Mastery criteria del nivel B1 (Job Ready)

Para considerar B1 Job Ready completado:
- ≥ 80% de los 22 bloques completados.
- Score promedio ≥ 70 en sub-skills B1 cubiertas.
- WPM 80-100 en producción libre profesional.
- < 6 fillers/min en speech profesional.
- Capaz de mantener conversación profesional de 5 min sin breakdown.

---

## 5. B1+: Consolidación profesional (12 bloques)

### 5.1 Module 5: Written communication (4 bloques)

#### `jr_b1plus_email_basics`

- **Título:** Emails profesionales básicos
- **Descripción:** Estructura email profesional: subject, greeting,
  body, sign-off. Casual register.
- **Sub-skills:** `vocab_business_general`,
  `flu_discourse_connectors`, `gram_articles`.
- **Prerequisites:** `jr_b1_short_self_pitch` (B1 capstone).
- **Estimated min:** 15
- **Asset sequence:**
  1. `listening_mc_email_examples` (3 min): identificar emails
     bien/mal estructurados.
  2. `vocab_drill_email_phrases` (3 min): "I'm writing to...", "Could
     you let me know...", "Best regards".
  3. `written_response_compose_email` (4 min): redactar email simple
     (text input + scoring).
  4. `roleplay_dictate_email` (5 min): dictar email a AI assistant.

#### `jr_b1plus_email_request`

- **Título:** Email de pedido / solicitud
- **Descripción:** Pedir información, recursos, tiempo, en email
  cortés.
- **Sub-skills:** `vocab_business_general`,
  `gram_conditionals_zero_first`, `flu_discourse_connectors`.
- **Prerequisites:** `jr_b1plus_email_basics`.
- **Estimated min:** 14

#### `jr_b1plus_email_followup`

- **Título:** Email de seguimiento
- **Descripción:** "Just following up on...", "Bumping this up...",
  "Per my last email...".
- **Sub-skills:** `vocab_business_general`,
  `gram_present_perfect`, `flu_discourse_connectors`.
- **Prerequisites:** `jr_b1plus_email_basics`.
- **Estimated min:** 14

#### `jr_b1plus_slack_register`

- **Título:** Mensajes de Slack/Teams
- **Descripción:** Register casual, abreviaciones comunes (FYI, EOD,
  ASAP), uso de emojis, threads.
- **Sub-skills:** `vocab_business_general`,
  `vocab_idioms_common` (intro), `gram_phrasal_verbs_basic`.
- **Prerequisites:** `jr_b1plus_email_basics`.
- **Estimated min:** 13

### 5.2 Module 6: Spoken communication consolidación (8 bloques)

#### `jr_b1plus_meeting_participation`

- **Título:** Participar en reuniones
- **Descripción:** Tomar y ceder turno, sumarse a discusión, hacer
  preguntas en reunión.
- **Sub-skills:** `flu_turn_taking`, `flu_extended_speech`,
  `flu_discourse_connectors`.
- **Prerequisites:** `jr_b1_disagreeing_politely`,
  `jr_b1_suggesting_ideas`.
- **Estimated min:** 16

#### `jr_b1plus_disagreeing_proposal`

- **Título:** Disagreement con propuestas
- **Descripción:** Disagree con propuestas concretas usando "I see
  the merit, however...", "My concern is...".
- **Sub-skills:** `flu_discourse_connectors`,
  `vocab_emotion_register`, `flu_extended_speech`.
- **Prerequisites:** `jr_b1_disagreeing_politely`,
  `jr_b1plus_meeting_participation`.
- **Estimated min:** 15

#### `jr_b1plus_suggesting_alternative`

- **Título:** Sugerir alternativas
- **Descripción:** "What if we tried...", "Another option could
  be...", "I'd suggest...".
- **Sub-skills:** `gram_conditionals_zero_first`,
  `flu_discourse_connectors`, `flu_extended_speech`.
- **Prerequisites:** `jr_b1_suggesting_ideas`,
  `jr_b1plus_disagreeing_proposal`.
- **Estimated min:** 14

#### `jr_b1plus_past_project_describe`

- **Título:** Describir proyecto pasado
- **Descripción:** Storytelling estructurado: contexto, rol, acción,
  resultado (proto-STAR).
- **Sub-skills:** `gram_past_simple_continuous`,
  `gram_present_perfect`, `flu_extended_speech`.
- **Prerequisites:** `jr_b1_what_you_learned`.
- **Estimated min:** 16

#### `jr_b1plus_reporting_others`

- **Título:** Reportar lo que dijo otro
- **Descripción:** "She said that...", "He mentioned...", "They
  asked if..." con shifts de tense.
- **Sub-skills:** `gram_reported_speech`,
  `gram_past_simple_continuous`, `flu_extended_speech`.
- **Prerequisites:** `jr_b1plus_meeting_participation`.
- **Estimated min:** 14

#### `jr_b1plus_expressing_concerns`

- **Título:** Expresar preocupaciones
- **Descripción:** "I'm worried that...", "My concern is...", "Have
  we considered the risk of...".
- **Sub-skills:** `flu_discourse_connectors`,
  `vocab_emotion_register`, `gram_conditionals_zero_first`.
- **Prerequisites:** `jr_b1plus_disagreeing_proposal`.
- **Estimated min:** 14

#### `jr_b1plus_giving_feedback_simple`

- **Título:** Dar feedback simple
- **Descripción:** Sandwich method básico, evitar "you're wrong",
  feedback constructivo en español → inglés.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_discourse_connectors`, `flu_extended_speech`.
- **Prerequisites:** `jr_b1plus_expressing_concerns`.
- **Estimated min:** 15

#### `jr_b1plus_recap_summarize`

- **Título:** Resumir y hacer recap
- **Descripción:** "To recap...", "So the key points are...", "Just
  to summarize...".
- **Sub-skills:** `flu_discourse_connectors`,
  `flu_extended_speech`, `flu_pause_management`.
- **Prerequisites:** `jr_b1plus_meeting_participation`.
- **Estimated min:** 13

### 5.3 Mastery criteria del nivel B1+ (Job Ready)

- ≥ 80% de los 12 bloques B1+ completados.
- Score promedio ≥ 75 en sub-skills B1 cubiertas.
- WPM 100-120 en producción libre profesional.
- < 5 fillers/min.
- Capaz de mantener reunión simulada de 10 min con AI sin breakdown.

---

## 6. B2: Job Ready completo (28 bloques)

### 6.1 Module 7: Interview mastery (10 bloques)

Este es el corazón del track. 10 bloques específicos para entrevistas
en USA / remote.

#### `jr_b2_interview_tell_me_about`

- **Título:** "Tell me about yourself" extendido
- **Descripción:** Versión 2-min de la pregunta inicial estándar.
  Estructura: present-past-future con relevant evidence.
- **Sub-skills:** `flu_extended_speech`, `gram_present_perfect`,
  `flu_thinking_in_english`.
- **Prerequisites:** `jr_b1_short_self_pitch`,
  `jr_b1plus_past_project_describe`.
- **Estimated min:** 18
- **Asset sequence:**
  1. `listening_mc_tell_me_about_yourself_examples` (4 min): 5
     ejemplos buenos vs malos, con análisis.
  2. `vocab_drill_interview_signposting` (2 min): "I started my
     career in...", "More recently, I've been...".
  3. `free_response_tell_me_v1` (4 min): primera versión 2 min.
  4. `free_response_tell_me_v2` (4 min): refinar.
  5. `roleplay_full_interview_opener` (4 min): roleplay con AI
     interviewer.

#### `jr_b2_interview_star_method`

- **Título:** STAR method para behavioral
- **Descripción:** Framework Situation-Task-Action-Result para
  preguntas behavioral.
- **Sub-skills:** `flu_extended_speech`, `flu_discourse_connectors`,
  `gram_past_perfect`.
- **Prerequisites:** `jr_b1plus_past_project_describe`.
- **Estimated min:** 18
- **Asset sequence:**
  1. `listening_mc_star_examples` (4 min).
  2. `fill_blank_star_structure` (3 min): identificar S-T-A-R en
     respuestas.
  3. `free_response_star_practice_3_topics` (5 min): 3 mini-stories
     en STAR.
  4. `roleplay_behavioral_interview_star` (6 min): AI hace 3
     behavioral, user responde con STAR.

#### `jr_b2_interview_behavioral_q`

- **Título:** Preguntas behavioral comunes
- **Descripción:** Banco de las 10 preguntas behavioral más comunes
  en USA tech / business interviews.
- **Sub-skills:** `flu_extended_speech`,
  `gram_conditionals_second_third`, `flu_thinking_in_english`.
- **Prerequisites:** `jr_b2_interview_star_method`.
- **Estimated min:** 17
- **Banco de preguntas cubierto:** "Tell me about a time you...":
  failed, disagreed with manager, led without authority, made a
  mistake, dealt with conflict, missed a deadline, gave critical
  feedback, learned from a peer, handled ambiguity, prioritized.

#### `jr_b2_interview_strengths_weaknesses`

- **Título:** Strengths / weaknesses refinado
- **Descripción:** Versión avanzada con evidence específica y
  weakness genuina + plan de mejora.
- **Sub-skills:** `flu_extended_speech`, `vocab_emotion_register`,
  `gram_present_perfect`.
- **Prerequisites:** `jr_b1_strengths_intro`,
  `jr_b2_interview_star_method`.
- **Estimated min:** 16

#### `jr_b2_interview_why_this_company`

- **Título:** Por qué esta empresa
- **Descripción:** Framework de research → product/values fit →
  growth opportunity.
- **Sub-skills:** `flu_extended_speech`,
  `vocab_business_general`, `flu_discourse_connectors`.
- **Prerequisites:** `jr_b2_interview_tell_me_about`.
- **Estimated min:** 15

#### `jr_b2_interview_difficult_situations`

- **Título:** Situaciones difíciles superadas
- **Descripción:** Conflict at work, missed deadline, project
  failure, contadas en STAR.
- **Sub-skills:** `flu_extended_speech`, `gram_past_perfect`,
  `vocab_emotion_register`.
- **Prerequisites:** `jr_b2_interview_star_method`.
- **Estimated min:** 17

#### `jr_b2_interview_leadership_story`

- **Título:** Historia de leadership
- **Descripción:** Una historia central de leadership (formal o
  informal), bien practicada para usar en interviews.
- **Sub-skills:** `flu_extended_speech`,
  `gram_conditionals_second_third`, `vocab_business_general`.
- **Prerequisites:** `jr_b2_interview_difficult_situations`.
- **Estimated min:** 17

#### `jr_b2_interview_technical_q_response`

- **Título:** Respuestas a preguntas técnicas
- **Descripción:** Estructura para responder preguntas técnicas
  (define → context → solve → verify).
- **Sub-skills:** `flu_thinking_in_english`,
  `vocab_tech_industry`, `flu_extended_speech`.
- **Prerequisites:** `jr_b2_interview_star_method`.
- **Estimated min:** 18
- **Variantes:** este bloque tiene 2 variantes (`_engineering`,
  `_non_engineering`). Roadmap asigna basado en
  `student_profile.professional_field`.

#### `jr_b2_interview_questions_to_ask`

- **Título:** Preguntas para hacer al final
- **Descripción:** Banco de 10 preguntas inteligentes para hacer al
  interviewer al final.
- **Sub-skills:** `pron_intonation_questions`,
  `flu_turn_taking`, `vocab_business_general`.
- **Prerequisites:** `jr_b2_interview_why_this_company`.
- **Estimated min:** 14

#### `jr_b2_salary_negotiation_basic`

- **Título:** Negociación salarial básica
- **Descripción:** Language para "What's your expected salary?",
  contraofertar, negotiate range.
- **Sub-skills:** `flu_extended_speech`,
  `gram_conditionals_second_third`,
  `vocab_emotion_register`.
- **Prerequisites:** `jr_b2_interview_tell_me_about`.
- **Estimated min:** 17

### 6.2 Module 8: Technical meetings (8 bloques)

#### `jr_b2_meeting_present_status`

- **Título:** Presentar status en reunión
- **Descripción:** Status update estructurado en reunión: progress,
  blockers, next steps.
- **Sub-skills:** `flu_extended_speech`,
  `flu_discourse_connectors`, `vocab_tech_industry`.
- **Prerequisites:** `jr_b1_status_update_short`,
  `jr_b1plus_meeting_participation`.
- **Estimated min:** 16

#### `jr_b2_meeting_explain_problem`

- **Título:** Explicar un problema técnico
- **Descripción:** Articular un bug / issue claramente: síntoma →
  impacto → causa supuesta → ayuda necesaria.
- **Sub-skills:** `flu_extended_speech`, `vocab_tech_industry`,
  `gram_past_perfect`.
- **Prerequisites:** `jr_b2_meeting_present_status`.
- **Estimated min:** 17

#### `jr_b2_meeting_propose_solution`

- **Título:** Proponer solución
- **Descripción:** Estructura "I'd recommend X because Y. The
  trade-off is Z, but...".
- **Sub-skills:** `flu_extended_speech`,
  `flu_discourse_connectors`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `jr_b2_meeting_explain_problem`,
  `jr_b1plus_suggesting_alternative`.
- **Estimated min:** 17

#### `jr_b2_meeting_diplomatic_pushback`

- **Título:** Pushback diplomático
- **Descripción:** Disagree con seniority/cliente sin parecer
  agresivo. "I hear you, and at the same time...".
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `jr_b1plus_disagreeing_proposal`.
- **Estimated min:** 16

#### `jr_b2_meeting_running_meeting`

- **Título:** Liderar una reunión
- **Descripción:** Abrir, dirigir, cerrar reunión. Manejar agenda,
  cortar tangentes con cortesía.
- **Sub-skills:** `flu_turn_taking`, `flu_extended_speech`,
  `flu_discourse_connectors`.
- **Prerequisites:** `jr_b2_meeting_present_status`,
  `jr_b1plus_recap_summarize`.
- **Estimated min:** 18

#### `jr_b2_meeting_qa_handling`

- **Título:** Manejar Q&A
- **Descripción:** Listening for the real question, dar respuesta
  estructurada, "I'm not sure, let me get back to you", "Great
  question".
- **Sub-skills:** `list_native_speed_general`, `list_inference`,
  `flu_thinking_in_english`.
- **Prerequisites:** `jr_b2_meeting_running_meeting`.
- **Estimated min:** 16

#### `jr_b2_cross_team_communication`

- **Título:** Comunicación cross-team
- **Descripción:** Hablar con teams que no conocen tu contexto:
  abstraer technical detail, dar background.
- **Sub-skills:** `flu_extended_speech`,
  `flu_discourse_connectors`, `vocab_business_general`.
- **Prerequisites:** `jr_b2_meeting_explain_problem`.
- **Estimated min:** 15

#### `jr_b2_handling_unexpected`

- **Título:** Manejar imprevistos en reunión
- **Descripción:** "Sorry, I missed that, could you...", recuperarse
  de tangentes, retomar agenda.
- **Sub-skills:** `flu_self_correction_control`,
  `flu_thinking_in_english`, `flu_turn_taking`.
- **Prerequisites:** `jr_b2_meeting_qa_handling`.
- **Estimated min:** 16

### 6.3 Module 9: Professional written register (4 bloques)

#### `jr_b2_email_formal_register`

- **Título:** Email register formal
- **Descripción:** Step up de B1+: tono formal para clientes,
  ejecutivos, externos. "I'd like to bring to your attention...",
  "Per our conversation...".
- **Sub-skills:** `vocab_business_general`,
  `vocab_emotion_register`, `flu_discourse_connectors`.
- **Prerequisites:** `jr_b1plus_email_basics`,
  `jr_b1plus_email_request`.
- **Estimated min:** 14

#### `jr_b2_email_difficult_topic`

- **Título:** Email sobre tema difícil
- **Descripción:** Bad news email, decline request, escalation.
  Hedging language, empathy.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_discourse_connectors`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `jr_b2_email_formal_register`.
- **Estimated min:** 16

#### `jr_b2_written_proposal`

- **Título:** Propuesta escrita corta
- **Descripción:** Estructura proposal de 1 página: context →
  problem → proposal → ask.
- **Sub-skills:** `flu_discourse_connectors`,
  `vocab_business_general`, `gram_relative_clauses`.
- **Prerequisites:** `jr_b2_email_formal_register`,
  `jr_b2_meeting_propose_solution`.
- **Estimated min:** 17

#### `jr_b2_status_report_written`

- **Título:** Status report escrito
- **Descripción:** Weekly/biweekly status report estructurado para
  manager.
- **Sub-skills:** `flu_discourse_connectors`,
  `gram_present_perfect`, `vocab_business_general`.
- **Prerequisites:** `jr_b1plus_email_followup`.
- **Estimated min:** 14

### 6.4 Module 10: Workplace dynamics (6 bloques)

#### `jr_b2_conflict_resolution`

- **Título:** Resolución de conflicto
- **Descripción:** Conflict con peer / manager. "Can we step back
  and align on...", "I want to understand your perspective".
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `jr_b2_meeting_diplomatic_pushback`.
- **Estimated min:** 17

#### `jr_b2_performance_review_receiving`

- **Título:** Recibir performance review
- **Descripción:** Recibir critical feedback con grace, hacer
  preguntas para entender, no defender.
- **Sub-skills:** `vocab_emotion_register`, `list_inference`,
  `flu_thinking_in_english`.
- **Prerequisites:** `jr_b1plus_giving_feedback_simple`.
- **Estimated min:** 16

#### `jr_b2_giving_critical_feedback`

- **Título:** Dar feedback crítico
- **Descripción:** Sandwich avanzado, SBI model (Situation-
  Behavior-Impact), evitar generalizaciones.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `flu_discourse_connectors`.
- **Prerequisites:** `jr_b1plus_giving_feedback_simple`.
- **Estimated min:** 17

#### `jr_b2_remote_work_communication`

- **Título:** Comunicación remota
- **Descripción:** Async-first, over-communication, async vs sync
  decisions, time zone awareness.
- **Sub-skills:** `vocab_business_general`,
  `flu_discourse_connectors`, `vocab_idioms_common`.
- **Prerequisites:** `jr_b1plus_slack_register`,
  `jr_b2_meeting_present_status`.
- **Estimated min:** 14

#### `jr_b2_business_idioms_us`

- **Título:** Idioms de business USA
- **Descripción:** "Circle back", "ping me", "EOD", "low-hanging
  fruit", "double down", "pivot", "pushback", "ramp up", etc.
- **Sub-skills:** `vocab_idioms_common`,
  `vocab_business_general`, `pron_word_stress`.
- **Prerequisites:** `jr_b1plus_slack_register`.
- **Estimated min:** 13

#### `jr_b2_directness_vs_diplomacy`

- **Título:** Directness vs diplomacia
- **Descripción:** Cultural awareness USA: directness es valor
  positivo; latam tendency a circumlocution puede leer como evasión.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `flu_thinking_in_english`.
- **Prerequisites:** `jr_b2_giving_critical_feedback`.
- **Estimated min:** 15

### 6.5 Mastery criteria del nivel B2 (Job Ready)

- ≥ 80% de los 28 bloques B2 completados.
- Score promedio ≥ 75 en sub-skills B2 cubiertas.
- WPM 110-130.
- < 4 fillers/min.
- Capaz de mantener interview simulada de 30 min con AI sin
  breakdown.
- Capaz de liderar reunión simulada de 15 min.

---

## 7. B2+: Senior / leadership transition (12 bloques)

### 7.1 Module 11: Senior interviews (4 bloques)

#### `jr_b2plus_senior_interview_vision`

- **Título:** Senior interview: visión
- **Descripción:** "Where do you see this team / product / industry
  in 3 years?". Strategic thinking articulado.
- **Sub-skills:** `flu_extended_speech`,
  `gram_conditionals_second_third`,
  `vocab_business_general`.
- **Prerequisites:** `jr_b2_interview_tell_me_about`.
- **Estimated min:** 18

#### `jr_b2plus_senior_interview_leadership`

- **Título:** Senior interview: leadership
- **Descripción:** Leadership stories con scope amplio: org-level,
  multi-team, ambiguity.
- **Sub-skills:** `flu_extended_speech`,
  `gram_past_perfect`,
  `flu_thinking_in_english`.
- **Prerequisites:** `jr_b2_interview_leadership_story`.
- **Estimated min:** 18

#### `jr_b2plus_senior_interview_strategy`

- **Título:** Senior interview: estrategia
- **Descripción:** "How would you approach building X / scaling Y?".
  Framework + trade-offs + iteration.
- **Sub-skills:** `flu_extended_speech`,
  `gram_conditionals_second_third`,
  `vocab_business_general`.
- **Prerequisites:** `jr_b2plus_senior_interview_vision`.
- **Estimated min:** 18

#### `jr_b2plus_senior_interview_failure`

- **Título:** Senior interview: fracaso aprendido
- **Descripción:** Failure story de senior level: ownership,
  systemic learning, organizational change derivada.
- **Sub-skills:** `flu_extended_speech`,
  `gram_past_perfect`,
  `vocab_emotion_register`.
- **Prerequisites:** `jr_b2_interview_difficult_situations`.
- **Estimated min:** 17

### 7.2 Module 12: Leadership communication (8 bloques)

#### `jr_b2plus_stakeholder_management`

- **Título:** Stakeholder management
- **Descripción:** Hablar con execs, manager, peers, reports en
  diferente register. Sensemaking.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`, `flu_discourse_connectors`.
- **Prerequisites:** `jr_b2_cross_team_communication`,
  `jr_b2_meeting_running_meeting`.
- **Estimated min:** 17

#### `jr_b2plus_influencing_without_authority`

- **Título:** Influir sin autoridad
- **Descripción:** Persuadir peers / cross-team sin ser su
  manager. Framing, alignment, asks.
- **Sub-skills:** `flu_extended_speech`,
  `vocab_emotion_register`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `jr_b2plus_stakeholder_management`.
- **Estimated min:** 17

#### `jr_b2plus_hedging_diplomatic`

- **Título:** Hedging diplomático
- **Descripción:** "It might be worth considering...", "I'm not 100%
  sure, but my sense is...", "It's possible that...".
- **Sub-skills:** `vocab_emotion_register`,
  `gram_conditionals_second_third`,
  `flu_extended_speech`.
- **Prerequisites:** `jr_b2_meeting_diplomatic_pushback`.
- **Estimated min:** 15

#### `jr_b2plus_persuasion_advanced`

- **Título:** Persuasión avanzada
- **Descripción:** Persuasion frameworks: appeal to logos / pathos /
  ethos. Anchoring. Question-based.
- **Sub-skills:** `flu_extended_speech`,
  `flu_discourse_connectors`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `jr_b2plus_influencing_without_authority`.
- **Estimated min:** 17

#### `jr_b2plus_difficult_conversations`

- **Título:** Conversaciones difíciles
- **Descripción:** Termination, promotion denial, scope reduction.
  Stuctured difficult conversation framework.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `flu_thinking_in_english`.
- **Prerequisites:** `jr_b2_conflict_resolution`,
  `jr_b2_giving_critical_feedback`.
- **Estimated min:** 18

#### `jr_b2plus_giving_performance_review`

- **Título:** Dar performance review
- **Descripción:** Estructurar perf review formal. Articular
  growth areas con specifics.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `flu_discourse_connectors`.
- **Prerequisites:** `jr_b2_giving_critical_feedback`,
  `jr_b2_performance_review_receiving`.
- **Estimated min:** 17

#### `jr_b2plus_mentoring_conversations`

- **Título:** Conversaciones de mentoring
- **Descripción:** Coaching questions, listening more than telling,
  reframing problems.
- **Sub-skills:** `flu_turn_taking`, `list_inference`,
  `flu_thinking_in_english`.
- **Prerequisites:** `jr_b2plus_giving_performance_review`.
- **Estimated min:** 16

#### `jr_b2plus_executive_presence`

- **Título:** Presencia ejecutiva (capstone B2+)
- **Descripción:** Capstone bloque: integrar pace, calm, gravitas,
  thinking-aloud sin filler. 3-min impromptu pitch a executive.
- **Sub-skills:** consolida todas las sub-skills B2 cubiertas.
- **Prerequisites:** `jr_b2plus_persuasion_advanced`,
  `jr_b2plus_stakeholder_management`,
  `jr_b2plus_difficult_conversations`.
- **Estimated min:** 18

### 7.3 Mastery criteria del nivel B2+ (Job Ready)

- ≥ 80% de los 12 bloques B2+ completados.
- Score promedio ≥ 80 en sub-skills B2 cubiertas.
- WPM 120-140.
- < 3 fillers/min.
- Capaz de mantener executive simulation de 20 min con AI senior
  interviewer sin breakdown notable.

---

## 8. Prerequisites graph y orden recomendado

### 8.1 Visualización resumida

```
B1 entry: jr_b1_intro_self_professional (raíz, sin prereqs)
   │
   ├── jr_b1_describe_role_company
   │      ├── jr_b1_education_background → jr_b1_career_path_so_far
   │      ├── jr_b1_work_experience_simple → jr_b1_career_path_so_far
   │      ├── jr_b1_team_structure
   │      ├── jr_b1_daily_work_routine
   │      │      ├── jr_b1_tools_you_use → jr_b1_typical_workday_story
   │      │      └── jr_b1_polite_requests_work → ...
   │      └── jr_b1_industry_vocab_intro
   │
   ├── jr_b1_polite_requests_work
   │      ├── jr_b1_asking_clarification → jr_b1_following_instructions
   │      ├── jr_b1_disagreeing_politely → jr_b1_suggesting_ideas
   │      └── jr_b1_work_small_talk
   │
   ├── jr_b1_career_path_so_far
   │      ├── jr_b1_career_goals_simple → jr_b1_future_plans_work
   │      └── jr_b1_strengths_intro → jr_b1_what_you_learned
   │
   └── jr_b1_short_self_pitch (capstone B1; depende de 4 bloques)
          │
          ▼
      B1+ entrada: jr_b1plus_email_basics
          │
          ├── jr_b1plus_email_request
          ├── jr_b1plus_email_followup
          └── jr_b1plus_slack_register
                 │
                 └── jr_b1plus_meeting_participation
                        ├── jr_b1plus_disagreeing_proposal → ...
                        ├── jr_b1plus_recap_summarize
                        └── jr_b1plus_reporting_others
                              │
                              ▼
                        B2 entrada: jr_b2_interview_tell_me_about
                              │
                              ├── Module 7 (interview mastery)
                              ├── Module 8 (technical meetings)
                              ├── Module 9 (written register)
                              └── Module 10 (workplace dynamics)
                                     │
                                     ▼
                               B2+ entrada: jr_b2plus_senior_interview_*
                                     │
                                     ▼
                               jr_b2plus_executive_presence (capstone)
```

### 8.2 Orden recomendado (path principal)

El roadmap inicial post-onboarding debería sugerir blocks en este
orden para usuarios B1 entry:

1. `jr_b1_intro_self_professional`
2. `jr_b1_describe_role_company`
3. `jr_b1_daily_work_routine`
4. `jr_b1_polite_requests_work`
5. `jr_b1_disagreeing_politely`
6. `jr_b1_suggesting_ideas`
7. `jr_b1_work_experience_simple`
8. `jr_b1_career_path_so_far`
9. `jr_b1_strengths_intro`
10. `jr_b1_career_goals_simple`
11. `jr_b1_short_self_pitch` (capstone)
12. ... continúa con B1+ y B2 según roadmap personalizado.

User puede saltarse bloques si demuestra mastery vía test-out
(`pedagogical-system.md` §4.5).

### 8.3 Variantes según `professional_field`

Algunos bloques tienen variantes por campo profesional:

| Block ID | Variantes |
|----------|-----------|
| `jr_b1_industry_vocab_intro` | `_tech`, `_finance`, `_healthcare`, `_education`, `_sales_marketing` |
| `jr_b2_interview_technical_q_response` | `_engineering`, `_non_engineering` |
| `jr_b2_business_idioms_us` | `_tech_startup`, `_corporate` (post-MVP) |

Roadmap selecciona variante basado en
`student_profiles.professional_field`.

---

## 9. Mastery criteria por bloque y por nivel

### 9.1 Por bloque (default rule)

Cada bloque se marca completed cuando:
- Todos los assets obligatorios completados con score ≥ 60.
- Score promedio del bloque ≥ 70.

Excepciones:
- Capstone blocks (`jr_b1_short_self_pitch`,
  `jr_b2plus_executive_presence`): score promedio ≥ 75.

### 9.2 Por nivel (transición CEFR)

Coordinar con `curriculum-by-cefr.md` §5.9, §6.8, §7.9, §8.8.

Triggers para transición Job Ready level X → X+1:
- ≥ 80% de los bloques del nivel completados (no necesariamente todos).
- Score promedio en sub-skills target del nivel ≥ threshold (70/75/80
  según level).
- WPM y fillers en rango target del nivel.

Trigger emite `user.cefr_changed` event y celebración
(`motivation-and-achievements.md` §6.2.3:
`level_up_b1_to_b2`, etc.).

### 9.3 Tested-out por bloque

User puede skipear un bloque si:
- Cumple test-out criteria de `pedagogical-system.md` §4.5.
- Bloque está marcado `testable_out: true` (default true para los 74).
- Excepción: capstone blocks no son testable_out (deben hacerse).

---

## 10. Assets compartidos cross-block (reuse)

### 10.1 Filosofía

Atómicos del catálogo `media_atomics` se reusan cross-block. El
mismo audio "let me think about that for a sec" puede aparecer en
20 bloques distintos como `media_atomic_id` referenciado.

### 10.2 Atómicos cross-block esperados (high reuse)

| Atomic ID | Tipo | Reuse esperado | Bloques |
|-----------|------|---------------:|---------|
| `audio_filler_let_me_think` | audio TTS | 30+ | varios |
| `audio_could_you_repeat` | audio TTS | 25+ | varios |
| `audio_sorry_to_interrupt` | audio TTS | 20+ | meeting blocks |
| `image_office_meeting_room` | image AI | 15+ | meeting blocks |
| `image_video_call` | image AI | 20+ | remote work blocks |
| `audio_recruiter_voice_neutral` | audio TTS | 30+ | interview blocks |
| `audio_manager_voice_warm` | audio TTS | 25+ | feedback blocks |

### 10.3 Estimación de atomics totales para Job Ready

A 4 atomics por bloque y reuse 2x (cada atomic en 2 bloques en
promedio):

```
74 bloques × 4 atomics / bloque / 2 reuse = ~148 atomics únicos.
```

Aproximadamente la mitad del MVP atomic library (~280-350 según
`curriculum-by-cefr.md` §11.1).

---

## 11. Decisiones cerradas

### 11.1 74 bloques (upper bound) como target MVP ✓

**Razón:** §15.6 de `curriculum-by-cefr.md` establece "upper bound
como target". Mejor sobre-cubrir y permitir test-out que sub-cubrir.
74 está en el upper end del rango B1+B1++B2+B2+ (~58-74).

### 11.2 Granularidad de bloques: ~13-18 min cada uno ✓

**Razón:** alineado con `student_profile.daily_minutes_available`
median de 15. User puede completar 1 bloque en una sesión.

### 11.3 Capstones por nivel CEFR (B1, B2+): SÍ ✓

**Razón:** capstone obliga al user a integrar todo lo aprendido.
Funciona como "soft assessment" interno antes de transición CEFR.

### 11.4 Variantes por `professional_field` solo en 2-3 bloques: SÍ ✓

**Razón:** evitar combinatorial explosion. Solo donde el contenido
es genuinamente distinto por field (vocabulario industria + technical
question response).

### 11.5 Job Ready B1 incluye 22 bloques (vs 18 lower bound): SÍ ✓

**Razón:** B1 es el nivel de entry típico del producto. Mejor
sobre-cubrir el entry point para reducir frustración y aumentar
retention en primeras semanas.

### 11.6 NO bloques cross-track en MVP ✓

**Razón:** Job Ready y Daily Conversation son tracks separados. Si
un bloque sirve para ambos (ej: `daily_small_talk` vs
`work_small_talk`), se duplica con contexto distinto en lugar de
compartir.

**Reconsiderar post-MVP:** si tracking shows muchos users haciendo
ambos tracks, podríamos tener bloques compartidos.

### 11.7 NO blocks de inglés escrito separados de orales ✓

**Razón:** integrar reading/writing dentro de bloques mixtos
(emails con audio dictation, propuestas con voiceover) es más
realista. Plan futuro: track Writing-only post-MVP si demand existe.

### 11.8 Asset sequence estándar 4 assets por bloque ✓

**Razón:** alineado con `content-creation-system.md` §6.2 ("3-5
assets de tipos variados"). Algunos capstones tendrán 5; algunos
bloques cortos 3.

---

## 12. Plan de implementación

### 12.1 Sprint 1-2 (semanas 1-4): B1 (22 bloques)

- Crear los 22 bloques B1 con composiciones de assets.
- ~88 assets únicos.
- ~44 atomics únicos (con reuse).
- Validación pedagógica con humano antes de aprobar para producción.

### 12.2 Sprint 3 (semanas 5-6): B1+ (12 bloques)

- 12 bloques B1+.
- ~48 assets únicos (~24 atomics nuevos).

### 12.3 Sprint 4-5 (semanas 7-10): B2 (28 bloques)

- 28 bloques B2.
- ~112 assets únicos (~56 atomics nuevos).
- Foco interview + meetings: assets interactive más complejos.

### 12.4 Sprint 6 (semanas 11-12): B2+ (12 bloques)

- 12 bloques B2+.
- ~48 assets únicos (~24 atomics nuevos).

### 12.5 Total estimado

- **74 bloques** generados.
- **~296 assets** únicos.
- **~148 atomics** únicos (con reuse 2x).
- **12 semanas** de creación + validación.

A 30 min de generación + revisión por asset (estimación de
`content-creation-system.md` §7.1): ~150 horas de trabajo de
content team.

### 12.6 Validación pedagógica obligatoria

Cada bloque pasa por:
1. Generación AI (assets atomics).
2. Composición humana (sequencing, mastery criteria).
3. **Validación pedagógica:** humano con background en TESOL revisa
   antes de `approved_for_production = true`.
4. Beta con 50 users antes de release general.

(Detalle de pipeline en `content-creation-system.md` §5.)

---

## 13. Métricas de éxito

### 13.1 Por bloque

- Completion rate (% de users que empezaron y completaron) ≥ 75%.
- Average score ≥ 70.
- Average time matchea estimated_minutes ± 30%.
- < 5% de users marcan "este bloque no fue útil" en feedback.

### 13.2 Agregadas track Job Ready

- % de users en Job Ready que completan B1 → 50% target.
- % que llegan a B2 → 25% target en primer año.
- % que llegan a B2+ → 10% target.
- Job placement: 30% de users que completan B2+ reportan haber
  conseguido un trabajo en inglés en los siguientes 6 meses (post-MVP
  metric, requiere follow-up survey).

### 13.3 Triggers de revisión

- Si un bloque tiene completion rate < 50%: investigar (muy difícil,
  poco engaging, o prerequisite gap).
- Si un módulo (ej: Module 7 interview) tiene drop-off > 40% entre
  bloque 1 y bloque 5: revisar dificultad progression.

---

## 14. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`curriculum-by-cefr.md`](curriculum-by-cefr.md) §11 | Matriz tracks × CEFR (Job Ready row es la cubierta acá). |
| [`curriculum-by-cefr.md`](curriculum-by-cefr.md) §5-§8 | Plan B1, B1+, B2, B2+ (sub-skills, vocab target, mastery). |
| [`pedagogical-system.md`](pedagogical-system.md) §2.4 | Catálogo de 50 sub-skills (referenciadas en cada bloque). |
| [`pedagogical-system.md`](pedagogical-system.md) §4.5 | Test-out criteria por bloque. |
| [`content-creation-system.md`](content-creation-system.md) §6 | Estructura de `learning_block`. |
| [`content-creation-system.md`](content-creation-system.md) §7 | Cobertura inicial y roadmap de creación. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) §5 | Generación de roadmap usa este desglose para sequenciar. |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §3.1 | `professional_field` determina variantes. |
| [`motivation-and-achievements.md`](motivation-and-achievements.md) §6.2.5 | Logros de Roadmap (track-related). |

---

*Documento vivo. Actualizar cuando se rebalancee el track basado en
data de producción, se agreguen variantes de `professional_field`, o
se separen bloques que resulten muy largos / unan los muy cortos.*

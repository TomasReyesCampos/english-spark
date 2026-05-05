# Track Daily Conversation — Bloques B1 → B2+

> Desglose bloque-por-bloque del **Daily Conversation track** (track
> social-cotidiano del MVP, el más grande de los tres). Define los 62
> bloques que componen el track desde B1 hasta B2+, con sub-skills
> target, prerequisites, duración estimada, composición de assets y
> criterios de mastery por bloque.

**Estado:** Diseño v1.0 (listo para implementación)
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador (al crear contenido) +
humano (validación pedagógica).
**Alcance:** Daily Conversation track, niveles MVP (B1, B1+, B2, B2+).

---

## 0. Cómo leer este documento

- §1 establece **filosofía** del track Daily Conversation.
- §2 cubre **arquitectura** (módulos por CEFR).
- §3 muestra la **tabla resumen** de los 62 bloques.
- §4 detalla **B1: Foundation conversacional** (22 bloques).
- §5 detalla **B1+: Conversación con profundidad** (12 bloques).
- §6 detalla **B2: Native-speed mastery** (18 bloques).
- §7 detalla **B2+: Conversational mastery total** (10 bloques).
- §8 cubre **prerequisites graph** y orden recomendado.
- §9 cubre **mastery criteria por bloque y por nivel**.
- §10 cubre **assets compartidos cross-block** y cross-track.
- §11 cubre **decisiones cerradas**.
- §12 cubre **plan de implementación**.

---

## 1. Filosofía del track Daily Conversation

### 1.1 Audiencia target

Hispanohablante latinoamericano que quiere "simplemente hablar inglés
con confianza" sin un objetivo profesional o de viaje específico:
- **Family ties:** familia en USA, conversaciones recurrentes con
  parientes que viven angloparlantes.
- **Social connections:** amigos angloparlantes, comunidades online,
  partners.
- **Personal growth:** dejar de sentir bloqueo, expresarse "como
  realmente soy".
- **Foundation universal:** base que sirve a todos los demás contextos
  (trabajo, viaje, estudio).

### 1.2 Diferenciador vs Job Ready y Travel Confident

| Dimensión | Job Ready | Travel Confident | Daily Conversation |
|-----------|-----------|------------------|--------------------|
| Vocabulario | Business + tech | Travel + service | High-frequency social + emocional |
| Roleplays | Reuniones, entrevistas | Aeropuerto, hotel, calle | Amigos, familia, casual chats |
| Register | Formal y semi-formal | Service-formal + tourist-casual | Casual / íntimo |
| Cultural | USA business | USA travel | LatAm-USA bridge social |
| Output target | Comunicación al trabajo | Resolver situaciones de viaje | Conexión humana auténtica |

### 1.3 Outcome al completar el track

User capaz de:
- Mantener cualquier conversación casual de 30+ minutos sin breakdown.
- Expresar emociones, opiniones, humor, sarcasmo en inglés.
- Sentirse "uno mismo" en inglés (no una versión disminuida).
- Entender native-speed conversation con cultural references.
- (B2+) Tener conversaciones de "podcast quality" en cualquier tema.

### 1.4 Lo que NO cubre este track

- **Specialized vocabulary** (medical, legal, technical): otros
  tracks.
- **Public speaking / presentations**: parcialmente cubierto en Job
  Ready.
- **Academic discourse**: scope de track Academic (post-MVP).
- **A1-A2 absolute beginner**: queda fuera de MVP (Fase 2-3 cubrirá
  con A2).

### 1.5 Principio de diseño: "be yourself in English"

Foco emocional: el bloqueo del hispanohablante no es solo lingüístico
sino emocional ("no soy yo en inglés, soy una versión peor"). Cada
bloque debe permitir al user expresar su personalidad genuina, no
solo "responder correctamente".

Esto se manifiesta en:
- Roleplays con AI que tiene personalidad (no neutral robot).
- Free responses centrados en su vida real, no temas abstractos.
- Capstones de "telling your story" auténticas.

---

## 2. Arquitectura del track

### 2.1 Módulos por CEFR

```
┌──────────────────────────────────────────────────────────────┐
│  Daily Conversation Track (62 bloques MVP)                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  B1 (22) — Foundation conversacional                         │
│    ├ Module 1: Self & social basics (5)                      │
│    ├ Module 2: Daily life topics (6)                         │
│    ├ Module 3: Casual interactions (6)                       │
│    └ Module 4: Personal expression (5)                       │
│                                                              │
│  B1+ (12) — Conversación con profundidad                     │
│    ├ Module 5: Emotional connection (6)                      │
│    └ Module 6: Conversation maintenance (6)                  │
│                                                              │
│  B2 (18) — Native-speed mastery                              │
│    ├ Module 7: Native-speed comprehension (6)                │
│    ├ Module 8: Cultural fluency (6)                          │
│    └ Module 9: Group dynamics (6)                            │
│                                                              │
│  B2+ (10) — Conversational mastery total                     │
│    ├ Module 10: Deep personal conversations (5)              │
│    └ Module 11: Mastery (5)                                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Nomenclatura de bloques

`dc_<cefr>_<short_slug>` donde:
- `dc` = Daily Conversation prefix.
- `<cefr>` = `b1`, `b1plus`, `b2`, `b2plus`.
- `<short_slug>` = snake_case descriptivo.

### 2.3 Tamaño estándar

- 4 assets por bloque (3 en cortos, 5 en capstones).
- Duración estimada: 11-17 minutos.
- Composición típica: listening (audio "real" con barullo) + vocab
  drill + free response personal + roleplay con AI con personalidad.

### 2.4 Énfasis en personalidad de AI personajes

Daily Conversation requiere AI roleplay partners con personalidad
distintiva (no genérico "AI assistant"). Catálogo de personajes
recurrentes:

| Character ID | Persona | Aparece en |
|--------------|---------|------------|
| `char_alex_friendly_friend` | Amigo casual, 25-30, USA | Module 3, 4, 5, 6 |
| `char_grandma_patient` | Abuela paciente, 65+, lenta | Module 1, 2 |
| `char_sarah_witty_coworker` | Coworker con humor, 30-35 | Module 5, 6, 8 |
| `char_mike_chatty_neighbor` | Vecino conversador, 50+ | Module 2, 3 |
| `char_jamie_deep_friend` | Mejor amigo, conversaciones profundas | Module 9, 10, 11 |
| `char_emma_native_fast` | Native speaker rápida, B2+ challenge | Module 7, 11 |

(Detalle de characters en `content-creation-system.md` §3.4.)

---

## 3. Tabla resumen de los 62 bloques

### 3.1 Por CEFR y módulo

| # | Block ID | Título | CEFR | Min |
|--:|----------|--------|------|----:|
| **B1: Foundation conversacional (22)** ||||
| 1 | `dc_b1_intro_self_casual` | Presentarte casualmente | B1 | 12 |
| 2 | `dc_b1_family_basics` | Hablar de tu familia | B1 | 13 |
| 3 | `dc_b1_hometown` | Tu ciudad o pueblo de origen | B1 | 13 |
| 4 | `dc_b1_friends_relationships` | Amigos y relaciones | B1 | 14 |
| 5 | `dc_b1_personality_basics` | Cómo eres tú (personalidad) | B1 | 14 |
| 6 | `dc_b1_daily_routine_personal` | Tu rutina diaria personal | B1 | 12 |
| 7 | `dc_b1_hobbies_interests` | Hobbies e intereses | B1 | 13 |
| 8 | `dc_b1_food_preferences` | Comida favorita y preferencias | B1 | 12 |
| 9 | `dc_b1_movies_tv_shows` | Películas y series | B1 | 13 |
| 10 | `dc_b1_music_preferences` | Música que escuchas | B1 | 12 |
| 11 | `dc_b1_sports_casual` | Sports casual chat | B1 | 12 |
| 12 | `dc_b1_phone_call_personal` | Llamadas personales | B1 | 14 |
| 13 | `dc_b1_texting_register` | Mensajes de texto / WhatsApp | B1 | 13 |
| 14 | `dc_b1_making_plans_friends` | Hacer planes con amigos | B1 | 13 |
| 15 | `dc_b1_cancelling_plans` | Cancelar planes cortésmente | B1 | 12 |
| 16 | `dc_b1_giving_opinions_casual` | Dar opiniones casualmente | B1 | 13 |
| 17 | `dc_b1_agreeing_disagreeing` | Acuerdo y desacuerdo casual | B1 | 13 |
| 18 | `dc_b1_short_personal_anecdote` | Anécdota personal corta | B1 | 14 |
| 19 | `dc_b1_describing_people` | Describir personas | B1 | 13 |
| 20 | `dc_b1_describing_places` | Describir lugares | B1 | 13 |
| 21 | `dc_b1_future_plans_personal` | Planes futuros personales | B1 | 14 |
| 22 | `dc_b1_capstone_about_my_life` | Capstone: "About my life" | B1 | 16 |
| **B1+: Conversación con profundidad (12)** ||||
| 23 | `dc_b1plus_sharing_good_news` | Compartir buenas noticias | B1+ | 13 |
| 24 | `dc_b1plus_sharing_bad_news` | Compartir malas noticias | B1+ | 14 |
| 25 | `dc_b1plus_comforting_friend` | Consolar a un amigo | B1+ | 14 |
| 26 | `dc_b1plus_inviting_accepting` | Invitar y aceptar / rechazar | B1+ | 13 |
| 27 | `dc_b1plus_recommending_things` | Recomendar cosas (libros, restaurants) | B1+ | 13 |
| 28 | `dc_b1plus_telling_jokes` | Contar chistes / humor casual | B1+ | 14 |
| 29 | `dc_b1plus_keeping_convo_going` | Mantener la conversación viva | B1+ | 15 |
| 30 | `dc_b1plus_thoughtful_questions` | Hacer preguntas interesantes | B1+ | 13 |
| 31 | `dc_b1plus_casual_debate` | Debate casual sin escalar | B1+ | 14 |
| 32 | `dc_b1plus_roommate_household` | Convivencia (roommate, hogar) | B1+ | 14 |
| 33 | `dc_b1plus_dating_casual` | Citas casuales / online dating | B1+ | 15 |
| 34 | `dc_b1plus_capstone_long_chat` | Capstone: chat de 30 min | B1+ | 16 |
| **B2: Native-speed mastery (18)** ||||
| 35 | `dc_b2_native_speed_listening` | Listening native-speed | B2 | 16 |
| 36 | `dc_b2_banter_humor` | Banter y humor | B2 | 16 |
| 37 | `dc_b2_sarcasm_irony_recognition` | Reconocer sarcasmo e ironía | B2 | 15 |
| 38 | `dc_b2_movie_tv_references` | Referencias a películas/TV | B2 | 14 |
| 39 | `dc_b2_meme_internet_culture` | Memes y cultura de internet | B2 | 13 |
| 40 | `dc_b2_following_fast_groups` | Seguir conversación grupal rápida | B2 | 17 |
| 41 | `dc_b2_heated_debate_respectful` | Debate acalorado pero respetuoso | B2 | 17 |
| 42 | `dc_b2_empathy_difficult_topics` | Empatía en temas difíciles | B2 | 16 |
| 43 | `dc_b2_hosting_gathering` | Ser el host de una reunión | B2 | 16 |
| 44 | `dc_b2_being_a_guest` | Ser invitado | B2 | 14 |
| 45 | `dc_b2_toast_speech_short` | Toast o discurso corto | B2 | 15 |
| 46 | `dc_b2_complex_phone_calls` | Llamadas complejas | B2 | 16 |
| 47 | `dc_b2_voice_messages_register` | Audios de WhatsApp / voice notes | B2 | 13 |
| 48 | `dc_b2_news_current_events` | Discutir noticias y current events | B2 | 16 |
| 49 | `dc_b2_politics_carefully` | Discutir política con cuidado | B2 | 16 |
| 50 | `dc_b2_long_phone_family` | Llamada larga con familia | B2 | 15 |
| 51 | `dc_b2_anecdote_with_twist` | Anécdota con giro | B2 | 16 |
| 52 | `dc_b2_cross_generational` | Conversación intergeneracional | B2 | 14 |
| **B2+: Conversational mastery total (10)** ||||
| 53 | `dc_b2plus_persuasive_personal` | Persuasión en lo personal | B2+ | 17 |
| 54 | `dc_b2plus_conflict_personal_rel` | Conflicto en relaciones personales | B2+ | 17 |
| 55 | `dc_b2plus_apologies_forgiveness` | Disculpas y perdón | B2+ | 16 |
| 56 | `dc_b2plus_big_life_decisions` | Discutir decisiones grandes de vida | B2+ | 17 |
| 57 | `dc_b2plus_grief_condolences` | Condolencias y duelo | B2+ | 16 |
| 58 | `dc_b2plus_existential_topics` | Temas existenciales / filosóficos | B2+ | 17 |
| 59 | `dc_b2plus_mentoring_personal` | Mentoría personal informal | B2+ | 16 |
| 60 | `dc_b2plus_cultural_deep_dive` | Diferencias culturales profundas | B2+ | 17 |
| 61 | `dc_b2plus_humor_advanced` | Humor avanzado (puns, dry, dark) | B2+ | 15 |
| 62 | `dc_b2plus_capstone_podcast_quality` | Capstone: conversación "podcast quality" | B2+ | 18 |

**Totales:**
- B1: 22 bloques, ~290 minutos.
- B1+: 12 bloques, ~168 minutos.
- B2: 18 bloques, ~275 minutos.
- B2+: 10 bloques, ~166 minutos.
- **Total Daily Conversation MVP: 62 bloques, ~899 minutos** (~15
  horas de contenido pedagógico).

---

## 4. B1: Foundation conversacional (22 bloques)

### 4.1 Module 1: Self & social basics (5 bloques)

#### `dc_b1_intro_self_casual`

- **Título:** Presentarte casualmente
- **Descripción:** "Hi, I'm Maria, nice to meet you" en contextos
  sociales (party, gym, app). Distinto del professional intro.
- **Sub-skills:** `vocab_high_frequency_b1`,
  `flu_speaking_pace`, `pron_intonation_questions`.
- **Prerequisites:** ninguno (entry point del track).
- **Estimated min:** 12
- **Asset sequence:**
  1. `listening_mc_casual_intros_at_party` (3 min): identificar
     tone shifts entre intros formal vs casual.
  2. `vocab_drill_casual_greeting_phrases` (2 min): "What's up?",
     "How's it going?", "Pleasure".
  3. `free_response_intro_yourself_at_party` (3 min): grabar tu
     intro en contexto de party.
  4. `roleplay_meet_at_friend_party` (4 min): char_alex te
     presenta a 2-3 personas.

#### `dc_b1_family_basics`

- **Título:** Hablar de tu familia
- **Descripción:** Estructura básica de familia, hermanos, edades,
  donde viven.
- **Sub-skills:** `vocab_high_frequency_b1`,
  `gram_present_simple_continuous`,
  `flu_pause_management`.
- **Prerequisites:** `dc_b1_intro_self_casual`.
- **Estimated min:** 13

#### `dc_b1_hometown`

- **Título:** Tu ciudad o pueblo de origen
- **Descripción:** Describir donde naciste/creciste, qué tiene,
  qué te gusta y no, comparación con donde vives ahora.
- **Sub-skills:** `gram_articles`,
  `vocab_high_frequency_b1`,
  `flu_discourse_connectors`.
- **Prerequisites:** `dc_b1_intro_self_casual`.
- **Estimated min:** 13

#### `dc_b1_friends_relationships`

- **Título:** Amigos y relaciones
- **Descripción:** Hablar de tu mejor amigo, cómo se conocieron,
  qué hacen juntos. Patrones "we met at...", "we've known each
  other for...".
- **Sub-skills:** `gram_present_perfect`,
  `gram_past_simple_continuous`,
  `flu_extended_speech`.
- **Prerequisites:** `dc_b1_family_basics`.
- **Estimated min:** 14

#### `dc_b1_personality_basics`

- **Título:** Cómo eres tú (personalidad)
- **Descripción:** Adjetivos de personalidad (introvert/extrovert,
  funny, shy, easy-going), describirte con ejemplos.
- **Sub-skills:** `vocab_emotion_register` (intro),
  `vocab_high_frequency_b1`, `flu_extended_speech`.
- **Prerequisites:** `dc_b1_intro_self_casual`.
- **Estimated min:** 14

### 4.2 Module 2: Daily life topics (6 bloques)

#### `dc_b1_daily_routine_personal`

- **Título:** Tu rutina diaria personal
- **Descripción:** Mañana, tarde, noche personal (no laboral).
  Patrones con adverbs of frequency.
- **Sub-skills:** `gram_present_simple_continuous`,
  `vocab_high_frequency_b1`, `pron_word_stress`.
- **Prerequisites:** ninguno.
- **Estimated min:** 12

#### `dc_b1_hobbies_interests`

- **Título:** Hobbies e intereses
- **Descripción:** "I'm into...", "I really enjoy...", "I've been
  doing X for Y years". Hobbies comunes con vocabulary.
- **Sub-skills:** `gram_present_perfect`,
  `vocab_high_frequency_b1`,
  `flu_discourse_connectors`.
- **Prerequisites:** `dc_b1_daily_routine_personal`.
- **Estimated min:** 13

#### `dc_b1_food_preferences`

- **Título:** Comida favorita y preferencias
- **Descripción:** What you love, hate, can't stand, are obsessed
  with. Cuisines preferences. Allergies.
- **Sub-skills:** `vocab_high_frequency_b1`,
  `gram_present_simple_continuous`,
  `pron_intonation_questions`.
- **Prerequisites:** ninguno.
- **Estimated min:** 12

#### `dc_b1_movies_tv_shows`

- **Título:** Películas y series
- **Descripción:** What you're watching now, all-time favorites,
  recommend / avoid. Entertainment vocabulary.
- **Sub-skills:** `gram_present_perfect`,
  `vocab_high_frequency_b1`,
  `flu_extended_speech`.
- **Prerequisites:** ninguno.
- **Estimated min:** 13

#### `dc_b1_music_preferences`

- **Título:** Música que escuchas
- **Descripción:** Genres, favorite artists, last concert, music
  for moods.
- **Sub-skills:** `vocab_high_frequency_b1`,
  `gram_present_simple_continuous`, `flu_pause_management`.
- **Prerequisites:** ninguno.
- **Estimated min:** 12

#### `dc_b1_sports_casual`

- **Título:** Sports casual chat
- **Descripción:** Sports played, watched, fan teams. USA-specific:
  football vs soccer terminology.
- **Sub-skills:** `vocab_high_frequency_b1`,
  `gram_present_simple_continuous`, `pron_word_stress`.
- **Prerequisites:** ninguno.
- **Estimated min:** 12

### 4.3 Module 3: Casual interactions (6 bloques)

#### `dc_b1_phone_call_personal`

- **Título:** Llamadas personales
- **Descripción:** Llamar a un amigo, family member. Saludo, small
  talk, motivo, cierre. Estructura completa de call.
- **Sub-skills:** `flu_turn_taking`,
  `pron_intonation_questions`, `vocab_high_frequency_b1`.
- **Prerequisites:** `dc_b1_intro_self_casual`.
- **Estimated min:** 14

#### `dc_b1_texting_register`

- **Título:** Mensajes de texto / WhatsApp
- **Descripción:** Casual register, abbreviations comunes (lol, ttyl,
  bc, w/), emoji usage.
- **Sub-skills:** `vocab_idioms_common` (intro),
  `vocab_high_frequency_b1`,
  `gram_present_simple_continuous`.
- **Prerequisites:** `dc_b1_intro_self_casual`.
- **Estimated min:** 13

#### `dc_b1_making_plans_friends`

- **Título:** Hacer planes con amigos
- **Descripción:** "Let's grab coffee", "Wanna meet up Saturday?",
  proposing time and place.
- **Sub-skills:** `gram_future_forms`,
  `flu_turn_taking`, `pron_intonation_questions`.
- **Prerequisites:** `dc_b1_phone_call_personal`.
- **Estimated min:** 13

#### `dc_b1_cancelling_plans`

- **Título:** Cancelar planes cortésmente
- **Descripción:** "I'm so sorry, something came up", "Can we
  reschedule?", evitar dejar al otro colgado.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_turn_taking`, `gram_present_perfect`.
- **Prerequisites:** `dc_b1_making_plans_friends`.
- **Estimated min:** 12

#### `dc_b1_giving_opinions_casual`

- **Título:** Dar opiniones casualmente
- **Descripción:** "I think...", "In my opinion...", "If you ask
  me...", "Honestly,...".
- **Sub-skills:** `flu_discourse_connectors`,
  `flu_extended_speech`, `vocab_emotion_register`.
- **Prerequisites:** `dc_b1_movies_tv_shows`.
- **Estimated min:** 13

#### `dc_b1_agreeing_disagreeing`

- **Título:** Acuerdo y desacuerdo casual
- **Descripción:** "Totally", "I'm with you", "Not really", "Hmm,
  I see it differently". Sin escalación.
- **Sub-skills:** `flu_turn_taking`,
  `flu_discourse_connectors`, `pron_intonation_questions`.
- **Prerequisites:** `dc_b1_giving_opinions_casual`.
- **Estimated min:** 13

### 4.4 Module 4: Personal expression (5 bloques)

#### `dc_b1_short_personal_anecdote`

- **Título:** Anécdota personal corta
- **Descripción:** 1-2 min historia personal. "So the other day...",
  "You won't believe what happened to me...".
- **Sub-skills:** `gram_past_simple_continuous`,
  `flu_extended_speech`, `flu_discourse_connectors`.
- **Prerequisites:** `dc_b1_giving_opinions_casual`.
- **Estimated min:** 14

#### `dc_b1_describing_people`

- **Título:** Describir personas
- **Descripción:** Appearance + personality. "She's really nice",
  "He's the kind of person who...".
- **Sub-skills:** `gram_relative_clauses` (intro),
  `vocab_emotion_register`, `vocab_high_frequency_b1`.
- **Prerequisites:** `dc_b1_personality_basics`,
  `dc_b1_friends_relationships`.
- **Estimated min:** 13

#### `dc_b1_describing_places`

- **Título:** Describir lugares
- **Descripción:** Casa donde vives, lugar favorito, donde te
  gustaría vivir. Sensory details.
- **Sub-skills:** `gram_articles`,
  `vocab_high_frequency_b1`,
  `flu_extended_speech`.
- **Prerequisites:** `dc_b1_hometown`.
- **Estimated min:** 13

#### `dc_b1_future_plans_personal`

- **Título:** Planes futuros personales
- **Descripción:** Próximas vacaciones, dream goals, "I'm hoping
  to...", "I've been thinking about...".
- **Sub-skills:** `gram_future_forms`,
  `gram_conditionals_zero_first`,
  `flu_extended_speech`.
- **Prerequisites:** `dc_b1_hobbies_interests`.
- **Estimated min:** 14

#### `dc_b1_capstone_about_my_life`

- **Título:** Capstone: "About my life"
- **Descripción:** Capstone B1 del track: 3-4 min coherentes sobre
  tu vida (familia, trabajo, hobbies, dreams) integrando todo.
- **Sub-skills:** consolida todas las sub-skills B1 cubiertas.
- **Prerequisites:** `dc_b1_intro_self_casual`,
  `dc_b1_family_basics`, `dc_b1_personality_basics`,
  `dc_b1_future_plans_personal`,
  `dc_b1_short_personal_anecdote`.
- **Estimated min:** 16
- **Asset sequence:**
  1. `listening_mc_about_my_life_examples` (4 min): 5 ejemplos
     de "tell me about your life" en distinto register.
  2. `vocab_drill_life_signposting` (2 min): "Born and raised
     in...", "Currently I'm...", "Looking ahead...".
  3. `free_response_about_my_life_v1` (4 min): primera grabación
     3-4 min.
  4. `free_response_about_my_life_v2` (3 min): refinada con
     feedback.
  5. `roleplay_share_with_new_friend` (3 min): char_alex con
     follow-ups.

### 4.5 Mastery criteria del nivel B1 (Daily Conversation)

- ≥ 80% de los 22 bloques B1 completados.
- Score promedio ≥ 70 en sub-skills B1 cubiertas.
- WPM 80-100.
- < 6 fillers/min.
- Capaz de mantener casual conversation de 5 min con AI char_alex
  sin breakdown.

---

## 5. B1+: Conversación con profundidad (12 bloques)

### 5.1 Module 5: Emotional connection (6 bloques)

#### `dc_b1plus_sharing_good_news`

- **Título:** Compartir buenas noticias
- **Descripción:** "Guess what!", "I have great news!", reacciones
  esperadas del otro.
- **Sub-skills:** `vocab_emotion_register`,
  `pron_intonation_questions`, `flu_turn_taking`.
- **Prerequisites:** `dc_b1_capstone_about_my_life`.
- **Estimated min:** 13

#### `dc_b1plus_sharing_bad_news`

- **Título:** Compartir malas noticias
- **Descripción:** "I have something hard to share", "I don't know
  how to say this, but...", manejar reaction del otro.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `gram_past_perfect`.
- **Prerequisites:** `dc_b1plus_sharing_good_news`.
- **Estimated min:** 14

#### `dc_b1plus_comforting_friend`

- **Título:** Consolar a un amigo
- **Descripción:** "I'm so sorry", "That sounds really hard",
  active listening, "Is there anything I can do?".
- **Sub-skills:** `vocab_emotion_register`,
  `flu_turn_taking`, `flu_extended_speech`.
- **Prerequisites:** `dc_b1plus_sharing_bad_news`.
- **Estimated min:** 14

#### `dc_b1plus_inviting_accepting`

- **Título:** Invitar y aceptar / rechazar
- **Descripción:** Invitations more elaborated than B1, accepting
  warmly, declining gracefully.
- **Sub-skills:** `gram_conditionals_zero_first`,
  `vocab_emotion_register`,
  `flu_turn_taking`.
- **Prerequisites:** `dc_b1_making_plans_friends`.
- **Estimated min:** 13

#### `dc_b1plus_recommending_things`

- **Título:** Recomendar cosas (libros, restaurants)
- **Descripción:** "You have to try...", "I can't recommend X
  enough", articulando why.
- **Sub-skills:** `flu_extended_speech`,
  `vocab_emotion_register`,
  `gram_relative_clauses`.
- **Prerequisites:** `dc_b1_giving_opinions_casual`,
  `dc_b1_movies_tv_shows`.
- **Estimated min:** 13

#### `dc_b1plus_telling_jokes`

- **Título:** Contar chistes / humor casual
- **Descripción:** Setup-punchline structure, observational humor,
  self-deprecating humor (USA convention).
- **Sub-skills:** `pron_intonation_questions`,
  `flu_pause_management`, `vocab_emotion_register`.
- **Prerequisites:** `dc_b1_short_personal_anecdote`.
- **Estimated min:** 14

### 5.2 Module 6: Conversation maintenance (6 bloques)

#### `dc_b1plus_keeping_convo_going`

- **Título:** Mantener la conversación viva
- **Descripción:** Bridges: "That reminds me of...", "Speaking of
  X...", "That's funny because...". Avoid awkward silences.
- **Sub-skills:** `flu_turn_taking`,
  `flu_discourse_connectors`,
  `flu_extended_speech`.
- **Prerequisites:** `dc_b1_agreeing_disagreeing`.
- **Estimated min:** 15

#### `dc_b1plus_thoughtful_questions`

- **Título:** Hacer preguntas interesantes
- **Descripción:** Beyond "How was your day?". Curiosity-driven
  follow-ups, hypotheticals.
- **Sub-skills:** `pron_intonation_questions`,
  `gram_conditionals_second_third` (intro),
  `flu_turn_taking`.
- **Prerequisites:** `dc_b1plus_keeping_convo_going`.
- **Estimated min:** 13

#### `dc_b1plus_casual_debate`

- **Título:** Debate casual sin escalar
- **Descripción:** Friendly disagreement on opinions: best pizza,
  movies, sports teams. Without offending.
- **Sub-skills:** `flu_extended_speech`,
  `vocab_emotion_register`,
  `flu_discourse_connectors`.
- **Prerequisites:** `dc_b1_agreeing_disagreeing`,
  `dc_b1plus_keeping_convo_going`.
- **Estimated min:** 14

#### `dc_b1plus_roommate_household`

- **Título:** Convivencia (roommate, hogar)
- **Descripción:** Negociar chores, dealing with annoyances, "Hey,
  could we talk about...".
- **Sub-skills:** `vocab_emotion_register`,
  `gram_conditionals_zero_first`,
  `flu_turn_taking`.
- **Prerequisites:** `dc_b1plus_comforting_friend`.
- **Estimated min:** 14

#### `dc_b1plus_dating_casual`

- **Título:** Citas casuales / online dating
- **Descripción:** First date conversation, swipes language,
  flirting básico, follow-ups.
- **Sub-skills:** `vocab_emotion_register`,
  `pron_intonation_questions`,
  `flu_turn_taking`.
- **Prerequisites:** `dc_b1plus_thoughtful_questions`.
- **Estimated min:** 15

#### `dc_b1plus_capstone_long_chat`

- **Título:** Capstone: chat de 30 min
- **Descripción:** Capstone B1+ del track: mantener conversation
  con AI char_alex de 30 min sobre 3-5 topics distintos sin
  breakdown.
- **Sub-skills:** consolida sub-skills B1+ cubiertas.
- **Prerequisites:** completar al menos 8 de los 11 bloques B1+
  previos.
- **Estimated min:** 16
- **Asset sequence:**
  1. `listening_mc_long_chat_examples` (3 min): identificar
     conversation maintenance techniques.
  2. `vocab_drill_topic_transitions` (2 min).
  3. `roleplay_30min_chat_with_alex` (10 min): char_alex con 5
     topic switches.
  4. `free_response_reflect_on_chat` (1 min): qué te costó más,
     qué fluyó mejor.

### 5.3 Mastery criteria del nivel B1+ (Daily Conversation)

- ≥ 80% de los 12 bloques B1+ completados.
- Score promedio ≥ 75 en sub-skills cubiertas.
- WPM 100-120.
- < 5 fillers/min.
- Capaz de mantener 30 min conversation con AI char sin breakdown
  notable.

---

## 6. B2: Native-speed mastery (18 bloques)

### 6.1 Module 7: Native-speed comprehension (6 bloques)

#### `dc_b2_native_speed_listening`

- **Título:** Listening native-speed
- **Descripción:** Audio sin slowdown, native speakers casual,
  reduced forms (gonna, wanna, kinda, sorta).
- **Sub-skills:** `list_native_speed_general`,
  `pron_linking`, `flu_thinking_in_english`.
- **Prerequisites:** `dc_b1plus_capstone_long_chat`.
- **Estimated min:** 16

#### `dc_b2_banter_humor`

- **Título:** Banter y humor
- **Descripción:** Quick witty exchanges, playful insults entre
  amigos cercanos, riffing.
- **Sub-skills:** `flu_thinking_in_english`,
  `flu_self_correction_control`,
  `vocab_idioms_common`.
- **Prerequisites:** `dc_b1plus_telling_jokes`,
  `dc_b2_native_speed_listening`.
- **Estimated min:** 16

#### `dc_b2_sarcasm_irony_recognition`

- **Título:** Reconocer sarcasmo e ironía
- **Descripción:** Tone clues for sarcasm, irony in friendly
  interactions, "Yeah, right" sarcastic.
- **Sub-skills:** `list_inference`,
  `pron_intonation_questions`,
  `vocab_emotion_register`.
- **Prerequisites:** `dc_b2_banter_humor`.
- **Estimated min:** 15

#### `dc_b2_movie_tv_references`

- **Título:** Referencias a películas/TV
- **Descripción:** Common cultural references: Friends, Office,
  Star Wars, Marvel. Recognizing y joining the reference.
- **Sub-skills:** `list_inference`,
  `vocab_idioms_common`,
  `vocab_emotion_register`.
- **Prerequisites:** `dc_b2_native_speed_listening`.
- **Estimated min:** 14

#### `dc_b2_meme_internet_culture`

- **Título:** Memes y cultura de internet
- **Descripción:** Twitter/X language, Reddit threads, common
  memes, "I can't even", "based", "ratio".
- **Sub-skills:** `vocab_idioms_common`,
  `vocab_high_frequency_b1`,
  `list_inference`.
- **Prerequisites:** `dc_b1_texting_register`,
  `dc_b2_native_speed_listening`.
- **Estimated min:** 13

#### `dc_b2_following_fast_groups`

- **Título:** Seguir conversación grupal rápida
- **Descripción:** 4+ speakers, overlapping speech, quick topic
  switches. No te bloquees, pickup contextual cues.
- **Sub-skills:** `list_overlapping_speech` (intro),
  `list_native_speed_general`,
  `flu_thinking_in_english`.
- **Prerequisites:** `dc_b2_native_speed_listening`.
- **Estimated min:** 17

### 6.2 Module 8: Cultural fluency (6 bloques)

#### `dc_b2_heated_debate_respectful`

- **Título:** Debate acalorado pero respetuoso
- **Descripción:** Strong opinions, disagree firmly, no insults
  personales. "I respect your view, but I really think...".
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `dc_b1plus_casual_debate`.
- **Estimated min:** 17

#### `dc_b2_empathy_difficult_topics`

- **Título:** Empatía en temas difíciles
- **Descripción:** Mental health, family struggles, life crises.
  Hold space, not fix.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_turn_taking`, `flu_thinking_in_english`.
- **Prerequisites:** `dc_b1plus_comforting_friend`.
- **Estimated min:** 16

#### `dc_b2_hosting_gathering`

- **Título:** Ser el host de una reunión
- **Descripción:** Welcome guests, intro people, manage
  conversation flow, refill drinks language.
- **Sub-skills:** `flu_turn_taking`,
  `flu_extended_speech`, `vocab_emotion_register`.
- **Prerequisites:** `dc_b1plus_keeping_convo_going`.
- **Estimated min:** 16

#### `dc_b2_being_a_guest`

- **Título:** Ser invitado
- **Descripción:** RSVP, gift expectations, thank you note.
  Cultural USA: "I brought a bottle", thank-you texts.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_turn_taking`, `vocab_idioms_common`.
- **Prerequisites:** `dc_b1plus_inviting_accepting`.
- **Estimated min:** 14

#### `dc_b2_toast_speech_short`

- **Título:** Toast o discurso corto
- **Descripción:** 1-2 min toast en birthday/wedding/farewell.
  Estructura, tone, tempo.
- **Sub-skills:** `flu_extended_speech`,
  `flu_pause_management`, `vocab_emotion_register`.
- **Prerequisites:** `dc_b1plus_telling_jokes`.
- **Estimated min:** 15

#### `dc_b2_voice_messages_register`

- **Título:** Audios de WhatsApp / voice notes
- **Descripción:** 1-3 min voice notes coherentes, organizing
  thought-aloud sin filler excesivo.
- **Sub-skills:** `flu_extended_speech`,
  `flu_filler_reduction`,
  `flu_thinking_in_english`.
- **Prerequisites:** `dc_b1_texting_register`.
- **Estimated min:** 13

### 6.3 Module 9: Group dynamics (6 bloques)

#### `dc_b2_complex_phone_calls`

- **Título:** Llamadas complejas
- **Descripción:** Customer service calls, billing disputes,
  cable issues. Phone-distorted audio.
- **Sub-skills:** `list_phone_distorted_audio`,
  `vocab_emotion_register`,
  `flu_self_correction_control`.
- **Prerequisites:** `dc_b1_phone_call_personal`.
- **Estimated min:** 16

#### `dc_b2_news_current_events`

- **Título:** Discutir noticias y current events
- **Descripción:** Headlines + opinion. "Did you hear about...",
  "What do you think of...".
- **Sub-skills:** `flu_extended_speech`,
  `vocab_emotion_register`,
  `flu_discourse_connectors`.
- **Prerequisites:** `dc_b1_giving_opinions_casual`.
- **Estimated min:** 16

#### `dc_b2_politics_carefully`

- **Título:** Discutir política con cuidado
- **Descripción:** Hot-button topics manejados con tact, evitando
  alienar. "I see it differently because of my background".
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `dc_b2_heated_debate_respectful`,
  `dc_b2_news_current_events`.
- **Estimated min:** 16

#### `dc_b2_long_phone_family`

- **Título:** Llamada larga con familia
- **Descripción:** 30+ min calls catching up. Storytelling,
  asking, listening. Family in USA y back home.
- **Sub-skills:** `flu_extended_speech`,
  `flu_turn_taking`,
  `vocab_emotion_register`.
- **Prerequisites:** `dc_b1_phone_call_personal`,
  `dc_b1plus_capstone_long_chat`.
- **Estimated min:** 15

#### `dc_b2_anecdote_with_twist`

- **Título:** Anécdota con giro
- **Descripción:** 5 min story con setup → expectation → twist →
  resolution. Storytelling pulido.
- **Sub-skills:** `flu_extended_speech`,
  `gram_past_perfect`,
  `flu_discourse_connectors`.
- **Prerequisites:** `dc_b1_short_personal_anecdote`.
- **Estimated min:** 16

#### `dc_b2_cross_generational`

- **Título:** Conversación intergeneracional
- **Descripción:** Hablar con grandparents-aged speakers (slower,
  más formal) vs Gen Z (slang, fast).
- **Sub-skills:** `flu_thinking_in_english`,
  `vocab_idioms_common`,
  `list_distinguishing_accents`.
- **Prerequisites:** `dc_b2_native_speed_listening`,
  `dc_b2_meme_internet_culture`.
- **Estimated min:** 14

### 6.4 Mastery criteria del nivel B2 (Daily Conversation)

- ≥ 80% de los 18 bloques B2 completados.
- Score promedio ≥ 75 en sub-skills cubiertas.
- WPM 110-130.
- < 4 fillers/min.
- Capaz de mantener group conversation de 4 speakers sin
  breakdown.
- Capaz de pickup sarcasm e irony en >70% de los casos.

---

## 7. B2+: Conversational mastery total (10 bloques)

### 7.1 Module 10: Deep personal conversations (5 bloques)

#### `dc_b2plus_persuasive_personal`

- **Título:** Persuasión en lo personal
- **Descripción:** Convencer a friend/partner sobre decisión
  importante. Empathy + reasoning + ask.
- **Sub-skills:** `flu_extended_speech`,
  `vocab_emotion_register`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `dc_b2_heated_debate_respectful`.
- **Estimated min:** 17

#### `dc_b2plus_conflict_personal_rel`

- **Título:** Conflicto en relaciones personales
- **Descripción:** Difficult conversation con pareja, amigo
  cercano, family member. "I" statements, no blame.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_self_correction_control`,
  `flu_thinking_in_english`.
- **Prerequisites:** `dc_b1plus_roommate_household`,
  `dc_b2_empathy_difficult_topics`.
- **Estimated min:** 17

#### `dc_b2plus_apologies_forgiveness`

- **Título:** Disculpas y perdón
- **Descripción:** Real apology language (vs fake), accepting
  apology, forgiveness rituals.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`, `flu_pause_management`.
- **Prerequisites:** `dc_b2plus_conflict_personal_rel`.
- **Estimated min:** 16

#### `dc_b2plus_big_life_decisions`

- **Título:** Discutir decisiones grandes de vida
- **Descripción:** Career change, moving, getting married,
  having kids. With friend or partner.
- **Sub-skills:** `flu_extended_speech`,
  `gram_conditionals_second_third`,
  `vocab_emotion_register`.
- **Prerequisites:** `dc_b2plus_persuasive_personal`.
- **Estimated min:** 17

#### `dc_b2plus_grief_condolences`

- **Título:** Condolencias y duelo
- **Descripción:** Death of someone close. Right things to say,
  what to avoid. Hold silence sin nervios.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_pause_management`,
  `flu_turn_taking`.
- **Prerequisites:** `dc_b2_empathy_difficult_topics`.
- **Estimated min:** 16

### 7.2 Module 11: Mastery (5 bloques)

#### `dc_b2plus_existential_topics`

- **Título:** Temas existenciales / filosóficos
- **Descripción:** Meaning of life, religion (with respect),
  death, purpose. Late-night philosophical chats.
- **Sub-skills:** `flu_extended_speech`,
  `flu_thinking_in_english`,
  `vocab_emotion_register`.
- **Prerequisites:** `dc_b2plus_big_life_decisions`.
- **Estimated min:** 17

#### `dc_b2plus_mentoring_personal`

- **Título:** Mentoría personal informal
- **Descripción:** Friend asking for advice, listen first, ask
  reflection questions, share experience without imposing.
- **Sub-skills:** `flu_turn_taking`,
  `list_inference`, `flu_thinking_in_english`.
- **Prerequisites:** `dc_b2_empathy_difficult_topics`.
- **Estimated min:** 16

#### `dc_b2plus_cultural_deep_dive`

- **Título:** Diferencias culturales profundas
- **Descripción:** LatAm vs USA cultural values articulated
  reflexivamente. "In my culture we...", "I noticed in USA...".
- **Sub-skills:** `flu_extended_speech`,
  `vocab_emotion_register`,
  `flu_thinking_in_english`.
- **Prerequisites:** `dc_b2_politics_carefully`.
- **Estimated min:** 17

#### `dc_b2plus_humor_advanced`

- **Título:** Humor avanzado (puns, dry, dark)
- **Descripción:** Wordplay, dry British-style humor, dark humor
  con tact, recognizing audience.
- **Sub-skills:** `flu_thinking_in_english`,
  `vocab_idioms_common`,
  `list_inference`.
- **Prerequisites:** `dc_b2_banter_humor`,
  `dc_b2_sarcasm_irony_recognition`.
- **Estimated min:** 15

#### `dc_b2plus_capstone_podcast_quality`

- **Título:** Capstone: conversación "podcast quality"
- **Descripción:** Capstone final del track Daily Conversation:
  conversación de 30 min con char_emma sobre topic libre que el
  user proponga, transcribida, evaluada por coherence + depth +
  fluency.
- **Sub-skills:** consolida todas las sub-skills B2 cubiertas.
- **Prerequisites:** completar al menos 7 de los 9 bloques B2+
  previos.
- **Estimated min:** 18
- **Asset sequence:**
  1. `listening_mc_podcast_excerpts` (4 min): muestras de
     conversaciones podcast-quality.
  2. `vocab_drill_long_form_signposting` (2 min).
  3. `roleplay_30min_podcast_with_emma` (10 min): char_emma como
     interviewer/co-host, user elige topic.
  4. `free_response_self_reflection` (2 min): user evalúa qué
     funcionó, qué no.

### 7.3 Mastery criteria del nivel B2+ (Daily Conversation)

- ≥ 80% de los 10 bloques B2+ completados.
- Score promedio ≥ 80 en sub-skills cubiertas.
- WPM 120-140.
- < 3 fillers/min.
- Capstone podcast-quality conversation con char_emma:
  - 30 min sin breakdown.
  - Coherence score ≥ 80 (LLM-evaluated).
  - Self-correction control demostrado.

---

## 8. Prerequisites graph y orden recomendado

### 8.1 Visualización

```
B1 entry: dc_b1_intro_self_casual (raíz)
   │
   ├── Module 1 (self & social basics)
   │      ├── dc_b1_family_basics → dc_b1_friends_relationships
   │      ├── dc_b1_hometown
   │      └── dc_b1_personality_basics
   │
   ├── Module 2 (daily life topics, todos paralelos)
   │      ├── dc_b1_daily_routine_personal → dc_b1_hobbies_interests
   │      ├── dc_b1_food_preferences
   │      ├── dc_b1_movies_tv_shows
   │      ├── dc_b1_music_preferences
   │      └── dc_b1_sports_casual
   │
   ├── Module 3 (casual interactions)
   │      ├── dc_b1_phone_call_personal → dc_b1_making_plans_friends → dc_b1_cancelling_plans
   │      ├── dc_b1_texting_register
   │      └── dc_b1_giving_opinions_casual → dc_b1_agreeing_disagreeing
   │
   └── Module 4 (personal expression)
          ├── dc_b1_short_personal_anecdote
          ├── dc_b1_describing_people / dc_b1_describing_places
          ├── dc_b1_future_plans_personal
          │
          ▼
      dc_b1_capstone_about_my_life (capstone B1)
          │
          ▼
      B1+ entrada: dc_b1plus_sharing_good_news
          │
          ├── Module 5 (emotional)
          │     ├── dc_b1plus_sharing_bad_news → dc_b1plus_comforting_friend
          │     ├── dc_b1plus_inviting_accepting
          │     ├── dc_b1plus_recommending_things
          │     └── dc_b1plus_telling_jokes
          │
          └── Module 6 (maintenance)
                ├── dc_b1plus_keeping_convo_going → dc_b1plus_thoughtful_questions
                ├── dc_b1plus_casual_debate
                ├── dc_b1plus_roommate_household
                ├── dc_b1plus_dating_casual
                │
                ▼
          dc_b1plus_capstone_long_chat (capstone B1+)
                │
                ▼
          B2 entrada: dc_b2_native_speed_listening
                │
                ├── Module 7 (native-speed)
                ├── Module 8 (cultural fluency)
                └── Module 9 (group dynamics)
                       │
                       ▼
                 B2+ entrada: dc_b2plus_persuasive_personal
                       │
                       ├── Module 10 (deep personal)
                       └── Module 11 (mastery)
                              │
                              ▼
                        dc_b2plus_capstone_podcast_quality (capstone final)
```

### 8.2 Orden recomendado (path principal)

Para user B1 entry típico que prioriza Daily Conversation:

1. `dc_b1_intro_self_casual`
2. `dc_b1_personality_basics`
3. `dc_b1_family_basics`
4. `dc_b1_daily_routine_personal`
5. `dc_b1_hobbies_interests`
6. `dc_b1_food_preferences`
7. `dc_b1_phone_call_personal`
8. `dc_b1_making_plans_friends`
9. `dc_b1_giving_opinions_casual`
10. `dc_b1_agreeing_disagreeing`
11. `dc_b1_short_personal_anecdote`
12. `dc_b1_capstone_about_my_life` (capstone)
13. ... continúa con B1+, B2, B2+ según roadmap.

### 8.3 Variantes por contexto

Daily Conversation es el track más universal — pocas variantes por
profile. Variante única:

| Block ID | Variantes |
|----------|-----------|
| `dc_b1_sports_casual` | `_us_sports` (NFL, NBA, MLB), `_global_sports` (soccer, F1) |

---

## 9. Mastery criteria por bloque y por nivel

### 9.1 Por bloque

Default rule (idéntica a otros tracks):
- Todos los assets obligatorios completados con score ≥ 60.
- Score promedio del bloque ≥ 70.

Excepciones capstones:
- `dc_b1_capstone_about_my_life`: score promedio ≥ 75.
- `dc_b1plus_capstone_long_chat`: score promedio ≥ 75 + completion
  efectiva del 30-min chat.
- `dc_b2plus_capstone_podcast_quality`: score promedio ≥ 80 +
  coherence LLM ≥ 80.

### 9.2 Por nivel

(Coordinar con `curriculum-by-cefr.md` §5.9-§8.8.)

Métrica especial Daily Conversation: **fluidez emocional**. Más allá
de WPM y fillers, capturar:
- Capacidad de expresar emoción sin bloqueo (sub-skill
  `vocab_emotion_register`).
- Comfort en silencios (no rellenar con "uhh").
- Personality match: AI roleplay partners reportan "sentí que
  hablaba con una persona real" (LLM-evaluated).

### 9.3 Tested-out por bloque

Default `testable_out: true` excepto capstones.

---

## 10. Assets compartidos cross-block y cross-track

### 10.1 Atómicos cross-block esperados (high reuse interno)

| Atomic ID | Tipo | Reuse esperado | Bloques |
|-----------|------|---------------:|---------|
| `audio_alex_friendly_greeting` | audio TTS char | 15+ | varios Module 1, 3 |
| `audio_grandma_slow_question` | audio TTS char | 8+ | Module 1, 2 |
| `audio_emma_native_fast_intro` | audio TTS char | 10+ | Module 7, 11 |
| `audio_party_ambient_noise` | audio variant | 15+ | social blocks |
| `audio_phone_call_quality` | audio variant | 10+ | phone blocks |
| `image_casual_party_scene` | image AI | 8+ | social blocks |
| `image_coffee_shop_chat` | image AI | 10+ | various |
| `image_video_call_friend` | image AI | 8+ | phone/long_chat blocks |

### 10.2 Estimación de atomics totales

A 4 atomics por bloque y reuse 2x:

```
62 bloques × 4 atomics / 2 reuse = ~124 atomics únicos
```

Aproximadamente 40% del MVP atomic library (~280-350).

### 10.3 Cross-track reuse

Daily Conversation es la base universal — atomics de DC se reusan
extensivamente en los otros tracks:

| Atomic ID | Daily Conv blocks | Job Ready blocks | Travel blocks |
|-----------|-------------------|------------------|---------------|
| `audio_filler_let_me_think` | 20+ | 15+ | 10+ |
| `audio_could_you_repeat` | 15+ | 10+ | 8+ |
| `image_phone_in_hand` | 10+ | 5+ | 8+ |
| `audio_turn_taking_phrases` | 15+ | 10+ | 5+ |
| `audio_casual_greeting_set` | 8+ | 5+ | 8+ |

Esta reusabilidad alta hace que crear primero Daily Conversation reduce
el costo subsiguiente de los otros tracks.

---

## 11. Decisiones cerradas

### 11.1 62 bloques (upper bound del MVP) ✓

**Razón:** consistente con §15.6 de `curriculum-by-cefr.md`. Daily
Conversation es el track más grande del MVP por ser foundation
universal.

### 11.2 Característico: AI characters con personalidad ✓

**Razón:** Daily Conversation requiere "be yourself in English". Si
roleplay partners son robots neutrales, user no practica expresar
personalidad. 6 characters recurrentes con voces y personalidades
distintas.

**Implementación:** ver `content-creation-system.md` §3.4 para
detalles del catálogo `characters`.

### 11.3 "Be yourself in English" como principio de diseño ✓

**Razón:** el bloqueo del hispanohablante es emocional, no solo
lingüístico. Cada bloque debe permitir expresión de personalidad
real, no solo "respuestas correctas".

### 11.4 Sub-skill `vocab_emotion_register` aparece en >50% de bloques ✓

**Razón:** la fluidez emocional es el outcome más importante. Mejor
sobre-cubrir que sub-cubrir.

### 11.5 Capstones progresivos: 3 capstones del track ✓

- B1 capstone: "About my life" (3-4 min sobre tu vida).
- B1+ capstone: "Chat de 30 min" (mantener conversación larga).
- B2+ capstone: "Podcast quality conversation" (30 min con
  evaluación LLM de coherencia).

**Razón:** progression demostrable. User ve avance concreto.

### 11.6 USA-first cultural references en MVP ✓

**Razón:** target market initial. Sub-track UK / global cultural
references post-MVP.

### 11.7 Memes y internet culture incluidos en B2 ✓

**Razón:** real-world casual conversation en USA incluye Twitter/X
language, Reddit, memes. Excluirlos sería incompleto.

**Limitación:** memes are dated rápido. Se prevé refresh cada 6
meses del block `dc_b2_meme_internet_culture`.

### 11.8 Daily Conversation crea atomics que otros tracks reusan ✓

**Razón:** orden recomendado de creación: DC primero, después JR y
TC, porque DC genera la base de atomics universales (greetings,
fillers, casual phrases) que aparecen en todos los tracks.

---

## 12. Plan de implementación

### 12.1 Sprint 1-3 (semanas 1-5): B1 (22 bloques)

- Crear los 22 bloques B1 con composiciones de assets.
- Crear los 6 characters recurrentes (audio TTS + visual asset por
  cada uno).
- ~88 assets únicos.
- ~44 atomics únicos.

### 12.2 Sprint 4 (semana 6): B1+ (12 bloques)

- 12 bloques B1+.
- ~48 assets únicos (~24 atomics nuevos).
- Capstone 30-min chat: roleplay infrastructure complejo.

### 12.3 Sprint 5-6 (semanas 7-9): B2 (18 bloques)

- 18 bloques B2.
- ~72 assets únicos (~36 atomics nuevos).
- Native-speed audio: validation con native speakers.
- Cultural references audio: validar relevance, refresh schedule.

### 12.4 Sprint 7 (semana 10): B2+ (10 bloques)

- 10 bloques B2+.
- ~40 assets únicos (~20 atomics nuevos).
- Capstone podcast-quality: LLM coherence evaluation infra.

### 12.5 Total estimado

- **62 bloques** generados.
- **~248 assets** únicos.
- **~124 atomics** únicos (con reuse 2x).
- **10 semanas** de creación + validación.

A 30 min de generación + revisión por asset: ~125 horas de content
team.

### 12.6 Validación pedagógica obligatoria

Validation idéntica a Job Ready y Travel Confident.

Validation cultural extra para bloques con USA-specific content
(memes, idioms, sports, política): revisor debe ser native speaker
o equivalente (5+ años en USA).

---

## 13. Métricas de éxito

### 13.1 Por bloque

- Completion rate ≥ 75%.
- Average score ≥ 70.
- Time matchea ± 30%.
- LLM personality-match score ≥ 70 (qualitative: "se siente como
  conversación real").

### 13.2 Agregadas track Daily Conversation

- % users en DC que completan B1 → 55% target (foundation, mucho
  contenido, drop-off natural).
- % que llegan a B2 → 20% target.
- % que llegan a B2+ → 8% target.
- Self-reported confidence: "me siento yo mismo en inglés" → 70%
  agree en users que completaron B2.

### 13.3 Métrica especial: emotional fluency

Coherence LLM evaluation cada 4 weeks:
- Coherencia narrativa.
- Expression de emoción.
- Personality-match.

Si coherencia plateau > 6 weeks: trigger personalized review session.

---

## 14. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`curriculum-by-cefr.md`](curriculum-by-cefr.md) §11 | Matriz tracks × CEFR (Daily Conversation row). |
| [`curriculum-by-cefr.md`](curriculum-by-cefr.md) §5-§8 | Plan B1, B1+, B2, B2+. |
| [`pedagogical-system.md`](pedagogical-system.md) §2.4 | Catálogo de 50 sub-skills. |
| [`content-creation-system.md`](content-creation-system.md) §3.4 | Catálogo de 6 characters recurrentes. |
| [`content-creation-system.md`](content-creation-system.md) §6 | Estructura de `learning_block`. |
| [`track-job-ready-blocks.md`](track-job-ready-blocks.md) | Track paralelo (Job Ready); cross-track atomic reuse. |
| [`track-travel-confident-blocks.md`](track-travel-confident-blocks.md) | Track paralelo (Travel Confident); cross-track reuse. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) §5 | Generación de roadmap usa este desglose. |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §3.1 | `primary_goals` determina si DC es track recomendado. |
| [`motivation-and-achievements.md`](motivation-and-achievements.md) §6.2.5 | Logros de Roadmap (track-related). |

---

*Documento vivo. Actualizar cuando se rebalancee el track basado en
data de producción, se actualice catálogo de memes/cultural
references, o se descubran gaps en cobertura emocional.*

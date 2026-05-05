# Track Travel Confident — Bloques B1 → B2+

> Desglose bloque-por-bloque del **Travel Confident track** (track de
> viaje, segundo en prioridad MVP). Define los 39 bloques que componen
> el track desde B1 hasta B2+, con sub-skills target, prerequisites,
> duración estimada, composición de assets y criterios de mastery por
> bloque.

**Estado:** Diseño v1.0 (listo para implementación)
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador (al crear contenido) +
humano (validación pedagógica).
**Alcance:** Travel Confident track, niveles MVP (B1, B1+, B2, B2+).

---

## 0. Cómo leer este documento

- §1 establece **filosofía** del track Travel Confident.
- §2 cubre **arquitectura** (módulos por CEFR).
- §3 muestra la **tabla resumen** de los 39 bloques.
- §4 detalla **B1: Travel basics** (15 bloques).
- §5 detalla **B1+: Travel real-world situations** (8 bloques).
- §6 detalla **B2: Travel deep + business travel** (10 bloques).
- §7 detalla **B2+: Cross-cultural + storytelling** (6 bloques).
- §8 cubre **prerequisites graph** y orden recomendado.
- §9 cubre **mastery criteria por bloque y por nivel**.
- §10 cubre **assets compartidos cross-block**.
- §11 cubre **decisiones cerradas**.
- §12 cubre **plan de implementación**.

---

## 1. Filosofía del track Travel Confident

### 1.1 Audiencia target

Hispanohablante latinoamericano que viaja (o quiere viajar) a
contextos donde el inglés es la lingua franca:
- **Tourism:** USA, UK, Asia, Europa.
- **Family visits:** family living abroad (US-resident relatives).
- **Business travel:** conferencias, client visits, work trips.
- **Long-term stay:** working holiday, study abroad, expat short-term.

### 1.2 Diferenciador vs Job Ready y Daily Conversation

| Dimensión | Daily Conversation | Job Ready | Travel Confident |
|-----------|--------------------|-----------|------------------|
| Vocabulario | High-frequency genérico | Business + tech | Travel + service interactions |
| Roleplays | Cotidianos | Reuniones, entrevistas | Aeropuerto, hotel, restaurant, calle |
| Register | Casual neutro | Formal y semi-formal | Mix: service-formal + tourist-casual |
| Cultural | LatAm-USA general | USA business culture | Cross-cultural awareness múltiples destinos |
| Output target | Conversación social | Comunicación al trabajo | Resolver situaciones de viaje sin asistencia |

### 1.3 Outcome al completar el track

User capaz de:
- Manejar todo un viaje a USA (o destino anglo) sin intérprete.
- Resolver situaciones imprevistas (vuelo cancelado, hotel issues,
  emergencia médica leve).
- Mantener conversaciones sociales con locales y otros viajeros.
- Negociar precios, solicitudes especiales, cambios de plan.
- (B2+) Contar historias de viaje con detalle y matiz, networking en
  conferencias internacionales, hacer business travel sin esfuerzo.

### 1.4 Lo que NO cubre este track

- **Vocabulario hyper-específico de un destino** (ej: jerga local
  cockney en UK). Foco en inglés "internacional" / "global English".
- **Aviation operations específico** (pilots, controllers): scope de
  Aviation English (post-MVP).
- **Tourism industry profesional** (tour guides, hotel staff):
  parcialmente solapa pero foco es viajero, no profesional del rubro.
- **A1-A2 entry point**: queda fuera de MVP (Fase 2-3 cubrirá).

### 1.5 Principio de diseño: "trip simulation"

Los bloques siguen el arc de un viaje real, no temas aislados. User
puede simular un viaje completo de inicio a fin (pre-trip → airport →
hotel → city → return) y el sistema rastrea su confidence en cada
fase.

---

## 2. Arquitectura del track

### 2.1 Módulos por CEFR

```
┌──────────────────────────────────────────────────────────────┐
│  Travel Confident Track (39 bloques MVP)                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  B1 (15) — Travel basics                                     │
│    ├ Module 1: Pre-trip & airport (4)                        │
│    ├ Module 2: Hotel & accommodation (3)                     │
│    ├ Module 3: Eating out (3)                                │
│    └ Module 4: Getting around city (5)                       │
│                                                              │
│  B1+ (8) — Travel real-world situations                      │
│    ├ Module 5: Travel problems & solutions (4)               │
│    └ Module 6: Social travel (4)                             │
│                                                              │
│  B2 (10) — Travel deep + business travel                     │
│    ├ Module 7: Business travel (5)                           │
│    └ Module 8: Cultural travel (5)                           │
│                                                              │
│  B2+ (6) — Cross-cultural + storytelling                     │
│    ├ Module 9: Travel storytelling avanzado (3)              │
│    └ Module 10: Cross-cultural mastery (3)                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Nomenclatura de bloques

`tc_<cefr>_<short_slug>` donde:
- `tc` = Travel Confident prefix.
- `<cefr>` = `b1`, `b1plus`, `b2`, `b2plus`.
- `<short_slug>` = snake_case descriptivo.

### 2.3 Tamaño estándar

- 4 assets por bloque (3 en bloques cortos, 5 en capstone).
- Duración estimada: 12-17 minutos.
- Composición típica: listening (audio "real" de aeropuerto/hotel) +
  vocab drill + free response + roleplay situacional.

### 2.4 Énfasis en listening "real-world"

A diferencia de Job Ready (donde la mayoría es speaking-output),
Travel Confident requiere mucho listening con audio "ruidoso":
- Aeropuerto con anuncios.
- Restaurante con barullo.
- Voz de hotel staff con acento.
- Phone calls (calidad reducida).

Sub-skill `list_phone_distorted_audio` (B2) y
`list_distinguishing_accents` (B2) son críticos en este track.

---

## 3. Tabla resumen de los 39 bloques

### 3.1 Por CEFR y módulo

| # | Block ID | Título | CEFR | Min |
|--:|----------|--------|------|----:|
| **B1: Travel basics (15)** ||||
| 1 | `tc_b1_pretrip_planning` | Planificar el viaje | B1 | 13 |
| 2 | `tc_b1_airport_checkin_security` | Check-in y security | B1 | 14 |
| 3 | `tc_b1_inflight_basics` | Conversaciones en el vuelo | B1 | 12 |
| 4 | `tc_b1_immigration_customs` | Inmigración y aduana USA | B1 | 14 |
| 5 | `tc_b1_hotel_checkin` | Check-in en hotel | B1 | 13 |
| 6 | `tc_b1_hotel_amenities_questions` | Preguntar por amenities | B1 | 12 |
| 7 | `tc_b1_hotel_basic_issues` | Problemas básicos en hotel | B1 | 14 |
| 8 | `tc_b1_restaurant_ordering` | Ordenar en restaurante | B1 | 13 |
| 9 | `tc_b1_restaurant_dietary_basic` | Restricciones dietarias básicas | B1 | 13 |
| 10 | `tc_b1_paying_tipping` | Pagar y propinas USA | B1 | 12 |
| 11 | `tc_b1_asking_directions` | Pedir direcciones | B1 | 13 |
| 12 | `tc_b1_public_transportation` | Transporte público | B1 | 14 |
| 13 | `tc_b1_taxi_rideshare` | Taxi y rideshare (Uber/Lyft) | B1 | 12 |
| 14 | `tc_b1_shopping_basic` | Compras básicas | B1 | 13 |
| 15 | `tc_b1_tourism_questions` | Preguntar sobre lugares turísticos | B1 | 14 |
| **B1+: Travel real-world situations (8)** ||||
| 16 | `tc_b1plus_lost_delayed_luggage` | Equipaje perdido o demorado | B1+ | 15 |
| 17 | `tc_b1plus_flight_cancellation` | Vuelo cancelado o reprogramado | B1+ | 16 |
| 18 | `tc_b1plus_hotel_complex_issues` | Problemas complejos en hotel | B1+ | 15 |
| 19 | `tc_b1plus_medical_minor` | Situaciones médicas menores | B1+ | 16 |
| 20 | `tc_b1plus_meeting_locals` | Conocer locales | B1+ | 14 |
| 21 | `tc_b1plus_meeting_other_travelers` | Otros viajeros (hostel, tour) | B1+ | 14 |
| 22 | `tc_b1plus_cultural_etiquette_basic` | Etiqueta cultural básica | B1+ | 13 |
| 23 | `tc_b1plus_short_travel_story` | Contar una experiencia de viaje | B1+ | 15 |
| **B2: Travel deep + business travel (10)** ||||
| 24 | `tc_b2_business_trip_arrival` | Business trip: llegada y check-in profesional | B2 | 15 |
| 25 | `tc_b2_business_trip_meetings` | Reuniones durante business trip | B2 | 16 |
| 26 | `tc_b2_business_trip_dinner` | Cena de negocios | B2 | 16 |
| 27 | `tc_b2_conference_networking` | Networking en conferencia | B2 | 17 |
| 28 | `tc_b2_business_travel_logistics` | Logística de business travel | B2 | 14 |
| 29 | `tc_b2_negotiating_prices_disputes` | Negociar precios y disputar cargos | B2 | 16 |
| 30 | `tc_b2_emergency_situations` | Emergencias (robo, médico serio) | B2 | 17 |
| 31 | `tc_b2_cultural_deep_topics` | Hablar de cultura con detalle | B2 | 16 |
| 32 | `tc_b2_travel_disagreements` | Desacuerdos al viajar en grupo | B2 | 15 |
| 33 | `tc_b2_long_travel_anecdote` | Anécdota larga de viaje | B2 | 16 |
| **B2+: Cross-cultural + storytelling (6)** ||||
| 34 | `tc_b2plus_cross_cultural_nuance` | Matices culturales sutiles | B2+ | 17 |
| 35 | `tc_b2plus_high_stakes_situation` | Situaciones high-stakes (deportación, hospital serio) | B2+ | 18 |
| 36 | `tc_b2plus_travel_narrative_extended` | Narrativa extendida (15 min historia) | B2+ | 18 |
| 37 | `tc_b2plus_business_travel_executive` | Business travel ejecutivo | B2+ | 17 |
| 38 | `tc_b2plus_diplomatic_intercultural` | Diplomacia intercultural | B2+ | 16 |
| 39 | `tc_b2plus_travel_writing_review` | Review / narrativa escrita de viaje | B2+ | 16 |

**Totales:**
- B1: 15 bloques, ~196 minutos.
- B1+: 8 bloques, ~118 minutos.
- B2: 10 bloques, ~158 minutos.
- B2+: 6 bloques, ~102 minutos.
- **Total Travel Confident MVP: 39 bloques, ~574 minutos** (~9.5
  horas de contenido pedagógico).

---

## 4. B1: Travel basics (15 bloques)

### 4.1 Module 1: Pre-trip & airport (4 bloques)

#### `tc_b1_pretrip_planning`

- **Título:** Planificar el viaje
- **Descripción:** Llamar a aerolínea, hablar con agencia, confirmar
  detalles. "I'd like to book...", "Could you confirm...".
- **Sub-skills:** `vocab_travel_practical`,
  `gram_present_simple_continuous`,
  `pron_intonation_questions`.
- **Prerequisites:** ninguno (entry point del track B1).
- **Estimated min:** 13
- **Asset sequence:**
  1. `listening_mc_travel_agency_calls` (3 min): identificar
     información clave de 4 llamadas.
  2. `vocab_drill_travel_phrases` (2 min): "round trip", "layover",
     "non-refundable", etc.
  3. `free_response_describe_planned_trip` (3 min): describir el
     próximo viaje en 90s.
  4. `roleplay_call_travel_agency` (5 min): roleplay AI agente,
     reservar viaje.
- **Mastery criteria:**
  - Score promedio ≥ 70 en pronunciation y fluency.
  - Capacidad de articular destino, fechas, número de pasajeros.

#### `tc_b1_airport_checkin_security`

- **Título:** Check-in y security
- **Descripción:** Diálogos con counter agent + TSA agent.
  Preguntas estándar: ID, equipaje, contenidos.
- **Sub-skills:** `vocab_travel_practical`,
  `list_detail_capture`, `pron_final_consonants`.
- **Prerequisites:** `tc_b1_pretrip_planning`.
- **Estimated min:** 14
- **Asset sequence:**
  1. `listening_mc_airport_counter_dialogues` (3 min).
  2. `vocab_drill_security_phrases` (2 min): "boarding pass",
     "carry-on", "liquids", "laptop out".
  3. `fill_blank_checkin_dialogue` (3 min): completar diálogo en
     mostrador.
  4. `roleplay_full_checkin_flow` (5 min): roleplay AI counter agent
     + TSA, end-to-end.

#### `tc_b1_inflight_basics`

- **Título:** Conversaciones en el vuelo
- **Descripción:** Flight attendants, otros pasajeros, requests
  básicos (water, blanket, headphones).
- **Sub-skills:** `vocab_travel_practical`,
  `flu_turn_taking`, `pron_intonation_questions`.
- **Prerequisites:** `tc_b1_airport_checkin_security`.
- **Estimated min:** 12

#### `tc_b1_immigration_customs`

- **Título:** Inmigración y aduana USA
- **Descripción:** Las 10 preguntas más comunes de US Customs +
  declaración aduanera. Tono importante: directo, no nervioso.
- **Sub-skills:** `list_detail_capture`,
  `vocab_travel_practical`,
  `flu_self_correction_control` (intro).
- **Prerequisites:** `tc_b1_airport_checkin_security`.
- **Estimated min:** 14
- **Asset sequence:**
  1. `listening_mc_customs_questions` (3 min): "Purpose of visit?",
     "How long will you stay?", etc.
  2. `vocab_drill_immigration_phrases` (2 min).
  3. `free_response_practice_answers` (4 min): grabar respuesta a 8
     preguntas comunes.
  4. `roleplay_us_customs_officer` (5 min): roleplay AI strict
     officer.
- **Nota cultural:** énfasis en NO entrar en explicaciones largas, no
  joke, respuestas breves y honestas.

### 4.2 Module 2: Hotel & accommodation (3 bloques)

#### `tc_b1_hotel_checkin`

- **Título:** Check-in en hotel
- **Descripción:** Diálogo estándar de check-in: ID, credit card,
  preferencias de habitación.
- **Sub-skills:** `vocab_travel_practical`,
  `list_detail_capture`, `pron_word_stress`.
- **Prerequisites:** `tc_b1_immigration_customs`.
- **Estimated min:** 13

#### `tc_b1_hotel_amenities_questions`

- **Título:** Preguntar por amenities
- **Descripción:** WiFi, breakfast, gym, pool, late check-out.
  Patrón "Does the hotel have..." / "What time is...".
- **Sub-skills:** `pron_intonation_questions`,
  `vocab_travel_practical`,
  `gram_present_simple_continuous`.
- **Prerequisites:** `tc_b1_hotel_checkin`.
- **Estimated min:** 12

#### `tc_b1_hotel_basic_issues`

- **Título:** Problemas básicos en hotel
- **Descripción:** WiFi no funciona, AC ruidoso, room sucia, towel
  faltante. "There's a problem with...", "Could you send someone...".
- **Sub-skills:** `vocab_emotion_register` (intro),
  `flu_turn_taking`, `vocab_travel_practical`.
- **Prerequisites:** `tc_b1_hotel_amenities_questions`.
- **Estimated min:** 14

### 4.3 Module 3: Eating out (3 bloques)

#### `tc_b1_restaurant_ordering`

- **Título:** Ordenar en restaurante
- **Descripción:** Sentarse, ver menú, ordenar entrée + drink, hacer
  follow-ups.
- **Sub-skills:** `vocab_travel_practical`,
  `flu_turn_taking`, `pron_intonation_questions`.
- **Prerequisites:** ninguno (paralelo a Module 2).
- **Estimated min:** 13

#### `tc_b1_restaurant_dietary_basic`

- **Título:** Restricciones dietarias básicas
- **Descripción:** Vegetariano, alergias comunes (nuts, gluten,
  dairy). "I'm allergic to...", "Does this contain...".
- **Sub-skills:** `vocab_travel_practical`,
  `pron_final_consonants`, `list_detail_capture`.
- **Prerequisites:** `tc_b1_restaurant_ordering`.
- **Estimated min:** 13

#### `tc_b1_paying_tipping`

- **Título:** Pagar y propinas USA
- **Descripción:** Cultural especial: tipping en USA (15-20% es
  default, expected). Pedir cuenta, dividir cuenta, pagar con tarjeta.
- **Sub-skills:** `vocab_travel_practical`, `list_detail_capture`,
  `pron_intonation_questions`.
- **Prerequisites:** `tc_b1_restaurant_ordering`.
- **Estimated min:** 12

### 4.4 Module 4: Getting around city (5 bloques)

#### `tc_b1_asking_directions`

- **Título:** Pedir direcciones
- **Descripción:** "Excuse me, how do I get to...", entender
  respuestas con left/right/straight + landmarks.
- **Sub-skills:** `list_detail_capture`,
  `pron_intonation_questions`, `gram_prepositions_place`.
- **Prerequisites:** ninguno.
- **Estimated min:** 13

#### `tc_b1_public_transportation`

- **Título:** Transporte público
- **Descripción:** Subway, bus, train. Buy ticket, ask for stops,
  transfer instructions. NYC + USA general.
- **Sub-skills:** `vocab_travel_practical`,
  `list_detail_capture`, `gram_prepositions_time`.
- **Prerequisites:** `tc_b1_asking_directions`.
- **Estimated min:** 14

#### `tc_b1_taxi_rideshare`

- **Título:** Taxi y rideshare (Uber/Lyft)
- **Descripción:** Llamar taxi, dirigir al driver, problema con
  ride. Spanish "taxi" vs USA conventions.
- **Sub-skills:** `vocab_travel_practical`,
  `gram_prepositions_place`, `flu_turn_taking`.
- **Prerequisites:** `tc_b1_asking_directions`.
- **Estimated min:** 12

#### `tc_b1_shopping_basic`

- **Título:** Compras básicas
- **Descripción:** Pedir tallas, colores, probarse, devolver. Sales
  tax en USA (no incluido en precio).
- **Sub-skills:** `vocab_travel_practical`,
  `flu_turn_taking`, `pron_intonation_questions`.
- **Prerequisites:** ninguno.
- **Estimated min:** 13

#### `tc_b1_tourism_questions`

- **Título:** Preguntar sobre lugares turísticos
- **Descripción:** En tourist info, museum desk, attraction. Hours,
  prices, recommendations, queue advice.
- **Sub-skills:** `vocab_travel_practical`,
  `list_detail_capture`, `pron_intonation_questions`.
- **Prerequisites:** `tc_b1_asking_directions`.
- **Estimated min:** 14

### 4.5 Mastery criteria del nivel B1 (Travel Confident)

- ≥ 80% de los 15 bloques B1 completados.
- Score promedio ≥ 70 en sub-skills B1 cubiertas.
- WPM 80-100 en producción libre relacionada a viaje.
- Capaz de manejar simulación de "primer día en USA" (aeropuerto +
  hotel + restaurant) con AI sin breakdown notable.

---

## 5. B1+: Travel real-world situations (8 bloques)

### 5.1 Module 5: Travel problems & solutions (4 bloques)

#### `tc_b1plus_lost_delayed_luggage`

- **Título:** Equipaje perdido o demorado
- **Descripción:** Reportar en counter de aerolínea, seguimiento,
  language para insistir politely.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`, `gram_present_perfect`.
- **Prerequisites:** `tc_b1_hotel_basic_issues`.
- **Estimated min:** 15

#### `tc_b1plus_flight_cancellation`

- **Título:** Vuelo cancelado o reprogramado
- **Descripción:** Re-book, compensación, voucher hotel, language
  para escalar al supervisor.
- **Sub-skills:** `vocab_emotion_register`,
  `gram_conditionals_zero_first`,
  `flu_extended_speech`.
- **Prerequisites:** `tc_b1plus_lost_delayed_luggage`.
- **Estimated min:** 16

#### `tc_b1plus_hotel_complex_issues`

- **Título:** Problemas complejos en hotel
- **Descripción:** Cobro indebido, room downgrade, noise complaint
  serio, demanding refund cortésmente.
- **Sub-skills:** `vocab_emotion_register`,
  `gram_present_perfect`, `flu_extended_speech`.
- **Prerequisites:** `tc_b1_hotel_basic_issues`.
- **Estimated min:** 15

#### `tc_b1plus_medical_minor`

- **Título:** Situaciones médicas menores
- **Descripción:** Pharmacy, urgent care, describir síntomas.
  Vocabulario corporal básico + síntomas comunes.
- **Sub-skills:** `vocab_travel_practical` (medical extension),
  `vocab_emotion_register`, `list_detail_capture`.
- **Prerequisites:** `tc_b1_asking_directions`.
- **Estimated min:** 16

### 5.2 Module 6: Social travel (4 bloques)

#### `tc_b1plus_meeting_locals`

- **Título:** Conocer locales
- **Descripción:** Small talk con bartender, store clerk, fellow
  pedestrians. Topics seguros (weather, sports, "where are you from").
- **Sub-skills:** `flu_turn_taking`, `flu_speaking_pace`,
  `pron_intonation_questions`.
- **Prerequisites:** `tc_b1_tourism_questions`.
- **Estimated min:** 14

#### `tc_b1plus_meeting_other_travelers`

- **Título:** Otros viajeros (hostel, tour)
- **Descripción:** Backpacker conversation, "Where have you been?",
  "Where are you going next?". Travel comparison.
- **Sub-skills:** `gram_present_perfect`,
  `flu_extended_speech`,
  `flu_discourse_connectors`.
- **Prerequisites:** `tc_b1plus_meeting_locals`.
- **Estimated min:** 14

#### `tc_b1plus_cultural_etiquette_basic`

- **Título:** Etiqueta cultural básica
- **Descripción:** USA-specific: tipping, smile to strangers,
  personal space, casual "How are you?" como saludo.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_thinking_in_english` (intro), `pron_intonation_questions`.
- **Prerequisites:** `tc_b1plus_meeting_locals`.
- **Estimated min:** 13

#### `tc_b1plus_short_travel_story`

- **Título:** Contar una experiencia de viaje
- **Descripción:** Capstone B1+ del track: 2-min historia de viaje
  con introducción + middle + ending. Storytelling estructurado.
- **Sub-skills:** consolida sub-skills B1+ travel.
- **Prerequisites:** `tc_b1plus_meeting_other_travelers`,
  `tc_b1plus_flight_cancellation`.
- **Estimated min:** 15
- **Asset sequence:**
  1. `listening_mc_travel_anecdotes` (4 min): 5 historias buenas vs
     malas.
  2. `vocab_drill_storytelling_phrases` (2 min): "It all started
     when...", "What I didn't know was...", "In the end...".
  3. `free_response_your_travel_story_v1` (4 min): primera versión.
  4. `free_response_your_travel_story_v2` (3 min): refinada.
  5. `roleplay_share_story_with_friend` (2 min): contar a amigo AI.

### 5.3 Mastery criteria del nivel B1+ (Travel Confident)

- ≥ 80% de los 8 bloques B1+ completados.
- Score promedio ≥ 75 en sub-skills cubiertas.
- WPM 100-120 en producción libre travel-related.
- Capaz de manejar problema de viaje (vuelo cancelado / luggage
  perdido) en simulación 10 min con AI sin breakdown.

---

## 6. B2: Travel deep + business travel (10 bloques)

### 6.1 Module 7: Business travel (5 bloques)

#### `tc_b2_business_trip_arrival`

- **Título:** Business trip: llegada y check-in profesional
- **Descripción:** Hotel check-in con corporate rate, gym y
  business center, daily breakfast voucher.
- **Sub-skills:** `vocab_business_general`,
  `vocab_travel_practical`, `flu_extended_speech`.
- **Prerequisites:** `tc_b1_hotel_checkin`.
- **Estimated min:** 15

#### `tc_b2_business_trip_meetings`

- **Título:** Reuniones durante business trip
- **Descripción:** Meeting room booking, intro a clientes locales,
  agenda explanation, post-meeting wrap-up.
- **Sub-skills:** `flu_extended_speech`,
  `vocab_business_general`, `flu_discourse_connectors`.
- **Prerequisites:** `tc_b2_business_trip_arrival`. Cross-track:
  asume haber visto `jr_b2_meeting_present_status` o equivalente.
- **Estimated min:** 16

#### `tc_b2_business_trip_dinner`

- **Título:** Cena de negocios
- **Descripción:** Restaurant con clientes, small talk apropiado,
  topics a evitar (politics, religion), levantar copa, pagar la cuenta.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_turn_taking`, `flu_extended_speech`.
- **Prerequisites:** `tc_b2_business_trip_meetings`.
- **Estimated min:** 16

#### `tc_b2_conference_networking`

- **Título:** Networking en conferencia
- **Descripción:** Coffee break, panel Q&A, after-hours mixer.
  Self-intro repetido con variaciones, asking smart questions.
- **Sub-skills:** `flu_extended_speech`,
  `flu_thinking_in_english`, `vocab_business_general`.
- **Prerequisites:** `tc_b2_business_trip_meetings`.
- **Estimated min:** 17

#### `tc_b2_business_travel_logistics`

- **Título:** Logística de business travel
- **Descripción:** Cambiar reservas, expense reporting language,
  preferred airlines/hotels, travel agent calls.
- **Sub-skills:** `vocab_business_general`,
  `flu_discourse_connectors`,
  `gram_conditionals_zero_first`.
- **Prerequisites:** `tc_b2_business_trip_arrival`.
- **Estimated min:** 14

### 6.2 Module 8: Cultural travel (5 bloques)

#### `tc_b2_negotiating_prices_disputes`

- **Título:** Negociar precios y disputar cargos
- **Descripción:** Markets en destinos donde haggling es expected,
  disputar cargos en hotel/restaurant.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `tc_b1plus_hotel_complex_issues`.
- **Estimated min:** 16

#### `tc_b2_emergency_situations`

- **Título:** Emergencias (robo, médico serio)
- **Descripción:** Llamar 911, reportar en policía, ER (emergency
  room) language. Tono claro, concise, urgente.
- **Sub-skills:** `vocab_emotion_register`,
  `list_detail_capture`,
  `flu_self_correction_control`.
- **Prerequisites:** `tc_b1plus_medical_minor`.
- **Estimated min:** 17

#### `tc_b2_cultural_deep_topics`

- **Título:** Hablar de cultura con detalle
- **Descripción:** Discutir tradiciones, comida, política
  general, history del lugar visitado vs casa. Articular diferencias
  diplomáticamente.
- **Sub-skills:** `flu_extended_speech`,
  `vocab_emotion_register`,
  `flu_thinking_in_english`.
- **Prerequisites:** `tc_b1plus_cultural_etiquette_basic`.
- **Estimated min:** 16

#### `tc_b2_travel_disagreements`

- **Título:** Desacuerdos al viajar en grupo
- **Descripción:** Travel companion fights ("we're going to be
  late", "I want to do X, you want Y"), llegar a compromise.
- **Sub-skills:** `vocab_emotion_register`,
  `flu_extended_speech`,
  `gram_conditionals_second_third`.
- **Prerequisites:** `tc_b1plus_meeting_other_travelers`.
- **Estimated min:** 15

#### `tc_b2_long_travel_anecdote`

- **Título:** Anécdota larga de viaje
- **Descripción:** 5-min story con build-up, complication, climax,
  resolution. Storytelling pulido.
- **Sub-skills:** `flu_extended_speech`,
  `flu_discourse_connectors`,
  `gram_past_perfect`.
- **Prerequisites:** `tc_b1plus_short_travel_story`.
- **Estimated min:** 16

### 6.3 Mastery criteria del nivel B2 (Travel Confident)

- ≥ 80% de los 10 bloques B2 completados.
- Score promedio ≥ 75 en sub-skills cubiertas.
- WPM 110-130.
- Capaz de manejar business trip simulado (3 días, multiple
  scenarios) con AI sin breakdown.
- Capaz de contar travel story de 5 min sin pausas largas.

---

## 7. B2+: Cross-cultural + storytelling (6 bloques)

### 7.1 Module 9: Travel storytelling avanzado (3 bloques)

#### `tc_b2plus_travel_narrative_extended`

- **Título:** Narrativa extendida (15 min historia)
- **Descripción:** Capstone storytelling: 15 min story con multiple
  characters, settings, themes. Inspirado en NPR / The Moth.
- **Sub-skills:** `flu_extended_speech`,
  `flu_discourse_connectors`,
  `flu_thinking_in_english`.
- **Prerequisites:** `tc_b2_long_travel_anecdote`.
- **Estimated min:** 18

#### `tc_b2plus_travel_writing_review`

- **Título:** Review / narrativa escrita de viaje
- **Descripción:** Tripadvisor / blog review escrito. Estructura,
  detail, tone.
- **Sub-skills:** `flu_discourse_connectors`,
  `gram_relative_clauses`, `vocab_emotion_register`.
- **Prerequisites:** `tc_b2_long_travel_anecdote`.
- **Estimated min:** 16

#### `tc_b2plus_business_travel_executive`

- **Título:** Business travel ejecutivo
- **Descripción:** Senior business trip: dinner con C-suite, public
  speaking en conferencia internacional, gala events.
- **Sub-skills:** `flu_extended_speech`,
  `vocab_business_general`,
  `vocab_emotion_register`.
- **Prerequisites:** `tc_b2_business_trip_dinner`,
  `tc_b2_conference_networking`.
- **Estimated min:** 17

### 7.2 Module 10: Cross-cultural mastery (3 bloques)

#### `tc_b2plus_cross_cultural_nuance`

- **Título:** Matices culturales sutiles
- **Descripción:** USA vs UK vs AU English subtle differences,
  East-Asia vs West-Europe expectations, faux pas avoidance.
- **Sub-skills:** `list_distinguishing_accents`,
  `vocab_emotion_register`,
  `flu_thinking_in_english`.
- **Prerequisites:** `tc_b2_cultural_deep_topics`.
- **Estimated min:** 17

#### `tc_b2plus_high_stakes_situation`

- **Título:** Situaciones high-stakes (deportación, hospital serio)
- **Descripción:** Situaciones legales/médicas serias requiriendo
  precisión absoluta. Cuándo pedir intérprete oficial. Lawyer up
  language.
- **Sub-skills:** `vocab_emotion_register`,
  `list_detail_capture`, `flu_self_correction_control`.
- **Prerequisites:** `tc_b2_emergency_situations`.
- **Estimated min:** 18

#### `tc_b2plus_diplomatic_intercultural`

- **Título:** Diplomacia intercultural (capstone B2+ track)
- **Descripción:** Manejo de situaciones cross-cultural con tact:
  responder ofensa cultural sin escalar, navegar tensiones
  multiculturales en grupo.
- **Sub-skills:** consolida sub-skills B2 + travel.
- **Prerequisites:** `tc_b2plus_cross_cultural_nuance`,
  `tc_b2plus_travel_narrative_extended`.
- **Estimated min:** 16

### 7.3 Mastery criteria del nivel B2+ (Travel Confident)

- ≥ 80% de los 6 bloques B2+ completados.
- Score promedio ≥ 80 en sub-skills cubiertas.
- WPM 120-140.
- Capaz de mantener narrativa de 15 min sobre experiencia de viaje
  sin pausas largas.
- Capaz de manejar high-stakes simulation (medical, legal) con
  precisión.

---

## 8. Prerequisites graph y orden recomendado

### 8.1 Visualización

```
B1 entry: tc_b1_pretrip_planning (raíz)
   │
   ├── tc_b1_airport_checkin_security
   │      ├── tc_b1_inflight_basics
   │      └── tc_b1_immigration_customs → tc_b1_hotel_checkin
   │             │
   │             ├── tc_b1_hotel_amenities_questions → tc_b1_hotel_basic_issues
   │             │
   │             └── tc_b1_restaurant_ordering
   │                    ├── tc_b1_restaurant_dietary_basic
   │                    └── tc_b1_paying_tipping
   │
   ├── tc_b1_asking_directions
   │      ├── tc_b1_public_transportation
   │      ├── tc_b1_taxi_rideshare
   │      └── tc_b1_tourism_questions → tc_b1plus_meeting_locals (B1+)
   │
   └── tc_b1_shopping_basic (rama paralela)
          │
          ▼
      B1+ entrada: tc_b1plus_lost_delayed_luggage
          │
          ├── tc_b1plus_flight_cancellation
          ├── tc_b1plus_hotel_complex_issues
          └── tc_b1plus_medical_minor
                 │
                 ├── tc_b1plus_meeting_locals
                 │     ├── tc_b1plus_meeting_other_travelers
                 │     └── tc_b1plus_cultural_etiquette_basic
                 │
                 └── tc_b1plus_short_travel_story (capstone B1+)
                        │
                        ▼
                  B2 entrada: tc_b2_business_trip_arrival
                        │
                        ├── Module 7 (business travel)
                        └── Module 8 (cultural travel)
                               │
                               ▼
                         B2+ entrada: tc_b2plus_travel_narrative_extended
                               │
                               ▼
                         tc_b2plus_diplomatic_intercultural (capstone)
```

### 8.2 Orden recomendado (path principal de "primer viaje")

Si user nunca viajó a destino angloparlante, este es el orden
tipo "trip simulation":

1. `tc_b1_pretrip_planning`
2. `tc_b1_airport_checkin_security`
3. `tc_b1_inflight_basics`
4. `tc_b1_immigration_customs`
5. `tc_b1_hotel_checkin`
6. `tc_b1_hotel_amenities_questions`
7. `tc_b1_restaurant_ordering`
8. `tc_b1_paying_tipping`
9. `tc_b1_asking_directions`
10. `tc_b1_public_transportation`
11. `tc_b1_tourism_questions`
12. `tc_b1plus_meeting_locals`
13. `tc_b1plus_short_travel_story` (capstone)
14. ... continúa con B2 según destino y necesidades.

### 8.3 Variantes / fork points

#### 8.3.1 "Tourist" vs "Business traveler" fork

A nivel B2, user elige (basado en `student_profile.primary_goals`):
- `'travel'` → Module 8 (Cultural travel) primero.
- `'remote_work'` o `'business_communication'` → Module 7 (Business
  travel) primero.

Ambos completan los 10 bloques al final.

#### 8.3.2 Variantes por destino (post-MVP)

Algunos bloques pueden tener variantes futuras:

| Block ID | Variantes potenciales (post-MVP) |
|----------|-----------------------------------|
| `tc_b1_immigration_customs` | `_us`, `_uk`, `_canada`, `_australia` |
| `tc_b1_paying_tipping` | `_us`, `_uk`, `_no_tipping_culture` |

MVP solo cubre USA. Post-MVP expansión según demanda.

---

## 9. Mastery criteria por bloque y por nivel

### 9.1 Por bloque

Default rule (idéntica a Job Ready, ver
`track-job-ready-blocks.md` §9.1):
- Todos los assets obligatorios completados con score ≥ 60.
- Score promedio del bloque ≥ 70.

Excepción capstones:
- `tc_b1plus_short_travel_story`: score promedio ≥ 75.
- `tc_b2plus_travel_narrative_extended`: score promedio ≥ 75.
- `tc_b2plus_diplomatic_intercultural`: score promedio ≥ 80.

### 9.2 Por nivel

(Coordinar con `curriculum-by-cefr.md` §5.9-§8.8.)

Triggers para transición Travel Confident X → X+1:
- ≥ 80% de los bloques del nivel completados.
- Score promedio en sub-skills target del nivel ≥ threshold.
- Métrica especial Travel: completion del "trip simulation"
  end-to-end del nivel (B1: aeropuerto → hotel → restaurant; B1+:
  trip + problema; B2: business trip).

### 9.3 Tested-out por bloque

Default `testable_out: true` excepto capstones (deben hacerse).

---

## 10. Assets compartidos cross-block (reuse)

### 10.1 Atómicos cross-block esperados (high reuse)

| Atomic ID | Tipo | Reuse esperado | Bloques |
|-----------|------|---------------:|---------|
| `audio_airport_announcement_generic` | audio TTS | 8+ | Module 1 |
| `audio_us_customs_officer_voice` | audio TTS | 6+ | immigration + emergency |
| `audio_hotel_receptionist_voice` | audio TTS | 10+ | hotel blocks |
| `audio_waiter_voice_neutral` | audio TTS | 8+ | restaurant blocks |
| `audio_phone_call_quality_reduced` | audio variant | 12+ | various |
| `image_airport_terminal` | image AI | 6+ | airport blocks |
| `image_hotel_lobby` | image AI | 6+ | hotel blocks |
| `image_restaurant_interior` | image AI | 5+ | restaurant blocks |
| `image_subway_map_nyc` | image AI | 4+ | transport blocks |

### 10.2 Estimación de atomics totales

A 4 atomics por bloque y reuse 2x:

```
39 bloques × 4 atomics / 2 reuse = ~78 atomics únicos
```

Aproximadamente 25% del MVP atomic library (estimado total
~280-350).

### 10.3 Cross-track reuse con Job Ready

Algunos atomics se reusan entre Job Ready y Travel Confident
(business meeting context aparece en ambos):

| Atomic ID | Job Ready blocks | Travel Confident blocks |
|-----------|------------------|-------------------------|
| `audio_business_meeting_intro_phrases` | jr_b1_describe_role_company | tc_b2_business_trip_meetings |
| `audio_dinner_smalltalk` | (—) | tc_b2_business_trip_dinner |
| `audio_filler_let_me_think` | varios | varios |

Esto reduce más el atomic count efectivo (sumando todos los tracks).

---

## 11. Decisiones cerradas

### 11.1 39 bloques (upper bound del MVP) ✓

**Razón:** consistente con §15.6 de `curriculum-by-cefr.md` ("upper
bound como target"). Lower bound era 30, upper 38 — usamos 39 con 1
bloque extra para no rotar abajo si se ajusta.

### 11.2 USA-first para variantes culturales en MVP ✓

**Razón:** target market initial de viajes desde LatAm es USA (data
de turismo/business travel). Otros destinos (UK, AU, Canada,
non-anglophone) son post-MVP variants.

### 11.3 "Trip simulation" como principio de diseño ✓

**Razón:** retención mayor cuando user ve aplicación inmediata vs
"learning vocabulary lists". User puede simular un viaje real
end-to-end en el track.

### 11.4 Storytelling como capstones (B1+ y B2+): SÍ ✓

**Razón:** travel storytelling es el outcome aspiracional del
track. Capstones obligan a integrar todo lo aprendido en una pieza
demostrable.

### 11.5 Business travel como módulo separado (no track distinto) ✓

**Razón:** business travel es subset de Travel Confident (el viaje
es el contexto, business overlay es la diferencia). No amerita
track propio en MVP. Reconsiderar en Fase 2-3 si demand alta.

### 11.6 NO bloque dedicado a "writing travel reviews" en B1-B1+ ✓

**Razón:** B1-B1+ writing es muy básico. Travel writing (incluido
review) está en B2+ (`tc_b2plus_travel_writing_review`) donde el
user tiene capacidad de output escrito relevante.

### 11.7 Listening con audio "ruidoso" como diferenciador ✓

**Razón:** travel implica audio degradado real (aeropuertos, calle,
phones). Sub-skill `list_phone_distorted_audio` y
`list_distinguishing_accents` aparecen más en este track que en Job
Ready. Atomics deben tener variants con noise.

### 11.8 NO variantes por destino en MVP ✓

**Razón:** complejidad combinatorial. USA es el target principal.
Post-MVP: variantes UK, AU, Canada según data de uso.

---

## 12. Plan de implementación

### 12.1 Sprint 1-2 (semanas 1-3): B1 (15 bloques)

- Crear los 15 bloques B1 con composiciones de assets.
- ~60 assets únicos.
- ~30 atomics únicos (con reuse).
- Foco listening "ruidoso": atomics audio con variants ambient
  noise.

### 12.2 Sprint 3 (semana 4): B1+ (8 bloques)

- 8 bloques B1+.
- ~32 assets únicos (~16 atomics nuevos).
- Capstone storytelling: review extra.

### 12.3 Sprint 4 (semana 5-6): B2 (10 bloques)

- 10 bloques B2.
- ~40 assets únicos (~20 atomics nuevos).
- Business travel module reusa atomics de Job Ready B2 meetings.

### 12.4 Sprint 5 (semana 7): B2+ (6 bloques)

- 6 bloques B2+.
- ~24 assets únicos (~12 atomics nuevos).
- High-stakes y storytelling capstones requieren validación
  pedagógica extra.

### 12.5 Total estimado

- **39 bloques** generados.
- **~156 assets** únicos.
- **~78 atomics** únicos (con reuse 2x).
- **7 semanas** de creación + validación.

A 30 min de generación + revisión por asset: ~78 horas de content
team (aprox. la mitad del esfuerzo de Job Ready).

### 12.6 Validación pedagógica obligatoria

Cada bloque pasa por validation idéntica a Job Ready (ver
`track-job-ready-blocks.md` §12.6).

Validación cultural extra para bloques con USA-specific content
(immigration, tipping, etiquette): revisor debe haber vivido en
USA o ser nativo.

---

## 13. Métricas de éxito

### 13.1 Por bloque

- Completion rate ≥ 70% (un poco más bajo que Job Ready: tracking
  shows users de Travel hacen bursts pre-trip, no consistencia).
- Average score ≥ 70.
- Time matchea ± 30%.

### 13.2 Agregadas track Travel Confident

- % users en Travel que completan B1 → 60% target (más alto que
  Job Ready B1 porque B1 Travel es más simple en outcome).
- % que llegan a B2+ → 5% target (sub-set pequeño).
- Trip success rate (post-MVP, requiere follow-up survey): "¿pudiste
  manejar tu viaje en inglés sin asistencia?" → 80% sí target en
  users que completaron B1.

### 13.3 Métricas pre-trip burst

Travel users tienen patrón distinto a Job Ready:
- Practice burst en 2-4 semanas pre-trip.
- Drop sustancial post-trip.
- Re-engage si planean próximo viaje.

Roadmap debería detectar este patrón (`student_profiles.observed_behavior`
puede capturar via `expected_trip_date` opcional) y ajustar
sequencing.

---

## 14. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`curriculum-by-cefr.md`](curriculum-by-cefr.md) §11 | Matriz tracks × CEFR (Travel Confident row es la cubierta acá). |
| [`curriculum-by-cefr.md`](curriculum-by-cefr.md) §5-§8 | Plan B1, B1+, B2, B2+ (sub-skills, vocab target, mastery). |
| [`pedagogical-system.md`](pedagogical-system.md) §2.4 | Catálogo de 50 sub-skills. |
| [`pedagogical-system.md`](pedagogical-system.md) §4.5 | Test-out criteria por bloque. |
| [`content-creation-system.md`](content-creation-system.md) §6 | Estructura de `learning_block`. |
| [`track-job-ready-blocks.md`](track-job-ready-blocks.md) | Track paralelo (Job Ready); reuse atomics. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) §5 | Generación de roadmap usa este desglose para sequenciar. |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §3.1 | `primary_goals` determina si Travel es track recomendado. |
| [`motivation-and-achievements.md`](motivation-and-achievements.md) §6.2.5 | Logros de Roadmap (track-related). |

---

*Documento vivo. Actualizar cuando se rebalancee el track basado en
data de producción, se agreguen variantes por destino post-MVP, o se
descubran gaps en cobertura cultural.*

# Curriculum by CEFR Level

> Plan de estudios completo organizado por nivel CEFR (A1 → C2). Define
> objetivos pedagógicos, vocabulario target, grammar focus, sub-skills
> cubiertas, blocks estimados por track, duración esperada y criterios
> de mastery para transición al siguiente nivel.

**Estado:** Diseño v1.0
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador (al crear contenido) +
humano (para validación pedagógica).
**Alcance:** Sistema completo. **MVP cubre B1, B1+, B2, B2+. A2, C1
son scope de expansión Fase 2-3. A1, C2 son scope expansión Fase 4+.**

---

## 0. Cómo leer este documento

- §1 establece **filosofía** del curriculum.
- §2 cubre **marco CEFR aplicado al producto**.
- §3-§10 detallan el **plan por nivel** (uno por sección).
- §11 es la **matriz tracks × CEFR**.
- §12 cubre **transiciones entre niveles**.
- §13 cubre **bloqueos y prerequisites cross-level**.
- §14 cubre **estimaciones de duración consolidadas**.
- §15 cubre **decisiones cerradas**.

---

## 1. Filosofía del curriculum

### 1.1 Principios

**Nivel CEFR como marco autoritativo.** Usamos CEFR (Common European
Framework of Reference for Languages) por:
- Estándar internacional reconocido.
- Permite comparación con tests externos (TOEFL, IELTS, Duolingo
  English Test).
- Mapping claro a empleabilidad (B2+ es target laboral típico).

**Sub-niveles `+` para granularidad.** B1+ y B2+ son intermedios entre
estándares CEFR. El producto los usa porque:
- Permiten transiciones más suaves.
- Reflejan realidad de aprendizaje (no se salta de B1 a B2 en un día).
- Permiten roadmaps más precisos.

**Track-orthogonal.** El curriculum por CEFR es un **framework
transversal**. Cada track (Job Ready, Travel, Daily Conversation)
selecciona y secuencia bloques de los niveles CEFR según sus
objetivos.

**Hispanohablante latinoamericano como audiencia primaria.** Errores
típicos del hispanohablante (ver `pedagogical-system.md` §3.1) están
priorizados en cada nivel.

### 1.2 Lo que NO es este documento

- **No es el contenido literal de los assets.** Para eso ver
  `content-creation-system.md` y la biblioteca de
  `media_atomics`/`learning_assets`.
- **No es un plan operativo de creación.** Para timeline ver
  `ai-roadmap-system.md` §13.
- **No es una rúbrica de scoring.** Para eso ver
  `pedagogical-system.md` §3.

---

## 2. Marco CEFR aplicado al producto

### 2.1 Niveles soportados

| Nivel CEFR | Nombre | Status en producto |
|-----------|--------|--------------------|
| A1 | Acceso | Scope expansión Fase 4+ |
| A2 | Plataforma | Scope expansión Fase 2-3 |
| **B1** | **Umbral** | **MVP** |
| **B1+** | **Umbral consolidado** | **MVP** |
| **B2** | **Avanzado** | **MVP** |
| **B2+** | **Avanzado consolidado** | **MVP** |
| C1 | Dominio Operativo Eficaz | Scope expansión Fase 2-3 |
| C2 | Maestría | Scope expansión Fase 4+ |

**Razón del foco MVP B1-B2+:**
- Mercado primario: profesionales latinos buscando trabajo remoto / job
  internacional. Requirement típico = B2.
- Mercado primario: usuarios ya con base (no absolute beginners).
- Onboarding asume self-perceived B1+ como minimum entry point.
- Users A1/A2 reales se identifican vía mini-test y se les ofrece path
  preparatorio (en MVP: contenido limitado; Fase 2-3: full coverage).
- Users C1+ no son target principal (ya hablan bien); MVP les ofrece
  refinamiento limitado.

### 2.2 Sub-niveles `+`

Sub-niveles intermedios:
- **B1+** = consolidación de B1, transición a B2.
- **B2+** = consolidación de B2, transición a C1.

NO usamos A2+, C1+, etc. Solo donde la diferencia de granularidad
agrega valor pedagógico (zona crítica B1-B2 donde la mayoría del
producto vive).

### 2.3 Vocabulario target por nivel

Word count acumulativo (vocabulario activo + reconocimiento):

| Nivel | Vocab acumulativo | Vocab nuevo del nivel |
|-------|------------------:|---------------------:|
| A1 | ~500 | ~500 |
| A2 | ~1.500 | ~1.000 |
| B1 | ~2.500 | ~1.000 |
| B1+ | ~3.000 | ~500 |
| B2 | ~4.000 | ~1.000 |
| B2+ | ~5.000 | ~1.000 |
| C1 | ~6.500 | ~1.500 |
| C2 | ~10.000+ | ~3.500+ |

Source de listas CEFR de vocabulario: CEFR-J + extensiones propias del
producto (tech, business USA, travel).

### 2.4 Mapping a sub-skills del catálogo

(Ver `pedagogical-system.md` §2.4 para catálogo de 50 sub-skills core.)

| Nivel | Sub-skills introducidas (catálogo actual) |
|-------|-------------------------------------------|
| A1 | (a expandir Fase 4+: pron_alphabet, gram_to_be_basic, vocab_top_500_a1, etc.) |
| A2 | 14 sub-skills (pron_th_voiceless, pron_v_vs_b, pron_short_long_i, pron_ae, pron_final_consonants, gram_present_simple_continuous, gram_past_simple_continuous, gram_prepositions_time, gram_prepositions_place, vocab_travel_practical, flu_speaking_pace, etc.) |
| B1 | 18 sub-skills (pron_zh, pron_schwa, pron_word_stress, pron_intonation_questions, pron_short_long_u, gram_present_perfect, gram_future_forms, gram_conditionals_zero_first, gram_articles, gram_phrasal_verbs_basic, gram_passive_voice, vocab_high_frequency_b1, vocab_collocations_general, vocab_business_general, flu_pause_management, flu_filler_reduction, flu_discourse_connectors, flu_turn_taking, list_detail_capture) |
| B2 | 16 sub-skills (pron_sentence_stress, pron_linking, gram_past_perfect, gram_conditionals_second_third, gram_relative_clauses, gram_reported_speech, vocab_idioms_common, vocab_tech_industry, vocab_emotion_register, flu_self_correction_control, flu_extended_speech, flu_thinking_in_english, list_native_speed_general, list_distinguishing_accents, list_inference, list_phone_distorted_audio) |
| C1 | 2 sub-skills (vocab_academic_discourse, list_overlapping_speech) |

Expansión a 200+ sub-skills en Fase 2 (ver
`pedagogical-system.md` §14.2) cubrirá mejor A1, A2, C1, C2.

---

## 3. Plan A1 — Acceso

> **Status MVP:** scope expansión Fase 4+. Esta sección es plan
> aspiracional.

### 3.1 Objetivos pedagógicos

Capacidades target al completar A1:
- Presentarse: nombre, edad, profesión, país.
- Saludar y despedirse en contextos básicos.
- Hacer preguntas simples (información personal, lugares, tiempo).
- Entender hablantes lentos sobre temas familiares concretos.
- Identificar palabras y frases familiares en textos cortos.

### 3.2 Vocabulario target

- **Count:** ~500 palabras.
- **Temáticas:** información personal, números 1-100, días, meses,
  colores, familia, comida básica, ropa, partes del cuerpo, lugares
  comunes (casa, escuela, trabajo), acciones cotidianas básicas.

### 3.3 Grammar focus

- Verbo `to be` en presente.
- Pronombres personales y posesivos básicos (my, your, his/her).
- Artículos básicos: a/an/the (uso indicativo).
- Plurales regulares.
- Present simple básico (yo, tú, él/ella).
- Question forms: what, where, who, how (básico).
- Numerales cardinales y ordinales.
- `there is / there are`.

### 3.4 Pronunciation focus

- Alfabeto inglés.
- Sonidos de vocales largas vs cortas (intro a `/iː/ vs /ɪ/`).
- Stress básico en palabras de 2-3 sílabas.
- Question intonation rising.

### 3.5 Cultural / pragmatic focus

- Greetings formales vs informales (Hi vs Hello vs Good morning).
- Self-introduction estándar USA.
- "How are you?" como saludo no-pregunta.

### 3.6 Sub-skills (a expandir Fase 4+)

Sub-skills tentativos para añadir al catálogo:
- `pron_alphabet`
- `gram_to_be_basic`
- `gram_articles_basic`
- `gram_present_simple_basic`
- `gram_question_forms_wh`
- `vocab_top_500_a1`
- `vocab_numbers_a1`
- `vocab_family_a1`
- `flu_speaking_pace_a1` (wpm 40-60 target)

### 3.7 Blocks por track

| Track | Blocks A1 estimados | Notas |
|-------|--------------------:|-------|
| Job Ready | 0 | A1 no toca contexto laboral |
| Travel Confident | 8-10 | Saludos, números, lugares básicos |
| Daily Conversation | 10-12 | Foundation de conversaciones |
| **Total A1** | **~20** | |

### 3.8 Duración estimada

A 15 min/día: **6-8 semanas**.
A 30 min/día: **3-4 semanas**.

### 3.9 Mastery criteria para pasar a A2

- Score promedio ≥ 70 en sub-skills A1 cubiertos.
- ≥ 80% de blocks A1 completados.
- Capacidad de mantener intercambio conversacional simple (3-4 turnos)
  sobre información personal.

---

## 4. Plan A2 — Plataforma

> **Status MVP:** scope expansión Fase 2-3.

### 4.1 Objetivos pedagógicos

Capacidades target:
- Describir aspectos sencillos de su entorno.
- Hablar de rutinas diarias y hábitos.
- Comunicarse en contextos predecibles (compras, transporte,
  restaurantes).
- Comprender frases y expresiones de uso frecuente.
- Escribir mensajes cortos sobre temas familiares.

### 4.2 Vocabulario target

- **Count acumulativo:** ~1.500 palabras.
- **Nuevas:** ~1.000 palabras.
- **Temáticas:** rutinas, work básico (oficina, profesión,
  responsabilidades), shopping, restaurants, transportation,
  directions, weather, hobbies, opiniones simples.

### 4.3 Grammar focus

- Past simple regular e irregular básico.
- Present continuous (acción en curso).
- Future con `going to` y `will` (intro).
- Comparatives y superlatives.
- Modal verbs básicos: can, can't, must, should, have to.
- Quantifiers: some, any, much, many, a lot of.
- Adverbs of frequency (always, sometimes, never).
- Prepositions of time: in, on, at.
- Prepositions of place: in, on, at, under, between, next to.

### 4.4 Pronunciation focus

- /θ/ (think, three) vs /s/ (interferencia hispana).
- /ð/ (this, mother) vs /d/.
- /v/ vs /b/ (very vs berry — clásico hispano).
- /ʃ/ (she, sure).
- /æ/ (cat, hand) vs /a/ española.
- /ɪ/ (ship) vs /iː/ (sheep).
- Final consonants: -ed, -s, -t.
- Stress en palabras de 2-3 sílabas.

### 4.5 Cultural / pragmatic focus

- Polite requests: "Could you...", "Would you mind...".
- Small talk básico (weather, weekend).
- Restaurant ordering en USA.
- Tipping culture.
- Time expressions (US-style: 5 PM vs 17:00).

### 4.6 Sub-skills cubiertos (del catálogo actual)

- `pron_th_voiceless`, `pron_th_voiced`, `pron_v_vs_b`, `pron_sh`,
  `pron_short_long_i`, `pron_ae`, `pron_final_consonants`.
- `flu_speaking_pace`.
- `gram_present_simple_continuous`, `gram_past_simple_continuous`,
  `gram_prepositions_time`, `gram_prepositions_place`.
- `vocab_travel_practical`.
- `list_detail_capture` (intro).

### 4.7 Blocks por track

| Track | Blocks A2 estimados |
|-------|--------------------:|
| Job Ready | 5-8 (intro a contexto profesional básico) |
| Travel Confident | 15-18 |
| Daily Conversation | 18-22 |
| **Total A2** | **~40-48** |

### 4.8 Duración estimada

A 15 min/día: **8-10 semanas**.
A 30 min/día: **4-5 semanas**.

### 4.9 Mastery criteria para pasar a B1

- Score promedio ≥ 70 en sub-skills A2 cubiertos.
- ≥ 80% de blocks A2 completados.
- WPM target: 60-80 en speech espontáneo.
- < 8 pausas/min en producción libre.

---

## 5. Plan B1 — Umbral

> **Status MVP:** ✅ COVERED. Nivel de entrada típico para users del
> producto.

### 5.1 Objetivos pedagógicos

Capacidades target:
- Entablar conversaciones cotidianas con confianza.
- Expresar opiniones y razones simples.
- Describir experiencias, eventos, sueños y aspiraciones.
- Manejar situaciones imprevistas en viajes.
- Comprender textos sobre temas familiares.
- Escribir cartas/emails personales describiendo experiencias.

### 5.2 Vocabulario target

- **Count acumulativo:** ~2.500 palabras.
- **Nuevas:** ~1.000 palabras.
- **Temáticas:** work tasks (más detalle), opiniones argumentadas,
  experiences, plans, basic professional contexts (meetings, emails),
  health, education, technology básica.

### 5.3 Grammar focus

- Present perfect simple (uses: experience, recent past, ongoing).
- Past simple vs present perfect distinction.
- Past continuous para acciones interrumpidas.
- Future forms expansion: will vs going to vs present continuous para
  futuro.
- Conditionals zero y first.
- Articles deepening (a/an/the — uso avanzado, especialmente difícil
  para hispanos).
- Phrasal verbs básicos (get up, look for, give up, run into).
- Passive voice en presente y pasado simple.
- Modals expansion: could, should, might.
- Reported speech intro.

### 5.4 Pronunciation focus

- /ʒ/ (measure, vision).
- Schwa /ə/ en sílabas átonas (about, photograph).
- Word stress en palabras multi-silábicas.
- Question intonation rising vs falling.
- Vowel length distinctions consolidación.

### 5.5 Cultural / pragmatic focus

- Email register casual vs formal.
- Phone conversations básicas.
- Disagreeing politely.
- Small talk extendido.
- US-specific cultural cues: weekend plans, hobbies questions.

### 5.6 Sub-skills cubiertos

- `pron_zh`, `pron_schwa`, `pron_word_stress`,
  `pron_intonation_questions`, `pron_short_long_u`.
- `flu_pause_management`, `flu_filler_reduction`,
  `flu_discourse_connectors`, `flu_turn_taking`.
- `gram_present_perfect`, `gram_future_forms`,
  `gram_conditionals_zero_first`, `gram_articles`,
  `gram_phrasal_verbs_basic`, `gram_passive_voice`.
- `vocab_high_frequency_b1`, `vocab_collocations_general`,
  `vocab_business_general`.
- `list_detail_capture`.

### 5.7 Blocks por track

| Track | Blocks B1 estimados |
|-------|--------------------:|
| Job Ready | 18-22 |
| Travel Confident | 12-15 |
| Daily Conversation | 18-22 |
| **Total B1** | **~50-60** |

### 5.8 Duración estimada

A 15 min/día: **10-12 semanas**.
A 30 min/día: **5-6 semanas**.

### 5.9 Mastery criteria para pasar a B1+

- Score promedio ≥ 70 en sub-skills B1 cubiertos.
- ≥ 80% de blocks B1 completados.
- WPM target: 80-100.
- < 6 fillers/min.
- Capaz de discutir temas familiares por 2+ minutos sin pausas largas.

---

## 6. Plan B1+ — Umbral consolidado

> **Status MVP:** ✅ COVERED.

### 6.1 Objetivos pedagógicos

Transición B1 → B2. Capacidades target:
- Mantener conversaciones de 5+ minutos sobre temas familiares con
  fluidez.
- Argumentar opiniones con conectores discursivos.
- Manejar la mayoría de situaciones laborales cotidianas (no
  reuniones técnicas complejas).
- Escribir emails profesionales claros.

### 6.2 Vocabulario target

- **Count acumulativo:** ~3.000 palabras.
- **Nuevas:** ~500 palabras.
- **Temáticas:** consolidación + work-specific vocabulary, abstract
  concepts intro, news/current events básico.

### 6.3 Grammar focus (consolidación)

- Present perfect continuous.
- Conditionals second (intro).
- Passive voice expansion.
- Reported speech con tiempos verbales.
- Modal verbs en passive (must be done, should be sent).
- Discourse markers: however, although, despite, on the other hand.

### 6.4 Pronunciation focus

- Sentence stress (content vs function words) intro.
- Linking entre palabras (basic).
- Schwa consolidación.

### 6.5 Sub-skills cubiertos (intersección B1-B2)

Sub-skills en consolidación más que nuevas. Foco en mastery de B1
sub-skills y intro a B2 sub-skills.

### 6.6 Blocks por track

| Track | Blocks B1+ estimados |
|-------|--------------------:|
| Job Ready | 8-12 |
| Travel Confident | 6-8 |
| Daily Conversation | 10-12 |
| **Total B1+** | **~24-32** |

### 6.7 Duración estimada

A 15 min/día: **6-8 semanas**.

### 6.8 Mastery criteria para pasar a B2

- Score promedio ≥ 75 en sub-skills B1 cubiertos.
- WPM target: 100-120.
- < 5 fillers/min.

---

## 7. Plan B2 — Avanzado

> **Status MVP:** ✅ COVERED. Target laboral (USA jobs, BPOs,
> international companies).

### 7.1 Objetivos pedagógicos

Capacidades target:
- Trabajar o estudiar en inglés con relativa fluidez.
- Comprender textos complejos y abstractos.
- Participar activamente en reuniones y discusiones técnicas.
- Mantener fluidez espontánea en interacciones con nativos.
- Producir escritura coherente sobre variedad de temas.
- Manejar entrevistas laborales con confianza.

### 7.2 Vocabulario target

- **Count acumulativo:** ~4.000 palabras.
- **Nuevas:** ~1.000 palabras.
- **Temáticas:** business communications, interviews, tech industry
  USA, debates, abstract topics, work scenarios deep, news.

### 7.3 Grammar focus

- Past perfect simple y continuous.
- Conditionals second y third.
- Mixed conditionals.
- Reported speech avanzado (con shifts más complejos).
- Relative clauses (defining, non-defining).
- Subjunctive básico (I suggest he be...).
- Passive voice avanzado (have something done).
- Inversion básica (rarely have I seen...).
- More phrasal verbs.

### 7.4 Pronunciation focus

- Sentence stress completo (content vs function).
- Linking en speech connected (consonant + vowel, etc.).
- Intonation patterns en yes/no questions vs statements vs choices.
- Reduced forms (gonna, wanna, gotta).

### 7.5 Cultural / pragmatic focus

- Business email register (formal vs casual).
- Job interview USA style: STAR method, behavioral questions.
- Negotiation language básico.
- Critical feedback recibido y dado.
- USA business idioms (circle back, ping me, EOD, ASAP).
- Cultural awareness: directness vs diplomacy.

### 7.6 Sub-skills cubiertos

- `pron_sentence_stress`, `pron_linking`.
- `flu_self_correction_control`, `flu_extended_speech`,
  `flu_thinking_in_english`.
- `gram_past_perfect`, `gram_conditionals_second_third`,
  `gram_relative_clauses`, `gram_reported_speech`.
- `vocab_idioms_common`, `vocab_tech_industry`,
  `vocab_emotion_register`.
- `list_native_speed_general`, `list_distinguishing_accents`,
  `list_inference`, `list_phone_distorted_audio`.

### 7.7 Blocks por track

| Track | Blocks B2 estimados |
|-------|--------------------:|
| Job Ready | 22-28 |
| Travel Confident | 8-10 |
| Daily Conversation | 15-18 |
| **Total B2** | **~45-56** |

### 7.8 Duración estimada

A 15 min/día: **12-14 semanas**.

### 7.9 Mastery criteria para pasar a B2+

- Score promedio ≥ 75 en sub-skills B2 cubiertos.
- WPM target: 110-130.
- < 4 fillers/min.
- Capaz de mantener interview de 30 min sin breakdown.

---

## 8. Plan B2+ — Avanzado consolidado

> **Status MVP:** ✅ COVERED.

### 8.1 Objetivos pedagógicos

Transición B2 → C1. Capacidades target:
- Reuniones técnicas / business sin esfuerzo notorio.
- Presentations con seguridad sobre tu área de expertise.
- Escritura business profesional sin asistencia.
- Manejo confiable de situaciones unexpected.

### 8.2 Vocabulario target

- **Count acumulativo:** ~5.000 palabras.
- **Nuevas:** ~1.000 palabras.
- **Temáticas:** specialized business, leadership, abstract reasoning,
  cross-cultural communication.

### 8.3 Grammar focus (consolidación + sofisticación)

- Inversion avanzada para énfasis.
- Cleft sentences (it was X who...).
- Hedging language para diplomacia.
- Subjunctive en wishes y conditionals.
- Discourse cohesion avanzada.

### 8.4 Pronunciation focus

- Stress en frases compuestas y collocations.
- Intonation para implicaciones y sarcasmo.
- Reducción y elisión en speech rápido.

### 8.5 Sub-skills cubiertos (consolidación)

Sub-skills B2 en mastery + intro a C1.

### 8.6 Blocks por track

| Track | Blocks B2+ estimados |
|-------|--------------------:|
| Job Ready | 10-12 |
| Travel Confident | 4-6 |
| Daily Conversation | 8-10 |
| **Total B2+** | **~22-28** |

### 8.7 Duración estimada

A 15 min/día: **8-10 semanas**.

### 8.8 Mastery criteria para pasar a C1

- Score promedio ≥ 80 en sub-skills B2 cubiertos.
- WPM target: 120-140.
- < 3 fillers/min.
- Pronunciación percibida como "comprensible siempre" por nativos.

---

## 9. Plan C1 — Dominio Operativo Eficaz

> **Status MVP:** scope expansión Fase 2-3.

### 9.1 Objetivos pedagógicos

Capacidades target:
- Uso flexible y efectivo del idioma para fines sociales, académicos
  y profesionales.
- Comprender una amplia gama de textos exigentes y largos.
- Producir textos claros, bien estructurados y detallados sobre temas
  complejos.
- Negociar, persuadir, presentar argumentos sofisticados.
- Senior leadership communication.

### 9.2 Vocabulario target

- **Count acumulativo:** ~6.500 palabras.
- **Nuevas:** ~1.500 palabras.
- **Temáticas:** academic, abstract reasoning, business strategy,
  negotiation, persuasion, advanced idioms, register switching.

### 9.3 Grammar focus

- Subjunctive avanzado.
- Inversions complejas.
- Cleft sentences avanzadas.
- Ellipsis y substitution.
- Modal perfects (could have, must have, should have).
- Discourse markers avanzados (notwithstanding, albeit, hence).
- Hedging y emphasis avanzados.

### 9.4 Pronunciation focus

- Native-like prosody.
- Regional awareness (entender a hablantes de UK, AU, etc.).
- Connected speech masterful.

### 9.5 Cultural / pragmatic focus

- Negotiation strategies USA business.
- Diplomatic language para feedback critical.
- Persuasion techniques.
- Academic discourse.
- Code-switching profesional vs casual.

### 9.6 Sub-skills cubiertos

- `vocab_academic_discourse`.
- `list_overlapping_speech`.
- (Expansión a 200+ sub-skills en Fase 2 cubrirá más explícitamente.)

### 9.7 Blocks por track

| Track | Blocks C1 estimados |
|-------|--------------------:|
| Job Ready | 15-20 (senior roles, leadership) |
| Travel Confident | 5-8 |
| Daily Conversation | 10-12 |
| **Total C1** | **~30-40** |

### 9.8 Duración estimada

A 15 min/día: **16-20 semanas**.

### 9.9 Mastery criteria para pasar a C2

- Score promedio ≥ 80 en sub-skills C1 cubiertos.
- WPM target: 130-160.
- < 2 fillers/min.
- Pronunciación percibida como "near-native" por nativos.

---

## 10. Plan C2 — Maestría

> **Status MVP:** scope expansión Fase 4+. Plan aspiracional.

### 10.1 Objetivos pedagógicos

Capacidades target:
- Dominio excepcional, similar a hablante nativo educado.
- Comprender prácticamente todo lo que se escucha o lee.
- Resumir información de fuentes diversas reconstruyendo argumentos.
- Expresarse con espontaneidad, fluidez y precisión, distinguiendo
  matices sutiles incluso en situaciones complejas.

### 10.2 Vocabulario target

- **Count acumulativo:** ~10.000+ palabras.
- **Nuevas:** ~3.500+ palabras.
- **Temáticas:** literature, advanced academia, specialized fields,
  archaic forms, regional variations, technical jargon.

### 10.3 Grammar focus

- Todas las formas dominadas con flexibilidad nativa.
- Literary structures.
- Stylistic choices (passive vs active para énfasis específico).
- Subtle register switching automático.

### 10.4 Pronunciation focus

- Indistinguibilidad de hablante nativo (objetivo aspiracional).
- Adaptación regional según interlocutor.

### 10.5 Sub-skills (a expandir Fase 4+)

Sub-skills tentativos:
- `vocab_literary`
- `vocab_archaic`
- `vocab_specialized_legal`, `vocab_specialized_medical`, etc.
- `gram_advanced_inversions`
- `pron_native_indistinguishable`
- `register_switching_automatic`

### 10.6 Blocks por track

| Track | Blocks C2 estimados |
|-------|--------------------:|
| Job Ready | 10-15 (executive level) |
| Travel Confident | 3-5 |
| Daily Conversation | 5-8 |
| **Total C2** | **~18-28** |

### 10.7 Duración estimada

Open-ended (años de práctica).

### 10.8 Mastery criteria

- Score promedio ≥ 90 en sub-skills C2.
- WPM target: 140-180.
- Pronunciación regularmente confundida con hablante nativo.
- Capacidad de manejar registers múltiples sin esfuerzo.

---

## 11. Matriz tracks × CEFR

Total estimado de blocks por intersección track × CEFR:

| Track \\ Nivel | A1 | A2 | B1 | B1+ | B2 | B2+ | C1 | C2 | **Total** |
|----------------|---:|---:|---:|----:|---:|----:|---:|---:|----------:|
| **Job Ready** | 0 | 5-8 | 18-22 | 8-12 | 22-28 | 10-12 | 15-20 | 10-15 | **88-117** |
| **Travel Confident** | 8-10 | 15-18 | 12-15 | 6-8 | 8-10 | 4-6 | 5-8 | 3-5 | **61-80** |
| **Daily Conversation** | 10-12 | 18-22 | 18-22 | 10-12 | 15-18 | 8-10 | 10-12 | 5-8 | **94-116** |
| **Total por nivel** | 18-22 | 38-48 | 48-59 | 24-32 | 45-56 | 22-28 | 30-40 | 18-28 | |

### 11.1 Foco MVP

MVP cubre B1, B1+, B2, B2+. Total blocks MVP target:

```
Job Ready B1+B1++B2+B2+:    18 + 8 + 22 + 10 = ~58 (conservador)
Job Ready upper bound:       22 + 12 + 28 + 12 = ~74
Travel Confident MVP:        12 + 6 + 8 + 4 = ~30 a 38
Daily Conversation MVP:      18 + 10 + 15 + 8 = ~51 a 62

Total blocks MVP:            ~140 a ~175
```

A ~4 atomics por block (con reuse 2x → 2 atomics realmente creados):

```
Atomics MVP:                 ~280 a ~350
```

Esto matchea aproximadamente con la estimación previa de ~700 assets
para MVP (`content-creation-system.md` §7.1) si contamos cada
ejercicio/exercise dentro de un bloque como "asset". Los **bloques**
son ~150 y cada uno tiene 4-5 ejercicios.

### 11.2 Nuevas tracks Fase 2-3 (referencia)

| Track | Niveles cubiertos | Blocks estimados |
|-------|-------------------|----------------:|
| Business English | B1-C1 | 80-100 |
| Tech Professional | B1+-C1 | 70-90 |
| Academic | B2-C2 | 60-80 |
| Customer Service | A2-B2 | 60-80 |
| Healthcare | A2-B2 | 60-80 |

---

## 12. Transiciones entre niveles

### 12.1 Detección automática de transición

Cuando un user en nivel X cumple los **mastery criteria** de §X.9 (o
§X.8 en sub-niveles), el sistema:

1. Emite evento `user.cefr_changed` (de `pedagogical-system.md` §12.4).
2. `ai-roadmap-system` evalúa: regenerar roadmap o continuar
   incrementalmente.
3. Cliente celebra (de `motivation-and-achievements.md` §6.2.3:
   `level_up_b1_to_b2`, etc.).

### 12.2 Salto de niveles

Posible si user demuestra mastery sin practicar todos los blocks:
- Test-out a nivel sub-skill (`pedagogical-system.md` §4.5).
- Test-out a nivel block (`pedagogical-system.md` §4.5).
- Re-evaluation oficial (`student-profile-and-assessment.md` §8).

### 12.3 Regresión

Si user cae en performance sostenidamente (decay + bajo score en
re-evaluation):
- Sub-skills a `developing` (de `pedagogical-system.md` §6.2).
- Roadmap se ajusta para reforzar.
- NO se reduce CEFR oficial del user (evita loss aversion brusca).

---

## 13. Bloqueos y prerequisites cross-level

### 13.1 Reglas

- Un user solo puede practicar blocks de su nivel CEFR actual ± 1.
- Blocks de nivel + 2 están locked (ej: B1 user no ve blocks B2+).
- Blocks de nivel - 2 están disponibles para revisión (decay).

### 13.2 Prerequisites entre tracks

- Job Ready B1 requiere mastery básico de Daily Conversation A2-B1.
- Travel Confident es independiente (puede iniciarse en cualquier
  nivel).
- Tracks especializados (Business, Tech, Academic) requieren B2+ de
  Daily Conversation.

### 13.3 Initial assessment determina el entry point

El mini-test del Day 0 + assessment del Day 7
(`student-profile-and-assessment.md`) determinan:
- CEFR estimado del user.
- Sub-skills débiles a fortalecer.
- Track recomendado según objetivo.

Roadmap inicial entonces selecciona blocks del nivel CEFR estimado
(no inferior, no superior).

---

## 14. Estimaciones de duración consolidadas

### 14.1 Tiempo total para llegar a B2 (target laboral típico)

Asumiendo entry point en A1 (worst case):

```
A1: 6-8 semanas
A2: 8-10 semanas
B1: 10-12 semanas
B1+: 6-8 semanas
B2: 12-14 semanas

Total A1 → B2: ~42-52 semanas (~10-12 meses)
```

Asumiendo entry point en B1 (typical user del producto):

```
B1: 10-12 semanas
B1+: 6-8 semanas
B2: 12-14 semanas

Total B1 → B2: ~28-34 semanas (~7-8 meses)
```

Asumiendo entry point en B2 (advanced):

```
B2 → B2+: 8-10 semanas
B2+ → C1: 16-20 semanas

Total B2 → C1: ~24-30 semanas (~6-7 meses)
```

### 14.2 Tiempo a 30 min/día (ambitious user)

Aproximadamente la mitad de los tiempos anteriores.

### 14.3 Tiempo a 5 min/día (casual user)

Aproximadamente el triple. Realísticamente puede no llegar a B2 en
1 año.

---

## 15. Decisiones cerradas

### 15.1 MVP cubre B1-B2+ (no A1, A2, C1, C2): **SÍ** ✓

**Razón:** target market = profesionales latinos buscando trabajo
internacional. Mayoría tiene ya base mínima (B1). C1 ya hablan bien
sin necesidad crítica de producto.

### 15.2 Sub-niveles `+` solo en B1 y B2: **SÍ** ✓

**Razón:** zona crítica del producto. Otros niveles no se beneficiarían
proporcionalmente de granularidad adicional.

### 15.3 CEFR oficial no se reduce ante regresión: **SÍ** ✓

**Razón:** loss aversion brusca daña retention. Sub-skills sí pueden
volver a `developing` sin afectar CEFR overall del user.

### 15.4 Track-orthogonal vs track-by-track curriculum: **track-orthogonal** ✓

**Razón:** mantiene CEFR como framework universal. Cada track filtra
del catálogo según objetivo, pero el lenguaje compartido es CEFR.

### 15.5 Vocab counts target oficiales del producto: **basados en CEFR-J + extensiones** ✓

**Razón:** CEFR-J es source open source bien validado. Extensiones del
producto (tech, business, travel) agregan vocabulary específico al
target.

### 15.6 Blocks per level upper bound vs lower bound: **upper bound como target** ✓

**Razón:** mejor sobre-cubrir un nivel que sub-cubrir. User puede
saltar via test-out si domina ya.

---

## 16. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`pedagogical-system.md`](pedagogical-system.md) §2.4 | Catálogo de 50 sub-skills (A2-C1). Expansión Fase 2 cubrirá A1, C2. |
| [`pedagogical-system.md`](pedagogical-system.md) §2.5 | Reglas de agregación score → CEFR. |
| [`pedagogical-system.md`](pedagogical-system.md) §3.2 | Targets de fluency (WPM) por CEFR. |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §6 | Assessment Day 7 determina CEFR inicial. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) §5 | Generación de roadmap usa este curriculum. |
| [`content-creation-system.md`](content-creation-system.md) §7 | Cobertura inicial de biblioteca. |
| [`motivation-and-achievements.md`](motivation-and-achievements.md) §6.2.3 | Logros de level-up CEFR. |

---

## 17. Plan de implementación

### 17.1 Fase 1: MVP (meses 0–3)

**Crítico:**
- Curriculum B1, B1+, B2, B2+ con blocks generados.
- ~140-175 blocks total (3 tracks × 4 niveles).
- ~280-350 atomics (con reuse 2x).

### 17.2 Fase 2: Expansión (meses 3–9)

- Agregar curriculum A2 (~38-48 blocks).
- Agregar curriculum C1 (~30-40 blocks).
- Expandir sub-skills catalog de 50 → 200+.

### 17.3 Fase 3: Especialización (meses 9–18)

- Tracks especializados: Business English, Tech Professional, Academic.
- Curriculum cross-track refinement.

### 17.4 Fase 4: Cobertura completa (año 2+)

- Curriculum A1 con sub-skills tentativos.
- Curriculum C2 con sub-skills tentativos.
- Aspiracional para users en extremos del espectro.

---

*Documento vivo. Actualizar cuando se calibren métricas en producción,
se descubran gaps en cobertura, o se expandan sub-skills del catálogo.*

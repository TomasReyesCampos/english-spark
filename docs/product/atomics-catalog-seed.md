# Atomics Catalog Seed — MVP

> Banco inicial autoritativo de los ~350 `media_atomics` del MVP listo
> para implementación. Define naming conventions, voice pool, los 6
> characters completos, atomics ambient/scene, atomics de imagen,
> atomics de fraseología genérica, y los pipelines de generación con
> costos.

**Estado:** Diseño v1.0 (listo para implementación)
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador (al ejecutar el batch
de generación inicial) + content team (al validar).
**Alcance:** seed inicial de `media_atomics` + `characters` para MVP.

---

## 0. Cómo leer este documento

- §1 establece **filosofía** y scope del seed.
- §2 cubre **naming conventions** estrictas.
- §3 cubre el **voice pool TTS** (12 voces base).
- §4 detalla los **6 characters recurrentes** completos.
- §5 cubre **character-bound atomics** (~180 audio+image+video).
- §6 cubre **ambient/scene audio** (~40 atomics).
- §7 cubre **scene image atomics** (~70 atomics).
- §8 cubre **generic phrase atomics** (~60 atomics).
- §9 cubre **generation pipelines** (proveedor, prompt, costo).
- §10 cubre **seed migration plan**.
- §11 cubre **costos totales**.
- §12 cubre **decisiones cerradas**.

---

## 1. Filosofía y scope

### 1.1 Objetivo del seed

Generar un catálogo inicial de atomics que:
- Cubra los **175 bloques MVP** (74 Job Ready + 39 Travel Confident +
  62 Daily Conversation).
- Maximice **reuse cross-block y cross-track**.
- Mantenga **identidad consistente** de characters (audio + visual).
- Permita **bootstrap rápido** del content team (semana 1
  productiva).

### 1.2 Conteo target

| Categoría | Atomics esperados | Reuse target |
|-----------|------------------:|-------------:|
| Character-bound (audio voice + visual avatar) | ~180 | 3-5x |
| Ambient/scene audio (background, distortions) | ~40 | 5-8x |
| Scene images (settings, contexts) | ~70 | 4-6x |
| Generic phrase audio (fillers, transitions, common phrases) | ~60 | 6-10x |
| **Total** | **~350** | promedio 4-5x |

**Reuse rate target inicial MVP:** ≥ 1.5 (de
`content-creation-system.md` §3.5). Con 175 bloques × 4 assets =
700 composites usando 350 atomics → reuse efectivo 2.0.

### 1.3 Scope del seed (qué SÍ y qué NO)

**SÍ incluye en seed:**
- Atomics requeridos por al menos 2 bloques del MVP.
- Atomics que componen los 3 capstones de cada track (alta visibilidad).
- Voice pool completo (12 voces base).
- Los 6 characters con ≥ 20 atomics each.

**NO incluye en seed (se generan on-demand):**
- Atomics requeridos por solo 1 bloque (long tail).
- Variantes de tono / speed específicas (se generan al detectar
  necesidad).
- Atomics post-MVP variants (UK English, otros destinos travel).

### 1.4 Validación previa al uso productivo

Cada atomic en el seed pasa por:
1. Generación AI (proveedor según media type).
2. Validación automática (ver `content-creation-system.md` §5.3).
3. **Revisión humana obligatoria** (TESOL background) antes de
   `approved = true`.
4. Beta con 50 users antes de marcar `approved_for_production` en
   composites que lo usan.

---

## 2. Naming conventions

### 2.1 Estructura del `id`

```
<media_format>_<descriptor>_<context>_<variant>_v<n>
```

- **media_format:** `audio`, `image`, `video`.
- **descriptor:** snake_case describiendo el contenido.
- **context:** opcional, agrega contexto track o character.
- **variant:** opcional, distingue versiones (slow, fast, noisy).
- **v<n>:** versión semver mayor (v1, v2).

**Ejemplos:**
- `audio_alex_friendly_intro_v1`
- `audio_filler_let_me_think_v1`
- `audio_phone_distortion_overlay_v1`
- `image_coffee_shop_interior_us_v1`
- `video_sarah_introduces_30s_v1`

### 2.2 Reglas estrictas

- **snake_case** siempre. Nunca camelCase ni kebab-case.
- **Inglés** para identifiers (estándar de codebase, ver
  `CLAUDE.md` §5.1).
- **Sin acentos, sin caracteres especiales** (R2 storage keys deben
  ser ASCII-safe).
- **Max 80 chars** total para el id.
- **Prefijo media_format obligatorio** (audio_, image_, video_).
- **Sufijo de versión obligatorio** (_v1, _v2, etc.) para permitir
  evolución sin romper referencias.

### 2.3 Convenciones por categoría

#### Character-bound atomics

```
audio_<character_short>_<context>_<variant>_v<n>
image_<character_short>_<context>_v<n>
video_<character_short>_<scenario>_<duration>s_v<n>
```

`character_short`: alias corto de 4-7 chars (alex, sarah, mike,
grandma, jamie, emma).

#### Ambient/scene audio

```
audio_ambient_<scene>_<duration>s_v<n>
audio_<distortion_type>_overlay_v<n>
```

#### Scene images

```
image_<scene_descriptor>_<region>_v<n>
```

`region`: `us`, `latam`, `neutral` (donde aplique culturalmente).

#### Generic phrase atomics

```
audio_<intent>_<phrase_short>_v<n>
```

`intent`: `filler`, `greeting`, `farewell`, `confirm`, `clarify`,
`turn_taking`, `transition`.

---

## 3. Voice pool TTS (12 voces base)

### 3.1 Filosofía del pool

12 voces seleccionadas de ElevenLabs (proveedor primario, ver §9.2)
cubriendo edad, género, accent USA. Cada character usa una voz
del pool consistente.

### 3.2 Voice pool

| voice_id | Provider | Edad | Género | Accent | Tono | Usado por character |
|----------|----------|------|--------|--------|------|---------------------|
| `voice_us_f_25_warm` | ElevenLabs | 22-28 | F | US-General | Warm, casual | char_alex (si F) |
| `voice_us_m_28_friendly` | ElevenLabs | 25-32 | M | US-General | Friendly, casual | char_alex (default) |
| `voice_us_f_32_professional` | ElevenLabs | 28-38 | F | US-General | Profesional, clear | char_sarah |
| `voice_us_m_35_calm` | ElevenLabs | 32-40 | M | US-General | Calm, measured | (general) |
| `voice_us_f_45_articulate` | ElevenLabs | 40-50 | F | US-General | Articulate, refined | char_emma |
| `voice_us_m_50_warm` | ElevenLabs | 45-55 | M | US-Midwest | Warm, conversational | char_mike |
| `voice_us_f_65_grandmother` | ElevenLabs | 60-70 | F | US-General | Slow, kind | char_grandma |
| `voice_us_m_30_witty` | ElevenLabs | 28-35 | M | US-Northeast | Witty, energetic | (banter blocks) |
| `voice_us_f_28_thoughtful` | ElevenLabs | 25-32 | F | US-General | Thoughtful, paused | char_jamie |
| `voice_us_m_55_authoritative` | ElevenLabs | 50-60 | M | US-General | Authoritative | (interview blocks) |
| `voice_us_f_40_neutral` | ElevenLabs | 38-45 | F | US-General | Neutral, narrator | (listening exercises) |
| `voice_us_m_45_neutral` | ElevenLabs | 42-50 | M | US-General | Neutral, narrator | (listening exercises) |

### 3.3 Reglas de uso del pool

- **Un character = una voz fija**. No alternar voces para el mismo
  character (rompe memoria episódica).
- **Speed range permitido:** 0.85x (slow), 1.0x (default), 1.15x
  (fast). Para B2+ native-speed atomics: 1.15x.
- **Voces 11 y 12 (`_neutral`)** se usan para listening exercises sin
  character (ej: news clips, podcast excerpts).
- **No mix de accents** dentro del mismo asset composite (consistency
  pedagógica).

### 3.4 Validación

- Cada voice_id se prueba con 5 frases representativas antes de
  asignar a un character.
- Native speakers humanos califican naturalness (1-10). Threshold:
  ≥ 8.

---

## 4. Los 6 characters recurrentes (catálogo completo)

### 4.1 `char_alex_friendly_friend`

```yaml
id: char_alex_friendly_friend
name: Alex
voice_id: voice_us_m_28_friendly  # default M; F variant disponible
age_range: 25-32
occupation: Marketing coordinator (no relevante en mayoría de blocks)
cultural_background: us_general

description_visual: |
  Casual mid-20s person, light brown hair, warm smile, casual hoodie or
  t-shirt. Approachable, looks like "friend you'd grab coffee with".
  Background neutro/casual.

description_voice: |
  Friendly mid-20s, casual cadence, occasional "you know", "totally",
  "I mean". Energetic but not overwhelming. Default native-natural
  speed (1.0x) for B1; can go to 1.1x for B2+.

personality_traits:
  - curious about your life (asks follow-ups)
  - casual humor (no sarcasm at B1, gentle banter at B2+)
  - inclusive language
  - non-judgmental

appears_in_tracks: [daily_conversation, job_ready_b1plus]

key_phrases_signature:
  - "Hey, what's up?"
  - "That's cool, tell me more"
  - "I totally get that"
  - "What do you think about..."

avoid:
  - profanity
  - heavy slang that dates fast
  - condescending tone
  - personal questions too forward at B1
```

### 4.2 `char_grandma_patient`

```yaml
id: char_grandma_patient
name: Grandma Helen
voice_id: voice_us_f_65_grandmother
age_range: 60-70
occupation: Retired teacher
cultural_background: us_general

description_visual: |
  Warm older woman, silver hair, gentle smile, comfortable cardigan.
  Background often homely (kitchen, living room with photos).

description_voice: |
  Slow, deliberate cadence (0.85x default). Natural pauses. Repeats key
  phrases for emphasis. Vocabulary B1-friendly even at B2 contexts.

personality_traits:
  - patient (waits for user response, doesn't rush)
  - encouraging ("That's wonderful, dear")
  - shares wisdom in stories
  - doesn't dominate conversation

appears_in_tracks: [daily_conversation_b1, daily_conversation_b1plus]

key_phrases_signature:
  - "Tell me about your day, dear"
  - "Oh, that reminds me of when..."
  - "Take your time, no rush"
  - "How wonderful"

avoid:
  - fast speech
  - tech jargon
  - aggressive corrections
  - dating / inappropriate topics (cross-generational appropriateness)
```

### 4.3 `char_sarah_witty_coworker`

```yaml
id: char_sarah_witty_coworker
name: Sarah
voice_id: voice_us_f_32_professional
age_range: 28-38
occupation: Senior project manager
cultural_background: us_general (with latino heritage option)

description_visual: |
  Professional 30s, polished but approachable. Often in business-casual.
  Office or video-call backgrounds.

description_voice: |
  Clear professional cadence, controlled humor. Native speed by B1+.
  Light banter at B2. Crisp pronunciation, models good speech for
  professional contexts.

personality_traits:
  - direct but warm
  - light humor (not sarcastic at B1)
  - efficient (gets to the point)
  - mentor energy (offers help without imposing)

appears_in_tracks: [job_ready, daily_conversation_b2]

key_phrases_signature:
  - "Got it, makes sense"
  - "Quick question..."
  - "Let me push back gently on that"
  - "Honestly, I think..."

avoid:
  - corporate jargon overload
  - condescension
  - flirtatious tone
```

### 4.4 `char_mike_chatty_neighbor`

```yaml
id: char_mike_chatty_neighbor
name: Mike
voice_id: voice_us_m_50_warm
age_range: 50-60
occupation: Owner of small business (cafe / hardware store)
cultural_background: us_midwest

description_visual: |
  Approachable 50s man, casual button-down or apron. Outdoor or shop
  backgrounds.

description_voice: |
  Warm midwestern cadence, conversational pace (1.0x). Uses idioms
  naturally. Some friendly small talk filler.

personality_traits:
  - chatty (talks a bit more than asks)
  - tells local stories
  - friendly to strangers (USA midwest convention)
  - opinions strong but stated softly

appears_in_tracks: [daily_conversation, travel_confident]

key_phrases_signature:
  - "How're you doing today?"
  - "You from around here?"
  - "Funny story about that..."
  - "Folks around here say..."

avoid:
  - politics
  - intense topics at B1
```

### 4.5 `char_jamie_deep_friend`

```yaml
id: char_jamie_deep_friend
name: Jamie
voice_id: voice_us_f_28_thoughtful
age_range: 26-34
occupation: Therapist / counselor (informal mention)
cultural_background: us_general

description_visual: |
  Late 20s, calm presence, often shown in cozy settings (couch, coffee
  shop, walking). Empathetic body language.

description_voice: |
  Thoughtful, paced cadence. Comfortable with silence. Doesn't fill
  pauses with filler.

personality_traits:
  - deep listener (asks reflective questions)
  - non-judgmental
  - shares vulnerably when invited
  - doesn't try to fix, holds space

appears_in_tracks: [daily_conversation_b2, daily_conversation_b2plus]

key_phrases_signature:
  - "How are you, really?"
  - "What does that bring up for you?"
  - "I hear you"
  - "Take your time"

avoid:
  - therapy-speak overload (sounds like script)
  - imposing diagnoses
  - small talk only (pulls toward depth)
```

### 4.6 `char_emma_native_fast`

```yaml
id: char_emma_native_fast
name: Emma
voice_id: voice_us_f_45_articulate
age_range: 40-50
occupation: Journalist / podcast host
cultural_background: us_general (east coast)

description_visual: |
  Polished 40s, often in studio or interview settings. Headphones in
  some shots.

description_voice: |
  Native-speed default (1.15x). Articulate, sophisticated vocabulary.
  Uses idioms naturally. Comfortable with overlap and quick exchanges.

personality_traits:
  - intellectually curious
  - challenges politely (Socratic)
  - well-informed on culture, politics, current events
  - high standard for conversation depth

appears_in_tracks: [daily_conversation_b2, daily_conversation_b2plus, job_ready_b2plus]

key_phrases_signature:
  - "That's a fascinating point"
  - "Push back on that for me"
  - "Let me steelman..."
  - "What's your honest take?"

avoid:
  - dumbing down (this character IS the challenge)
  - interrupting (despite native speed, respects turns)
  - condescension
```

### 4.7 Tabla resumen characters

| Character | Voice | Tracks principales | # atomics esperados |
|-----------|-------|--------------------|--------------------:|
| char_alex_friendly_friend | voice_us_m_28_friendly | DC, JR B1+ | 35 |
| char_grandma_patient | voice_us_f_65_grandmother | DC B1, B1+ | 20 |
| char_sarah_witty_coworker | voice_us_f_32_professional | JR, DC B2 | 35 |
| char_mike_chatty_neighbor | voice_us_m_50_warm | DC, TC | 25 |
| char_jamie_deep_friend | voice_us_f_28_thoughtful | DC B2, B2+ | 30 |
| char_emma_native_fast | voice_us_f_45_articulate | DC B2+, JR B2+ | 35 |
| **Total** | | | **180** |

---

## 5. Character-bound atomics (~180)

### 5.1 Distribución por character

Cada character tiene mix de:
- **Audio TTS atomics** (~70% del volumen): saludos, frases comunes,
  lines de roleplay scripts, follow-ups.
- **Image atomics** (~20%): avatar variants (different facial
  expressions, settings).
- **Video atomics** (~10%): talking-head clips de 15-30s para listening
  exercises.

### 5.2 Template de generación por character

Cada character genera atomics en estas categorías:

#### Audio (per character, ~25 atomics promedio)

| Categoría | Cantidad | Ejemplos |
|-----------|---------:|----------|
| Saludos / opener phrases | 5 | "Hey, what's up?", "Hi there!", "How are you doing?" |
| Closer phrases | 3 | "Talk soon", "Have a good one", "See ya" |
| Filler natural del character | 4 | "I mean...", "you know", "totally" (variantes) |
| Follow-up question patterns | 5 | "Tell me more about...", "What was that like?" |
| Reactions (positive, neutral, surprise) | 4 | "Oh wow", "That's awesome", "Hmm interesting" |
| Roleplay scenario openers (track-specific) | 4-8 | "So I heard you...", "I wanted to chat about..." |

#### Image (per character, ~5 atomics)

| Variant | Descripción |
|---------|-------------|
| Avatar neutral | Default expression, neutral background |
| Avatar listening | Head slightly tilted, attentive |
| Avatar laughing / smiling | For positive moments |
| Avatar thoughtful | For deep conversation moments |
| Avatar in context | Specific setting (office, kitchen, café) |

#### Video (per character, ~3 atomics)

| Variant | Duración | Uso |
|---------|---------:|-----|
| Talking-head intro | 15s | Listening MC exercises |
| Talking-head extended | 30s | Long-form listening |
| Roleplay loop background | 5s loop | Roleplay UI background |

### 5.3 Listado de atomics character-bound (sample completo de char_alex)

#### `char_alex_friendly_friend` — atomics audio

| atomic_id | Texto / contenido | duration_ms | Used in |
|-----------|-------------------|------------:|---------|
| `audio_alex_greeting_hey_whats_up_v1` | "Hey, what's up?" | 1500 | DC Module 1, 3, 5 |
| `audio_alex_greeting_hi_there_v1` | "Hi there! Good to see you" | 2200 | DC Module 1 |
| `audio_alex_greeting_how_are_you_v1` | "How are you doing today?" | 1800 | DC Module 1, 3 |
| `audio_alex_greeting_long_time_v1` | "Hey! Long time no see" | 1800 | DC Module 1 |
| `audio_alex_greeting_morning_v1` | "Morning! How's it going?" | 1700 | DC Module 1 |
| `audio_alex_closer_talk_soon_v1` | "Alright, talk soon!" | 1400 | DC many |
| `audio_alex_closer_have_good_one_v1` | "Have a good one" | 1300 | DC many |
| `audio_alex_closer_see_you_around_v1` | "See you around" | 1200 | DC many |
| `audio_alex_filler_i_mean_v1` | "I mean..." | 700 | DC many |
| `audio_alex_filler_you_know_v1` | "you know..." | 600 | DC many |
| `audio_alex_filler_totally_v1` | "totally" | 500 | DC many |
| `audio_alex_filler_let_me_think_v1` | "let me think for a sec..." | 1500 | DC many |
| `audio_alex_followup_tell_me_more_v1` | "Tell me more about that" | 1700 | DC Module 5, 6 |
| `audio_alex_followup_what_was_that_like_v1` | "What was that like?" | 1500 | DC Module 5, 6 |
| `audio_alex_followup_how_did_you_v1` | "How did you handle that?" | 1900 | DC B1+ |
| `audio_alex_followup_what_do_you_think_v1` | "What do you think about it?" | 2000 | DC many |
| `audio_alex_followup_no_way_v1` | "No way, really?" | 1300 | DC casual |
| `audio_alex_reaction_oh_wow_v1` | "Oh wow" | 800 | DC many |
| `audio_alex_reaction_thats_awesome_v1` | "That's awesome!" | 1200 | DC positive |
| `audio_alex_reaction_hmm_interesting_v1` | "Hmm, interesting" | 1500 | DC neutral |
| `audio_alex_reaction_thats_rough_v1` | "Oh, that's rough" | 1400 | DC negative |
| `audio_alex_dc_party_intro_v1` | "Hey! I don't think we've met. I'm Alex" | 3000 | DC Module 1 party |
| `audio_alex_dc_meet_at_party_v1` | "I'm friends with the host. How do you know them?" | 4000 | DC Module 1 |
| `audio_alex_dc_make_plans_v1` | "We should grab coffee sometime, what do you say?" | 3500 | DC Module 3 |
| `audio_alex_dc_long_chat_opener_v1` | "Got 30 minutes? I want to actually catch up" | 3500 | DC B1+ capstone |

(Total atomics audio para `char_alex`: ~25.)

#### `char_alex_friendly_friend` — atomics image

| atomic_id | Descripción | Generation prompt key |
|-----------|-------------|----------------------|
| `image_alex_avatar_neutral_v1` | Default avatar, neutral expression | (ver §9.3) |
| `image_alex_avatar_smiling_v1` | Smiling, friendly | |
| `image_alex_avatar_thoughtful_v1` | Slight head tilt, listening | |
| `image_alex_avatar_at_coffee_shop_v1` | In coffee shop setting | |
| `image_alex_avatar_at_party_v1` | At social party | |

#### `char_alex_friendly_friend` — atomics video

| atomic_id | Duración | Descripción | Used in |
|-----------|---------:|-------------|---------|
| `video_alex_talking_head_intro_15s_v1` | 15s | Self-intro at party | DC Module 1 listening |
| `video_alex_talking_head_30s_v1` | 30s | Extended sharing | DC B1+ listening |
| `video_alex_roleplay_bg_loop_v1` | 5s loop | Background for roleplay UI | DC roleplays |

### 5.4 Pattern para los otros 5 characters

Usar el mismo template (saludos, closers, fillers, follow-ups,
reactions + character-specific scenario openers + 5 image avatars +
3 videos).

Los atomics de cada character tienen **prefijo de id consistente**:
`audio_<short>_*`, `image_<short>_*`, `video_<short>_*`.

Distribución exacta:
- char_alex: 25 audio + 5 image + 3 video = **33** (sample arriba; ~35 con extras)
- char_grandma: 15 audio + 4 image + 1 video = **20**
- char_sarah: 25 audio + 5 image + 5 video = **35**
- char_mike: 18 audio + 4 image + 3 video = **25**
- char_jamie: 22 audio + 5 image + 3 video = **30**
- char_emma: 26 audio + 5 image + 4 video = **35**

**Total character-bound: ~180 atomics.**

---

## 6. Ambient/scene audio (~40 atomics)

### 6.1 Filosofía

Audio de ambient noise para overlay en composites. Crear "world" en
audio sin requerir nuevas grabaciones.

### 6.2 Listado completo

#### Travel-related (12 atomics)

| atomic_id | Descripción | Duración | Used in tracks |
|-----------|-------------|---------:|----------------|
| `audio_ambient_airport_hall_30s_v1` | Aeropuerto general, anuncios distantes | 30s loop | TC |
| `audio_ambient_airport_security_15s_v1` | TSA línea, beeps | 15s | TC B1 |
| `audio_announcement_boarding_call_v1` | "Now boarding flight 1234..." | 12s | TC B1 |
| `audio_announcement_gate_change_v1` | "Attention passengers, gate change..." | 10s | TC B1+ |
| `audio_announcement_delay_v1` | "Flight 567 has been delayed..." | 12s | TC B1+ |
| `audio_ambient_airplane_cabin_v1` | Cabin engine hum | 20s loop | TC B1 |
| `audio_ambient_hotel_lobby_v1` | Lobby chatter, soft music | 25s loop | TC B1 |
| `audio_ambient_restaurant_busy_v1` | Restaurant chatter | 20s loop | TC, DC |
| `audio_ambient_restaurant_quiet_v1` | Quiet restaurant | 20s loop | TC |
| `audio_ambient_subway_station_v1` | NYC subway ambient | 20s loop | TC B1 |
| `audio_ambient_busy_street_us_v1` | NYC/SF street | 20s loop | TC, DC |
| `audio_ambient_hotel_room_v1` | Hotel room quiet | 15s loop | TC |

#### Workplace (10 atomics)

| atomic_id | Descripción | Duración | Used in |
|-----------|-------------|---------:|---------|
| `audio_ambient_office_open_v1` | Open office chatter | 20s loop | JR |
| `audio_ambient_office_quiet_v1` | Quiet office, keyboard | 15s loop | JR |
| `audio_ambient_meeting_room_v1` | Meeting room before/after | 15s | JR |
| `audio_ambient_zoom_intro_chime_v1` | Zoom join sound | 2s | JR remote work |
| `audio_ambient_video_call_hold_v1` | Hold music | 10s loop | JR |
| `audio_ambient_coffee_shop_workspace_v1` | Coffee shop work | 25s loop | JR remote |
| `audio_ambient_phone_ringing_us_v1` | US phone ring | 4s | JR, TC |
| `audio_ambient_phone_busy_signal_v1` | US busy signal | 3s | JR, TC |
| `audio_ambient_typing_keyboard_v1` | Typing | 8s loop | JR |
| `audio_ambient_office_door_open_v1` | Door opens, footsteps | 4s | JR |

#### Social / casual (10 atomics)

| atomic_id | Descripción | Duración | Used in |
|-----------|-------------|---------:|---------|
| `audio_ambient_party_indoor_v1` | House party | 25s loop | DC Module 1 |
| `audio_ambient_party_outdoor_v1` | Outdoor BBQ | 25s loop | DC |
| `audio_ambient_coffee_shop_casual_v1` | Coffee shop casual | 25s loop | DC |
| `audio_ambient_living_room_v1` | Quiet living room | 15s loop | DC |
| `audio_ambient_kitchen_cooking_v1` | Cooking sounds | 15s loop | DC |
| `audio_ambient_walk_park_v1` | Park walk | 20s loop | DC |
| `audio_ambient_car_inside_v1` | Inside car | 15s loop | DC, TC |
| `audio_ambient_gym_v1` | Gym ambient | 20s loop | DC |
| `audio_ambient_bar_pub_v1` | Bar / pub | 25s loop | DC, TC |
| `audio_ambient_doorbell_v1` | US doorbell | 2s | DC, TC |

#### Distortion overlays (8 atomics)

| atomic_id | Descripción | Used in |
|-----------|-------------|---------|
| `audio_phone_distortion_overlay_v1` | Apply over any voice for "phone call" | TC, DC, JR |
| `audio_video_call_distortion_v1` | Slight compression for "video call" | JR remote |
| `audio_walkie_talkie_overlay_v1` | Old-school walkie | (post-MVP) |
| `audio_room_reverb_overlay_v1` | Big room reverb | TC |
| `audio_outdoor_wind_overlay_v1` | Light outdoor wind | TC, DC |
| `audio_radio_overlay_v1` | Radio quality | DC news |
| `audio_voice_memo_quality_v1` | WhatsApp voice memo quality | DC B2 |
| `audio_loud_overlay_v1` | Loud environment makes voice harder | TC, DC |

**Total ambient/scene audio: 40 atomics.**

---

## 7. Scene image atomics (~70 atomics)

### 7.1 Filosofía

Imágenes generadas por IA que actúan como "settings" para roleplays
y como prompts para image_description exercises.

### 7.2 Listado por categoría

#### Travel scenes (20 atomics)

| atomic_id | Descripción |
|-----------|-------------|
| `image_airport_terminal_us_v1` | US airport terminal, signage |
| `image_airport_security_line_v1` | TSA line |
| `image_airport_immigration_booth_v1` | Immigration booth USA |
| `image_airport_baggage_claim_v1` | Baggage claim |
| `image_airplane_cabin_economy_v1` | Economy seats |
| `image_airplane_window_view_v1` | Sky from window |
| `image_hotel_lobby_modern_v1` | Modern hotel lobby |
| `image_hotel_room_standard_v1` | Standard hotel room |
| `image_hotel_room_messy_v1` | Hotel room with issue |
| `image_restaurant_us_diner_v1` | US diner |
| `image_restaurant_upscale_v1` | Upscale restaurant |
| `image_subway_train_nyc_v1` | NYC subway interior |
| `image_subway_map_nyc_v1` | NYC subway map illustration |
| `image_taxi_yellow_us_v1` | NYC yellow taxi |
| `image_uber_ride_interior_v1` | Inside Uber |
| `image_street_corner_us_v1` | US street corner with stoplight |
| `image_tourist_info_desk_v1` | Tourist info desk |
| `image_souvenir_shop_v1` | Touristy souvenir shop |
| `image_pharmacy_us_v1` | US pharmacy |
| `image_emergency_room_us_v1` | ER waiting room |

#### Workplace scenes (15 atomics)

| atomic_id | Descripción |
|-----------|-------------|
| `image_office_open_modern_v1` | Modern open-plan office |
| `image_office_meeting_room_v1` | Meeting room with whiteboard |
| `image_office_conference_table_v1` | Conference table empty |
| `image_video_call_grid_v1` | Zoom-style grid of 4 |
| `image_coworker_desk_setup_v1` | Coworker desk |
| `image_coffee_shop_workspace_v1` | Coffee shop laptop work |
| `image_home_office_setup_v1` | Home office |
| `image_business_dinner_v1` | Dinner with colleagues |
| `image_conference_audience_v1` | Conference audience |
| `image_conference_networking_v1` | Networking break |
| `image_interview_panel_v1` | Interview panel of 3 |
| `image_interview_one_on_one_v1` | 1-on-1 interview |
| `image_interview_video_call_v1` | Interview via video |
| `image_office_kitchen_v1` | Office kitchen / break room |
| `image_office_elevator_v1` | Office elevator |

#### Social / casual scenes (20 atomics)

| atomic_id | Descripción |
|-----------|-------------|
| `image_house_party_v1` | House party indoor |
| `image_outdoor_bbq_v1` | Backyard BBQ |
| `image_living_room_casual_v1` | Casual living room |
| `image_coffee_shop_casual_v1` | Coffee shop casual chat |
| `image_park_walk_v1` | Park walk |
| `image_gym_us_v1` | US gym |
| `image_bar_pub_v1` | Pub setting |
| `image_kitchen_cooking_v1` | Kitchen cooking together |
| `image_dining_table_family_v1` | Family dinner table |
| `image_phone_video_call_friend_v1` | Video call with friend |
| `image_text_message_screen_v1` | Phone with text messages |
| `image_movie_theater_v1` | Movie theater |
| `image_concert_crowd_v1` | Concert crowd |
| `image_grocery_store_v1` | US grocery store |
| `image_neighborhood_street_v1` | Suburban street |
| `image_apartment_building_v1` | US apartment building |
| `image_dating_app_screen_v1` | Generic dating app UI |
| `image_holiday_dinner_v1` | Holiday dinner table |
| `image_birthday_party_v1` | Birthday party |
| `image_funeral_respectful_v1` | Funeral (B2+ grief block) |

#### Generic / abstract (10 atomics)

| atomic_id | Descripción |
|-----------|-------------|
| `image_thought_bubble_neutral_v1` | Thought bubble icon |
| `image_clock_calendar_v1` | Clock and calendar generic |
| `image_world_map_us_centric_v1` | World map highlighting US |
| `image_pronunciation_mouth_diagram_v1` | Mouth/tongue position |
| `image_grammar_chart_tenses_v1` | Tenses chart illustration |
| `image_emotion_emoji_set_v1` | Generic emotion icons |
| `image_arrow_progression_v1` | Progress arrows |
| `image_decision_branching_v1` | Y-fork decision |
| `image_speech_bubble_pair_v1` | Two people with bubbles |
| `image_idea_lightbulb_v1` | Lightbulb idea generic |

#### Cultural reference (5 atomics)

| atomic_id | Descripción |
|-----------|-------------|
| `image_us_flag_v1` | US flag |
| `image_thanksgiving_dinner_v1` | Thanksgiving table |
| `image_super_bowl_party_v1` | Super Bowl viewing party |
| `image_us_road_trip_v1` | Iconic US road trip |
| `image_nyc_skyline_v1` | NYC skyline |

**Total scene image atomics: 70.**

---

## 8. Generic phrase atomics (~60 atomics)

### 8.1 Filosofía

Phrases comunes (no character-bound) usadas across muchos blocks y
characters. Voz neutral.

### 8.2 Listado por intent

#### Filler phrases (12 atomics)

Voces: 6 voces alternando (`voice_us_f_25_warm`,
`voice_us_m_28_friendly`, `voice_us_f_32_professional`,
`voice_us_m_35_calm`, `voice_us_f_40_neutral`, `voice_us_m_45_neutral`).

| atomic_id | Texto | duration_ms |
|-----------|-------|------------:|
| `audio_filler_um_v1` | "um..." | 400 |
| `audio_filler_uh_v1` | "uh..." | 400 |
| `audio_filler_well_v1` | "well..." | 500 |
| `audio_filler_so_v1` | "so..." | 400 |
| `audio_filler_let_me_think_v1` | "let me think..." | 1100 |
| `audio_filler_let_me_see_v1` | "let me see..." | 1000 |
| `audio_filler_thats_a_good_question_v1` | "that's a good question" | 1500 |
| `audio_filler_how_should_i_say_v1` | "how should I say this..." | 1600 |
| `audio_filler_kind_of_v1` | "kind of..." | 700 |
| `audio_filler_sort_of_v1` | "sort of..." | 700 |
| `audio_filler_in_a_way_v1` | "in a way..." | 800 |
| `audio_filler_actually_v1` | "actually..." | 800 |

#### Greeting / opener (8 atomics)

| atomic_id | Texto |
|-----------|-------|
| `audio_greeting_hello_neutral_v1` | "Hello" |
| `audio_greeting_hi_neutral_v1` | "Hi" |
| `audio_greeting_good_morning_v1` | "Good morning" |
| `audio_greeting_good_afternoon_v1` | "Good afternoon" |
| `audio_greeting_good_evening_v1` | "Good evening" |
| `audio_greeting_pleasure_to_meet_v1` | "Pleasure to meet you" |
| `audio_greeting_nice_to_meet_v1` | "Nice to meet you" |
| `audio_greeting_thanks_for_meeting_v1` | "Thanks for taking the time" |

#### Farewell / closer (6 atomics)

| atomic_id | Texto |
|-----------|-------|
| `audio_farewell_goodbye_v1` | "Goodbye" |
| `audio_farewell_have_a_nice_day_v1` | "Have a nice day" |
| `audio_farewell_take_care_v1` | "Take care" |
| `audio_farewell_see_you_later_v1` | "See you later" |
| `audio_farewell_talk_soon_v1` | "Talk soon" |
| `audio_farewell_have_a_good_one_v1` | "Have a good one" |

#### Confirm / acknowledge (8 atomics)

| atomic_id | Texto |
|-----------|-------|
| `audio_confirm_okay_v1` | "Okay" |
| `audio_confirm_got_it_v1` | "Got it" |
| `audio_confirm_makes_sense_v1` | "That makes sense" |
| `audio_confirm_understood_v1` | "Understood" |
| `audio_confirm_alright_v1` | "Alright" |
| `audio_confirm_perfect_v1` | "Perfect" |
| `audio_confirm_sounds_good_v1` | "Sounds good" |
| `audio_confirm_will_do_v1` | "Will do" |

#### Clarify / repair (8 atomics)

| atomic_id | Texto |
|-----------|-------|
| `audio_clarify_could_you_repeat_v1` | "Could you repeat that?" |
| `audio_clarify_sorry_missed_v1` | "Sorry, I missed that" |
| `audio_clarify_what_do_you_mean_v1` | "What do you mean by that?" |
| `audio_clarify_say_again_v1` | "Say that again, please" |
| `audio_clarify_louder_v1` | "Could you speak up a bit?" |
| `audio_clarify_slower_v1` | "Could you speak more slowly?" |
| `audio_clarify_im_not_sure_follow_v1` | "I'm not sure I follow" |
| `audio_clarify_one_more_time_v1` | "One more time, please" |

#### Turn taking (8 atomics)

| atomic_id | Texto |
|-----------|-------|
| `audio_turn_can_i_jump_in_v1` | "Can I jump in for a sec?" |
| `audio_turn_one_quick_thing_v1` | "Just one quick thing" |
| `audio_turn_before_we_move_on_v1` | "Before we move on..." |
| `audio_turn_to_add_to_that_v1` | "To add to that..." |
| `audio_turn_picking_up_on_v1` | "Picking up on what you said..." |
| `audio_turn_go_ahead_v1` | "Go ahead" |
| `audio_turn_after_you_v1` | "After you" |
| `audio_turn_im_sorry_interrupt_v1` | "I'm sorry to interrupt" |

#### Transition phrases (10 atomics)

| atomic_id | Texto |
|-----------|-------|
| `audio_transition_speaking_of_v1` | "Speaking of..." |
| `audio_transition_that_reminds_me_v1` | "That reminds me..." |
| `audio_transition_on_a_related_note_v1` | "On a related note..." |
| `audio_transition_changing_topic_v1` | "Changing the topic for a sec..." |
| `audio_transition_going_back_to_v1` | "Going back to what you said..." |
| `audio_transition_to_summarize_v1` | "To summarize..." |
| `audio_transition_first_of_all_v1` | "First of all..." |
| `audio_transition_finally_v1` | "Finally..." |
| `audio_transition_however_v1` | "However..." |
| `audio_transition_on_the_other_hand_v1` | "On the other hand..." |

**Total generic phrase atomics: 60.**

---

## 9. Generation pipelines

### 9.1 Resumen por categoría

| Categoría | Proveedor primario | Costo aprox por atomic | Tiempo |
|-----------|--------------------|-----------------------:|-------:|
| Audio TTS (character + generic) | ElevenLabs | $0.02-0.05 | 5-10s |
| Audio ambient | Freesound (CC) + edición | $0 (license) + 5-10 min edit | manual |
| Audio distortion overlays | DSP / FFmpeg | $0 | manual once |
| Image AI | DALL-E 3 / Midjourney | $0.04-0.08 | 10-30s |
| Video talking-head | HeyGen / Hedra | $0.50-1.50 | 1-3 min |

### 9.2 Audio TTS (ElevenLabs)

**Proveedor:** ElevenLabs (decisión cerrada — calidad superior para
ES-MX hispanohablantes input no aplica acá; voces nativas EN
incluidas en pool).

**Pipeline:**
1. Input: `voice_id` + `text`.
2. Call ElevenLabs API con `model_id = 'eleven_multilingual_v2'`.
3. Output: MP3 22050Hz mono.
4. Post-process: trim silence at ends, normalize -16 LUFS.
5. Upload to R2 con `storage_key = atomic_id + '.mp3'`.
6. Insert `media_atomics` row con `generated_by = 'ai_elevenlabs'`.

**Prompt template:**
```
text: "<exact text>"
voice_id: <voice_id from §3.2>
model_id: "eleven_multilingual_v2"
voice_settings:
  stability: 0.5
  similarity_boost: 0.75
  style: 0.4    # for character expressiveness
  use_speaker_boost: true
```

**Costo:** $0.02 per 1k chars (free tier 10k/month). Atomic
promedio 25 chars = ~$0.0005 per atomic. **240 character + 60
generic atomics × $0.0005 = $0.15 total.** Despreciable.

### 9.3 Image AI (DALL-E 3 primario)

**Proveedor:** OpenAI DALL-E 3 vía AI Gateway task `generate_image`
(ver `ai-gateway-strategy.md` §4).

**Pipeline:**
1. Input: prompt + size (`1024x1024` default).
2. Call DALL-E 3 con `quality: 'hd'`.
3. Output: PNG.
4. Post-process: optional crop, compress to WebP.
5. Upload to R2.

**Prompt template para character avatars:**
```
A {character.age_range} year old {character.gender}, {ethnicity_descriptor},
{character.description_visual}. {expression_modifier}. Photorealistic,
soft natural lighting, neutral background, professional portrait,
casual style. No text. Centered face, mid-body shot.
```

**Prompt template para scenes:**
```
{scene_type}, photorealistic, natural lighting, mid-day,
{cultural_region} setting, no people in frame (or generic background
people if scene requires), 16:9 composition, no text overlay.
```

**Determinismo:**
- Mismo character → mismo `generation_seed` para mantener consistency
  visual.
- Variantes (smile, thoughtful) → seed +1, +2, etc.
- Si DALL-E 3 no acepta seed, generar 4 variants con `n=4` y
  seleccionar manualmente la mejor (curaduría manual).

**Costo:** ~$0.04 per 1024x1024 HD. **70 scene + 30 character image =
100 × $0.04 = $4.** Despreciable.

### 9.4 Video talking-head (HeyGen / Hedra)

**Proveedor:** HeyGen (primario) o Hedra (alternativa) vía AI Gateway.

**Pipeline:**
1. Input: `character.default_video_avatar_id` (HeyGen avatar) +
   `text` + `voice_id`.
2. Call HeyGen API.
3. Output: MP4 1080p.
4. Post-process: convert to multiple bitrates (720p, 480p para
   conexiones lentas).
5. Upload to R2.

**Costo:** $0.50-1.50 per minute de video. **6 characters × ~3
videos × 30s avg = 9 minutos = $4.50 - $13.50.**

### 9.5 Audio ambient (manual + Freesound)

**Pipeline:**
1. Search Freesound.org con CC0 license filter.
2. Download .wav.
3. Edit en Audacity: trim a duración target, loop seamless si aplica.
4. Normalize -20 LUFS (background, no overpowering).
5. Convert a MP3.
6. Upload a R2.

**Costo:** $0 license + ~10 min de trabajo manual por atomic. **40
ambient × 10 min = ~7 horas content team.**

### 9.6 Distortion overlays (DSP / FFmpeg)

**Pipeline:**
1. Tomar audio source (puede ser cualquier voice atomic).
2. Aplicar FFmpeg filter chain:
   - Phone: `aresample=8000,aresample=22050,highpass=300,lowpass=3400`
   - Video call: light compression + slight reverb.
   - Etc.
3. Generar como overlay reusable.

**Costo:** $0 + setup once.

---

## 10. Seed migration plan

### 10.1 Sprint 1 (semana 1): voice pool + fundación

**Tareas:**
- [ ] Configurar ElevenLabs API + 12 voice_ids del pool en `voice_pool`
  table.
- [ ] Validar cada voice con 5 frases test (TESOL native rev).
- [ ] Configurar HeyGen account con 6 avatares mapeados a characters.
- [ ] Configurar DALL-E 3 vía AI Gateway.
- [ ] Configurar bucket R2 con prefix `atomics/v1/`.
- [ ] Migración SQL inicial: `media_atomics` + `characters` schemas.

### 10.2 Sprint 2 (semana 2): characters core (alex + sarah + grandma)

**Tareas:**
- [ ] Generar atomics audio + image + video para char_alex (33).
- [ ] Generar atomics para char_sarah (35).
- [ ] Generar atomics para char_grandma (20).
- [ ] Validation pedagógica + native review.
- [ ] Insertar en BD con `approved = true` post-review.

**Output:** ~88 atomics character-bound.

### 10.3 Sprint 3 (semana 3): characters restantes (mike + jamie + emma)

**Tareas:**
- [ ] Generar atomics para char_mike (25), char_jamie (30),
  char_emma (35).
- [ ] Validation + review.

**Output:** ~90 atomics adicionales (total 178).

### 10.4 Sprint 4 (semana 4): ambient + scene images

**Tareas:**
- [ ] Curar 40 atomics ambient audio (manual editing).
- [ ] Generar 70 scene images (DALL-E 3).
- [ ] Curaduría humana de imagen (rechazo + regeneration si quality
  issues).

**Output:** ~110 atomics adicionales (total ~290).

### 10.5 Sprint 5 (semana 5): generic phrase atomics + cleanup

**Tareas:**
- [ ] Generar 60 generic phrase atomics (ElevenLabs).
- [ ] Generar overlays de distorsión (FFmpeg setup).
- [ ] Auditoría de naming (verificar todos los IDs siguen §2.2).
- [ ] Verificar use_count tracker funciona (insertar test composites).
- [ ] Documentation final del seed.

**Output:** ~60 atomics adicionales (total ~350).

### 10.6 Total

```
350 atomics totales.
5 semanas content team (~200 horas a 40h/sem).
```

---

## 11. Costos totales

### 11.1 Generation costs (one-time)

| Categoría | Cantidad | Costo unitario | Subtotal |
|-----------|---------:|---------------:|---------:|
| Audio TTS (character + generic) | 240 | $0.0005 | $0.12 |
| Image AI (DALL-E 3 HD) | 100 | $0.04 | $4.00 |
| Video talking-head (HeyGen) | ~20 (6 chars × ~3 each) | $1.00 promedio | $20.00 |
| Audio ambient (Freesound CC) | 40 | $0 | $0 |
| Distortion overlays | 8 | $0 | $0 |
| **Total generation** | | | **~$24** |

### 11.2 Storage costs (mensual recurring)

```
350 atomics × ~500KB promedio = ~175 MB.
R2 storage: $0.015/GB/mes → ~$0.003/mes.
R2 bandwidth: gratis para deliveries.
```

**Despreciable.**

### 11.3 Human time

- Content team manual editing (ambient curation, image curation):
  ~30 horas.
- TESOL validation review: ~15 horas.
- Native speaker validation: ~10 horas.
- **Total ~55 horas humano.**

A $50/hora promedio: ~$2.750.

### 11.4 Total seed cost

```
Generation:           $24
Human time:           $2.750
Total:                ~$2.774
```

Comparado con el costo de generar 700 composites individualmente sin
reuse: ahorro estimado ~50% en assets ($1k+ ahorrados solo en
generation costs cuando se construyen los 175 bloques).

---

## 12. Decisiones cerradas

### 12.1 12 voces base en voice pool ✓

**Razón:** balance entre variedad (suficientes para 6 characters +
narrators + variantes) y manejabilidad (no se vuelve combinatorial
explosion). Post-MVP: agregar 4-6 voces UK / AU para variantes.

### 12.2 6 characters como seed inicial ✓

**Razón:** suficiente para cubrir los 3 tracks MVP con consistency.
Más characters al inicio diluye la memoria episódica del user.

### 12.3 ElevenLabs como TTS primario ✓

**Razón:** calidad superior vs Google/Amazon para casual conversational
register. Pricing aceptable. Multilingual v2 model permite agregar
es-MX voices en futuro si necesitamos AI characters que hablen
español.

### 12.4 DALL-E 3 como image generator primario, Midjourney secundario ✓

**Razón:** DALL-E 3 vía API es más controllable; Midjourney requires
manual workflow. DALL-E 3 también está integrado en AI Gateway.
Midjourney como fallback para casos donde DALL-E rechaza prompt o
calidad insuficiente.

### 12.5 HeyGen como video primario ✓

**Razón:** mejor para talking-head con voice cloning. Avatares
realistas. Hedra como alternativa post-MVP si pricing escala mal.

### 12.6 Freesound CC0 para ambient audio ✓

**Razón:** licencia clara, sin costo. Calidad suficiente para
background. Curado manual evita issues de audio no apropiado para
producción.

### 12.7 Versioning con _v<n> en ID ✓

**Razón:** permite evolucionar atomics sin romper composites
existentes. Cuando un atomic se mejora significativamente, _v2 se
crea y composites pueden migrar gradualmente.

### 12.8 No incluir post-MVP variants en seed ✓

**Razón:** scope creep. UK English variants, otros destinos travel,
specialized industry vocab → todo se genera on-demand cuando los
bloques que los necesitan se construyan.

---

## 13. Validación post-seed

### 13.1 Métricas a verificar al completar seed

| Métrica | Target |
|---------|-------:|
| Total atomics aprobados | ≥ 340 (97% del seed planeado) |
| Audio: % con duration_ms entre 0.5s y 30s | 100% |
| Audio: % normalizado a -16 LUFS ± 1 | ≥ 95% |
| Image: % aspect ratio 1:1 o 16:9 | 100% |
| Video: % en 1080p o 720p | 100% |
| Naming convention compliance | 100% |
| Voice_id assigned (audio only) | 100% |
| Character_id assigned (character-bound) | 100% |
| `approved = true` post-review | ≥ 95% |

### 13.2 Smoke tests

Tests automáticos a correr después del seed:
- Cada atomic ID es resoluble desde R2 (URL retorna 200).
- Cada audio se reproduce sin errors.
- Cada image renderiza correctamente.
- Cada video tiene audio sync OK (manual sample 10%).

---

## 14. Plan de extensión post-MVP

### 14.1 Fase 2: ampliar a A2 + C1

- Generar atomics para A2 entry-level y C1 advanced level.
- Estimado: +120 atomics.

### 14.2 Fase 2: variantes UK / AU

- 4-6 voces UK y AU adicionales en pool.
- Variantes de greeting, farewell, idioms.
- Estimado: +80 atomics.

### 14.3 Fase 3: characters adicionales

- 3-4 characters nuevos para tracks especializados (Business
  English, Tech, Healthcare).
- Estimado: +120 atomics.

### 14.4 Refresh schedule

- `dc_b2_meme_internet_culture` block: refresh cada 6 meses
  (memes envejecen rápido).
- Cultural reference atomics: revisión anual.

---

## 15. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`content-creation-system.md`](content-creation-system.md) §3 | Modelo atomic + composite + characters. |
| [`content-creation-system.md`](content-creation-system.md) §8.1 | Schema `media_atomics`. |
| [`content-creation-system.md`](content-creation-system.md) §8.2 | Schema `characters`. |
| [`track-job-ready-blocks.md`](track-job-ready-blocks.md) §10 | Atomics esperados para Job Ready. |
| [`track-travel-confident-blocks.md`](track-travel-confident-blocks.md) §10 | Atomics esperados para Travel Confident. |
| [`track-daily-conversation-blocks.md`](track-daily-conversation-blocks.md) §10 | Atomics esperados para Daily Conversation (referencia primaria de cross-track reuse). |
| [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) §4 | Tasks `generate_image`, `generate_audio_tts`, `generate_video`. |
| [`../explorations/multimedia-assets.md`](../explorations/multimedia-assets.md) | Detalles específicos de generación multimedia. |

---

*Documento vivo. Actualizar cuando se agreguen characters, se cambien
proveedores de generación, o se descubran patterns de atomics
faltantes en producción.*

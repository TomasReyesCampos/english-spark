# Exploración: Sporadic questions durante la fase "unknown"

> Micro-preguntas (5-30s) intercaladas en sesiones del trial Day 1-2
> que calibran el contenido en tiempo real y pre-arman data para el
> assessment formal del Day 3+. Bajo costo de fricción, alto valor de
> calibración.

**Status:** Exploración (no es spec final).
**Owner:** —
**Target de implementación:** Fase 1.5 (semanas 5-8 del MVP, después
del onboarding básico está sólido).
**Audiencia:** agente AI implementador + humano revisor.

---

## 1. Origen del concepto

### 1.1 Problema actual

Hoy en Day 1-2 (fase "unknown"):

- **Roadmap provisional** generado del onboarding mini-test de 3 min.
- Sistema captura `observed_behavior` (passive: WPM, pauses, fillers,
  exercises completed).
- **Assessment formal locked** hasta Day 3+ (`student-profile-and-assessment.md`
  §13.1).

**Gap:** entre Day 1-2 ofrecemos contenido al user con calibración
imprecisa. Si el user es muy distinto al "promedio del bucket", está
practicando contenido sub-óptimo durante 2 días.

### 1.2 Solución propuesta

**Sporadic questions:** micro-interacciones que ocurren **entre
ejercicios** (no durante) durante Day 1-2. Cada una toma 5-30s y
captura una pieza específica de calibración.

Ejemplos:

```
[Después de completar un ejercicio]

┌─────────────────────────────────────┐
│   Pregunta rápida:                  │
│                                     │
│   ¿Qué tan bien entiendes inglés    │
│   cuando lo escuchas?               │
│                                     │
│   😰 No entiendo casi nada           │
│   😟 Entiendo si hablan despacio    │
│   😐 Entiendo lo importante         │
│   😊 Entiendo bien casi todo        │
│   😎 Entiendo todo, incluso jerga   │
│                                     │
│   [Responder]    [Saltar]           │
│                                     │
└─────────────────────────────────────┘
```

```
[Otra sesión, otro día]

┌─────────────────────────────────────┐
│   Pregunta rápida:                  │
│                                     │
│   ¿Puedes leer esta frase en voz    │
│   alta?                             │
│                                     │
│   "Although the meeting was         │
│    productive, several issues       │
│    remained unresolved."            │
│                                     │
│   [▶ Grabar 10s]    [Saltar]        │
│                                     │
└─────────────────────────────────────┘
```

### 1.3 Beneficios

| Beneficio | Mecanismo |
|-----------|-----------|
| Calibración del contenido Day 1-2 | Sistema ajusta orden y dificultad en base a respuestas |
| Pre-data para assessment | Day 3+ ya viene con respuestas parciales → assessment más rico o más corto |
| Agency del usuario | "Nos importa lo que tú piensas", no solo medición pasiva |
| Detección de mismatch | Self-perception vs measured score → flag para assessment |
| Friction baja | Skip siempre disponible, max 1-2 por sesión |

---

## 2. Concept core

### 2.1 Definición

**Sporadic question** = un item del pool que se presenta al user en un
momento natural de pausa (post-ejercicio, app open con sesión ya
empezada), bajo throttling estricto, con opción de skip.

### 2.2 Diferencias con otras interacciones

| Aspecto | Sporadic question | Mini-test (onboarding) | Assessment formal |
|---------|-------------------|------------------------|-------------------|
| Duración | 5-30s | 3 min | 20-25 min |
| Frecuencia | 1-2/sesión, max 5-7 total | Una vez Day 0 | Una vez Day 3+ |
| Compromiso del user | Mínimo (skip allowed) | Onboarding mandatory | Day 5+ obligatorio |
| Scoring | Lightweight (signal incremental) | Sí (initial CEFR estimate) | Sí (definitive scoring) |
| Modifica roadmap | Ajuste suave | Genera roadmap provisional | Genera roadmap definitivo |

### 2.3 Cuándo se usan (timeline)

```
Day 0: Onboarding + mini-test (3 min)
       ↓
Day 1: 1-2 sporadic questions (entre ejercicios)
       ↓
Day 2: 1-2 sporadic questions
       ↓
Day 3-6: 0-1 sporadic questions (assessment ya disponible o
        próximo, foco cambia a assessment)
       ↓
Day 7+: Sporadic questions para re-calibration ocasional
        (1/semana max, opcional)
```

---

## 3. Categorías del pool

### 3.1 Self-assessment dimensional (~15 items)

Preguntas con escala 1-5 (con emojis para warmth) sobre dimensiones
específicas.

#### Listening self-perception

```
"¿Qué tan bien entiendes inglés cuando lo escuchas?"

😰 No entiendo casi nada (1)
😟 Entiendo si hablan despacio (2)
😐 Entiendo lo importante (3)
😊 Entiendo bien casi todo (4)
😎 Entiendo todo, incluso jerga (5)
```

#### Reading self-perception

```
"¿Cómo te sientes leyendo en inglés?"

😰 Necesito traducir cada palabra (1)
😟 Leo lento, traduzco mucho (2)
😐 Leo bien si es algo simple (3)
😊 Leo cómodo casi cualquier cosa (4)
😎 Leo igual que en español (5)
```

#### Speaking intelligibility (perceived)

```
"Cuando hablas inglés, ¿te entienden los demás?"

😰 Casi nadie me entiende (1)
😟 Tengo que repetir mucho (2)
😐 Me entienden si me escuchan bien (3)
😊 Me entienden casi siempre (4)
😎 Me entienden sin problema (5)
```

#### Speaking confidence

```
"¿Qué tan cómodo te sientes hablando inglés?"

(misma escala)
```

#### Vocabulary self-perception

```
"¿Cuántas palabras crees que sabes en inglés?"

[ ] Pocas, las básicas
[ ] Suficientes para situaciones simples
[ ] Muchas, manejo trabajo cotidiano
[ ] Bastantes, incluso técnicas de mi área
[ ] Muchísimas, casi todo lo necesario
```

#### Grammar self-perception

```
"¿Cómo te sientes con la gramática?"

[ ] Me cuesta mucho, me pierdo
[ ] Sé lo básico pero me trabo
[ ] Manejo lo principal con dudas
[ ] Bien, errores ocasionales
[ ] Muy bien, errores raros
```

#### Pronunciación self-perception

```
"¿Cómo crees que es tu pronunciación?"

😬 Me da vergüenza pronunciar (1)
😕 Tengo problemas con varios sonidos (2)
🙂 Pronuncio bien la mayoría (3)
😊 Buena pronunciación, acento ligero (4)
😎 Me confunden con nativo a veces (5)
```

#### Frustración primaria

```
"¿Qué te frustra más cuando intentas hablar inglés?"

[ ] No encontrar las palabras
[ ] Bloquearme con la gramática
[ ] No me entiendan por mi acento
[ ] No entender lo que me dicen
[ ] Mi velocidad (hablo muy lento)
[ ] Otra cosa
```

(Esta misma pregunta también está en Parte 4 del assessment formal —
acá se pregunta antes para tener data temprana.)

### 3.2 Micro audio captures (~10 items)

Frases cortas (5-15 palabras) que el user lee en voz alta. Análisis:
acento, claridad, pronunciation phonemes específicos.

#### Item: micro_pronunciation_th_phonemes

```
Lee esta frase en voz alta:

"I think these things are difficult to remember."

[▶ Grabar 10s]
```

**Targets phonemes:** /θ/, /ð/.

#### Item: micro_pronunciation_v_b

```
"Very well, very busy, very brave."
```

**Targets:** /v/ vs /b/ (interferencia hispana clásica).

#### Item: micro_pronunciation_vowels_long_short

```
"The ship and the sheep look similar from far away."
```

**Targets:** /ɪ/ vs /iː/.

#### Item: micro_speaking_short_intro

```
Habla 15 segundos sobre lo que hiciste hoy:

[▶ Grabar 15s]
```

**Mide:** fluidez en producción libre corta, WPM, fillers.

#### Item: micro_pronunciation_connected_speech

```
"Could you tell me where you live?"
```

**Targets:** linking, connected speech.

#### Item: micro_speaking_yes_no_question

```
Hazme una pregunta de sí o no en inglés sobre cualquier tema:

[▶ Grabar 8s]
```

**Mide:** intonation patterns, gramática de question forms.

#### Item: micro_listening_repeat

(Variante audio-input: user escucha frase corta y la repite.)

```
[▶ Reproducir audio]

"The annual report was thoroughly reviewed last week."

Ahora repite:

[▶ Grabar 10s]
```

**Mide:** listening + pronunciation conjuntamente.

### 3.3 Vocabulary quick checks (~10 items)

Una sola pregunta MC, ~5s.

```
"What does 'reach out' mean?"

[ ] To extend your arm
[ ] To contact someone ✓
[ ] To finish a project
[ ] To leave a place
```

(Calibrado a varios niveles CEFR. Pool diverso.)

### 3.4 Grammar quick checks (~10 items)

Igual, una pregunta de correct/wrong, ~5s.

```
Is this sentence correct?

"She's been working here since 5 years."

[ ] Yes, correct
[ ] No, has an error ✓ (correct: "for 5 years", not "since")
```

### 3.5 Goal validation (~5 items, low frequency)

```
"¿Sigues enfocada en mejorar tu inglés para entrevistas
laborales, o cambió tu objetivo?"

[ ] Sí, sigo enfocada en eso
[ ] Cambió, ahora me importa más: [select]
[ ] Tengo varios objetivos al mismo tiempo
```

(Una vez por trial, max. Se usa para detectar `significant_change`.)

---

## 4. Cuándo se preguntan (throttling)

### 4.1 Reglas duras

| Restricción | Valor |
|-------------|-------|
| Max sporadic questions por sesión | 2 |
| Max sporadic questions por día | 3 |
| Max sporadic questions Day 1-2 total | 5-7 |
| Min tiempo entre 2 sporadic questions de la misma sesión | 5 min |
| Min tiempo después de app open antes de la primera | 2 min |
| NO durante un ejercicio en curso | Hard rule |
| NO antes de que user complete su primer ejercicio del día | Hard rule |

### 4.2 Momentos óptimos para preguntar

1. **Post-ejercicio completado con éxito:** user está en momentum
   positivo, momentum micro-pausa antes del siguiente.
2. **Antes de cerrar la sesión voluntariamente** (tap "exit"): user
   ya decidió cortar; pregunta sumar 30s no daña la experiencia.
3. **Tras un logro desbloqueado:** celebración + pregunta natural.

### 4.3 Momentos a evitar

- **Mid-ejercicio:** rompe flujo, frustra.
- **Inmediato a app open:** se siente como gate.
- **Después de ejercicio fallido (score muy bajo):** user ya está
  frustrado, no agregar fricción.
- **Si user lleva > 30 min en sesión:** está cansado, terminar la
  sesión limpio.

### 4.4 Skip behavior

- "Saltar" siempre disponible.
- Skip no penaliza, ni reduce future questions.
- Skip se loggea (telemetry: `sporadic_question.skipped`) para detectar
  patrones (algunas preguntas pueden ser molestas y deberíamos retirarlas).

### 4.5 Throttle automático adaptativo

Si user salta 3+ sporadic questions consecutivas: pausar sporadic
questions por 24h. Indica que no quiere ese tipo de interacción.

---

## 5. Adaptive logic: cómo afectan el contenido

### 5.1 Respuestas → ajustes al roadmap provisional Day 1-2

Sporadic questions NO regeneran el roadmap (eso solo lo hace el
assessment formal). Pero pueden hacer **ajustes suaves**:

| Respuesta | Ajuste al contenido Day 1-2 |
|-----------|------------------------------|
| Listening self ≤ 2 | Priorizar bloques de listening más slow-speed |
| Listening self ≥ 4 | Permitir bloques de listening native-speed |
| Speaking confidence ≤ 2 | Empezar con drills, no roleplay free |
| Speaking confidence ≥ 4 | Permitir roleplay free temprano |
| Vocabulary self ≤ 2 | Más vocab in context exercises |
| Pronunciación self ≤ 2 | Más drill, menos free response |
| Frustración = "no encontrar palabras" | Vocab building blocks priorizados |

### 5.2 Mismatch detection

Si user dice self-perception alta pero scores observados son bajos
(o viceversa):

- Flag `mismatch_detected = true` en `student_profile`.
- Cuando el assessment formal corra, considerar la discrepancia:
  - Si self > measured: tono más reasurador en results screen.
  - Si self < measured: celebrar "estás más arriba de lo que creías"
    (caso del §3.5 en `post-assessment-flow.md`).

### 5.3 Pre-data para el assessment del Day 3+

Cuando se construye el assessment session (`assessment-content-bank.md`
§1.2), considerar las respuestas previas:

```typescript
// Si user ya respondió self-listening en sporadic Q
// → no preguntar de nuevo en Parte 1 del assessment
const sporadic_answers = await getSporadicResponsesByUser(user_id);
const skip_in_assessment = inferRedundantQuestions(sporadic_answers);

return {
  parte_1_questions: sampleProfileQuestions(userProfile, skip_in_assessment),
  // ...
};
```

Reduce duración del assessment de 20 min a 15-17 min sin perder
calidad de data.

### 5.4 Audio captures contribute to phoneme calibration

Las audio captures (§3.2) generan scores phoneme-level que entran a
`user_subskill_mastery` como **datos preliminares** (con flag
`source = 'sporadic'`). Esto:

- Permite que el roadmap inicial Day 1-2 ya tenga señal de
  pronunciation por fonema.
- Si user fue malo en /θ/ en una sporadic question → bloque de
  pronunciation /θ/ se prioriza.

---

## 6. Pool size y composición target

### 6.1 MVP

| Categoría | Items | Por CEFR level | Total |
|-----------|------:|---------------:|------:|
| Self-assessment dimensional | 7 (1 por dimensión + frustración + goal) | n/a | 7 |
| Micro audio captures | 10 | distribuidos | 10 |
| Vocabulary quick checks | 5 por CEFR × 4 levels | 5 | 20 |
| Grammar quick checks | 5 por CEFR × 4 levels | 5 | 20 |
| Goal validation | 1 (estática) | n/a | 1 |
| **Total MVP** | | | **~58** |

### 6.2 Fase 2

Expansión a 150+ items para evitar repetición en re-evaluations
periódicas.

---

## 7. Schema

### 7.1 Tabla `sporadic_questions` (pool)

```sql
CREATE TABLE sporadic_questions (
  id              TEXT PRIMARY KEY,             -- 'sq_self_listening_v1'
  category        TEXT NOT NULL CHECK (category IN (
                    'self_assessment',
                    'micro_audio',
                    'vocab_quick',
                    'grammar_quick',
                    'goal_validation'
                  )),

  -- Targeting
  cefr_level      TEXT,                          -- NULL si aplica a todos
  goal_relevance  TEXT[] DEFAULT '{}',           -- ['all'] o ['job_interview']
  target_dimension TEXT,                         -- 'listening', 'pronunciation', etc.

  -- Contenido
  content         JSONB NOT NULL,                -- shape varía por category

  -- Behavior config
  estimated_seconds INT NOT NULL,
  can_be_skipped  BOOLEAN NOT NULL DEFAULT true,
  weight_in_calibration NUMERIC(3,2) DEFAULT 0.5, -- 0-1, peso vs measured score

  -- Lifecycle
  approved        BOOLEAN NOT NULL DEFAULT false,
  archived        BOOLEAN NOT NULL DEFAULT false,
  use_count       INT NOT NULL DEFAULT 0,
  skip_count      INT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sq_category_cefr ON sporadic_questions(category, cefr_level)
  WHERE approved = true AND archived = false;
```

### 7.2 Tabla `sporadic_responses`

```sql
CREATE TABLE sporadic_responses (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  question_id     TEXT NOT NULL REFERENCES sporadic_questions(id),
  asked_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  responded_at    TIMESTAMPTZ,
  skipped         BOOLEAN NOT NULL DEFAULT false,

  -- Response data
  response_data   JSONB,                         -- shape varía por question

  -- Si es micro_audio, ref al atomic
  audio_atomic_id TEXT REFERENCES media_atomics(id),
  scoring_result  JSONB,                          -- phoneme scores, etc.

  -- Context
  session_id      TEXT,                          -- en qué sesión apareció
  trigger_context TEXT                           -- 'post_exercise', 'pre_session_close', 'achievement_unlock'
);

CREATE INDEX idx_sr_user_date ON sporadic_responses(user_id, asked_at DESC);
CREATE INDEX idx_sr_user_question ON sporadic_responses(user_id, question_id);
```

### 7.3 Modificación a `student_profiles`

Agregar campo:

```sql
ALTER TABLE student_profiles
  ADD COLUMN self_reported_dimensions JSONB DEFAULT '{}',
  ADD COLUMN sporadic_mismatch_flags TEXT[] DEFAULT '{}',
  ADD COLUMN sporadic_paused_until TIMESTAMPTZ;

-- self_reported_dimensions shape:
-- {
--   "listening": { "value": 3, "responded_at": "2026-05-01T..." },
--   "speaking_confidence": { "value": 4, "responded_at": "..." },
--   ...
-- }

-- sporadic_mismatch_flags ej:
-- ['listening_self_high_measured_low']
```

---

## 8. UX y mockups

### 8.1 Trigger: post-ejercicio

```
[User completa ejercicio con score 85]

┌─────────────────────────────────────┐
│         ✓ ¡Bien hecho!               │
│         Score: 85                    │
│                                     │
│   ─────────────────                  │
│                                     │
│   ✏️ Pregunta rápida (10s)           │
│                                     │
│   ¿Qué tan bien entiendes cuando    │
│   escuchas inglés?                  │
│                                     │
│   😰 😟 😐 😊 😎                       │
│                                     │
│   [ Responder ]    [ Saltar ]       │
│                                     │
└─────────────────────────────────────┘
```

### 8.2 Trigger: micro audio capture

```
┌─────────────────────────────────────┐
│   ✏️ Pregunta rápida (15s)           │
│                                     │
│   Lee esto en voz alta para ver     │
│   tu pronunciación:                 │
│                                     │
│   ┌───────────────────────────────┐ │
│   │ "Very well, very busy,        │ │
│   │  very brave."                 │ │
│   └───────────────────────────────┘ │
│                                     │
│   [▶ Grabar 10s]    [ Saltar ]      │
│                                     │
└─────────────────────────────────────┘
```

### 8.3 Confirmation post-respuesta

Sutil, no intrusivo:

```
┌─────────────────────────────────────┐
│              ✓ Gracias               │
│                                     │
│      Vuelvo a tu siguiente          │
│      ejercicio                      │
└─────────────────────────────────────┘
[Auto-dismiss en 1.5s]
```

### 8.4 Settings

User puede desactivar sporadic questions desde Settings:

```
Settings → Preguntas rápidas

[ON] Recibir preguntas esporádicas
     Te ayudan a recibir contenido mejor calibrado para ti.
     Máximo 1-2 por sesión.

     [ Pausar 1 semana ]    [ Desactivar ]
```

Default: ON.

---

## 9. Integration con sistemas existentes

### 9.1 `student-profile-and-assessment.md`

Agregar sección §X "Sporadic data collection durante el trial":

- Schema de `sporadic_questions` y `sporadic_responses`.
- Pre-data para assessment (§5.3 de este doc).
- Mismatch detection.

### 9.2 `ai-roadmap-system.md`

Agregar a §7 (adaptación nocturna):

- Sporadic responses como input adicional al ajuste suave del roadmap
  Day 1-2.
- No regenera, solo reordena bloques.

### 9.3 `pedagogical-system.md`

Audio captures generan scores phoneme-level con flag
`source = 'sporadic'`. Estos scores entran a `user_subskill_mastery`
con peso reducido (0.5x vs 1.0x de exercise normal).

### 9.4 `notifications-system.md`

NO usar sporadic questions como push. Solo in-app.

### 9.5 `assessment-content-bank.md`

§1.2 sampling logic: si hay `sporadic_responses` para self-perception,
skip preguntas redundantes en Parte 1 del assessment formal.

### 9.6 `cross-cutting/data-and-events.md`

Agregar eventos:
- `sporadic_question.shown`
- `sporadic_question.answered`
- `sporadic_question.skipped`
- `sporadic_question.mismatch_detected`

---

## 10. Decisiones a tomar antes de promover a spec

### 10.1 ¿Por qué no usar sporadic questions también en Day 7+?

**Pro:** mantener engagement, calibración continua.
**Contra:** después del assessment ya tenemos data formal completa;
sporadic agrega ruido marginal.

**Pendiente:** decidir si sporadic se mantiene como mecanismo
permanente (1 por semana max) o solo Day 1-2.

### 10.2 ¿Las audio captures cuestan Sparks?

**No en la propuesta actual.** Son data collection, no premium feature.
Pero usan AI Gateway (TTS para frase + STT del user + scoring).

**Costo estimado:** ~$0.01 por audio capture procesado. A 5 por user
en trial = $0.05. Aceptable.

**Pendiente:** confirmar que `sporadic` es free path en sparks-system,
no cobra.

### 10.3 ¿Throttle adaptativo más sofisticado?

**Propuesta actual:** si user salta 3+ consecutivos, pausa 24h.

**Alternativa:** ML que aprende qué tipos de sporadic questions
funcionan mejor para cada user (algunos prefieren self-assessment,
otros micro audio).

**Pendiente:** mantener simple en MVP, sofisticación post-MVP.

### 10.4 ¿Sporadic questions también para users premium pasados Day 7?

**Si SÍ:** es feature permanente, requiere settings.
**Si NO:** solo Day 1-2 (lifespan corto).

**Pendiente:** decidir scope final.

### 10.5 ¿Audio captures requieren consent explícito?

**Default actual:** consent_audio_processing del onboarding ya cubre.

**Pendiente:** verificar si las capturas sporadic requieren consent
adicional según privacy/legal review.

### 10.6 ¿Cómo manejar respuestas claramente fake?

Ej: user responde "5" a todas las preguntas de self-perception en 2s.

**Pendiente:** detección automática + reduce weight de las respuestas
en calibration.

---

## 11. Métricas de éxito

| Métrica | Target MVP |
|---------|-----------:|
| Response rate (vs skip) | > 60% |
| Skip rate sostenido por user | < 30% |
| Mismatch detection accuracy | > 70% (validado contra assessment) |
| Reducción de duración del assessment formal | -2 a -4 min |
| Conversión Day 1-2 → assessment voluntario Day 3 | +5-10% vs sin sporadic |
| User satisfaction post-trial | NPS no afectado negativo |

---

## 12. Plan de implementación si se promueve

### 12.1 Sprint A (semana 1)

- Schema `sporadic_questions`, `sporadic_responses`.
- 30 items de pool inicial.
- API contracts: `getSporadicQuestion`, `submitSporadicResponse`.

### 12.2 Sprint B (semana 2)

- UI components (modal post-exercise).
- Throttling logic (cron + state).
- Settings toggle.

### 12.3 Sprint C (semana 3)

- Audio capture pipeline (reuse `generate_audio_tts` para prompts +
  `score_pronunciation` para análisis).
- Mismatch detection.
- Telemetry events.

### 12.4 Sprint D (semana 4)

- Adaptive logic en ai-roadmap (reordenar Day 1-2 blocks por
  responses).
- Skip de preguntas redundantes en assessment formal.
- A/B test priority: con vs sin sporadic questions.

---

## 13. Riesgos

| Riesgo | Probabilidad | Mitigación |
|--------|:-----------:|-----------|
| User percibe sporadic como spam | Media | Throttle estricto + skip prominente + adaptive throttle |
| Self-reports son ruidosos / unreliable | Alta | Cross-validate con observed; weight reducido |
| Audio captures fallan technically | Media | Skip silencioso si STT falla; no bloquear flow |
| Aumenta complejidad del código sin payoff | Media | Métricas claras (reducción de assessment duration, mismatch detection) |
| Privacy concern por audio capture extra | Baja | Cubierto por consent existente |

---

## 14. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`../product/student-profile-and-assessment.md`](../product/student-profile-and-assessment.md) §5, §6 | Trial y assessment. Sporadic complementa. |
| [`../product/assessment-content-bank.md`](../product/assessment-content-bank.md) §1.2 | Pre-data para sampling del assessment. |
| [`../product/ai-roadmap-system.md`](../product/ai-roadmap-system.md) §7 | Adaptación suave del roadmap Day 1-2. |
| [`../product/pedagogical-system.md`](../product/pedagogical-system.md) §3 | Scoring de audio captures (con peso reducido). |
| [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) §4.2 | Tasks reused: transcribe_user_audio, score_pronunciation. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) | Eventos `sporadic_question.*` a agregar. |

---

*Documento de exploración. Promover a specs (modificando los docs
de §9) cuando esté validado el approach con el dueño.*

# Onboarding Flow

> Detalle completo de pantallas, copy y comportamiento desde "user
> abre la app por primera vez" hasta "user empieza su primer
> ejercicio". Todo copy en español mexicano con tuteo.

**Estado:** Diseño v1.0
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador (UI + copy) +
diseñador.
**Alcance:** Onboarding del Day 0 (también primer login después de
registro abandonado).

---

## 0. Cómo leer este documento

- §1 establece **visión general** del flujo.
- §2-§14 detallan cada pantalla con mockup, copy literal,
  comportamiento.
- §15 cubre **edge cases**.
- §16 cubre **variantes** (anonymous, detected country override).
- §17 cubre **telemetry**.
- §18 cubre **decisiones cerradas**.

**Convención de copy:** español mexicano con tuteo (ver
`reglas.md` §4.7 y `i18n.md` §2.2). Todo en este doc YA está en
tuteo.

---

## 1. Visión general

### 1.1 Las 16 pantallas del flujo

```
1. Auth options (Google / Apple / Email / Anonymous)
        ↓
2. Welcome + saludo (post-auth)
        ↓
3-7. Pre-questions:
     3. Ubicación
     4. Objetivo principal (multi-select)
     5. Deadline
     6. Contexto profesional
     7. Autoevaluación + tiempo diario
        ↓
8. Permisos micrófono
        ↓
9. Intro mini-test
        ↓
10-12. Mini-test (3 ejercicios)
        ↓
13. Procesamiento (mientras se analiza)
        ↓
14. Roadmap inicial reveal
        ↓
15. Permisos push notifications
        ↓
16. Primer ejercicio intro
```

### 1.2 Tiempo total esperado

- Pantallas 1-7 (auth + questions): **~2 min**.
- Pantalla 8 (permisos): **~10s**.
- Pantallas 9-12 (mini-test): **~3 min**.
- Pantalla 13 (procesamiento): **~30s-1 min**.
- Pantallas 14-16 (reveal + cierre): **~1 min**.

**Total target: 5-7 min.**

### 1.3 KPIs críticos

(Ver `metrics-and-experimentation.md` §3.2.)

| Métrica | Target |
|---------|--------|
| % completion del onboarding (auth → primer ejercicio) | > 85% |
| Drop-off en cualquier paso individual | < 5% por pantalla |
| Tiempo desde abrir app hasta primer ejercicio | < 30s post-auth a < 7 min total |
| % users que dan permiso de micrófono | > 90% |
| % users que dan permiso de push | > 70% |

### 1.4 Filosofía

- **Fricción mínima:** cada paso debe sentirse natural, no como gate.
- **Anonymous-first option visible:** "Empezar sin registrarme"
  prominente para los que dudan.
- **Pre-questions cortas:** max 3-7 preguntas, max 30s cada una.
- **Mini-test es la culminación, no obstáculo:** se siente como "vamos
  a conocernos", no como examen.
- **Tono cálido pero profesional:** trato adulto al usuario.

---

## 2. Pantalla 1: Auth options

### 2.1 Cuándo

Primera pantalla al abrir la app por primera vez (sin sesión
existente).

### 2.2 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│         [Logo de la app]            │
│                                     │
│                                     │
│   Empieza a hablar inglés hoy       │
│                                     │
│   Tu plan personalizado, lecciones  │
│   adaptadas a ti, conversación con  │
│   IA. Todo en 15 minutos al día.    │
│                                     │
│                                     │
│   ┌───────────────────────────────┐ │
│   │  🔍 Continuar con Google      │ │
│   └───────────────────────────────┘ │
│                                     │
│   ┌───────────────────────────────┐ │  <- iOS only
│   │   Continuar con Apple         │ │
│   └───────────────────────────────┘ │
│                                     │
│   ──────────  o  ──────────         │
│                                     │
│   📧  Continuar con email           │
│                                     │
│   ─────────────────────────         │
│                                     │
│   ¿Quieres probar primero?          │
│   [Empezar sin registrarme]         │
│                                     │
└─────────────────────────────────────┘
```

### 2.3 Copy específico

**Hero title:**
> Empieza a hablar inglés hoy

**Subtitle:**
> Tu plan personalizado, lecciones adaptadas a ti, conversación con
> IA. Todo en 15 minutos al día.

**Primary CTAs:**
- "Continuar con Google"
- "Continuar con Apple" (solo iOS)

**Secondary CTA:**
- "Continuar con email"

**Tertiary (anonymous):**
- "¿Quieres probar primero?"
- "Empezar sin registrarme"

### 2.4 Comportamiento

- Tap Google/Apple → SSO flow nativo → §3 si éxito.
- Tap email → modal con sign-up form (email + password + nombre) → §3.
- Tap "Empezar sin registrarme" → anonymous auth (Firebase) → §3.
  Banner en home después: "Guarda tu progreso registrándote" sutil.

### 2.5 Variantes

**Si user vuelve después de cerrar app sin registrarse:**

```
Bienvenido de vuelta

¿Quieres seguir donde te quedaste?

[Continuar como invitado]
[Crear cuenta para guardar mi progreso]
```

---

## 3. Pantalla 2: Welcome + saludo

### 3.1 Cuándo

Inmediatamente post-auth exitoso (cualquier método).

### 3.2 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│         👋  ¡Hola, María!           │
│                                     │
│                                     │
│   Vamos a conocernos en 5 minutos   │
│   para armar el plan perfecto       │
│   para ti.                          │
│                                     │
│                                     │
│   Te voy a hacer algunas preguntas  │
│   y al final una pequeña prueba     │
│   para entender tu nivel actual.    │
│                                     │
│                                     │
│              [Empezar →]            │
│                                     │
└─────────────────────────────────────┘
```

### 3.3 Copy

**Saludo:** "¡Hola, [nombre]!" (si SSO trajo nombre) / "¡Hola!"
(anonymous o email sin nombre).

**Promesa:**
> Vamos a conocernos en 5 minutos para armar el plan perfecto para ti.

**Qué viene:**
> Te voy a hacer algunas preguntas y al final una pequeña prueba para
> entender tu nivel actual.

**CTA:** "Empezar"

### 3.4 Comportamiento

- Tap "Empezar" → §4.
- No back. Esta pantalla no es dismisible (puede cerrar app pero
  vuelve a esta).
- Animación sutil del emoji 👋 (wave).

---

## 4. Pantalla 3: Ubicación

### 4.1 Cuándo

Tap "Empezar" desde §3.

### 4.2 Mockup

```
┌─────────────────────────────────────┐
│   1 de 5                            │
│   ━━━━━░░░░░░░░░░░░░░░               │
│                                     │
│   ¿Dónde estás?                     │
│                                     │
│   Esto nos ayuda a darte ejemplos   │
│   relevantes a tu día a día.        │
│                                     │
│                                     │
│   ┌─────────────┐  ┌─────────────┐  │
│   │   🇲🇽         │  │   🇦🇷         │  │
│   │   México    │  │  Argentina   │  │
│   └─────────────┘  └─────────────┘  │
│                                     │
│   ┌─────────────┐  ┌─────────────┐  │
│   │   🇨🇴         │  │   🇨🇱         │  │
│   │  Colombia   │  │   Chile     │  │
│   └─────────────┘  └─────────────┘  │
│                                     │
│   ┌─────────────┐  ┌─────────────┐  │
│   │   🇵🇪         │  │   🌎         │  │
│   │    Perú     │  │ Otro Latam  │  │
│   └─────────────┘  └─────────────┘  │
│                                     │
└─────────────────────────────────────┘
```

### 4.3 Copy

**Progress indicator:** "1 de 5"

**Pregunta:** "¿Dónde estás?"

**Subtítulo:** "Esto nos ayuda a darte ejemplos relevantes a tu día a
día."

**Opciones:** 6 países (México, Argentina, Colombia, Chile, Perú,
Otro Latam).

### 4.4 Comportamiento

- Pre-selección automática según `cf.country` del request.
  Si detectado, opción aparece **highlighted** con ring verde:
  "Detectamos que estás en México [highlighted]" — user puede
  confirmar o cambiar.
- Tap opción → fade transition a §5.
- Auto-avance al país pre-seleccionado tras 3s si user no
  interactúa (UX: si la detección es correcta, no hace falta tappear).

### 4.5 Variantes

**Si detección falla:**

```
1 de 5
━━━━━░░░░░░░░░░░░░░░

¿Dónde estás?

(No pudimos detectar tu ubicación
automáticamente.)

[Selecciona tu país de la lista]
```

**Si selecciona "Otro Latam":**

Modal expandido con lista completa:

```
Selecciona tu país:

[ Bolivia ]      [ Costa Rica ]
[ Cuba ]         [ Ecuador ]
[ El Salvador ]  [ Guatemala ]
[ Honduras ]     [ Nicaragua ]
[ Panamá ]       [ Paraguay ]
[ Rep. Dom. ]    [ Uruguay ]
[ Venezuela ]    [ Otro fuera Latam ]
```

---

## 5. Pantalla 4: Objetivo principal

### 5.1 Cuándo

Tap país desde §4.

### 5.2 Mockup

```
┌─────────────────────────────────────┐
│   2 de 5                            │
│   ━━━━━━━━━━░░░░░░░░░░               │
│                                     │
│   ¿Por qué quieres mejorar tu       │
│   inglés?                           │
│                                     │
│   Puedes elegir más de uno.         │
│                                     │
│                                     │
│   ☐ Conseguir trabajo en empresa    │
│      internacional                  │
│   ☐ Trabajo remoto desde donde estoy│
│   ☐ Comunicarme mejor en mi trabajo │
│      actual                         │
│   ☐ Viajar                          │
│   ☐ Estudios o intercambio          │
│   ☐ Examen (TOEFL, IELTS, etc.)     │
│   ☐ Crecimiento personal            │
│                                     │
│                                     │
│       [Continuar →]                 │
│                                     │
└─────────────────────────────────────┘
```

### 5.3 Copy

**Pregunta:** "¿Por qué quieres mejorar tu inglés?"

**Subtítulo:** "Puedes elegir más de uno."

**Opciones (multi-select):**
- Conseguir trabajo en empresa internacional → maps a `job_interview`
- Trabajo remoto desde donde estoy → `remote_work`
- Comunicarme mejor en mi trabajo actual → `business_communication`
- Viajar → `travel`
- Estudios o intercambio → `studies`
- Examen (TOEFL, IELTS, etc.) → `exam_prep`
- Crecimiento personal → `personal_growth`

### 5.4 Comportamiento

- User puede tappear múltiples opciones.
- "Continuar" disabled hasta que seleccione al menos 1.
- Tap "Continuar" → §6.

### 5.5 Variantes

**Si selecciona "Examen":**

Sub-pregunta inmediata:

```
¿Qué examen?

[ TOEFL ]
[ IELTS ]
[ Duolingo English Test ]
[ Cambridge (FCE/CAE/CPE) ]
[ Otro ]
```

Si responde, capturamos `exam_target`.

---

## 6. Pantalla 5: Deadline

### 6.1 Cuándo

Tap "Continuar" desde §5.

### 6.2 Mockup

```
┌─────────────────────────────────────┐
│   3 de 5                            │
│   ━━━━━━━━━━━━━━━░░░░░               │
│                                     │
│   ¿Tienes alguna fecha en mente?    │
│                                     │
│   Saber cuándo lo necesitas         │
│   nos ayuda a priorizar.            │
│                                     │
│                                     │
│   ⚪ Sí, en menos de 1 mes          │
│   ⚪ En 1 a 3 meses                 │
│   ⚪ En 3 a 6 meses                 │
│   ⚪ No tengo apuro                 │
│                                     │
│                                     │
│       [Continuar →]                 │
│                                     │
└─────────────────────────────────────┘
```

### 6.3 Copy

**Pregunta:** "¿Tienes alguna fecha en mente?"

**Subtítulo:** "Saber cuándo lo necesitas nos ayuda a priorizar."

**Opciones (single-select):**
- Sí, en menos de 1 mes → `deadline_urgent`
- En 1 a 3 meses → `deadline_soon`
- En 3 a 6 meses → `deadline_medium`
- No tengo apuro → `deadline_none`

### 6.4 Comportamiento

- Tap opción → fade a §7.
- Si selecciona "menos de 1 mes" o "1-3 meses", capturamos
  `has_deadline = true` y mostramos sub-prompt opcional:

```
¿Para qué exactamente?
(opcional, te puedes saltar)

[ Texto libre, max 100 chars ]

Ejemplo: "Entrevista en TechCorp el 5 de junio."
```

---

## 7. Pantalla 6: Contexto profesional

### 7.1 Cuándo

Tap continúe desde §6.

### 7.2 Mockup

```
┌─────────────────────────────────────┐
│   4 de 5                            │
│   ━━━━━━━━━━━━━━━━━━━━░               │
│                                     │
│   ¿En qué área trabajas o           │
│   estudias?                         │
│                                     │
│                                     │
│   ⚪ Tecnología                     │
│   ⚪ Salud                          │
│   ⚪ Finanzas                       │
│   ⚪ Ventas                         │
│   ⚪ Marketing                      │
│   ⚪ Educación                      │
│   ⚪ Servicios                      │
│   ⚪ Otra área                      │
│   ⚪ Aún estudiando                 │
│                                     │
│                                     │
│       [Continuar →]                 │
│                                     │
└─────────────────────────────────────┘
```

### 7.3 Copy

**Pregunta:** "¿En qué área trabajas o estudias?"

**Opciones (single-select):**
- Tecnología, Salud, Finanzas, Ventas, Marketing, Educación,
  Servicios, Otra área, Aún estudiando.

### 7.4 Comportamiento

- Tap opción → §8.
- "Otra área" muestra input opcional para describir.

---

## 8. Pantalla 7: Autoevaluación + tiempo diario

### 8.1 Cuándo

Tap opción desde §7. Esta es la última de las pre-questions, contiene
**dos sub-preguntas**.

### 8.2 Mockup

```
┌─────────────────────────────────────┐
│   5 de 5                            │
│   ━━━━━━━━━━━━━━━━━━━━━━━━━           │
│                                     │
│   ¿Cómo te sientes hablando         │
│   inglés?                           │
│                                     │
│   ⚪ 😰  Me bloqueo completamente    │
│   ⚪ 😟  Muy nervioso, evito hablar  │
│   ⚪ 😐  Nervioso pero hablo        │
│   ⚪ 😊  Hablo con errores pero fluyo│
│   ⚪ 😎  Hablo bien, quiero pulir   │
│                                     │
│   ─────────────────────────         │
│                                     │
│   ¿Cuánto tiempo puedes dedicar     │
│   al día?                           │
│                                     │
│   ⚪ 5 min  (mantenimiento)         │
│   ⚪ 10-15 min (avance constante)   │
│   ⚪ 20-30 min (avance rápido)      │
│   ⚪ +30 min (avance intensivo)     │
│                                     │
│       [Continuar →]                 │
│                                     │
└─────────────────────────────────────┘
```

### 8.3 Copy primera pregunta

**Pregunta:** "¿Cómo te sientes hablando inglés?"

**Opciones (single-select, mapeadas a `speaking_confidence` 1-5 +
`self_perceived_level`):**

| Emoji | Texto | confidence | CEFR mapeado |
|:-:|------|:--:|:--:|
| 😰 | Me bloqueo completamente | 1 | A2 |
| 😟 | Muy nervioso, evito hablar | 2 | A2/B1 |
| 😐 | Nervioso pero hablo | 3 | B1 |
| 😊 | Hablo con errores pero fluyo | 4 | B1+/B2 |
| 😎 | Hablo bien, quiero pulir | 5 | B2+/C1 |

### 8.4 Copy segunda pregunta

**Pregunta:** "¿Cuánto tiempo puedes dedicar al día?"

**Opciones:**
- 5 min (mantenimiento) → `daily_minutes_available = 5`
- 10-15 min (avance constante) → `daily_minutes_available = 15`
- 20-30 min (avance rápido) → `daily_minutes_available = 30`
- +30 min (avance intensivo) → `daily_minutes_available = 45`

### 8.5 Comportamiento

- "Continuar" disabled hasta que ambas preguntas tengan respuesta.
- Tap "Continuar" → §9 (permisos micrófono).

---

## 9. Pantalla 8: Permisos micrófono

### 9.1 Cuándo

Tap "Continuar" desde §8. **Crítico:** sin micrófono no podemos hacer
mini-test ni la mayoría del producto.

### 9.2 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│         🎙️                          │
│                                     │
│   Necesitamos tu micrófono          │
│                                     │
│                                     │
│   Para evaluar tu pronunciación y   │
│   ayudarte a mejorar tu speaking,   │
│   necesitamos grabar audios cortos  │
│   cuando hables en la app.          │
│                                     │
│   Estos audios:                     │
│                                     │
│   ✓ Se almacenan encriptados        │
│   ✓ Se borran automáticamente       │
│      después de 30 días             │
│   ✓ Puedes desactivar el            │
│      almacenamiento en cualquier    │
│      momento                        │
│                                     │
│                                     │
│       [Permitir micrófono]          │
│                                     │
│       [Más detalles]                │
│                                     │
└─────────────────────────────────────┘
```

### 9.3 Copy

**Hero:** "Necesitamos tu micrófono"

**Body:**
> Para evaluar tu pronunciación y ayudarte a mejorar tu speaking,
> necesitamos grabar audios cortos cuando hables en la app.

**Garantías (lista):**
- ✓ Se almacenan encriptados
- ✓ Se borran automáticamente después de 30 días
- ✓ Puedes desactivar el almacenamiento en cualquier momento

**Primary CTA:** "Permitir micrófono"

**Secondary:** "Más detalles" (abre privacy policy en webview)

### 9.4 Comportamiento

- Tap "Permitir" → trigger system permission prompt nativo.
  - Si user permite → §10.
  - Si user deniega → modal recovery (§9.5).
- Tap "Más detalles" → privacy policy en webview, vuelve al modal.

### 9.5 Recovery si deniega

```
┌─────────────────────────────────────┐
│         😕                          │
│                                     │
│   Sin micrófono no podemos hacer    │
│   gran parte del producto.          │
│                                     │
│   Lo que SÍ funciona sin micrófono: │
│                                     │
│   ✓ Lecciones de comprensión       │
│   ✓ Vocabulario                     │
│   ✓ Gramática                       │
│                                     │
│   Lo que NO funciona:               │
│                                     │
│   ✗ Práctica de pronunciación      │
│   ✗ Conversación con IA             │
│   ✗ Análisis de tu speaking         │
│                                     │
│   ─────────────────────             │
│                                     │
│   [Activar micrófono en Settings]   │
│                                     │
│   [Continuar sin micrófono]         │
│                                     │
└─────────────────────────────────────┘
```

User puede continuar sin micrófono con feature limitado, o deep-link a
Settings para activarlo.

---

## 10. Pantalla 9: Intro mini-test

### 10.1 Cuándo

Permisos otorgados desde §9.

### 10.2 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│   Última parte 🎯                   │
│                                     │
│                                     │
│   Te voy a pedir que hagas 3        │
│   pequeños ejercicios:              │
│                                     │
│   1️⃣  Repetir una frase              │
│   2️⃣  Responder una pregunta        │
│   3️⃣  Nombrar 3 imágenes             │
│                                     │
│   Toma 3 minutos en total.          │
│   No hay respuestas "correctas",    │
│   solo queremos escucharte.         │
│                                     │
│                                     │
│        [Empezar →]                  │
│                                     │
└─────────────────────────────────────┘
```

### 10.3 Copy

**Hero:** "Última parte 🎯"

**Body:**
> Te voy a pedir que hagas 3 pequeños ejercicios:

**Lista numerada:**
1. Repetir una frase
2. Responder una pregunta
3. Nombrar 3 imágenes

**Reassurance:**
> Toma 3 minutos en total. No hay respuestas "correctas", solo
> queremos escucharte.

### 10.4 Comportamiento

- Tap "Empezar" → §11 (primer ejercicio del mini-test).

---

## 11. Pantallas 10-12: Mini-test ejercicios

### 11.1 Mini-test ejercicio 1: Repetir frase

**Mockup:**

```
┌─────────────────────────────────────┐
│   Ejercicio 1 de 3                  │
│                                     │
│   Repite esta frase en voz alta:    │
│                                     │
│   ┌───────────────────────────────┐ │
│   │  "Thank you for the           │ │
│   │   opportunity to interview."  │ │
│   └───────────────────────────────┘ │
│                                     │
│   [▶ Escuchar pronunciación]        │
│                                     │
│   ─────────────────────             │
│                                     │
│   [🎙️ Grabar (10s)]                 │
│                                     │
│        [Saltar este]                │
│                                     │
└─────────────────────────────────────┘
```

**Copy:**
- Header: "Ejercicio 1 de 3"
- Instruction: "Repite esta frase en voz alta:"
- Phrase mostrada (selected del item bank, varía por user level
  inicial estimado).
- "Escuchar pronunciación" reproduce TTS de la frase con voz nativa.
- "Grabar (10s)" inicia recording.
- "Saltar este" disponible siempre.

**Comportamiento:**
- Audio se uploadea a R2.
- Llama `pedagogical.submitExerciseAttempt` con `block_id =
  'onboarding_minitest'`, `source = 'mini_test'`.
- Después de submit (success o skip): §11.2.

### 11.2 Mini-test ejercicio 2: Responder pregunta

**Mockup:**

```
┌─────────────────────────────────────┐
│   Ejercicio 2 de 3                  │
│                                     │
│   Escucha la pregunta y responde    │
│   en inglés (60s):                  │
│                                     │
│   [▶ Reproducir pregunta]           │
│                                     │
│   "How was your day yesterday?"     │
│                                     │
│   ─────────────────────             │
│                                     │
│   [🎙️ Grabar (60s)]                 │
│                                     │
│        [Saltar este]                │
│                                     │
└─────────────────────────────────────┘
```

**Copy:**
- "Escucha la pregunta y responde en inglés (60s):"
- "How was your day yesterday?" mostrado + audio play.
- "Grabar (60s)" timer cuenta regresiva.

**Comportamiento:**
- Mide fluidez espontánea, vocabulario activo, tiempos verbales.

### 11.3 Mini-test ejercicio 3: Nombrar imágenes

**Mockup:**

```
┌─────────────────────────────────────┐
│   Ejercicio 3 de 3                  │
│                                     │
│   Mira las imágenes y di qué son    │
│   en inglés:                        │
│                                     │
│   ┌───────┐  ┌───────┐  ┌───────┐  │
│   │ [img] │  │ [img] │  │ [img] │  │
│   │ café  │  │ libro │  │  reloj│  │
│   └───────┘  └───────┘  └───────┘  │
│                                     │
│   [🎙️ Grabar 15s]                   │
│                                     │
│        [Saltar este]                │
│                                     │
└─────────────────────────────────────┘
```

**Copy:**
- "Mira las imágenes y di qué son en inglés:"
- Tres imágenes con labels en español debajo (helper visual).
- "Grabar 15s" para responder oralmente.

**Comportamiento:**
- Mide vocabulario receptivo y producción inmediata.

---

## 12. Pantalla 13: Procesamiento

### 12.1 Cuándo

Tap finish del último ejercicio o auto-trigger después del último
recording.

### 12.2 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│     [Animación: pulso suave]        │
│                                     │
│      Armando tu plan…               │
│                                     │
│   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━     │
│                                     │
│   ✓ Analizando tus respuestas       │
│   ✓ Calibrando tu nivel             │
│   ⏳ Eligiendo tus primeras         │
│      lecciones                      │
│                                     │
└─────────────────────────────────────┘
```

### 12.3 Copy

**Hero:** "Armando tu plan…"

**Pasos visibles:**
- ✓ Analizando tus respuestas
- ✓ Calibrando tu nivel
- ⏳ Eligiendo tus primeras lecciones

### 12.4 Comportamiento

- Procesamiento backend: STT, scoring, LLM análisis,
  `generateInitialRoadmap`.
- Latencia esperada: 15-45s.
- Si > 60s: agregar mensaje "esto está tomando un poquito más de lo
  normal, no te vayas".
- Auto-transition a §13 al completar.

---

## 13. Pantalla 14: Roadmap inicial reveal

### 13.1 Cuándo

Procesamiento completado (§12).

### 13.2 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│       ✨ ¡Listo, María!             │
│                                     │
│                                     │
│   Detectamos que tu nivel está      │
│   alrededor de B1 y tu mayor        │
│   desafío es la fluidez.            │
│                                     │
│   ─────────────────────             │
│                                     │
│   📋 Tu plan inicial                │
│                                     │
│   📅 4 semanas                       │
│   📚 32 lecciones                    │
│   🎯 Foco: confianza al hablar      │
│                                     │
│   ─────────────────────             │
│                                     │
│   ⚡ 50 Sparks gratuitos             │
│      por 7 días para probar todo.   │
│                                     │
│   📅 El día 5 podrás hacer un       │
│      assessment más profundo que    │
│      ajustará tu plan.              │
│                                     │
│        [Vamos →]                    │
│                                     │
└─────────────────────────────────────┘
```

### 13.3 Copy

**Hero:** "✨ ¡Listo, [nombre]!"

**Diagnóstico generado:**
> Detectamos que tu nivel está alrededor de [CEFR] y tu mayor desafío
> es [dimension más débil].

(Generado vía AI Gateway task `generate_onboarding_summary` desde
`initial_test_results` + `self_perceived_level`.)

**Plan inicial:**
- 📅 [N] semanas
- 📚 [N] lecciones
- 🎯 Foco: [foco principal]

**Promesas del trial:**
- ⚡ 50 Sparks gratuitos por 7 días para probar todo.
- 📅 El día 5 podrás hacer un assessment más profundo que ajustará tu
  plan.

**CTA:** "Vamos"

### 13.4 Comportamiento

- Tap "Vamos" → §14 (push permissions).

---

## 14. Pantalla 15: Permisos push

### 14.1 Cuándo

Después del reveal del plan (§13).

### 14.2 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│         🔔                          │
│                                     │
│   ¿Te aviso cada día?               │
│                                     │
│                                     │
│   El inglés vive de la              │
│   consistencia. Te aviso una vez    │
│   al día a la hora que elijas.      │
│                                     │
│                                     │
│   Hora preferida:                   │
│                                     │
│      [ 7:00 PM  ▾ ]                 │
│                                     │
│                                     │
│       [Activar recordatorios]       │
│                                     │
│       [Después]                     │
│                                     │
└─────────────────────────────────────┘
```

### 14.3 Copy

**Hero:** "¿Te aviso cada día?"

**Body:**
> El inglés vive de la consistencia. Te aviso una vez al día a la hora
> que elijas.

**Selector:** "Hora preferida: [time picker]"

**Default:** 7:00 PM hora local.

**Primary:** "Activar recordatorios"
**Secondary:** "Después"

### 14.4 Comportamiento

- Tap "Activar" → trigger system push permission prompt nativo.
  Persistir hora preferida.
  - Si user permite → §15.
  - Si user deniega → §15 directo (puede activar después en
    Settings).
- Tap "Después" → §15.

---

## 15. Pantalla 16: Primer ejercicio intro

### 15.1 Cuándo

Después de §14 (push perm decision).

### 15.2 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│      Vamos a tu primera lección     │
│                                     │
│                                     │
│   Vas a empezar con algo cómodo:    │
│                                     │
│   ┌───────────────────────────────┐ │
│   │   Past Perfect en             │ │
│   │   storytelling                │ │
│   │                               │ │
│   │   ⏱ 12 minutos                │ │
│   │   🎯 Pronunciación + Gramática │ │
│   │                               │ │
│   │   Escucha, identifica el uso, │ │
│   │   y practica con un roleplay. │ │
│   └───────────────────────────────┘ │
│                                     │
│        [Empezar lección →]          │
│                                     │
│        [Después, gracias]           │
│                                     │
└─────────────────────────────────────┘
```

### 15.3 Copy

**Hero:** "Vamos a tu primera lección"

**Body:** "Vas a empezar con algo cómodo:"

**Card de la lección:**
- Título del primer block del roadmap.
- Tiempo estimado.
- Foco pedagógico.
- Descripción breve de qué hace.

**Primary:** "Empezar lección"
**Secondary:** "Después, gracias"

### 15.4 Comportamiento

- Tap "Empezar lección" → entra al flujo normal del producto (block
  page).
- Tap "Después, gracias" → home screen del producto en estado
  "trial Day 1, no exercise yet".

---

## 16. Edge cases

### 16.1 Auth

1. **User cierra app durante auth:** sesión persistida si Firebase
   completó. Próximo open vuelve a §3 (welcome) si user existe.
2. **Anonymous user pero device tiene cuenta no-anonymous existente:**
   default a la cuenta existente, ofrecer "iniciar sesión" o "empezar
   nuevo".
3. **Detección de país falla:** §4 muestra todos los países sin
   pre-selección.

### 16.2 Permisos

4. **User deniega micrófono:** §9.5 modal recovery; flow continúa con
   features limitadas.
5. **User deniega push:** flow continúa, mostrar banner sutil después
   "activa recordatorios para no perder tu racha".

### 16.3 Mini-test

6. **User skip los 3 ejercicios:** sistema usa solo respuestas a
   pre-questions para roadmap. Nivel inicial estimado por
   `self_perceived_level` solo. Roadmap más genérico pero válido.
7. **Audio del ejercicio falla (mic broken, network):** retry una vez;
   si falla, skip.
8. **User cierra app durante mini-test:** progreso preservado por
   24h. Vuelve al ejercicio que estaba haciendo.

### 16.4 Procesamiento

9. **Procesamiento toma > 60s:** mostrar "esto está tomando un
   poquito más, no te vayas".
10. **Backend falla:** fallback a roadmap template del bucket más
    cercano (CEFR + goal). Disclaimer en pantalla 14: "Tu plan inicial
    es genérico; lo personalizamos al hacer el assessment".

### 16.5 Drop-off

11. **User abandona en el medio:** estado preservado por 24h. Email
    "Tu onboarding está incompleto, ¿continuamos?" si tiene email.
12. **User vuelve después de 24h:** se reinicia onboarding desde §3
    (no auth otra vez).

---

## 17. Variantes

### 17.1 Anonymous user

- Skip de auth via §2 con "Empezar sin registrarme".
- §3 saluda con "¡Hola!" (no nombre).
- Banner sutil en §13 (post-roadmap-reveal): "Guarda tu progreso —
  solo toma 30s".
- Resto del flow idéntico.

### 17.2 User con goal `exam_prep`

§5 incluye sub-pregunta de qué examen.

§13 ajusta el diagnóstico:
> "Tu plan está enfocado en preparación para [examen]. 8 semanas de
> práctica intensiva."

### 17.3 User con país detectado correctamente

§4 auto-avanza tras 3s de mostrar el país pre-seleccionado.
Reduce 1 step.

### 17.4 User con anxiety alta (5 = "me bloqueo")

Pantallas posteriores con tono más calmante:

§13 incluye:
> "Recuerda: vas a tu propio ritmo. No hay presión."

§15 sugiere primer ejercicio de baja-presión:
> "Empezamos con algo simple: solo escuchar y elegir respuestas."

### 17.5 User que rechaza permisos críticos

Si rechaza micrófono Y push, mostrar mensaje en §15:

> "Para aprovechar todo el producto te recomendamos activar:
> - 🎙️ Micrófono (para análisis de pronunciación)
> - 🔔 Notificaciones (para mantener tu racha)
>
> Puedes activarlos en Settings cuando quieras."

---

## 18. Telemetry / events

(Detalle en `cross-cutting/data-and-events.md`.)

| Event | Cuándo |
|-------|--------|
| `onboarding.started` | Tap "Empezar" en §3 |
| `onboarding.country_selected` | Tap país en §4 |
| `onboarding.goals_selected` | Tap "Continuar" en §5 |
| `onboarding.deadline_selected` | Tap opción en §6 |
| `onboarding.context_selected` | Tap opción en §7 |
| `onboarding.self_assessment_completed` | Tap "Continuar" en §8 |
| `onboarding.mic_permission_granted` | User permite mic |
| `onboarding.mic_permission_denied` | User deniega mic |
| `onboarding.minitest_started` | Tap "Empezar" en §10 |
| `onboarding.minitest_exercise_completed` | Por cada ejercicio (1-3) |
| `onboarding.minitest_completed` | Último ejercicio submitted |
| `onboarding.processing_completed` | Backend retorna roadmap |
| `onboarding.roadmap_revealed` | §13 mostrada |
| `onboarding.push_permission_granted` | User permite push |
| `onboarding.push_permission_denied` | User deniega push |
| `onboarding.completed` | Tap "Empezar lección" en §15 |
| `onboarding.abandoned` | User cierra app sin completar (timeout 30 min) |

### 18.1 Funnel principal

```
auth_completed (100%)
    ↓
onboarding.started (98%)
    ↓
country_selected (97%)
    ↓
goals_selected (95%)
    ↓
deadline_selected (94%)
    ↓
context_selected (93%)
    ↓
self_assessment_completed (92%)
    ↓
mic_permission_granted (88%)
    ↓
minitest_completed (85%)
    ↓
roadmap_revealed (84%)
    ↓
onboarding.completed (82%)
```

Targets:
- `mic_permission_granted` ≥ 90% (crítico).
- `minitest_completed` ≥ 85% (crítico).
- `onboarding.completed` ≥ 80% (crítico).

---

## 19. Decisiones cerradas

### 19.1 Total de pre-questions: **5 pasos** ✓

**Razón:** balance entre data útil y fricción. 7+ pasos genera
drop-off, 3 pasos da info insuficiente.

### 19.2 Mini-test al final del onboarding (no antes): **Sí** ✓

**Razón:** user ya invirtió tiempo en preguntas; mini-test se siente
como culminación, no obstáculo. Dropoff bajo.

### 19.3 Permisos micrófono ANTES de mini-test: **Sí** ✓

**Razón:** sin mic el mini-test no funciona. Mejor preguntar
explícitamente con contexto que mid-flow.

### 19.4 Permisos push DESPUÉS del reveal: **Sí** ✓

**Razón:** user ve valor primero (su plan), después se pide push como
"para mantenerte". Higher conversion.

### 19.5 Anonymous auth como opción principal: **Sí, secundaria pero visible** ✓

**Razón:** reduce fricción para users que dudan. Mejora completion del
onboarding +5-10% según benchmarks de apps similares.

### 19.6 Pre-selección de país por IP: **Sí, con confirmation visual** ✓

**Razón:** ahorra 1 tap, no es invasivo, user puede cambiar fácil.

### 19.7 Skip de mini-test exercises permitido: **Sí** ✓

**Razón:** evita que user con mic broken / problema técnico abandone
todo el onboarding. Roadmap se genera con menos data pero válido.

---

## 20. Plan de implementación

### 20.1 Sprint 1 (semana 1)

- [ ] Pantallas 1-7 (auth + pre-questions) implementadas.
- [ ] Detección de país por IP.
- [ ] Validación de progreso (back/next).
- [ ] Telemetry events.

### 20.2 Sprint 2 (semana 2)

- [ ] Pantalla 8 (permisos micrófono) con recovery flow.
- [ ] Pantallas 10-12 (mini-test) con item pool inicial (~5 items por
  ejercicio).
- [ ] Audio recording + upload a R2.
- [ ] Integration con `pedagogical-system.submitExerciseAttempt`.

### 20.3 Sprint 3 (semana 3)

- [ ] Pantalla 13 (procesamiento) con estados de loading.
- [ ] Pantalla 14 (roadmap reveal) con generación AI.
- [ ] Pantalla 15 (push permissions).
- [ ] Pantalla 16 (primer ejercicio intro).
- [ ] Edge cases tests.

### 20.4 Sprint 4 (semana 4)

- [ ] Variantes (anonymous, exam_prep, anxiety alta).
- [ ] Persistencia de progreso 24h.
- [ ] Email de "onboarding incompleto" para registered users.
- [ ] A/B test: anxiety variants.

---

## 21. Métricas y A/B tests

### 21.1 A/B tests prioritarios

| Test | Hipótesis | Métrica primary |
|------|-----------|----------------|
| Pre-questions: 5 vs 7 vs 3 | Menos preguntas → menos dropoff sin perder calidad | `onboarding.completed` |
| Mic permission con vs sin lista de garantías | Garantías aumentan grant rate | `mic_permission_granted` |
| Roadmap reveal con números específicos vs genéricos | Específicos engagement más alto | `onboarding.completed` rate |
| Default push hora (7 PM) vs prompt al user | Asking aumenta engagement y permission grant | `push_permission_granted` |
| Mini-test 3 ejercicios vs 2 vs 4 | 3 es óptimo balance data vs friction | `minitest_completed` |

---

## 22. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §4 | Estructura del onboarding (este doc lo expande). |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §3 | Schema de `student_profiles` que se popula durante onboarding. |
| [`pedagogical-system.md`](pedagogical-system.md) §3 | Scoring del mini-test. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) §5 | `generateInitialRoadmap` post-processing. |
| [`../architecture/authentication-system.md`](../architecture/authentication-system.md) §3-§5 | Auth flow nativo (Google, Apple, Email, Anonymous). |
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) §6 | Push permissions setup. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.2 | Eventos `onboarding.*`. |
| [`reglas.md`](../reglas.md) §4.7 | Tono mexicano-tuteo. |
| [`../cross-cutting/i18n.md`](../cross-cutting/i18n.md) §2 | Convenciones de copy. |
| [`post-assessment-flow.md`](post-assessment-flow.md) | Flow paralelo del Day 7+. |

---

*Documento vivo. Actualizar cuando se observen patrones de drop-off
en el funnel, se A/B-testen variantes, o se cambien las pre-questions.*

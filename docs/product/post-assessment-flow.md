# Post-Assessment Flow

> Detalle completo de pantallas, copy y comportamiento desde "termina
> el último ejercicio del assessment" hasta "user paga / no paga".
> Esta es la zona de conversión más crítica del producto — copy y UX
> aquí mueven la aguja del LTV/CAC ratio.

**Estado:** Diseño v1.0
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador (UI + copy) +
diseñador.
**Alcance:** Flujo post-assessment del Day 7 (también re-evaluations
posteriores, con variantes mínimas).

---

## 0. Cómo leer este documento

- §1 establece **visión general** del flujo.
- §2-§10 detallan **cada pantalla** con mockup, copy, comportamiento.
- §11 cubre **path alternativo** si user no paga.
- §12 cubre **variantes** según perfil.
- §13 cubre **telemetry**.
- §14 enumera **edge cases**.
- §15 cubre **decisiones cerradas**.

**Convención de copy:** español mexicano con tuteo (ver
`reglas.md` §4.7 y `i18n.md` §2.2). Todo copy en este doc YA está en
tuteo.

---

## 1. Visión general

### 1.1 Las 8 pantallas del flujo

```
[Última pregunta del assessment]
            ↓
1. Procesamiento (5-15s)
            ↓
2. Reveal de resultado CEFR (emocional)
            ↓
3. Scores por dimensión (data)
            ↓
4. Strengths & weaknesses (insights)
            ↓
5. Roadmap reveal (tu plan personalizado)
            ↓
6. Paywall (3 planes)
            ↓
7a. Selección plan + pago         7b. "Continuar gratis"
            ↓                              ↓
8a. Welcome to plan          8b. Limited mode + re-engagement
```

### 1.2 Tiempo total esperado

- Pantallas 1-5 (results + roadmap): **~3 min**.
- Pantalla 6 (paywall): variable, **30s-2min**.
- Pantallas 7-8 (compra y welcome): **~1-2 min**.

Total post-assessment: **~5-7 min**.

### 1.3 KPIs críticos

(Ver `metrics-and-experimentation.md` §3.2.)

| Métrica | Target |
|---------|--------|
| % completion del flow (ver hasta paywall) | > 90% |
| % conversion en paywall | > 40% |
| Tiempo en paywall (mediana) | 45-90s |
| % re-conversion D14 (no convertidos) | > 5% |

### 1.4 Filosofía

- **Honesto, no manipulativo.** Mostramos resultados reales (incluyendo
  weaknesses), no ocultamos datos para vender más fuerte.
- **Pico de motivación.** Aprovechar el momento (user invirtió 20 min,
  está engaged) sin ser agresivo.
- **No cerrar la puerta.** User que no paga puede continuar usando
  preassets gratis indefinidamente.
- **Copy en tono cercano-profesional.** Tuteo mexicano-neutral.

---

## 2. Pantalla 1: Procesamiento (5-15s)

### 2.1 Cuándo

Aparece inmediatamente después de que el user completa el último
ejercicio del assessment.

### 2.2 Comportamiento técnico

- Cliente hace `submitAssessmentExercise(last_exercise)`.
- Backend procesa: STT + scoring + LLM análisis (vía AI Gateway).
- Latencia esperada: 5-15s. Si > 20s, mostrar mensaje extra.
- Cliente hace polling cada 2s a `getAssessmentStatus(assessment_id)`.

### 2.3 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│     [Animación: pulso suave]        │
│                                     │
│      Analizando tu inglés…          │
│                                     │
│   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━     │
│                                     │
│   ✓ Procesando audios               │
│   ✓ Analizando pronunciación        │
│   ⏳ Calibrando tu nivel…           │
│   ⏳ Armando tu plan personalizado  │
│                                     │
│                                     │
└─────────────────────────────────────┘
```

### 2.4 Copy variantes

**Si latencia normal (5-15s):**

```
Analizando tu inglés…

✓ Procesando audios
✓ Analizando pronunciación
⏳ Calibrando tu nivel
⏳ Armando tu plan personalizado
```

**Si > 20s (toma más de lo esperado):**

```
Esto está tardando un poquito más de lo normal.

No te preocupes, estamos siendo extra cuidadosos
con tu análisis.

[Mensaje cambia cada 5s entre frases motivadoras]
```

**Si > 60s (problema técnico):**

```
Estamos teniendo un retraso técnico.

Tu assessment se guardó perfectamente. Te avisaremos
en cuanto esté listo el análisis.

[Continuar después]
```

### 2.5 No-cerrable

User no puede dismissar esta pantalla. Background apps OK pero al
volver, sigue mostrando estado actual.

---

## 3. Pantalla 2: Reveal de resultado CEFR (emocional)

### 3.1 Cuándo

Backend retorna `assessment_results.measured_cefr_level`. Cliente
transiciona desde §2.

### 3.2 Comportamiento

- Animación de "drumroll": fade in del nivel grande.
- 3-4 segundos para el reveal completo.
- Tap "Ver mis scores" para continuar.

### 3.3 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│         🎉 ¡Listo, María!           │
│                                     │
│      Tu nivel actual de inglés:     │
│                                     │
│  ┌───────────────────────────────┐  │
│  │                               │  │
│  │           B1+                 │  │
│  │                               │  │
│  │      Umbral consolidado       │  │
│  │                               │  │
│  └───────────────────────────────┘  │
│                                     │
│   Estás más avanzada de lo que     │
│   muchos creen. Vamos a ver        │
│   exactamente cómo estás.          │
│                                     │
│       [Ver mis scores →]           │
│                                     │
└─────────────────────────────────────┘
```

### 3.4 Copy por CEFR resultado

**A2:**

```
Tu base está sentada. Vamos a construir
desde acá hacia donde quieres llegar.
```

**B1:**

```
Estás en el punto donde el inglés deja de
ser un obstáculo y empieza a ser una herramienta.
```

**B1+:**

```
Estás más avanzado/a de lo que muchos creen.
Vamos a ver exactamente cómo estás.
```

**B2:**

```
Tu inglés ya te alcanza para mucho. Ahora viene
la parte de pulirlo y volverlo natural.
```

**B2+:**

```
Tu inglés es sólido. Vamos a hacer que sea
indistinguible de natural.
```

**C1:**

```
Manejas el inglés a un nivel avanzado. Vamos a
refinarte para los matices que separan al fluido
del experto.
```

### 3.5 Edge: resultado contradice self-perceived

(Se determina al `determineCefrForAssessment`, ver
`student-profile-and-assessment.md` §13.2.)

Si user predijo más alto que el medido:

```
Tu nivel actual: B1+

Habías sentido que estabas en B2. Eso es muy
común — el inglés "del trabajo" es distinto al
inglés "del momento", y nuestra prueba mide
ambos.

[Ver mis scores →]
```

Si user predijo más bajo que el medido:

```
Tu nivel actual: B2

¡Sorpresa! Estás más arriba de lo que pensabas.
Tu inseguridad puede estar limitándote más que
tu nivel real.

[Ver mis scores →]
```

---

## 4. Pantalla 3: Scores por dimensión

### 4.1 Cuándo

Tap "Ver mis scores" desde §3.

### 4.2 Mockup

```
┌─────────────────────────────────────┐
│  ← Atrás        Tus scores       ⋯  │
│                                     │
│   Cómo te fue por habilidad:        │
│                                     │
│   📢 Pronunciación      ▓▓▓▓▓░░  72 │
│   💬 Fluidez            ▓▓▓▓░░░  65 │
│   📝 Gramática          ▓▓▓▓▓▓░  78 │
│   📚 Vocabulario        ▓▓▓▓▓░░  70 │
│   👂 Comprensión        ▓▓▓▓▓▓░  76 │
│                                     │
│   Promedio:              ▓▓▓▓▓░░  72│
│                                     │
│   [Ver qué significa cada uno]      │
│                                     │
│        [Continuar →]                │
│                                     │
└─────────────────────────────────────┘
```

### 4.3 Detalle expandible (tap "Ver qué significa cada uno")

```
┌─────────────────────────────────────┐
│  ← Volver         Detalle           │
│                                     │
│   📢 Pronunciación: 72              │
│                                     │
│   Cómo te suena tu inglés. Nos      │
│   fijamos en sonidos específicos    │
│   que son difíciles para            │
│   hispanohablantes:                 │
│                                     │
│   • /θ/ (think, three)         84 ✓ │
│   • /ð/ (this, mother)         71 ⚡ │
│   • /v/ vs /b/ (very vs berry) 65 ⚡ │
│   • Vocales largas vs cortas   78 ✓ │
│   • Stress en palabras         70 ⚡ │
│                                     │
│   ⚡ = áreas con oportunidad        │
│   ✓ = áreas que dominas             │
│                                     │
│   [Volver]                          │
└─────────────────────────────────────┘
```

(Análogo para otras 4 dimensiones.)

### 4.4 Comportamiento

- Animación de barras: llenan progresivamente al cargar (1.5s).
- Tap "Continuar" → §5.
- Tap "Ver qué significa cada uno" → modal con detalle por dimensión.

---

## 5. Pantalla 4: Strengths & weaknesses

### 5.1 Cuándo

Tap "Continuar" desde §4.

### 5.2 Mockup

```
┌─────────────────────────────────────┐
│  ← Atrás       Tu perfil        ⋯  │
│                                     │
│   💪 Lo que ya dominas:             │
│                                     │
│   ✓ Comprendes textos complejos     │
│     bien (78%)                      │
│   ✓ Tu pronunciación de /θ/ es      │
│     casi nativa                     │
│   ✓ Vocabulario business sólido     │
│                                     │
│                                     │
│   🎯 Tu mayor oportunidad:          │
│                                     │
│   📌 Fluidez en producción libre    │
│      (65/100)                       │
│                                     │
│   Hablas con seguridad cuando       │
│   tienes guion, pero te trabás      │
│   en conversaciones espontáneas.    │
│   Es lo más común en tu nivel.     │
│                                     │
│        [Y ahora, mi plan →]         │
│                                     │
└─────────────────────────────────────┘
```

### 5.3 Generación del contenido

- 3 strengths: top 3 sub-skills con score más alto.
- 1 weakness primary: sub-skill o dimensión con score más bajo
  *relativo* (no absoluto — si todas están altas, la "más baja"
  igual puede ser 80).
- Mensaje contextual generado vía AI Gateway task
  `generate_assessment_summary` (output validado contra schema Zod).

### 5.4 Tono

- Strengths: directo, factual, sin exagerar.
- Weakness: empático, normalizador ("es lo más común en tu nivel"),
  enfocado en oportunidad no en problema.

---

## 6. Pantalla 5: Roadmap reveal

### 6.1 Cuándo

Tap "Y ahora, mi plan" desde §5.

Background: `ai-roadmap.generateDefinitiveRoadmap` ya se llamó
durante §2 (procesamiento). Resultado está cacheado.

### 6.2 Mockup

```
┌─────────────────────────────────────┐
│  ← Atrás      Tu plan          ⋯   │
│                                     │
│  ✨ Tu plan personalizado            │
│                                     │
│   Para tu objetivo de               │
│   "entrevistas en inglés":          │
│                                     │
│   📅 14 semanas                      │
│   📚 68 lecciones específicas       │
│   🎯 Foco: confianza al hablar      │
│                                     │
│   Tu camino:                        │
│                                     │
│   ┌─────────────────────────────┐   │
│   │ ① Fundamentos de Job Ready  │   │
│   │   12 lecciones · 2 semanas  │   │
│   └─────────────────────────────┘   │
│   ┌─────────────────────────────┐   │
│   │ ② Conversaciones B1+        │   │
│   │   14 lecciones · 3 semanas  │   │
│   └─────────────────────────────┘   │
│   ┌─────────────────────────────┐   │
│   │ ③ Fluidez en interview     │   │
│   │   18 lecciones · 4 semanas  │   │
│   └─────────────────────────────┘   │
│        ↓ ver más niveles            │
│                                     │
│        [Empezar mi plan →]          │
│                                     │
└─────────────────────────────────────┘
```

### 6.3 Comportamiento

- Animación de aparición de niveles en cascada (1s total).
- Scroll para ver todos los niveles.
- Tap "Empezar mi plan" → §6 (paywall).
- Tap en cualquier nivel → preview de los blocks (read-only).

### 6.4 Nivel detail (al tap)

```
┌─────────────────────────────────────┐
│  ← Volver     Nivel 1              │
│                                     │
│  ① Fundamentos de Job Ready         │
│                                     │
│  Tu primer nivel está enfocado en   │
│  bases sólidas para conversaciones  │
│  laborales: vocabulario clave,      │
│  pronunciación específica, y        │
│  formato de respuestas STAR.        │
│                                     │
│  Lecciones (12):                    │
│                                     │
│  • Past Perfect en storytelling     │
│  • Vocabulario de carrera           │
│  • Pronunciación /θ/ y /ð/ pulida   │
│  • Roleplay: tell me about yourself │
│  • Fluidez con discourse markers    │
│  • ... (7 más)                       │
│                                     │
│  Tiempo estimado: ~2 semanas a      │
│  15 min/día.                        │
│                                     │
│        [Volver al plan]             │
│                                     │
└─────────────────────────────────────┘
```

### 6.5 Razón de mostrar el plan ANTES del paywall

User invierte interés en el plan → siente "esto es para mí" → más
abierto a pagar. Comparado con mostrar paywall primero (que mata
engagement con dinero antes de mostrar valor).

---

## 7. Pantalla 6: Paywall

### 7.1 Cuándo

Tap "Empezar mi plan" desde §6.

### 7.2 Mockup

```
┌─────────────────────────────────────┐
│  ← Atrás                   ⋯       │
│                                     │
│  Tu plan está listo. ¿Cómo lo      │
│  quieres practicar?                 │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  Plan Básico                │    │
│  │  $30 MXN /mes               │    │
│  │  ─────────────────          │    │
│  │  ✓ Tu plan completo          │    │
│  │  ✓ 30 Sparks/mes            │    │
│  │  ✓ Análisis semanal         │    │
│  │  ─────────────────          │    │
│  │  [ Elegir Básico ]          │    │
│  └─────────────────────────────┘    │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  ⭐ RECOMENDADO              │    │
│  │  Plan Pro                   │    │
│  │  $100 MXN /mes              │    │
│  │  ─────────────────          │    │
│  │  ✓ Todo lo del Básico        │    │
│  │  ✓ 200 Sparks/mes           │    │
│  │  ✓ Insights con IA          │    │
│  │  ✓ Re-evaluación cada 4 sem │    │
│  │  ─────────────────          │    │
│  │  [ Elegir Pro ]             │    │
│  └─────────────────────────────┘    │
│                                     │
│  ┌─────────────────────────────┐    │
│  │  Plan Premium               │    │
│  │  $250 MXN /mes              │    │
│  │  ─────────────────          │    │
│  │  ✓ Todo lo del Pro          │    │
│  │  ✓ 600 Sparks/mes           │    │
│  │  ✓ Roleplays personalizados │    │
│  │  ✓ Prioridad procesamiento  │    │
│  │  ─────────────────          │    │
│  │  [ Elegir Premium ]         │    │
│  └─────────────────────────────┘    │
│                                     │
│  💡 Cancelas cuando quieras.        │
│  Los Sparks comprados no expiran    │
│  por 6 meses.                       │
│                                     │
│  [No quiero pagar ahora]            │
│                                     │
└─────────────────────────────────────┘
```

### 7.3 Pricing localizado

(De `cross-cutting/i18n.md` §4.)

Cliente lee `student_profiles.country` para mostrar precios:
- México: $30 / $100 / $250 MXN.
- Argentina: $1.50 / $5 / $13 USD (pricing dinámico).
- Colombia: $7.000 / $22.000 / $55.000 COP.
- Etc.

Si web (no in-app): mostrar 15% descuento si plataforma + país lo
permiten:

```
Plan Pro:
  $5.00 USD/mes (pago in-app)
  $4.25 USD/mes (pago en web) ← Ahorras 15%
```

### 7.4 Element copy

#### "Recomendado" badge

Solo en Pro. Decisión basada en:
- Anchoring efectivo (user ve Pro como "el del medio").
- 200 Sparks/mes es suficiente para usuario activo.
- Premium parece overkill para mayoría.

#### "Cancelas cuando quieras"

Risk reversal explícito. Reduce ansiedad de compromiso.

#### "Sparks comprados no expiran por 6 meses"

Risk reversal sobre packs. Importante para users que dudan.

#### "[No quiero pagar ahora]"

Discreto pero accesible. NO oculto. Filosofía de "no cerrar la puerta".

### 7.5 Comportamiento

- Tap "Elegir [plan]" → §8 (selección + payment).
- Tap "No quiero pagar ahora" → §11 (path alternativo, free mode).
- Scroll: si user no ha tappeado en 30s, scroll automático suave para
  mostrar todos los planes.

### 7.6 Variantes por user

#### High anxiety user (`speaking_confidence ≤ 2`)

Agregar mensaje calmante arriba de los planes:

```
💛 Vas a tu propio ritmo. Cualquier plan que
elijas, no hay presión de avanzar más rápido
de lo que te sientes cómodo.
```

#### Trial Sparks runout (Day 3-4 caso)

```
Tus 50 Sparks gratuitos se acabaron pero tu
plan recién empieza. Para seguir avanzando:
```

#### Day 7+ con assessment ya completado

```
¡Excelente assessment! Tu plan está listo.
¿Cómo lo quieres practicar?
```

---

## 8. Pantalla 7a: Selección de plan + payment

### 8.1 Mobile (in-app purchase)

#### iOS (Apple Pay / Sign In)

```
┌─────────────────────────────────────┐
│                                     │
│   Plan Pro                          │
│   $100 MXN /mes                     │
│                                     │
│   [ Apple Pay ]                     │
│   [ Tarjeta ]                       │
│                                     │
│   Cancelas en cualquier momento     │
│   desde Configuración → Suscripciones │
│                                     │
└─────────────────────────────────────┘
```

#### Android (Google Pay)

Análogo con Google Pay.

#### Backend flow

1. User tap "Apple Pay".
2. Cliente llama Apple In-App Purchase.
3. Apple retorna `transaction_id`.
4. Cliente llama `POST /payments/verify-iap` con transaction_id.
5. Backend verifica con Apple Server-Side, llama
   `sparks-system.processPackPurchase` o
   `sparks-system.activateSubscription`.
6. Cliente recibe confirm.

### 8.2 Web (Stripe / MercadoPago)

```
┌─────────────────────────────────────┐
│                                     │
│   Plan Pro                          │
│   $4.25 USD /mes (15% off web)      │
│                                     │
│   Información de pago:              │
│                                     │
│   [ Tarjeta de crédito ]            │
│   [ MercadoPago ]                   │
│   [ PayPal ]                        │
│                                     │
└─────────────────────────────────────┘
```

### 8.3 Estados de payment

#### Pending

Modal con loading mientras se verifica:

```
Procesando tu pago…

Esto suele tomar unos segundos.
```

#### Success

```
✅ ¡Listo!

Tu Plan Pro está activo. Acabamos de
acreditar 200 Sparks a tu cuenta.

[Empezar a practicar →]
```

#### Failure

```
😕 Algo salió mal con tu pago

[Detalles del error específico, ej:
 "Tu tarjeta fue rechazada por tu banco.
 Probá con otra tarjeta o método."]

[Intentar de nuevo]
[Probar otro método]
[Contactar soporte]
```

### 8.4 Edge cases

- Tarjeta rechazada → mostrar razón general (no detalles del banco).
- Webhook de pago tarda > 60s → mostrar "Procesando, te avisamos en
  minutos. Tu plan se activará automáticamente cuando confirmemos."
- Doble tap en "Apple Pay" → idempotency en backend evita doble cobro.

---

## 9. Pantalla 8a: Welcome to plan

### 9.1 Cuándo

Después de pago exitoso (§8 success).

### 9.2 Mockup

```
┌─────────────────────────────────────┐
│                                     │
│       🎉 ¡Bienvenida a Pro!         │
│                                     │
│   Acabamos de:                      │
│                                     │
│   ✓ Activar tu Plan Pro             │
│   ✓ Acreditar 200 Sparks            │
│   ✓ Desbloquear tu plan completo    │
│   ✓ Activar insights semanales      │
│                                     │
│   ─────────────────────────         │
│                                     │
│   Tu primera lección está lista:    │
│                                     │
│   ┌─────────────────────────────┐   │
│   │ Past Perfect en storytelling │   │
│   │ ⏱ 12 minutos                │   │
│   │ 🎯 Pronunciación + Gramática │   │
│   └─────────────────────────────┘   │
│                                     │
│        [Empezar →]                  │
│                                     │
│   [Después, gracias]                │
│                                     │
└─────────────────────────────────────┘
```

### 9.3 Variantes

#### Si user pagó tras Day 7 (postergado mucho)

```
🎉 ¡Bienvenida a Pro, María!

Sé que te tomó pensarlo. Está bien — es una
decisión importante.

Tu plan completo está activo. Vamos cuando
quieras.
```

#### Si user pagó por error y se nota (poco probable)

(N/A — Apple/Google manejan esto en sus stores.)

---

## 10. Tras "Empezar" del welcome

> **Detalle completo del welcome flow post-paywall** (5 pantallas
> con copy literal mexicano-tuteo, algoritmo de selección del primer
> ejercicio, edge cases, variantes por perfil) vive en
> [`welcome-flow-post-paywall.md`](welcome-flow-post-paywall.md).
> Esa es la fuente de verdad para los primeros ~6-7 minutos
> post-pago, hasta que el user entra al flow normal del producto.

User entra al **welcome flow post-paywall** (5 pantallas):
1. Home premium reveal (animación + Sparks balance + plan badge).
2. First exercise intro (preparación con expectativas claras).
3. El ejercicio en sí (curado para max success: pronunciation drill
   preferido).
4. Success celebration (+ 5 Sparks bonus, +10 si Premium).
5. What's next (próximos 2 bloques del roadmap).

Después del welcome flow termina: user entra al **flujo normal del
producto**:
- Pantalla home con daily goal.
- Notificaciones activadas (preferences ya configuradas en
  onboarding).
- Roadmap navegable.
- Ejercicios disponibles.

(Detalle del producto en uso en otros docs: `ai-roadmap-system.md`,
`pedagogical-system.md`, etc.)

---

## 11. Path alternativo: user NO paga

### 11.1 Cuándo

Tap "No quiero pagar ahora" en §7 paywall, o cierre de la pantalla
sin pagar.

### 11.2 Pantalla de free mode

```
┌─────────────────────────────────────┐
│                                     │
│   Está bien. Vas a tu ritmo.        │
│                                     │
│   Lo que SIGUE disponible:          │
│                                     │
│   ✓ Acceso a todas las lecciones    │
│     pre-grabadas (preassets)        │
│   ✓ Tu plan personalizado visible   │
│   ✓ Streaks y logros                │
│                                     │
│   Lo que NO está disponible:        │
│                                     │
│   ✗ Conversación 1 a 1 con IA       │
│   ✗ Roleplays personalizados        │
│   ✗ Pronunciación con feedback IA   │
│   ✗ Insights semanales              │
│                                     │
│   ─────────────────────────         │
│                                     │
│   Cuando quieras desbloquear todo:  │
│                                     │
│   [Ver planes de nuevo]             │
│                                     │
│        [Continuar gratis]           │
│                                     │
└─────────────────────────────────────┘
```

### 11.3 Comportamiento

- Tap "Continuar gratis" → home con free mode.
- Tap "Ver planes de nuevo" → §7 paywall.
- En home: banner discreto "Desbloquea tu plan completo" siempre
  visible pero no intrusivo.

### 11.4 Re-engagement campaigns

(Coordinado con `notifications-system.md`.)

| Cuándo | Trigger | Mensaje |
|--------|---------|---------|
| Day 3 sin pago | Cron | "Llevas 3 días en modo gratis. Tu plan completo te espera." |
| Day 7 sin pago | Cron | "Esta semana mejoraste X% en pronunciación. Imagina si tuvieras conversación con IA." |
| Day 14 | Cron | "Te ofrecemos 30% de descuento tu primer mes. Solo por 48h." |
| Logro desbloqueado | Event-based | "¡Logro nuevo! Con Plan Pro hubieras ganado +30 Sparks bonus." |

### 11.5 Cuándo desistir

Después de **Day 30 sin conversion** y sin engagement:
- `trial_status = abandoned`.
- Re-engagement deja de enviar (circuit breaker, ver
  `notifications-system.md` §9.3).
- User puede volver cuando quiera; sus datos se mantienen.

---

## 12. Variantes por perfil del usuario

### 12.1 Por self-perceived anxiety

| Anxiety (1-5) | Tono general |
|:-:|------|
| 1-2 (cómodo) | Directo, foco en mejora rápida |
| 3 (medio) | Default — balanceado |
| 4-5 (alto) | Más calmante, normalizador, foco en proceso |

User con anxiety 5 ve frases extra como:
- "Vas a tu propio ritmo."
- "No hay presión."
- "Es lo más común en tu nivel."

### 12.2 Por edad

(Inferida desde `age_range` si declarado.)

| Edad | Ajustes |
|------|---------|
| 18-24 | Tono ligeramente más casual, emojis OK |
| 25-44 | Default profesional-cercano |
| 45+ | Menos emojis, lenguaje algo más formal |

### 12.3 Por país

(De `cross-cutting/i18n.md` §5.4.)

- México: vocabulario default. Mexicanismos suaves OK ("padrísimo" en
  feedback positivo, ej).
- Argentina: tuteo igual, sin mexicanismos. Tono universal.
- Colombia: tuteo + frases neutrales.
- Resto: tuteo + neutral.

### 12.4 Por goal primario

Mensajes contextualizados:

| Goal | Frase ejemplo en pantalla 5 (strengths/weaknesses) |
|------|----|
| `job_interview` | "Tu mayor oportunidad para conseguir esa entrevista..." |
| `travel` | "Tu mayor oportunidad para sentirte cómodo viajando..." |
| `daily_conversation` | "Tu mayor oportunidad para fluir con amigos..." |
| `studies` | "Tu mayor oportunidad para tu próximo curso..." |

---

## 13. Telemetry / events emitidos

(Detalle del shape en `cross-cutting/data-and-events.md` §5.2 y §5.5.)

| Event | Cuándo |
|-------|--------|
| `assessment.completed` | Al terminar último exercise (already documented) |
| `post_assessment.results_viewed` | Pantalla 2 mostrada |
| `post_assessment.scores_detail_expanded` | Tap "Ver qué significa cada uno" |
| `post_assessment.roadmap_viewed` | Pantalla 5 mostrada |
| `post_assessment.roadmap_level_expanded` | Tap en un nivel del plan |
| `paywall.viewed` | Pantalla 6 mostrada |
| `paywall.plan_selected` | Tap en "Elegir [plan]" |
| `paywall.dismissed` | Tap "No quiero pagar ahora" |
| `payment.attempted` | User inició payment flow |
| `payment.succeeded` | Webhook confirmó pago (de sparks-system) |
| `payment.failed` | Pago rechazado |
| `subscription.activated` | Plan activado post-pago |

### 13.1 Funnel principal

```
assessment.completed (100%)
   ↓
post_assessment.results_viewed (~98%)
   ↓
post_assessment.roadmap_viewed (~85%)
   ↓
paywall.viewed (~75%)
   ↓
paywall.plan_selected (~45%)
   ↓
payment.attempted (~42%)
   ↓
payment.succeeded (~38%)
```

Targets de optimización:
- `roadmap → paywall`: > 80% (si baja, paywall sale demasiado pronto).
- `paywall → plan_selected`: > 40% (conversion clave).
- `attempted → succeeded`: > 90% (technical issues si baja).

---

## 14. Edge cases (tests obligatorios)

### 14.1 Procesamiento

1. **Procesamiento toma > 60s:** mostrar mensaje de "tarda", permitir
   leave app y retornar via push notification cuando esté listo.
2. **Procesamiento falla (LLM down):** assessment se marca pending,
   user ve "te avisamos cuando esté listo", retry automático en
   background hasta 24h.
3. **User cierra app durante procesamiento:** push notification al
   completar.

### 14.2 Resultados

4. **CEFR resultado es null (failed scoring):** fallback a CEFR
   estimado del mini-test del onboarding. Disclaimer en pantalla.
5. **Score muy bajo (<30) en cualquier dimensión:** softening copy
   adicional ("estas dimensiones se construyen con el tiempo").
6. **Score muy alto (>95) en todas:** copy de "estás cerca de C1, te
   recomendamos retos avanzados".

### 14.3 Roadmap

7. **Roadmap generation falla:** mostrar plan template fallback (el
   más cercano por bucket goal+CEFR). Notificar via Sentry.
8. **User refresh durante roadmap reveal:** estado preservado
   (cached server-side por 24h).

### 14.4 Paywall

9. **User llega a paywall, abandona, vuelve dentro de 24h:** ve
   directo paywall sin re-mostrar resultados (already viewed).
10. **User en país no soportado para pricing:** fallback a USD.
11. **Apple/Google IAP rechazado por reason X:** mensaje específico
    según reason (insufficient funds, declined, etc.).

### 14.5 Payment

12. **Webhook de payment llega DESPUÉS de que user cerró app:** push
    notification "Tu Plan Pro está activo."
13. **User paga 2 veces (race condition):** idempotency en backend
    evita doble cobro. Refund automático del segundo si pasa.

### 14.6 Free mode

14. **User en free mode intenta usar feature premium:** modal "esta
    feature es de Plan Pro o superior. ¿Querés ver planes?".
15. **User en free mode usa preassets pesados:** no bloqueo, todos
    accesibles.

---

## 15. Decisiones cerradas

### 15.1 Mostrar plan ANTES del paywall: **Sí** ✓

**Razón:** user invierte interés en el plan → más abierto a pagar.
Mostrar paywall primero mata engagement con dinero antes de mostrar
valor.

### 15.2 "No quiero pagar ahora" visible en paywall: **Sí, discreto pero accesible** ✓

**Razón:** filosofía "no cerrar la puerta". User que no paga puede
volver. Si lo escondemos, generamos resentimiento.

### 15.3 Plan recomendado: **Pro** ✓

**Razón:** anchoring efectivo (user ve Pro como "el del medio").
200 Sparks/mes cubre user activo típico. Premium es overkill para
mayoría. Básico es starter para users con presupuesto restringido.

### 15.4 Re-engagement post-paywall-decline: **Sí, ramped** ✓

**Razón:** D3, D7, D14 con escalation suave. D14 con descuento. D30
desistir.

### 15.5 Pricing dinámico Argentina: **USD con conversion en runtime** ✓

(Coordinado con `i18n.md` §4.2.)

### 15.6 Variantes por anxiety / age / país: **Sí, en copy ajustes finos** ✓

**Razón:** marginal effort, marginal gain. No re-rendering completo;
solo strings change.

---

## 16. Plan de implementación

### 16.1 Sprint A (semana 1)

- [ ] Pantallas 1-5 (results flow) implementadas.
- [ ] Animaciones de reveal CEFR.
- [ ] Generación AI Gateway task `generate_assessment_summary`.
- [ ] Telemetry events §13.

### 16.2 Sprint B (semana 2)

- [ ] Pantalla 6 (paywall) con 3 planes.
- [ ] Pricing localizado por país.
- [ ] Variantes por anxiety/age/goal.

### 16.3 Sprint C (semana 3)

- [ ] Apple IAP integration.
- [ ] Google Pay integration.
- [ ] Stripe (web) integration.
- [ ] Pantalla 7 (payment states).
- [ ] Pantalla 8 (welcome).

### 16.4 Sprint D (semana 4)

- [ ] Path alternativo §11 (free mode).
- [ ] Re-engagement campaigns.
- [ ] Edge cases tests.
- [ ] A/B test priority: anxiety variants.

---

## 17. Métricas y A/B tests

### 17.1 A/B tests prioritarios

| Test | Hipótesis | Métrica primary |
|------|-----------|----------------|
| Plan recomendado: Pro vs Premium vs none | Pro highlighted aumenta conversion | paywall_to_plan_selected |
| Mostrar pricing dinámico Argentina vs fijo | Dinámico aumenta trust | conversion AR |
| Variant copy alta-anxiety | Tono calmante reduce churn alta-anxiety | conversion segmento alta-anxiety |
| "No quiero pagar" CTA visible vs hidden | Visible reduce conversion pero aumenta retención long-term | LTV |
| Roadmap reveal con preview de blocks vs solo niveles | Preview aumenta engagement con plan | paywall_viewed_to_plan_selected |

### 17.2 Cohortes a comparar

- Por país (MX vs AR vs CO).
- Por device (iOS vs Android vs Web).
- Por goal (job_interview vs travel vs daily).
- Por anxiety bucket (low/mid/high).
- Por nivel CEFR resultado.

---

## 18. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §6 | Estructura del assessment (input para este flow). |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §7 | Conversion al pago (overview, este doc lo expande). |
| [`assessment-content-bank.md`](assessment-content-bank.md) | Items concretos del assessment. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) §6 | `generateDefinitiveRoadmap` consumido en §2. |
| [`pedagogical-system.md`](pedagogical-system.md) §3 | Scoring del assessment. |
| [`../architecture/sparks-system.md`](../architecture/sparks-system.md) §8.4 | `processPackPurchase` post-pago. |
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) §4.4 | Re-engagement campaigns post-decline. |
| [`../cross-cutting/i18n.md`](../cross-cutting/i18n.md) §4 | Pricing localizado. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.2, §5.5 | Eventos `post_assessment.*`, `paywall.*`, `payment.*`. |
| [`../business/metrics-and-experimentation.md`](../business/metrics-and-experimentation.md) §3 | Conversion metrics. |
| [`reglas.md`](../reglas.md) §4.7 | Tono mexicano-tuteo. |

---

*Documento vivo. Actualizar cuando se observen patrones de drop-off
en el funnel, se A/B-testen variantes nuevas, o se cambien planes /
pricing.*

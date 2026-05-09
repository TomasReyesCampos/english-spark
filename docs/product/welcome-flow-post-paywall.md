# Welcome Flow Post-Paywall — Primer ejercicio post-conversion

> Spec del flujo desde "tap Empezar del welcome screen" (post-pago)
> hasta completar el primer ejercicio del plan pago. Este es el momento
> más crítico para validar la decisión de compra del user. Cada
> pantalla y cada copy está diseñada para que el user sienta:
> "tomé la decisión correcta."

**Estado:** Diseño v1.0 (listo para implementación)
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** desde §10 de `post-assessment-flow.md` ("Tras Empezar del
welcome") hasta el final del primer ejercicio post-pago.

---

## 0. Cómo leer este documento

- §1 establece **filosofía** del momento post-paywall.
- §2 cubre las **5 pantallas** del flujo.
- §3 cubre el **algoritmo de selección del primer ejercicio**.
- §4 cubre **copy literal mexicano-tuteo** de cada pantalla.
- §5 cubre **edge cases**.
- §6 cubre **telemetría / eventos emitidos**.
- §7 cubre **variantes por perfil del usuario**.
- §8 cubre **decisiones cerradas**.
- §9 cubre **plan de implementación**.
- §10 cubre **A/B tests prioritarios**.

---

## 1. Filosofía

### 1.1 Por qué el primer ejercicio post-pago importa más que cualquier otro

El user acaba de:
1. Completar 7 días de trial.
2. Hacer un assessment de 20 min.
3. Ver su roadmap personalizado.
4. **Pagar dinero real.**
5. Tomar la decisión emocional de comprometerse.

Su **expectativa está en el techo**. Si el primer ejercicio post-pago:
- Es genérico → "es lo mismo del trial, ¿por qué pagué?".
- Es demasiado fácil → "qué pago tan caro, esto era gratis".
- Es demasiado difícil → "no estoy listo, me bajo".
- Es buggy → cancelación inmediata + refund request.

Si el primer ejercicio:
- **Demuestra mejora vs trial:** sentir que el plan es superior.
- **Es retador pero alcanzable:** sentirse capaz, valorado.
- **Tiene feedback rico:** entender que la IA "ve" su nivel.
- **Termina con mini-victoria:** dopamina + commitment refuerzo.

→ user vuelve mañana. **Día 2 post-pago retention** es la métrica
crítica que predice 30-day retention.

### 1.2 Principios de diseño

1. **No es "el primer bloque del roadmap":** es un **ejercicio
   curado especial** que mostramos antes del flow normal.
2. **Optimizado para success:** seleccionado del nivel CEFR del user
   pero del lado fácil del rango (no del difícil).
3. **Showcase de IA:** debe involucrar feedback rico de IA (pronunciación
   con análisis fonema-por-fonema, o roleplay con character con
   personalidad).
4. **Termina con celebración explícita:** confetti + score + Sparks
   bonus.
5. **No requiere Sparks:** el primer ejercicio post-pago es free
   regalo (refund Sparks si los cobramos).

### 1.3 Lo que NO es

- **No es el primer bloque del roadmap.** Eso viene después.
- **No es un tutorial.** El user ya pasó tutorial en onboarding.
- **No es opcional.** Es parte obligatoria del welcome flow para
  garantizar primer success.
- **No es largo.** 5-7 minutos máximo.

---

## 2. Las 5 pantallas del flujo

### 2.1 Resumen

| # | Pantalla | Duración | Tipo |
|--:|----------|---------:|------|
| 1 | Home premium reveal | 10s | Visual + animación |
| 2 | First exercise intro | 20s | Copy explicativa |
| 3 | El ejercicio en sí | 4-6 min | Interactive |
| 4 | Success celebration | 30s | Resultado + Sparks bonus |
| 5 | "What's next" | 30s | Bridge al flow normal |

Total: ~6-7 min desde tap "Empezar" hasta home normal.

### 2.2 Pantalla 1: Home premium reveal

**Cuándo:** inmediatamente después de tap "Empezar" en
`post-assessment-flow.md` §9 (welcome to plan).

**Objetivo:** primera impresión del producto pago. Visualmente
distinta del trial (animación + glow + "premium" feel).

**Mockup:**

```
┌─────────────────────────────────────┐
│                                     │
│     ✨ ¡Bienvenido a tu plan! ✨     │
│                                     │
│   ┌─────────────────────────────┐   │
│   │                             │   │
│   │      [Animation: streak     │   │
│   │       counter starts 1,     │   │
│   │       confetti subtle]      │   │
│   │                             │   │
│   │      Día 1 de tu plan       │   │
│   │                             │   │
│   └─────────────────────────────┘   │
│                                     │
│   200 Sparks ⚡  •  Plan Pro 🌟     │
│                                     │
│   Vamos a empezar con un ejercicio  │
│   especial, hecho para ti.          │
│                                     │
│         [Empezar →]                 │
│                                     │
└─────────────────────────────────────┘
```

**Comportamiento:**
- Auto-advance permitido pero no automático (user tapea CTA).
- Animation: streak counter visible "Día 1" con confetti subtle.
- Sparks balance visible ("200 Sparks ⚡" para Plan Pro).
- Plan badge visible.

**Tap CTA:** → §2.3 (first exercise intro).

### 2.3 Pantalla 2: First exercise intro

**Cuándo:** después de tap CTA en §2.2.

**Objetivo:** preparar al user para el ejercicio sin abrumar.
Generar expectativa positiva.

**Mockup:**

```
┌─────────────────────────────────────┐
│                                     │
│        [Icon: target 🎯]            │
│                                     │
│   Tu primer ejercicio               │
│                                     │
│   Detectamos que tu mayor           │
│   oportunidad es {area}.            │
│                                     │
│   Vamos a trabajar eso ahora con    │
│   un ejercicio diseñado para tu     │
│   nivel B1+.                        │
│                                     │
│   ⏱️  5-7 minutos                    │
│   🎙️  Vas a hablar                  │
│   🎁  Sin costo de Sparks           │
│                                     │
│         [Empezar el ejercicio]      │
│                                     │
└─────────────────────────────────────┘
```

**Comportamiento:**
- `{area}` es la weakness primaria de
  `assessment_results.weakest_areas[0]` (ej: "fluidez al hablar").
- Nivel CEFR del user mostrado explícitamente.
- 3 expectativas claras: tiempo, formato, sin costo.

**Tap CTA:** → §2.4 (el ejercicio).

### 2.4 Pantalla 3: El ejercicio en sí

**Selección del ejercicio:** ver §3.

**UI:** la estándar de ejercicios del producto. Pero con flag
`is_first_paid_exercise = true` que activa:
- No se cobran Sparks (override de la lógica normal).
- LLM scoring tiene `priority = high` (mejor calidad de feedback).
- Audio quality validation más estricta (rechaza audio < 1s útil con
  retry sin penalización).

**Tipos preferidos (en orden de preferencia):**

1. **Pronunciation drill con feedback fonema-por-fonema** — showcase
   de Azure Pronunciation Assessment + IA. User ve gráfico de scores
   por fonema target.
2. **Free response corto (60s) sobre topic personal** — showcase de
   IA que entiende y responde con feedback estructurado.
3. **Roleplay structured con character recurrente** (preferiblemente
   `char_alex_friendly_friend` por ser warm + universal).

NO usar para primer ejercicio:
- Listening MC (passive, low engagement).
- Vocabulary in context (text-heavy, no demuestra IA).
- Capstones (demasiado largos).

### 2.5 Pantalla 4: Success celebration

**Cuándo:** completion del ejercicio.

**Objetivo:** celebración explícita + Sparks bonus + reinforcement.

**Mockup:**

```
┌─────────────────────────────────────┐
│                                     │
│       🎉  ¡Excelente!  🎉           │
│                                     │
│   Tu pronunciación de /θ/ mejoró    │
│   18% en este ejercicio.            │
│                                     │
│   Score:  ████████████░  82/100     │
│                                     │
│                                     │
│       + 5 Sparks de bienvenida ⚡    │
│                                     │
│   "Sigue así. Tu plan está          │
│   personalizado para llevarte de    │
│   B1+ a B2 en 14 semanas."          │
│                                     │
│         [Ver mi plan →]             │
│                                     │
└─────────────────────────────────────┘
```

**Comportamiento:**
- Confetti animation (más fuerte que cualquier success normal).
- Métrica concreta de mejora ("18%", "+5 Sparks").
- Cita corta del roadmap reveal (continuidad emocional).
- 5 Sparks bonus reales (acreditados a balance, queda en 205 si era
  Plan Pro de 200).

**Reglas:**
- Si score < 60 (raro pero posible): override a tono empático sin
  exageración. "Buen primer paso. Vamos a trabajar más en esto."
- Bonus Sparks mismo en todos los casos (no condicionado a score).

**Tap CTA:** → §2.6 (what's next).

### 2.6 Pantalla 5: What's next

**Cuándo:** después de §2.5.

**Objetivo:** transición suave al flow normal del producto. User
entiende qué viene después y se siente listo para usar la app por su
cuenta.

**Mockup:**

```
┌─────────────────────────────────────┐
│                                     │
│        [Icon: roadmap 🗺️]            │
│                                     │
│   Tu próximo paso                   │
│                                     │
│   Esta semana vamos a trabajar:     │
│                                     │
│   1. Bloque 1: {block_name}         │
│      ~15 min  •  Mañana             │
│                                     │
│   2. Bloque 2: {block_name}         │
│      ~13 min  •  Pasado mañana      │
│                                     │
│                                     │
│   Te aviso a las {hour} cada día,   │
│   como pediste.                     │
│                                     │
│       [Ir a mi plan]                │
│                                     │
└─────────────────────────────────────┘
```

**Comportamiento:**
- Mostrar los próximos 2 bloques del roadmap real.
- Hora del recordatorio del user (de
  `user_notification_preferences.preferred_reminder_hour`).
- CTA lleva al home normal del producto (`englishspark://home`).

**Tap CTA:** → home del producto. Welcome flow termina.

---

## 3. Algoritmo de selección del primer ejercicio

### 3.1 Inputs

```typescript
interface FirstExerciseSelectionInput {
  user_id: string;
  assessment_results: AssessmentResults;
  active_track: TrackId;
  cefr_level: string;
  weakest_areas: string[];
}
```

### 3.2 Lógica

```python
def select_first_paid_exercise(input):
    weakest_subskill = map_weakness_to_subskill(input.weakest_areas[0])
    candidate_blocks = filter_blocks(
        track=input.active_track,
        cefr=input.cefr_level,
        target_subskills_includes=weakest_subskill,
        is_first_block_friendly=True,
    )

    # Prefer types in this order
    PREFERRED_TYPES = [
        'pronunciation_drill',
        'free_response',
        'roleplay_structured',
    ]
    for type in PREFERRED_TYPES:
        candidates = [a for a in get_assets_in_blocks(candidate_blocks)
                      if a.type == type]
        if candidates:
            # Pick easier-end of CEFR range
            return min(candidates, key=lambda a: a.difficulty)

    # Fallback: any first asset of first compatible block
    return get_assets_in_blocks(candidate_blocks)[0]
```

### 3.3 Marcado de assets como `is_first_block_friendly`

Un asset es elegible para primer ejercicio post-pago si:
- `difficulty <= 5` (de 10).
- `estimated_minutes` entre 4 y 7.
- `target_subskills` no excede 2 (foco claro).
- No es un capstone block.
- `approved_for_production = true`.

### 3.4 Pre-warming

El backend pre-selecciona el ejercicio **al momento del payment
success** (no on-demand cuando user tapea Empezar). Esto evita lag.

```typescript
// Triggered by payment.succeeded event
async function preWarmFirstExercise(user_id: string) {
  const profile = await getProfile(user_id);
  const exercise = selectFirstPaidExercise(profile);
  await cache.set(
    `first_paid_exercise:${user_id}`,
    exercise.id,
    { ttl: 86400 } // 24h
  );
}
```

Cuando user tapea Empezar, se sirve desde cache instantáneamente.

---

## 4. Copy literal mexicano-tuteo de cada pantalla

### 4.1 Pantalla 1: Home premium reveal

| Elemento | Copy |
|----------|------|
| Hero | `✨ ¡Bienvenido a tu plan! ✨` |
| Sub-hero | `Día 1 de tu plan` |
| Stats line | `{sparks} Sparks ⚡  •  Plan {plan_name} 🌟` |
| Body | `Vamos a empezar con un ejercicio especial, hecho para ti.` |
| Primary CTA | `Empezar →` |

### 4.2 Pantalla 2: First exercise intro

| Elemento | Copy |
|----------|------|
| Hero | `Tu primer ejercicio` |
| Body | `Detectamos que tu mayor oportunidad es {area}.\n\nVamos a trabajar eso ahora con un ejercicio diseñado para tu nivel {cefr}.` |
| Bullet 1 | `⏱️  {minutes} minutos` |
| Bullet 2 | `🎙️  Vas a hablar` |
| Bullet 3 | `🎁  Sin costo de Sparks` |
| Primary CTA | `Empezar el ejercicio` |

**Variantes de `{area}`:**
- `pronunciation` → "tu pronunciación, especialmente del sonido /θ/" (ajustar fonema según data del assessment).
- `fluency` → "tu fluidez al hablar".
- `grammar` → "el uso de tiempos verbales".
- `vocabulary` → "ampliar tu vocabulario en contexto".
- `listening` → "tu comprensión de hablantes nativos".

### 4.3 Pantalla 4: Success celebration

| Elemento | Copy |
|----------|------|
| Hero | `🎉  ¡Excelente!  🎉` |
| Body principal | `Tu {dimension} mejoró {improvement_pct}% en este ejercicio.` |
| Score line | `Score:  {score_bar}  {score}/100` |
| Bonus line | `+ 5 Sparks de bienvenida ⚡` |
| Quote | `"Sigue así. Tu plan está personalizado para llevarte de {cefr_current} a {cefr_target} en {weeks} semanas."` |
| Primary CTA | `Ver mi plan →` |

**Variante para score < 60:**
| Elemento | Copy |
|----------|------|
| Hero | `Buen primer paso 💪` |
| Body principal | `Esto es exactamente lo que vamos a trabajar juntos.` |
| Quote | `"No te preocupes por el score. La práctica diaria es lo que cuenta. Tu plan está hecho para esto."` |
| Resto | (igual) |

### 4.4 Pantalla 5: What's next

| Elemento | Copy |
|----------|------|
| Hero | `Tu próximo paso` |
| Section title | `Esta semana vamos a trabajar:` |
| Bloque item | `{n}. Bloque {n}: {block_name}\n   ~{minutes} min  •  {when}` |
| Footer | `Te aviso a las {hour} cada día, como pediste.` |
| Primary CTA | `Ir a mi plan` |

**Variantes de `{when}`:**
- Bloque 1: `Mañana`
- Bloque 2: `Pasado mañana`
- Si user pidió 7 días: usar nombres de día ("El sábado", etc.).

---

## 5. Edge cases

### 5.1 Payment success pero user cierra app antes de welcome

- Payment exitoso.
- User no llegó a §2.1.

**Behavior:**
- Marcar `welcome_flow_pending = true` en student_profile.
- Próxima vez que abra app: redirect directo al §2.1 (no al home).
- Banner: "Tu plan ya está activo. Vamos a tu primer ejercicio →".

### 5.2 User cierra app durante §2.4 (mid-ejercicio)

- Audio uploaded → score parcial computed.
- Mark `first_paid_exercise_partially_completed = true`.
- Próxima sesión: NO repetir el ejercicio. Mostrar §2.5 directamente
  con el partial score. "Volviste, ¡bien hecho! Aquí está tu progreso
  hasta ahora."

### 5.3 Pre-warming falló (cache miss)

- Selection on-demand cuando user tapea Empezar.
- UI muestra loading "Preparando tu ejercicio..." (max 3s).
- Si selection toma > 3s: timeout → fallback a un asset hardcoded
  por CEFR + track (ver §3.5).

### 5.4 No hay assets `is_first_block_friendly` para ese CEFR + track

(Improbable si el catálogo está bien curado, pero edge case
defensive.)

- Fallback al primer asset del primer bloque del roadmap (no curado
  pero funcional).
- Log warning para que content team curate más assets `friendly`.

### 5.5 User pagó vía web pero está en mobile (cross-device)

- Web pay → email confirmation → user abre app móvil.
- App detecta plan activo desde Stripe webhook que sincronizó BD.
- App muestra welcome flow móvil normal.

### 5.6 Refund request mid-welcome-flow

- Improbable (5 min después del pago) pero posible si user cambia de
  opinión.
- Permitir cancelación durante primeros 14 días (Stripe estándar).
- UI no muestra refund button en welcome flow (avoid friction). Solo
  vía Settings → Account.

---

## 6. Telemetría / eventos emitidos

### 6.1 Eventos del flow

| Evento | Cuándo |
|--------|--------|
| `welcome_post_paywall.started` | User llega a §2.1 |
| `welcome_post_paywall.intro_seen` | Pantalla §2.2 mostrada |
| `welcome_post_paywall.exercise_started` | User tapea CTA en §2.2 |
| `welcome_post_paywall.exercise_completed` | User completa el ejercicio |
| `welcome_post_paywall.exercise_abandoned` | User cierra mid-ejercicio |
| `welcome_post_paywall.celebration_seen` | §2.5 mostrada |
| `welcome_post_paywall.bridge_seen` | §2.6 mostrada |
| `welcome_post_paywall.completed` | User tapea "Ir a mi plan" |
| `welcome_post_paywall.bonus_sparks_credited` | 5 Sparks acreditados |

### 6.2 Funnel principal

```
welcome_post_paywall.started        → 100%
welcome_post_paywall.exercise_started → ≥ 95% (target)
welcome_post_paywall.exercise_completed → ≥ 85% (target)
welcome_post_paywall.completed      → ≥ 80% (target)
```

### 6.3 Métricas críticas

| Métrica | Target |
|---------|-------:|
| Completion rate (started → completed) | ≥ 80% |
| Time-to-complete promedio | 6-7 min |
| First exercise score promedio | ≥ 70 |
| Day 2 retention de users que completaron welcome | ≥ 75% |
| Day 2 retention de users que NO completaron welcome | (medir) |
| Refund rate primeras 24h post-paywall | < 2% |

Si Day 2 retention de "no completaron welcome" es significativamente
menor: fortalecer el flow (más fricción para abandonar).

---

## 7. Variantes por perfil del usuario

### 7.1 Por anxiety (`student_profile.language_anxiety`)

#### Anxiety alta (4-5)

Tono más empático en §2.5 success.

**§2.5 Hero:** `Lo lograste. Sin presión 💚`
**§2.5 Body:** `Empezar es la parte más difícil, y ya lo hiciste.`

#### Anxiety baja (1-2)

Tono más high-energy.

**§2.5 Hero:** `🔥 ¡Lo rompiste! 🔥`
**§2.5 Body:** `Tu pronunciación de /θ/ mejoró {pct}%. ¡Vamos por más!`

### 7.2 Por country

Pricing display y bonus Sparks no varían. Sí varían referencias
culturales en quote del §2.5:

| País | Quote variant |
|------|---------------|
| MX | (default) |
| AR | (default; tuteo respetado) |
| CO | (default) |
| Otros | (default) |

### 7.3 Por plan elegido

| Plan | Bonus Sparks | Sparks display |
|------|-------------:|----------------|
| Básico | +5 | Balance: 35/30 (ya tienes los 30 del plan + 5 bonus) |
| Pro | +5 | Balance: 205/200 |
| Premium | +10 (premium tier extra) | Balance: 610/600 |

Premium recibe **+10 Sparks de bienvenida** vs +5 de Básico/Pro.
Refuerza valor del plan top.

### 7.4 Por goal primario

El `{area}` mostrado en §2.2 puede priorizar según goal:

| primary_goal | weakness preference order |
|--------------|---------------------------|
| `job_interview` | fluency > pronunciation > grammar |
| `travel` | listening > pronunciation > vocabulary |
| `business_communication` | grammar > vocabulary > fluency |
| `personal_growth` | (cualquiera, según assessment) |

---

## 8. Decisiones cerradas

### 8.1 Welcome flow obligatorio (no opcional) ✓

**Razón:** primer ejercicio post-pago es crítico para retention.
Permitir skip = perder la oportunidad de demostrar valor en pico de
motivación.

### 8.2 Sin costo de Sparks ✓

**Razón:** psicológicamente, cobrar el primer ejercicio post-pago
sería percibido como mezquino. Costo real para nosotros (~$0.05 IA)
es despreciable comparado con el lift en retention.

### 8.3 Tipo de ejercicio: pronunciation > free response > roleplay ✓

**Razón:** pronunciation drill demuestra IA visualmente (gráfico
fonema-por-fonema) — más "wow factor" que free response. Free
response segundo porque demuestra IA conversacional. Roleplay
tercero porque es más largo (riesgo de abandon en este momento
crítico).

### 8.4 Pre-warming en payment.succeeded ✓

**Razón:** evitar lag entre tap y delivery. Cache 24h.

### 8.5 +5 Sparks bonus universal (Básico/Pro), +10 Premium ✓

**Razón:** símbolo de bienvenida. Premium recibe +10 para reforzar
diferencia. No es ROI-significant pero psicológicamente refuerza.

### 8.6 Welcome flow distinto del onboarding flow ✓

**Razón:** son momentos pedagógicamente y emocionalmente distintos.
Onboarding = "presenta el producto". Welcome post-pago = "valida tu
decisión de compra". Conflar los flows diluye ambos.

### 8.7 Si abandona welcome, próxima sesión re-entra al welcome ✓

**Razón:** queremos que TODOS los users pagados tengan el primer
success. Si abandonó por X razón, re-ofrecer es la mejor segunda
oportunidad.

---

## 9. Plan de implementación

### 9.1 Sprint A (semana 1)

- Pantalla §2.1 (home premium reveal) con animation.
- Pantalla §2.2 (first exercise intro) con copy variants.
- Pantalla §2.5 (success celebration) base.
- Pantalla §2.6 (what's next) con next blocks display.
- Eventos básicos del funnel.

### 9.2 Sprint B (semana 2)

- Algoritmo §3 de selección.
- Pre-warming en payment.succeeded handler.
- Marking de assets `is_first_block_friendly`.
- Cache TTL + fallback logic.

### 9.3 Sprint C (semana 3)

- Variantes por anxiety/plan/goal (§7).
- Edge cases (§5).
- A/B test infra para §10.

### 9.4 Sprint D (semana 4)

- QA cross-device (mobile vs web payment).
- Refund flow integration.
- Métricas dashboards.

---

## 10. A/B tests prioritarios

| # | Test | Hipótesis | Métrica primaria |
|--:|------|-----------|------------------|
| 1 | Tipo de ejercicio: pronunciation vs free_response vs roleplay | Pronunciation maximiza completion | Completion rate |
| 2 | Bonus Sparks: +5 vs +10 vs 0 | +5 es sweet spot (no notado vs ignorado) | Day 2 retention |
| 3 | Mostrar score-quote vs solo score | Quote refuerza commitment | Day 7 retention |
| 4 | Welcome flow vs skip directo a home | Welcome flow significantly mejor en retention | Day 2 + Day 30 retention |
| 5 | Confetti animation vs subtle vs none | Subtle es mejor para anxiety alta | Completion + NPS |

Test #4 es el más crítico — establece si el welcome flow vale la
inversión de implementación. Run con 10% de users primero.

---

## 11. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`post-assessment-flow.md`](post-assessment-flow.md) §10 | Punto de entrada (post-welcome screen). |
| [`onboarding-flow.md`](onboarding-flow.md) | Flow paralelo en Day 0 (no post-pago). |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §6.4 | `assessment_results.weakest_areas` consumido en §3. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) | Roadmap mostrado en §2.6 (próximos bloques). |
| [`pedagogical-system.md`](pedagogical-system.md) §5 | Asset types disponibles para selección. |
| [`content-creation-system.md`](content-creation-system.md) §6 | Estructura de bloques + assets. |
| [`atomics-catalog-seed.md`](atomics-catalog-seed.md) §5.7 | Character `char_alex_friendly_friend` usado en roleplay fallback. |
| [`../architecture/sparks-system.md`](../architecture/sparks-system.md) | Bonus Sparks acreditados. |
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) | `preferred_reminder_hour` mostrado en §2.6. |
| [`push-notifications-copy-bank.md`](push-notifications-copy-bank.md) | Tono y conventions de copy. |

---

*Documento vivo. Actualizar cuando se rebalancee qué ejercicio es
mejor para primer success, cambien copys, o se agreguen variantes
post-MVP.*

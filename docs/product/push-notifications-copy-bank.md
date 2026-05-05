# Push Notifications Copy Bank

> Banco autoritativo de copys literales para todas las push
> notifications del MVP. Mexicano-tuteo, todas las variantes, todos los
> casos. Esta es la **fuente de verdad** que un agente AI puede pegar
> directamente como seed data o templates en los Workers.

**Estado:** Diseño v1.0 (listo para implementación)
**Última actualización:** 2026-05
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** MVP (15 notification_ids del §4.1 de notifications-system) + 80 logros del §6.2 de motivation-and-achievements.

---

## 0. Cómo leer este documento

- §1 establece **convenciones** (tono, longitud, variables, deeplinks).
- §2 cubre **daily_reminder** con templates por foco pedagógico.
- §3 cubre **streak_at_risk** con variantes por longitud.
- §4 cubre **onboarding** (welcome_d1, d3, d5, d7).
- §5 cubre **re-engagement** (inactivity_d3, d7, d14).
- §6 cubre **achievement_unlocked** con copy para los 80 logros.
- §7 cubre **level_completed**.
- §8 cubre **transactional** (low_sparks, pack_expiring, payment_failed, restriction_applied).
- §9 cubre **edge cases** (sin nombre, anonymous, multi-device, fallbacks).
- §10 cubre **reglas de personalización IA**.
- §11 cubre **backlog de A/B tests**.
- §12 cubre **decisiones cerradas**.

---

## 1. Convenciones

### 1.1 Tono y voz

- **Idioma:** español mexicano-tuteo (tú, no vos). No mexicanismos
  excluyentes (evitar "chido", "padrísimo", "neta"). Preferir
  vocabulario universal latam.
- **Persona:** segunda persona singular informal ("tú").
- **Tono base:** cálido, motivador, breve. Nunca culposo, nunca
  agresivo, nunca con urgencia falsa.
- **Emojis:** uno por título es OK. Cero o uno en cuerpo. Nunca >2 en
  un mismo push.
- **Mayúsculas:** título capitalize first letter only (no ALL CAPS, no
  Title Case).
- **Signos:** evitar `!!`, `??`, `…`. Un signo es suficiente.

### 1.2 Longitud

| Campo | Max chars | Razón |
|-------|----------:|-------|
| `title` | **50** | iOS lock screen trunca a ~40-50 según device |
| `body` | **90** | iOS preview trunca a ~110; dejamos margen |
| `body` (con expansión) | **180** | Si user expande la notif (long-press) |

Cada copy se valida contra estos límites en CI antes de mergear.

### 1.3 Variables disponibles

Las plantillas usan `{{var}}` con resolución en el Worker antes del
envío.

| Variable | Origen | Fallback si null |
|----------|--------|------------------|
| `{{name}}` | `users.display_name` (primer nombre) | `""` (omitir frase) |
| `{{streak}}` | `streaks.current` | `0` |
| `{{focus_today}}` | `roadmaps.today_focus` (foco pedagógico calculado batch nocturno) | `"tu plan de hoy"` |
| `{{minutes}}` | `student_profiles.daily_minutes_available` | `15` |
| `{{sparks_remaining}}` | `sparks_balances.current` | `0` |
| `{{sparks_total}}` | `sparks_balances.cycle_allotment` | — |
| `{{percent_remaining}}` | calculado | — |
| `{{achievement_name}}` | `achievements_catalog.name` | — |
| `{{achievement_sparks}}` | `achievements_catalog.sparks_reward` | — |
| `{{level_name}}` | `roadmap_levels.name` | — |
| `{{days_inactive}}` | calculado | — |
| `{{pack_expiry_days}}` | calculado | — |
| `{{recent_improvement}}` | dimensión + delta de últimos 14 días | `null` (omitir) |
| `{{tracker_metric}}` | "pronunciación de /θ/", "fluidez", etc. | `"tu inglés"` |

**Regla:** si una variable resulta `null` y no tiene fallback, el copy
debe poder leerse omitiendo la frase completa. Nunca renderizar
literalmente `{{var}}` al usuario.

### 1.4 Deeplinks

Todas las notifs incluyen `data.deeplink` que el cliente resuelve
al abrir.

| `notification_id` | Deeplink |
|-------------------|----------|
| `daily_reminder` | `englishspark://exercise/today` |
| `streak_at_risk` | `englishspark://exercise/today?reason=streak` |
| `welcome_d1` / `welcome_d3` | `englishspark://home` |
| `welcome_d5` | `englishspark://assessment/preview` |
| `welcome_d7` | `englishspark://assessment/start` |
| `inactivity_d3` / `d7` / `d14` | `englishspark://home?reason=comeback` |
| `level_completed` | `englishspark://achievements/recent` |
| `achievement_unlocked` | `englishspark://achievements/{{achievement_id}}` |
| `low_sparks` | `englishspark://sparks/buy` |
| `pack_expiring` | `englishspark://sparks/balance` |
| `payment_failed` | `englishspark://billing/retry` |
| `restriction_applied` | `englishspark://settings/restrictions` |

### 1.5 Categorías y opt-out

(Recap de `notifications-system.md` §4.2.)

| Categoría | Opt-out user | Override |
|-----------|:-:|----------|
| `reminder` | Sí | — |
| `achievement` | Sí | — |
| `onboarding` | Sí | — |
| `reengagement` | Sí | — |
| `transactional` | **No** (a menos que push global off) | — |

### 1.6 ID conventions

Cada copy tiene `copy_id` único: `{notification_id}.{variant}.{seq}`.
Ej: `daily_reminder.with_streak.01`.

Esto permite A/B test, tracking de open rate por variante y rotación.

---

## 2. `daily_reminder` (la más importante)

### 2.1 Reglas de selección

El batch nocturno (`generate_notification_content` task del AI Gateway)
elige un template basado en este árbol de decisión:

```
1. ¿streak >= 3?
   - Sí: usar variante `with_streak`
   - No: usar variante `no_streak`

2. ¿focus_today está disponible?
   - Sí: variante `with_focus`
   - No: variante `generic`

3. ¿es lunes o viernes?
   - Sí: rotar variantes `weekday_*`
   - No: rotar variantes `weekday_neutral`

4. Aplicar fallback si todo lo anterior falla: `fallback_template`.
```

La IA puede generar variaciones nuevas siempre que cumpla §1.1-§1.2 y
**los criterios de §10**. Las variantes acá son baseline + seed para el
prompt.

### 2.2 Variante: `with_streak` + `with_focus`

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `daily_reminder.streak_focus.01` | `Tu racha de {{streak}} días, {{name}}` | `Hoy: {{focus_today}}. {{minutes}} minutos para mantenerla.` |
| `daily_reminder.streak_focus.02` | `🔥 {{streak}} días seguidos` | `{{focus_today}} hoy. Tú decides la hora.` |
| `daily_reminder.streak_focus.03` | `{{name}}, hora de inglés` | `Llevas {{streak}} días. Hoy tocamos {{focus_today}}.` |
| `daily_reminder.streak_focus.04` | `No rompas la cadena` | `{{streak}} días. {{focus_today}} en {{minutes}} minutos.` |
| `daily_reminder.streak_focus.05` | `Tu cita diaria con el inglés` | `{{focus_today}} hoy. Mantén tu racha de {{streak}} días.` |

### 2.3 Variante: `with_streak` + `generic`

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `daily_reminder.streak_generic.01` | `Tu racha de {{streak}} días` | `{{minutes}} minutos hoy y sigue creciendo.` |
| `daily_reminder.streak_generic.02` | `🔥 {{streak}} días sin parar` | `Hoy te toca. Solo {{minutes}} minutos.` |
| `daily_reminder.streak_generic.03` | `{{name}}, tu plan te espera` | `{{streak}} días seguidos. Vamos por uno más.` |

### 2.4 Variante: `no_streak` + `with_focus`

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `daily_reminder.focus.01` | `Tu plan te espera, {{name}}` | `Hoy: {{focus_today}} en {{minutes}} minutos.` |
| `daily_reminder.focus.02` | `{{minutes}} minutos de inglés` | `Hoy tocamos {{focus_today}}.` |
| `daily_reminder.focus.03` | `Hora de practicar` | `{{focus_today}} hoy. Empezamos cuando quieras.` |
| `daily_reminder.focus.04` | `{{name}}, ¿listo?` | `{{focus_today}} hoy. Hagamos los {{minutes}} minutos.` |

### 2.5 Variante: `no_streak` + `generic` (fallback)

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `daily_reminder.generic.01` | `Tu plan te espera` | `{{minutes}} minutos de práctica hoy.` |
| `daily_reminder.generic.02` | `Hora de inglés, {{name}}` | `Solo {{minutes}} minutos. Tu plan está listo.` |
| `daily_reminder.generic.03` | `{{minutes}} minutos hoy` | `Sigue tu plan. Cada día cuenta.` |

### 2.6 Variantes especiales por contexto

#### 2.6.1 Lunes (motivación de inicio de semana)

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `daily_reminder.monday.01` | `Lunes, ¿empezamos?` | `Tu semana arranca con {{focus_today}}. {{minutes}} minutos.` |
| `daily_reminder.monday.02` | `Nueva semana, {{name}}` | `Empieza fuerte. {{focus_today}} hoy.` |

#### 2.6.2 Viernes (cerrar la semana)

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `daily_reminder.friday.01` | `Cierra la semana fuerte` | `{{focus_today}} hoy. Tu plan de fin de semana te espera.` |
| `daily_reminder.friday.02` | `Viernes y tu inglés` | `{{minutes}} minutos para terminar la semana en racha.` |

#### 2.6.3 Domingo (foco en avance global)

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `daily_reminder.sunday.01` | `Domingo de práctica` | `{{minutes}} minutos suaves. Mañana es lunes.` |
| `daily_reminder.sunday.02` | `Cierra tu semana, {{name}}` | `Una sesión hoy y mantienes tu racha.` |

### 2.7 Recent improvement (post-batch detecta delta positivo)

Si `{{recent_improvement}}` está disponible (ej: "+12 puntos en
pronunciación últimas 2 semanas"), prefiere estos:

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `daily_reminder.improvement.01` | `Estás mejorando, {{name}}` | `{{recent_improvement}}. Hoy: {{focus_today}}.` |
| `daily_reminder.improvement.02` | `Tu progreso se nota` | `{{recent_improvement}} en las últimas semanas. Sigamos.` |

---

## 3. `streak_at_risk`

### 3.1 Reglas

- Solo si `streak >= 5` (no urgir rachas cortas).
- Se envía 4h antes de medianoche en TZ del user.
- Se re-verifica antes del envío que user no haya practicado.
- Tono urgente pero amigable. Nunca culposo.

### 3.2 Variantes por longitud

#### Streak 5-9 días

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `streak_at_risk.short.01` | `🔥 Tu racha de {{streak}} días en juego` | `Quedan 4 horas. Solo {{minutes}} minutos para salvarla.` |
| `streak_at_risk.short.02` | `{{name}}, no la dejes ir` | `{{streak}} días seguidos. Te alcanza con una sesión corta.` |

#### Streak 10-29 días

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `streak_at_risk.medium.01` | `🔥 {{streak}} días en peligro` | `Quedan 4 horas. {{minutes}} minutos y la mantienes.` |
| `streak_at_risk.medium.02` | `Casi un mes, {{name}}` | `{{streak}} días no se construyen en un día. {{minutes}} min.` |
| `streak_at_risk.medium.03` | `Tu racha vale demasiado` | `{{streak}} días. Aún tienes 4 horas para salvarla.` |

#### Streak 30+ días

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `streak_at_risk.long.01` | `🔥 {{streak}} días, {{name}}` | `No la pierdas hoy. {{minutes}} minutos alcanzan.` |
| `streak_at_risk.long.02` | `Una racha histórica en juego` | `{{streak}} días seguidos. Te quedan 4 horas.` |
| `streak_at_risk.long.03` | `Tu disciplina de {{streak}} días` | `Una sesión rápida y la salvas.` |

### 3.3 Body con expansión (long-press)

Cuando user expande, mostramos contexto adicional. Ejemplo:

> "Tu racha de {{streak}} días es una de las cosas más difíciles de
> construir en idiomas. {{minutes}} minutos hoy y mañana sigue creciendo.
> Si la pierdes, los freeze tokens te dan 1 día gratis al mes."

(Detalle de freeze tokens en `motivation-and-achievements.md` §4.5.)

---

## 4. `onboarding` (welcome series)

Secuencia escalonada D1 → D7 que acompaña al user durante el trial.

### 4.1 `welcome_d1` (24h post-registro)

**Trigger:** 24h después de `onboarding.completed`.
**Contenido:** template (no IA-personalizado).

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `welcome_d1.01` | `¡Buen primer día, {{name}}!` | `¿Listo para el segundo? Tu plan te espera.` |
| `welcome_d1.02` | `Bienvenido, {{name}}` | `Hoy es tu día 2. {{minutes}} minutos y empiezas a notar el cambio.` |

### 4.2 `welcome_d3`

**Trigger:** 72h post-registro.
**Contenido:** IA-personalizado si user practicó al menos 1 vez (mostrar
qué mejoró). Template si no.

#### Si practicó (mostrar progreso):

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `welcome_d3.with_progress.01` | `Llevas 3 días, {{name}}` | `Tu {{tracker_metric}} ya mejoró. Sigue así.` |
| `welcome_d3.with_progress.02` | `3 días y se nota` | `{{recent_improvement}}. ¿Vamos por más hoy?` |

#### Si no practicó:

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `welcome_d3.no_progress.01` | `Tu plan sigue esperando` | `Llevas 3 días con la app. {{minutes}} minutos hoy y arrancas.` |
| `welcome_d3.no_progress.02` | `{{name}}, todo está listo` | `Solo falta que tú empieces. {{minutes}} minutos.` |

### 4.3 `welcome_d5`

**Trigger:** 5d post-registro.
**Objetivo:** crear expectativa del assessment Day 7.
**Modal subyacente:** assessment ya está disponible (modo "obligatorio
suave"; ver `student-profile-and-assessment.md` §13.1).

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `welcome_d5.01` | `Faltan 2 días para tu assessment` | `Va a desbloquear tu plan completo, hecho para ti.` |
| `welcome_d5.02` | `{{name}}, casi terminamos` | `En 2 días: tu assessment. Va a transformar tu plan.` |
| `welcome_d5.03` | `¿Te animas hoy?` | `Tu assessment ya está disponible. 20 minutos y plan personalizado.` |

### 4.4 `welcome_d7`

**Trigger:** 7d post-registro.
**Objetivo:** trigger del assessment.
**Modal subyacente:** assessment "obligatorio firme" (premium features
gated hasta completarlo).

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `welcome_d7.01` | `Llegó el día, {{name}}` | `Haz tu assessment ahora. 20 minutos y tu plan definitivo está listo.` |
| `welcome_d7.02` | `Tu assessment te espera` | `7 días observando cómo aprendes. Hoy lo medimos.` |
| `welcome_d7.03` | `Día 7: hora del assessment` | `Tu plan completo está a 20 minutos de distancia.` |

---

## 5. `reengagement` (D3, D7, D14)

Tres niveles con tono distinto. Después de D14: no se envían más.

### 5.1 `inactivity_d3` (suave, motivador)

**Trigger:** 72h sin actividad.
**Tono:** "te extrañamos, pero sin presión".

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `inactivity_d3.01` | `Tu plan te está esperando` | `3 días sin práctica. {{minutes}} minutos hoy y retomas.` |
| `inactivity_d3.02` | `{{name}}, ¿volvemos?` | `Tu inglés no se olvida en 3 días. Solo retomar.` |
| `inactivity_d3.03` | `Volver es fácil` | `{{minutes}} minutos hoy. Tu progreso te espera.` |

### 5.2 `inactivity_d7` (personal, mostrar progreso perdido)

**Trigger:** 7d sin actividad.
**Tono:** recordar que estaba mejorando, sin culpa.

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `inactivity_d7.with_progress.01` | `Mejorabas tu {{tracker_metric}}` | `7 días sin práctica. Vuelve y sigue donde dejaste.` |
| `inactivity_d7.with_progress.02` | `{{name}}, ibas muy bien` | `{{recent_improvement}}. No pierdas el avance.` |
| `inactivity_d7.no_progress.01` | `Una semana es nada` | `Tu plan sigue intacto. {{minutes}} minutos hoy.` |
| `inactivity_d7.no_progress.02` | `Volver toma 5 minutos` | `Tu inglés te espera, {{name}}.` |

### 5.3 `inactivity_d14` (último intento, posible incentivo)

**Trigger:** 14d sin actividad.
**Tono:** abierto, ofrecer Sparks bonus si vuelve.
**Coordinación:** triggers `sparks.awardBonus(20, reason='reengagement_d14_bonus')` cuando user abre.

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `inactivity_d14.01` | `Vuelve y te regalamos 20 Sparks` | `2 semanas sin verte. Tu plan + 20 Sparks bonus.` |
| `inactivity_d14.02` | `{{name}}, te extrañamos` | `Vuelve esta semana y tienes 20 Sparks de bienvenida.` |
| `inactivity_d14.03` | `Una segunda oportunidad` | `Tu plan sigue ahí. + 20 Sparks si vuelves hoy.` |

---

## 6. `achievement_unlocked`

### 6.1 Estructura

Para los 80 logros, cada uno tiene:
- `title`: nombre del logro + emoji.
- `body`: criterio cumplido + recompensa.
- Cuerpo expandido (long-press): celebración + raridad + % usuarios que
  lo desbloquearon.

### 6.2 Template base

```
title: "🏆 {{achievement_name}}"
body: "{{achievement_criterion_short}}. +{{achievement_sparks}} Sparks ⚡"
```

### 6.3 Banco completo de los 80 logros

#### 6.3.1 Constancia (15)

| achievement_id | Título | Cuerpo |
|----------------|--------|--------|
| `streak_3` | `🌱 Empezando` | `3 días seguidos. ¡Vas en camino! +5 Sparks ⚡` |
| `streak_7` | `🔥 Primera semana` | `7 días seguidos. Una semana entera. +10 Sparks ⚡` |
| `streak_14` | `💎 Hábito formándose` | `14 días seguidos. El hábito es tuyo. +20 Sparks ⚡` |
| `streak_30` | `⚡ Mes de hierro` | `30 días seguidos. Un mes completo. +30 Sparks ⚡` |
| `streak_60` | `🚀 Imparable` | `60 días seguidos. Imparable. +75 Sparks ⚡` |
| `streak_100` | `💯 Centenario` | `100 días seguidos. Histórico. +100 Sparks ⚡` |
| `streak_180` | `🏔️ Medio año` | `180 días seguidos. Medio año de constancia. +250 Sparks ⚡` |
| `streak_365` | `👑 Año completo` | `365 días seguidos. Un año entero. +500 Sparks ⚡` |
| `daily_goal_7` | `✅ Cumplidor` | `7 daily goals cumplidos. +10 Sparks ⚡` |
| `daily_goal_30` | `📅 Disciplinado` | `30 daily goals. Disciplina total. +30 Sparks ⚡` |
| `daily_goal_100` | `🎯 Constante` | `100 daily goals. La constancia paga. +100 Sparks ⚡` |
| `weekend_warrior` | `🛡️ Guerrero del finde` | `4 fines de semana seguidos. +25 Sparks ⚡` |
| `early_bird` | `🌅 Madrugador` | `10 sesiones antes de las 7am. +25 Sparks ⚡` |
| `night_owl` | `🦉 Búho nocturno` | `10 sesiones después de las 11pm. +25 Sparks ⚡` |
| `comeback_kid` | `💪 Resurrección` | `Volviste después de 14 días. ¡Bienvenido! +30 Sparks ⚡` |

#### 6.3.2 Volumen (10)

| achievement_id | Título | Cuerpo |
|----------------|--------|--------|
| `first_steps` | `👶 Primer paso` | `Completaste tu primer ejercicio. +5 Sparks ⚡` |
| `talker_10` | `🗣️ Conversador` | `10 conversaciones 1 a 1. +15 Sparks ⚡` |
| `talker_50` | `💬 Hablador` | `50 conversaciones. +50 Sparks ⚡` |
| `talker_200` | `🎤 Locuaz` | `200 conversaciones. Locuaz total. +150 Sparks ⚡` |
| `talker_500` | `🏆 Maestro de la palabra` | `500 conversaciones. Maestría. +400 Sparks ⚡` |
| `exercises_50` | `📚 Estudioso` | `50 ejercicios completados. +15 Sparks ⚡` |
| `exercises_200` | `📖 Dedicado` | `200 ejercicios. Dedicación pura. +40 Sparks ⚡` |
| `exercises_1000` | `🏃 Maratonista` | `1.000 ejercicios. Eres maratonista. +200 Sparks ⚡` |
| `minutes_1000` | `⏱️ Mil minutos` | `1.000 minutos totales de práctica. +50 Sparks ⚡` |
| `minutes_5000` | `⏰ Cinco mil` | `5.000 minutos totales. Inspiración. +200 Sparks ⚡` |

#### 6.3.3 Maestría (15)

| achievement_id | Título | Cuerpo |
|----------------|--------|--------|
| `pronunciation_master_th` | `🎯 TH dominado` | `Score >9 en /θ/ por 5 sesiones. +40 Sparks ⚡` |
| `pronunciation_master_v` | `🎯 V perfecta` | `Score >9 en /v/ por 5 sesiones. +40 Sparks ⚡` |
| `pronunciation_master_all` | `🌟 Pronunciación impecable` | `Score promedio >8.5 en 30 días. +150 Sparks ⚡` |
| `fluency_breakthrough` | `💨 Fluidez liberada` | `WPM >120 por 5 sesiones. +50 Sparks ⚡` |
| `fluency_master` | `🏎️ Fluido como nativo` | `WPM >150 mantenido. Nivel nativo. +150 Sparks ⚡` |
| `no_filler` | `🎙️ Sin muletillas` | `Reduciste 80% las muletillas en 4 semanas. +60 Sparks ⚡` |
| `vocab_b1_complete` | `📘 Vocabulario B1` | `Dominas el vocabulario B1 completo. +50 Sparks ⚡` |
| `vocab_b2_complete` | `📗 Vocabulario B2` | `Dominas el vocabulario B2 completo. +100 Sparks ⚡` |
| `grammar_pro` | `📐 Gramática pro` | `Menos del 2% de errores en 4 semanas. +100 Sparks ⚡` |
| `level_up_b1_to_b2` | `📈 De B1 a B2` | `Subiste de B1 a B2. Hito real. +200 Sparks ⚡` |
| `level_up_b2_to_c1` | `🚀 De B2 a C1` | `Subiste de B2 a C1. Avanzado. +400 Sparks ⚡` |
| `listening_native_speed` | `👂 Comprensión nativa` | `Comprendes audio nativo >90%. +100 Sparks ⚡` |
| `accent_match` | `🌍 Acento auténtico` | `Tu pronunciación se reconoce como nativa. +300 Sparks ⚡` |
| `improvement_50` | `📊 Gran progreso` | `Mejoraste tu score general >50%. +150 Sparks ⚡` |
| `improvement_100` | `🌠 Transformación` | `Duplicaste tu score general. +350 Sparks ⚡` |

#### 6.3.4 Variedad (8)

| achievement_id | Título | Cuerpo |
|----------------|--------|--------|
| `try_5_types` | `🔍 Curioso` | `Probaste 5 tipos de ejercicio. +10 Sparks ⚡` |
| `try_all_types` | `🗺️ Explorador` | `Probaste todos los tipos. +30 Sparks ⚡` |
| `versatile` | `🔄 Versátil` | `Ejercicios en 5 categorías. +25 Sparks ⚡` |
| `multilevel` | `🪜 Multinivel` | `Bloques en 3 niveles distintos. +30 Sparks ⚡` |
| `roleplay_5` | `🎭 Actor` | `5 roleplays distintos. +20 Sparks ⚡` |
| `roleplay_20` | `🎬 Camaleón` | `20 roleplays distintos. +60 Sparks ⚡` |
| `pronunciation_drill_master` | `🎯 Pronunciación intensiva` | `100 ejercicios de pronunciación. +50 Sparks ⚡` |
| `assessment_dedicated` | `🔬 Auto-conocedor` | `3 re-evaluaciones completas. +75 Sparks ⚡` |

#### 6.3.5 Roadmap (10)

| achievement_id | Título | Cuerpo |
|----------------|--------|--------|
| `first_block` | `🎉 Primera lección` | `Completaste tu primer bloque. +5 Sparks ⚡` |
| `first_level` | `🏁 Primer nivel` | `Terminaste el primer nivel del track. +25 Sparks ⚡` |
| `halfway` | `🎖️ Medio camino` | `50% del track completado. Vas a buen ritmo. +75 Sparks ⚡` |
| `track_complete` | `🏆 Track completo` | `Completaste el track entero. +200 Sparks ⚡` |
| `track_perfect` | `💎 Track perfecto` | `Track con >85% de mastery. Excelencia. +300 Sparks ⚡` |
| `multi_track` | `🌟 Polyglot path` | `Completaste 2 tracks. +400 Sparks ⚡` |
| `all_tracks` | `👑 Maestro de inglés` | `Completaste todos los tracks. +1.000 Sparks ⚡` |
| `fast_track` | `⚡ Vía rápida` | `Track terminado antes del estimado. +100 Sparks ⚡` |
| `goal_achieved` | `🎯 Misión cumplida` | `Lograste el objetivo que te propusiste. +250 Sparks ⚡` |
| `deadline_made` | `⏰ A tiempo` | `Cumpliste antes de tu deadline. +200 Sparks ⚡` |

#### 6.3.6 Social (10) [Fase 2+]

| achievement_id | Título | Cuerpo |
|----------------|--------|--------|
| `referral_1` | `🤝 Embajador` | `Invitaste a tu primer amigo. +20 Sparks ⚡` |
| `referral_5` | `📣 Influencer` | `Referiste a 5 amigos. +100 Sparks ⚡` |
| `referral_10` | `🎙️ Líder de comunidad` | `10 referidos. Creas comunidad. +250 Sparks ⚡` |
| `referral_paid_1` | `💰 Promotor` | `Tu primer referido pagó. +50 Sparks ⚡` |
| `referral_paid_5` | `💎 Súper promotor` | `5 referidos pagaron. +250 Sparks ⚡` |
| `friend_first` | `👋 Primer amigo` | `Agregaste tu primer amigo. +10 Sparks ⚡` |
| `challenge_won` | `🏅 Ganador` | `Ganaste un friend challenge. +30 Sparks ⚡` |
| `league_top3` | `🥉 Podio` | `Top 3 en la liga semanal. +50 Sparks ⚡` |
| `league_first` | `🥇 Campeón` | `Primer puesto en la liga semanal. +100 Sparks ⚡` |
| `event_winner` | `🎊 Ganador de evento` | `Top 10 en evento comunitario. +150 Sparks ⚡` |

#### 6.3.7 Especiales y secretos (12)

| achievement_id | Título | Cuerpo |
|----------------|--------|--------|
| `holiday_practitioner` | `🎄 Espíritu navideño` | `Practicaste el 25 de diciembre. +30 Sparks ⚡` |
| `new_year_starter` | `🎆 Año nuevo, hábito nuevo` | `Practicaste el 1 de enero. +30 Sparks ⚡` |
| `birthday_bash` | `🎂 Cumpleaños bilingüe` | `Practicaste en tu cumpleaños. +50 Sparks ⚡` |
| `marathon_session` | `🏃 Maratón` | `Sesión de 90+ minutos. +100 Sparks ⚡` |
| `lightning_fast` | `⚡ Velocista` | `Daily goal en menos de 5 minutos. +25 Sparks ⚡` |
| `perfectionist` | `💯 Perfeccionista` | `10 ejercicios perfectos seguidos. +100 Sparks ⚡` |
| `night_marathon` | `🌙 Trasnochada productiva` | `60+ minutos después de medianoche. +40 Sparks ⚡` |
| `weekend_double` | `🏖️ Doble fin de semana` | `+60 min sábado y domingo. +30 Sparks ⚡` |
| `early_assessor` | `🎯 Adelantado` | `Hiciste el assessment antes del Day 7. +40 Sparks ⚡` |
| `feedback_giver` | `💬 Voz crítica` | `Enviaste feedback útil 5 veces. +50 Sparks ⚡` |
| `bug_hunter` | `🐛 Cazador de bugs` | `Reportaste un bug confirmado. +100 Sparks ⚡` |
| `legendary_first_year` | `🏛️ Pionero` | `Eres uno de los primeros 1.000 usuarios. +500 Sparks ⚡` |

### 6.4 Body expandido (long-press)

Si user expande, mostramos contexto adicional. Patrón:

> "{{achievement_name}}: {{full_description}}. Es un logro
> {{rarity_label}}. {{total_unlocked}} personas lo desbloquearon."

Donde `rarity_label`:

| rarity | Label en español |
|--------|------------------|
| `common` | "común" |
| `rare` | "raro 💎" |
| `epic` | "épico 🌟" |
| `legendary` | "legendario 👑" |
| `unique` | "único ✨" |

---

## 7. `level_completed`

### 7.1 Reglas

- Trigger: `roadmap.level_completed` event.
- Solo si user no está en sesión activa (si está en sesión, UI muestra
  in-app celebration; suprimir push).
- Cooldown 1h después del evento (evita push si user sigue practicando).

### 7.2 Variantes

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `level_completed.01` | `🎉 Nivel completado, {{name}}` | `Terminaste "{{level_name}}". Próximo nivel desbloqueado.` |
| `level_completed.02` | `🏁 "{{level_name}}" completado` | `{{name}}, sigues avanzando. ¿Vamos por el siguiente?` |
| `level_completed.03` | `Nivel terminado 🎯` | `Ya pasaste "{{level_name}}". Tu plan sigue creciendo.` |

---

## 8. Transactional

### 8.1 `low_sparks` (balance < 20% del plan)

#### 8.1.1 Reglas

- Trigger: `sparks.balance_low` event.
- Cooldown: 7 días (no enviar más de 1 por semana).
- Solo si user tiene plan pagado (free trial usa flujo distinto, ver
  §5.5 de student-profile-and-assessment.md).

#### 8.1.2 Variantes

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `low_sparks.01` | `⚡ Te quedan {{sparks_remaining}} Sparks` | `Solo {{percent_remaining}}% del mes. ¿Compras un pack?` |
| `low_sparks.02` | `Sparks bajos, {{name}}` | `{{sparks_remaining}} de {{sparks_total}}. Pack o esperar al próximo ciclo.` |
| `low_sparks.03` | `Casi sin Sparks` | `{{sparks_remaining}} restantes. Los preassets siguen gratis.` |

### 8.2 `pack_expiring`

#### 8.2.1 Reglas

- Trigger: cron diario detecta packs con `expires_at` en 7 días.
- Una notif por pack (no agrupar).
- No reenviar (single shot).

#### 8.2.2 Variantes

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `pack_expiring.01` | `⏳ Tu pack expira en {{pack_expiry_days}} días` | `Tienes {{sparks_remaining}} Sparks sin usar. Aprovecha.` |
| `pack_expiring.02` | `{{pack_expiry_days}} días para usar tus Sparks` | `{{sparks_remaining}} ⚡ vencen pronto, {{name}}.` |

### 8.3 `payment_failed`

#### 8.3.1 Reglas

- Trigger: webhook de Stripe/RevenueCat.
- Categoría transactional crítica: bypass rate limit.
- 3 reintentos automáticos en 5 días (Stripe Smart Retries). Push en
  cada retry fallido.

#### 8.3.2 Variantes

| copy_id | Título | Cuerpo |
|---------|--------|--------|
| `payment_failed.first.01` | `Tu pago no pudo procesarse` | `Actualiza tu método y mantén tu plan activo, {{name}}.` |
| `payment_failed.retry_2.01` | `Segundo intento fallido` | `Tu plan se pausará si no actualizas tu método de pago.` |
| `payment_failed.final.01` | `⚠️ Última oportunidad` | `Tu plan se pausa mañana. Actualiza tu método ahora.` |

### 8.4 `restriction_applied`

#### 8.4.1 Reglas

- Trigger: anti-fraud aplicó restricción.
- Categoría transactional crítica.
- Tono claro, no acusatorio.

#### 8.4.2 Variantes por restriction_type

| restriction_type | Título | Cuerpo |
|------------------|--------|--------|
| `no_trial_sparks` | `Cuenta detectada como duplicada` | `Ya tienes cuenta con nosotros. Inicia sesión con tu email original.` |
| `rate_limited` | `Pausa temporal en tu cuenta` | `Detectamos uso fuera de lo común. Vuelve en 24 horas.` |
| `suspended` | `Tu cuenta está suspendida` | `Revisa nuestro mensaje y responde para resolver, {{name}}.` |
| `banned` | `Tu cuenta fue cerrada` | `Revisa los Términos de Uso. Puedes apelar desde el email.` |

---

## 9. Edge cases

### 9.1 Sin `display_name`

Si `users.display_name IS NULL` o vacío:
- Omitir frase con `{{name}}` (no decir "Hola , ..." con coma sola).
- Usar variante sin nombre cuando exista.
- Fallback genérico: omitir saludo personal.

**Ejemplo:** `daily_reminder.streak_focus.01` con `name=null`:
- ❌ "Tu racha de 8 días, "
- ✅ "Tu racha de 8 días"

### 9.2 Anonymous user (sin auth)

Anonymous users tienen `display_name = "Invitado"` por default.
Tratar como sin nombre (omitir `{{name}}`).

### 9.3 Multi-device

Si user tiene 3 tokens activos, FCM envía a los 3. Una sola
`notifications_log` entry con array de tokens. `opened_at` del primero
que abra.

### 9.4 Notificación llega tarde (FCM delay)

FCM puede demorar minutos. Si scheduled_for = 19:00 y FCM entrega
19:35:
- Daily reminder: aún relevante, mostrar.
- Streak at risk: si pasaron las 23:00 (hora de corte), suprimir
  client-side (`data.expires_at` field).

### 9.5 Streak quebrado entre schedule y send

Scheduler verifica nuevamente antes del envío. Si la racha se quebró,
no se envía.

### 9.6 Logro desbloqueado mid-session

Cliente FE muestra in-app celebration (UI propia). Worker NO envía
push si user está en sesión activa (`exercise_attempts.in_progress=true`
en últimos 5 min).

### 9.7 IA generation falla

Fallback a templates de §2 (variantes hardcoded). Cada
`notification_id` debe tener al menos 1 template no-IA.

### 9.8 Idioma futuro (post-MVP)

Estructura preparada para multi-idioma:
- `copy_bank` table con columna `locale`.
- MVP solo `es-MX`.
- Post-MVP: `pt-BR`, `en` (sí, para usuarios avanzados que cambian
  preferencia).

(Detalle en `cross-cutting/i18n.md`.)

---

## 10. Reglas de personalización IA

### 10.1 Qué SÍ puede hacer la IA

- Generar variaciones nuevas siguiendo §1.1 (tono) y §1.2 (longitud).
- Usar las variables disponibles (§1.3).
- Adaptar al `focus_today` específico del user.
- Mencionar `recent_improvement` si existe.
- Usar emojis del set permitido (§10.3).

### 10.2 Qué NO puede hacer la IA

- ❌ Generar guilt-tripping ("¿No te importa tu plan?", "Vas a fallar
  si no...").
- ❌ Usar voseo argentino o vocabulario excluyente.
- ❌ Mencionar otros usuarios por nombre (privacidad).
- ❌ Mentir sobre métricas (ej: inventar improvement que no existe).
- ❌ Promesas que no podemos cumplir ("dominarás el inglés en 30
  días").
- ❌ Comparaciones con otros usuarios ("la mayoría ya practicó hoy").
- ❌ Tono agresivo, urgente falso, FOMO manipulador.
- ❌ Más de 1 emoji por título o más de 1 por cuerpo.
- ❌ Romper límites de chars (validación bloquea).

### 10.3 Set de emojis permitidos

```
Streaks/fuego:    🔥 ⚡ 💪 🚀
Logros:           🏆 🎉 🎊 🥇 🥈 🥉 💎 🌟 ✨ 👑
Constancia:       📅 ✅ 🎯 ⏰ ⏳
Aprendizaje:      📚 📖 🎓 🧠 💡
Voz/audio:        🎤 🎙️ 👂 🗣️
Tiempo:           🌅 🌙 🦉 🌞
Especiales:       🎄 🎆 🎂 🎁
Notificaciones:   🔔 📣 💬
Pago/sparks:      ⚡ 💰 ⚠️
```

**Prohibidos:** 😡 😤 🤬 😭 😢 ⚰️ 💀 (negativos), 💩 🍆 (vulgares),
🚨 🆘 (alarma falsa).

### 10.4 Validación pre-envío

Pipeline en Worker antes de mandar a FCM:

```typescript
function validateCopy(copy: NotificationCopy): ValidationResult {
  const issues: string[] = [];
  if (copy.title.length > 50) issues.push('title_too_long');
  if (copy.body.length > 90) issues.push('body_too_long');
  if (containsVoseo(copy.title + copy.body)) issues.push('voseo_detected');
  if (containsBlockedEmoji(copy)) issues.push('emoji_blocked');
  if (containsGuiltTrigger(copy)) issues.push('guilt_pattern');
  if (countEmojis(copy.title) > 1) issues.push('too_many_emojis_title');
  if (containsUnresolvedVar(copy)) issues.push('unresolved_var');
  return { valid: issues.length === 0, issues };
}
```

Si falla validación: usar template hardcoded de §2-§8 como fallback.

### 10.5 Detección de patrones culposos

Lista de palabras/frases que activan el guilt detector:

```
Bloqueadas:
- "no te importa"
- "vas a fallar"
- "perdiste"
- "abandonaste"
- "te olvidaste de mí"
- "última oportunidad" (excepto en payment_failed.final)
- "no tienes excusa"
- "no seas..." (cualquier negación de identidad)

Sospechosas (review):
- "tienes que"
- "debes"
- "obligado"
```

---

## 11. Backlog de A/B tests

(Coordinar con `notifications-system.md` §15.9: A/B con Firebase Remote
Config es post-MVP. PostHog feature flags suplen para empezar.)

### 11.1 Priorizado por impacto esperado

| # | Test | Hipótesis | Métrica primaria | Tamaño muestra |
|---|------|-----------|------------------|---------------:|
| 1 | Emoji en título daily_reminder vs sin emoji | Emoji aumenta open rate (~10-15%) | Open rate | 2.000/var |
| 2 | Mencionar focus_today vs solo "tu plan" | Específico aumenta intent (~20%) | Open rate + sesión iniciada | 2.000/var |
| 3 | Streak_at_risk a 4h antes vs 2h antes | Más tiempo mejor para casos borderline | Streak salvada % | 1.000/var |
| 4 | inactivity_d14 con 20 sparks vs sin oferta | Incentivo aumenta comeback | Comeback rate D14-D21 | 500/var |
| 5 | welcome_d5 mencionando assessment vs no | Menciona crea expectativa | Assessment completed % | 1.000/var |
| 6 | Daily reminder personalizada IA vs template | IA mejora open rate >5% justifica costo | Open rate | 5.000/var |
| 7 | Achievement con/sin emoji en título | Emoji captura atención | Open rate | 2.000/var |
| 8 | Tono "tú decides" vs "vamos" en daily | Autonomy frame aumenta intent | Sesión iniciada | 2.000/var |

### 11.2 Tests **explícitamente fuera de scope MVP**

- Smart timing aprendido (post-MVP, ver §15.6 notifications-system).
- Notificaciones grupales (ver §15.7).
- Rich notifications con imágenes (ver §15.8).
- Emoji solo en preview platform-specific (iOS vs Android).

---

## 12. Decisiones cerradas

### 12.1 Mexicano-tuteo como locale único en MVP ✓

**Razón:** target inicial México + tuteo es más universal en LatAm
(no excluye usuarios de Colombia, Chile, Perú; sí excluiría a usuarios
mexicanos si usáramos voseo). Coherente con `CLAUDE.md` §10 y
`reglas.md`.

**Cuándo agregar otros locales:**
- pt-BR: cuando expandamos a Brasil (post-MVP).
- es neutro vs es-MX: si data muestra fricción específica con
  expresiones mexicanas (no esperado).

### 12.2 Emoji set restringido ✓

**Razón:** evitar tono agresivo (alarma 🚨), vulgar (💩) o ambivalente
(😢). Lista de §10.3 cubre todos los casos del MVP.

**Razón adicional:** consistencia visual de marca. Set predefinido es
revisable trimestralmente.

### 12.3 Banco de copys autoritativo, IA es seed ✓

**Razón:** IA puede generar variaciones, pero siempre sobre la base de
templates validados manualmente. Esto permite:
- Fallback determinístico si AI falla (§9.7).
- Validación más estricta (§10.4).
- A/B test contra baseline conocido (§11).
- Auditoría legal (sabemos qué se le mandó al user).

### 12.4 Cuerpo principal max 90 chars, expansion 180 ✓

**Razón:** preview de iOS lock screen y Android collapsed trunca a
~110 chars. 90 deja margen para "..." de la plataforma. Para users
que long-press, body extendido aporta contexto sin pesar en preview.

### 12.5 Logros usan template `🏆 {{name}}` para los 80 ✓ (MVP)

**Razón:** consistencia visual en stack de notificaciones. User
aprende a reconocer "logro" por el patrón. Banco custom (§6.3) es por
si más adelante queremos diferenciar por categoría con emojis
distintos — el banco ya tiene emoji custom por logro lista para usar.

**Decisión cerrada (post-revisión):** usar el banco custom de §6.3
desde MVP (cada logro con su emoji propio). Más memorable, mismo costo.

### 12.6 Welcome series escalonada D1, D3, D5, D7 ✓

**Razón:** alineado con timing del trial assessment de
`student-profile-and-assessment.md` §13.1 (3 niveles según día). D5
es donde modal "obligatorio suave" aparece; D7 donde es "obligatorio
firme". Push refuerza en cada hito.

### 12.7 Re-engagement detiene después de D14 ✓

**Razón:** después de D14 sin actividad, probabilidad de comeback
< 5%. Insistir genera fatigue y opt-outs. Mejor invertir esfuerzo en
features que mantengan a los usuarios activos.

**Excepción (post-MVP):** WhatsApp para D30+ con tono distinto
(notifications-system.md §14.2).

### 12.8 NO mencionar otros usuarios por nombre ni comparar ✓

**Razón:** privacidad + nuestra brand voice es individual ("tu
plan, hecho para ti"). Comparaciones generan ansiedad sin mejorar
retention en data conocida.

**Excepciones permitidas:** "X usuarios desbloquearon este logro" en
body expandido de achievement (anonimizado, agregado).

---

## 13. Plan de implementación

### 13.1 Fase 1 (semanas 1-2 del sprint de notifications)

- Schema `notifications_copy_bank` (tabla con todos los copys de §2-§8).
- Seed inicial con todo el banco.
- Worker resolve template + variables + valida (§10.4).
- Templates hardcoded como fallback en código (§9.7).

### 13.2 Fase 2 (semana 3-4)

- Integrar AI Gateway task `generate_notification_content` con seeds
  de §2.
- Validation pipeline pre-FCM.
- A/B test infra con PostHog feature flags.

### 13.3 Fase 3 (mes 2)

- Tests A/B priorizados §11.1 #1 y #2.
- Métricas de open rate por copy_id.
- Iteración del banco basada en data.

### 13.4 Schema sugerido

```sql
CREATE TABLE notifications_copy_bank (
  copy_id           TEXT PRIMARY KEY,
  notification_id   TEXT NOT NULL,
  variant           TEXT NOT NULL,
  locale            TEXT NOT NULL DEFAULT 'es-MX',
  title_template    TEXT NOT NULL,
  body_template     TEXT NOT NULL,
  body_expanded     TEXT,
  required_vars     TEXT[] NOT NULL DEFAULT '{}',
  optional_vars     TEXT[] NOT NULL DEFAULT '{}',
  emoji_count       INT NOT NULL DEFAULT 0,
  is_fallback       BOOLEAN NOT NULL DEFAULT false,
  is_active         BOOLEAN NOT NULL DEFAULT true,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_copy_bank_lookup
  ON notifications_copy_bank(notification_id, variant, locale)
  WHERE is_active = true;

-- Tracking de uso para rotación + A/B
CREATE TABLE notifications_copy_usage (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  copy_id           TEXT NOT NULL REFERENCES notifications_copy_bank(copy_id),
  user_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  sent_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
  delivered_at      TIMESTAMPTZ,
  opened_at         TIMESTAMPTZ,
  experiment_id     TEXT,                    -- para A/B
  variant_label     TEXT
);

CREATE INDEX idx_copy_usage_copy_date
  ON notifications_copy_usage(copy_id, sent_at DESC);
```

---

## 14. Métricas de éxito del banco

(Coordinar con `notifications-system.md` §13.1.)

### 14.1 Por copy_id

- Open rate (target depende de notification_id, ver §13.1
  notifications-system).
- Conversion rate (sesión iniciada en próximas 2h).
- Opt-out rate atribuido a este copy.

### 14.2 Agregadas

- % de notifs que pasan validación al primer intento (target > 95%).
- % de envíos que usan fallback (target < 5%).
- Costo IA por copy generado (target < $0.001).

### 14.3 Triggers de revisión

- Si open rate de cualquier copy_id cae > 30% vs baseline en 14 días:
  rotar a otra variante automáticamente.
- Si AI genera > 10% de copys que fallan validación: revisar prompt
  o restringir.

---

## 15. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) | Sistema técnico que envía estos copys. |
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) §4.1 | Catálogo de notification_ids. |
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) §8 | Personalización IA y prompt template. |
| [`motivation-and-achievements.md`](motivation-and-achievements.md) §6.2 | Catálogo de los 80 logros. |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §5.4 | Comunicación durante el trial Day 1-7. |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) §13.1 | Timing del assessment (welcome_d5 y d7). |
| [`onboarding-flow.md`](onboarding-flow.md) §14 | Pantalla de permisos de push. |
| [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) §4.2.7 | Task `generate_notification_content`. |
| [`../cross-cutting/i18n.md`](../cross-cutting/i18n.md) | Convenciones de mexicano-tuteo. |
| [`../reglas.md`](../reglas.md) §4.7 | Tono de voz autoritativo. |

---

*Documento vivo. Actualizar cuando se agreguen notification_ids
nuevos, logros nuevos al catálogo, o se agreguen locales post-MVP.*

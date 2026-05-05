# Motivation, Achievements and Social System

> Sistema integral de motivación, logros, gamificación y comunidad
> diseñado para sostener engagement a largo plazo. 5 capas: hábito
> diario, progreso visible, logros, social, personalización IA.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** Sistema completo (con fases marcadas)

---

## 0. Cómo leer este documento

- §1 establece **por qué la motivación es crítica** + marco teórico.
- §2 cubre **boundaries**.
- §3 cubre las **5 capas del sistema**.
- §4 cubre **streaks y daily goals** (Capa 1).
- §5 cubre **progreso visible** (Capa 2).
- §6 cubre **catálogo de 80 logros** (Capa 3).
- §7 cubre **achievement engine** (cómo se detectan).
- §8 cubre **social y comunidad** (Capa 4, post-MVP mayoría).
- §9 cubre **personalización IA** (Capa 5).
- §10 cubre **diferenciación cultural latinoamericana**.
- §11 cubre **API contracts**.
- §12 enumera **edge cases**.
- §13 cubre **eventos**.
- §14 cubre **decisiones cerradas**.

---

## 1. Por qué la motivación es crítica

### 1.1 El problema fundamental

80% de personas que empiezan a aprender un idioma abandonan en menos
de 60 días. Razón típica: pérdida de motivación, no contenido malo.

El aprendizaje de idiomas tiene un problema único: la **curva de
progreso percibida es plana** durante semanas o meses, aunque
objetivamente el user esté mejorando. Sin sistema externo de
motivación, el cerebro abandona porque no detecta progreso.

### 1.2 Apps que sobreviven motivan mejor

Duolingo no ganó por contenido superior (mediocre académicamente).
Ganó por su sistema de motivación. Cientos de millones mantienen
rachas por mecánicas, no por mejor pedagogía.

**Lección aplicable:** invertir en motivación es invertir en retención.

### 1.3 Marco teórico: Self-Determination Theory

Cuatro motivadores principales:

**Autonomía:** sentir que controlo qué y cómo aprendo.
- En el sistema: roadmap personalizado, opciones de orden, capacidad
  de saltar bloques que ya domino.

**Competencia:** sentir que estoy mejorando, que soy capaz.
- En el sistema: logros, progreso visualizado, feedback positivo,
  comparación con yo mismo en el pasado.

**Relatedness:** sentir que soy parte de algo, no estoy solo.
- En el sistema: leagues, eventos comunitarios, friend system.

**Propósito:** conectar lo que hago hoy con un objetivo significativo.
- En el sistema: tracks orientados a metas (Job Ready, Travel),
  recordatorios del por qué empecé, certificados.

Un buen sistema motivacional ataca los 4 ejes. Apps que solo atacan uno
(típicamente competencia con XP/badges) fallan a mediano plazo.

---

## 2. Boundaries

### 2.1 Es responsable de

- Streaks, freeze tokens, repair de streaks.
- Daily goals (definir, trackear cumplimiento).
- Catálogo de 80 logros.
- Achievement engine (detectar criterios cumplidos consumiendo events).
- Otorgar Sparks bonus via `sparks-system.awardBonus`.
- Visualizaciones de progreso (gráficos, heatmap).
- Insights y celebrations (genera mensajes via AI Gateway).
- Sistema social (referidos, friends, leagues post-MVP).
- Detección de perfil motivacional.
- Detección de burnout y vacation mode.

### 2.2 NO es responsable de

- **Cobrar Sparks ni mantener balance:** delega a `sparks-system`.
- **Determinar mastery:** consume desde `pedagogical-system`.
- **Enviar push:** delega a `notifications-system`.
- **Definir bloques o assets:** consumido de `content-creation-system`.
- **Decidir qué bloque ofrecer:** eso es `ai-roadmap-system`.

### 2.3 Tensiones

| Tensión | Resolución |
|---------|-----------|
| Logro requiere data de múltiples sistemas | Achievement engine consume eventos cross-system; queries puntuales si necesario |
| User cumple logro raro pero tiene fraud_score alto | `anti-fraud` puede aplicar `restriction = no_premium_features`; engine respeta restriction |
| Streak roto a las 23:59 PM (TZ del user) | Cálculo en TZ del user, no UTC |
| User reactiva trial y rompe streak | Streak se mantiene si no han pasado >24h reales |

---

## 3. Arquitectura: 5 capas

```
┌─────────────────────────────────────────────────────────────┐
│                  CAPA 1: HÁBITO DIARIO                      │
│  Daily goals, streaks, freeze tokens, anti-burnout          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  CAPA 2: PROGRESO VISIBLE                   │
│  Visualizaciones, comparaciones temporales, milestones      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  CAPA 3: LOGROS Y RECOMPENSAS               │
│  80 achievements, raridad, Sparks bonus                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  CAPA 4: SOCIAL Y COMUNIDAD                 │
│  Referidos, leagues, friend system, eventos                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  CAPA 5: PERSONALIZACIÓN IA                 │
│  Detección de qué motiva a cada user, mensajes únicos       │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 Principios de diseño

**No saturar:** todas las capas son opt-in (excepto básicas). Sin
bombardeo simultáneo de celebraciones, leaderboards, notificaciones.

**Calibrar magnitud:** "completaste 1 ejercicio" no debe sentirse igual
que "completaste tu primer track". Si todo es épico, nada lo es.

**Aliado, no acosador:** detectar burnout, sugerir descanso, respetar
vacaciones. Apps que se sienten obligación se abandonan.

**Personalizar el motivador:** algunos users responden a streaks, otros
a logros, otros a comunidad. Sistema aprende.

---

## 4. Capa 1: Hábito diario

### 4.1 Daily Goals

#### Concepto

Cada día tiene objetivo concreto, alcanzable, específico al user.
"Hoy: 12 minutos de práctica con foco en pronunciación de /θ/".

NO es genérico "practica inglés". Es objetivo claro con criterio
binario.

#### Fuente

Generado por job nocturno basado en:
- `daily_minutes_available` declarado.
- Plan del roadmap actual.
- Performance reciente.
- Foco pedagógico de hoy según roadmap.

#### Visualización

```
┌─────────────────────────────────────┐
│  TU DÍA — Martes                    │
│                                     │
│  🎯 Objetivo de hoy                 │
│  12 minutos                         │
│  Foco: pronunciación de /θ/         │
│                                     │
│  ▓▓▓▓▓▓░░░░░░ 6/12 min              │
│                                     │
│  [Continuar]                        │
└─────────────────────────────────────┘
```

#### Mecánica

- Cumplir objetivo: +2-5 Sparks bonus.
- 7 días seguidos: badge especial + 20 Sparks.
- Exceder no recompensa proporcionalmente. Cumplir es el objetivo.

### 4.2 Streaks (Rachas)

#### Concepto

Días consecutivos cumpliendo daily goal. Mostrado prominentemente.

#### Por qué funcionan

3 mecánicas psicológicas:
- **Endowment effect:** lo que tengo, no quiero perder.
- **Loss aversion:** perder 23 días duele más que la satisfacción de
  ganar 1 más.
- **Visible progress:** un número que sube es satisfactorio.

Lado oscuro: cuando se rompe streak largo, mucha gente abandona
definitivamente. Necesitan mitigación.

#### Mitigaciones

**Streak Freeze:**

```typescript
const STREAK_FREEZE_RULES = {
  awarded_at_streak_7: 1,           // 1 freeze gratis al alcanzar 7 días
  awarded_per_30_days: 1,            // 1 freeze adicional cada 30 días
  max_accumulated: 3,
  applied_automatically: true,       // si user pierde día y tiene freeze
  notification_after_use: true,
};
```

```
🧊 Tienes 2 Streak Freezes
Estos te salvan automáticamente si te pierdes un día.
```

**Streak Repair:**

Si se rompe sin freezes disponibles, opción de "comprarlo de vuelta"
pagando Sparks dentro de 24h:

```
😔 Tu racha de 23 días se rompió ayer
Pero puedes recuperarla pagando 30 Sparks
[Recuperar mi racha]
```

Convierte pérdida en agencia.

#### Visualización: fire icon evolutivo

| Días | Icono |
|-----:|-------|
| 1–6 | 🔥 (llama pequeña) |
| 7–29 | 🔥🔥 (dos llamas) |
| 30–99 | ⚡ (electricidad) |
| 100–364 | 💎 (diamante) |
| 365+ | 👑 (corona) |

#### Pantalla de racha

```
🔥 Racha actual
23 días

📅 Tu calendario de práctica
[heatmap visual estilo GitHub contributions]

⚡ Logros de constancia
✅ Primera semana (7 días)
✅ Hábito formándose (14 días)
🔒 Hábito sólido (30 días) - 7 días para desbloquear

🛡️ Streak Freezes disponibles: 2
```

### 4.3 Anti-burnout

#### Detección de fatiga

Señales monitoreadas:
- Scores cayendo en sesiones consecutivas.
- Tiempo por ejercicio aumentando significativamente.
- Tasa de abandono subiendo.
- Sesiones más cortas que el promedio del user.

Si 3+ señales en 2 días consecutivos:

```
Notamos que las últimas sesiones fueron más difíciles.
Tu cerebro consolida mejor con descanso.

¿Querés tomarte el día?
[Sí, descansar] [No, sigo]

Si descansás, no afecta tu racha.
```

#### Vacation Mode

Opción explícita en Settings:

```
🌴 Modo Vacaciones
"No me molesten por X días"
- Pausa notificaciones
- Pausa cuenta de streak (no se rompe)
- Mantiene Sparks y progreso intactos

[Activar 7 días] [Activar 14 días] [Personalizar]
```

Sin culpa, sin penalización.

#### Sugerencia de bajar metas

Si user consistentemente no cumple daily goal:

```
Notamos que tu objetivo de 30 min/día puede estar muy alto.
¿Querés ajustar a 15 min/día? Es mejor cumplir 15 min que fallar 30.

[Sí, ajustar] [No, lo intento]
```

---

## 5. Capa 2: Progreso visible

### 5.1 El problema de la progresión percibida

En idiomas, progreso se siente plano aunque sea real. Esta capa hace
**VISIBLE el progreso invisible**.

### 5.2 Visualizaciones core

#### Gráfico de evolución de habilidades

Línea temporal con 4 dimensiones (pronunciación, fluidez, gramática,
vocabulario). 4–12 semanas visualizadas.

```
Tu evolución últimas 8 semanas
                                    ●
Pronunciación  ●━━●━━●━━●━━●━━●━━●━●
              60     70     80    +12

Fluidez       ●━━━━●━━━●━━●━━●━━●━●
              55      62        68  +6
```

#### "Tú vs tú hace 30 días"

Comparación side-by-side con audios reales:

```
🎙️ ESCUCHATE PROGRESAR

Hace 30 días:
"thank you for the... uh... opportunity"
[▶ Reproducir]

Hoy:
"thank you for the opportunity to interview"
[▶ Reproducir]

Pronunciación: 65 → 82 (+17)
Fluidez: 55 → 71 (+16)
```

Audios reales son la prueba más potente de progreso.

#### Heatmap anual de actividad

```
Tu año en inglés - 2026

[grid 52 semanas x 7 días]

🟩 Día épico (>30 min)
🟢 Día completado
🟡 Día parcial
⬛ Día sin práctica

Total: 184 días practicados
Mejor mes: Mayo (28 días)
Racha más larga: 47 días
```

#### Mapa visual del roadmap

Roadmap completo con:
- Niveles completados (verde, checkmark).
- Nivel actual (destacado, animación).
- Niveles futuros (grises, preview).
- Estimación: "estás en el 35% del camino".

### 5.3 Insights semanales

Cada domingo (email + in-app):

```
📊 TU SEMANA EN INGLÉS

Esta semana practicaste:
• 124 minutos (objetivo: 105) ✅
• 7/7 días con racha activa 🔥
• 18 ejercicios completados

🎯 Tu mayor mejora:
Vocabulary in business context (+15%)

🔍 Tu desafío de la semana:
Pronunciación de /ʒ/ (palabras como "measure", "vision")

✨ Tu próximo objetivo:
Llegar al nivel "Conversational Confidence"
(estás a 5 lecciones)

Top 3 logros de la semana:
🏆 Primera entrevista simulada
⚡ 7 días consecutivos
📚 50 ejercicios totales

[Ver detalle completo]
```

Generado por job nocturno via AI Gateway task `generate_weekly_summary`.

### 5.4 Milestones celebrations

| Magnitud | Trigger | Celebración |
|----------|---------|-------------|
| Pequeño | 1 ejercicio completed | Mensaje breve, feedback positivo |
| Medio | Daily goal cumplido | Animación + Sparks + toast |
| Grande | Logro desbloqueado | Pantalla completa, animación, mensaje |
| Épico | Nivel del roadmap completed | Pantalla, audio, certificado parcial |
| Legendario | Track completo | Experiencia completa, certificado, opción compartir |

Cada celebración debe sentirse genuina, no robótica. Mensaje
personalizado y acorde al esfuerzo.

---

## 6. Capa 3: Catálogo de 80 logros

### 6.1 Estructura

#### Categorías

1. **Constancia** - rachas y consistencia.
2. **Volumen** - acumulación.
3. **Maestría** - logros cualitativos en habilidades.
4. **Variedad** - exploración del producto.
5. **Roadmap** - hitos del progreso estructurado.
6. **Social** - interacción con comunidad.
7. **Especiales** - únicos o secretos.

#### Niveles de raridad

| Raridad | Color | Sparks bonus | % users que lo desbloquean |
|---------|-------|-------------:|---------------------------:|
| Común | Gris | 5–10 | > 50% |
| Raro | Azul | 20–30 | 20–50% |
| Épico | Morado | 50–100 | 5–20% |
| Legendario | Dorado | 200–500 | < 5% |
| Único | Iridiscente | Variable | < 1% |

### 6.2 Catálogo completo (80 logros)

#### 6.2.1 Constancia (15)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|-------:|
| `streak_3` | Empezando | 3 días seguidos | Común | 5 |
| `streak_7` | Primera semana | 7 días seguidos | Común | 10 |
| `streak_14` | Hábito formándose | 14 días seguidos | Raro | 20 |
| `streak_30` | Mes de hierro | 30 días seguidos | Raro | 30 |
| `streak_60` | Imparable | 60 días seguidos | Épico | 75 |
| `streak_100` | Centenario | 100 días seguidos | Épico | 100 |
| `streak_180` | Medio año | 180 días seguidos | Legendario | 250 |
| `streak_365` | Año completo | 365 días seguidos | Legendario | 500 |
| `daily_goal_7` | Cumplidor | 7 daily goals cumplidos | Común | 10 |
| `daily_goal_30` | Disciplinado | 30 daily goals | Raro | 30 |
| `daily_goal_100` | Constante | 100 daily goals | Épico | 100 |
| `weekend_warrior` | Guerrero del finde | 4 fines de semana seguidos | Raro | 25 |
| `early_bird` | Madrugador | 10 sesiones antes de 7am | Raro | 25 |
| `night_owl` | Búho nocturno | 10 sesiones después de 11pm | Raro | 25 |
| `comeback_kid` | Resurrección | Volver después de 14+ días | Raro | 30 |

#### 6.2.2 Volumen (10)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|-------:|
| `first_steps` | Primer paso | Completar primer ejercicio | Común | 5 |
| `talker_10` | Conversador | 10 conversaciones 1 a 1 | Común | 15 |
| `talker_50` | Hablador | 50 conversaciones | Raro | 50 |
| `talker_200` | Locuaz | 200 conversaciones | Épico | 150 |
| `talker_500` | Maestro de la palabra | 500 conversaciones | Legendario | 400 |
| `exercises_50` | Estudioso | 50 ejercicios | Común | 15 |
| `exercises_200` | Dedicado | 200 ejercicios | Raro | 40 |
| `exercises_1000` | Maratonista | 1.000 ejercicios | Épico | 200 |
| `minutes_1000` | Mil minutos | 1.000 minutos totales | Raro | 50 |
| `minutes_5000` | Cinco mil | 5.000 minutos totales | Épico | 200 |

#### 6.2.3 Maestría (15)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|-------:|
| `pronunciation_master_th` | TH dominado | Score > 9 en /θ/ por 5 sesiones | Raro | 40 |
| `pronunciation_master_v` | V perfecta | Score > 9 en /v/ por 5 sesiones | Raro | 40 |
| `pronunciation_master_all` | Pronunciación impecable | Score promedio > 8.5 mantenido 30 días | Épico | 150 |
| `fluency_breakthrough` | Fluidez liberada | WPM > 120 por 5 sesiones | Raro | 50 |
| `fluency_master` | Fluido como nativo | WPM > 150 mantenido | Épico | 150 |
| `no_filler` | Sin muletillas | Reducir 80% en 4 semanas | Raro | 60 |
| `vocab_b1_complete` | Vocabulario B1 | Dominar B1 completo | Raro | 50 |
| `vocab_b2_complete` | Vocabulario B2 | Dominar B2 completo | Épico | 100 |
| `grammar_pro` | Gramática pro | <2% errores en 4 semanas | Épico | 100 |
| `level_up_b1_to_b2` | De B1 a B2 | Subir CEFR oficialmente | Épico | 200 |
| `level_up_b2_to_c1` | De B2 a C1 | Subir CEFR | Legendario | 400 |
| `listening_native_speed` | Comprensión nativa | Comprender audio nativo > 90% | Épico | 100 |
| `accent_match` | Acento auténtico | Pronunciación reconocida como nativa | Legendario | 300 |
| `improvement_50` | Gran progreso | Mejorar score general > 50% | Épico | 150 |
| `improvement_100` | Transformación | Duplicar score general | Legendario | 350 |

#### 6.2.4 Variedad (8)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|-------:|
| `try_5_types` | Curioso | Probar 5 tipos de ejercicio | Común | 10 |
| `try_all_types` | Explorador | Probar todos los tipos | Raro | 30 |
| `versatile` | Versátil | Ejercicios en 5 categorías | Raro | 25 |
| `multilevel` | Multinivel | Bloques en 3 niveles distintos | Raro | 30 |
| `roleplay_5` | Actor | 5 roleplays distintos | Común | 20 |
| `roleplay_20` | Camaleón | 20 roleplays distintos | Raro | 60 |
| `pronunciation_drill_master` | Pronunciación intensiva | 100 ejercicios de pronunciación | Raro | 50 |
| `assessment_dedicated` | Auto-conocedor | Completar 3 re-evaluaciones | Raro | 75 |

#### 6.2.5 Roadmap (10)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|-------:|
| `first_block` | Primera lección | Completar primer bloque | Común | 5 |
| `first_level` | Primer nivel | Completar primer nivel del track | Común | 25 |
| `halfway` | Medio camino | 50% del track completado | Raro | 75 |
| `track_complete` | Track completo | Completar track entero | Épico | 200 |
| `track_perfect` | Track perfecto | Track con > 85% mastery | Épico | 300 |
| `multi_track` | Polyglot path | Completar 2 tracks | Legendario | 400 |
| `all_tracks` | Maestro de inglés | Completar todos los tracks | Legendario | 1000 |
| `fast_track` | Vía rápida | Completar track antes del estimado | Raro | 100 |
| `goal_achieved` | Misión cumplida | Lograr el objetivo declarado | Épico | 250 |
| `deadline_made` | A tiempo | Cumplir antes del deadline | Épico | 200 |

#### 6.2.6 Social (10) [Fase 2+]

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|-------:|
| `referral_1` | Embajador | Referir 1 amigo que se registre | Común | 20 |
| `referral_5` | Influencer | Referir 5 amigos | Raro | 100 |
| `referral_10` | Líder de comunidad | Referir 10 | Épico | 250 |
| `referral_paid_1` | Promotor | Primer referido paga | Raro | 50 |
| `referral_paid_5` | Súper promotor | 5 referidos pagan | Épico | 250 |
| `friend_first` | Primer amigo | Agregar primer amigo | Común | 10 |
| `challenge_won` | Ganador | Ganar friend challenge | Raro | 30 |
| `league_top3` | Podio | Top 3 en liga semanal | Raro | 50 |
| `league_first` | Campeón | Primer puesto liga semanal | Épico | 100 |
| `event_winner` | Ganador de evento | Top 10 en evento comunitario | Épico | 150 |

#### 6.2.7 Especiales y secretos (12)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|-------:|
| `holiday_practitioner` | Espíritu navideño | Practicar el 25 dic | Raro | 30 |
| `new_year_starter` | Año nuevo, hábito nuevo | Practicar el 1 ene | Raro | 30 |
| `birthday_bash` | Cumpleaños bilingüe | Practicar en tu cumpleaños | Raro | 50 |
| `marathon_session` | Maratón | Sesión de 90+ min | Épico | 100 |
| `lightning_fast` | Velocista | Daily goal en <5 min | Raro | 25 |
| `perfectionist` | Perfeccionista | 10 ejercicios consecutivos perfectos | Épico | 100 |
| `night_marathon` | Trasnochada productiva | 60+ min después de medianoche | Raro | 40 |
| `weekend_double` | Doble fin de semana | +60 min sáb y dom | Raro | 30 |
| `early_assessor` | Adelantado | Hacer assessment antes del Day 7 | Raro | 40 |
| `feedback_giver` | Voz crítica | Enviar feedback útil 5 veces | Raro | 50 |
| `bug_hunter` | Cazador de bugs | Reportar bug confirmado | Épico | 100 |
| `legendary_first_year` | Pionero | Usuario de los primeros 1000 | Único | 500 |

**Total: 80 logros** en 7 categorías y 5 niveles de raridad.

---

## 7. Achievement Engine

### 7.1 Engine spec

```typescript
interface AchievementEngine {
  // Verificar todos los logros pendientes para un user en respuesta a evento
  checkAchievements(userId: string, trigger: TriggerEvent): Promise<Achievement[]>;

  // Otorgar un logro
  grantAchievement(userId: string, achievementId: string): Promise<void>;

  // Calcular progreso hacia logros aún no obtenidos
  getProgressTowardsAchievements(userId: string): Promise<AchievementProgress[]>;
}

interface TriggerEvent {
  type: 'session_completed' | 'streak_updated' | 'level_completed' |
        'assessment_done' | 'referral_signup' | 'manual_check';
  data: any;
}
```

### 7.2 Estrategia de evaluación

**Evaluación inmediata (en respuesta a eventos):**
- Logros que dependen de evento puntual (completar primer ejercicio,
  ganar liga).
- Triggered desde lógica de negocio que generó el evento.

**Evaluación batch (job nocturno):**
- Logros acumulativos (totales, promedios).
- Logros de mantenimiento (mantener score por X días).
- Logros temporales (eventos calendario).

### 7.3 Notificación al usuario

Pantalla de logro desbloqueado:

```
┌─────────────────────────────────────┐
│         🏆 LOGRO DESBLOQUEADO        │
│                                     │
│         [Ícono del logro]           │
│                                     │
│        "Hábito formándose"          │
│                                     │
│   Practicaste 14 días seguidos      │
│                                     │
│         + 20 Sparks ⚡               │
│                                     │
│         Raridad: Raro 💎             │
│   2.847 personas lo desbloquearon   │
│                                     │
│   [Compartir] [Ver mi colección]    │
└─────────────────────────────────────┘
```

### 7.4 Pantalla de colección

Sección dedicada que muestra:
- Total: 23/80 logros desbloqueados (29%).
- Filtro por categoría.
- Filtro por raridad.
- Logros bloqueados muestran progreso ("12/14 días" para `streak_14`).
- Logros secretos muestran "???" hasta desbloquearlos.

Funciona como museo personal, fuente de orgullo.

### 7.5 Schema Postgres

```sql
CREATE TABLE achievements_catalog (
  id              TEXT PRIMARY KEY,
  category        TEXT NOT NULL CHECK (category IN (
                    'constancia', 'volumen', 'maestria', 'variedad',
                    'roadmap', 'social', 'especiales'
                  )),
  name            TEXT NOT NULL,
  description     TEXT NOT NULL,
  rarity          TEXT NOT NULL CHECK (rarity IN (
                    'common', 'rare', 'epic', 'legendary', 'unique'
                  )),
  sparks_reward   INT NOT NULL,
  icon_url        TEXT,
  criteria        JSONB NOT NULL,
  is_secret       BOOLEAN NOT NULL DEFAULT false,
  is_active       BOOLEAN NOT NULL DEFAULT true,
  total_unlocked  INT NOT NULL DEFAULT 0,         -- denormalized counter
  country_filter  TEXT[]                          -- ['MX', 'AR'] o NULL para global
);

CREATE TABLE user_achievements (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  achievement_id  TEXT NOT NULL REFERENCES achievements_catalog(id),
  unlocked_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  shared_count    INT NOT NULL DEFAULT 0,
  flagged_for_review BOOLEAN NOT NULL DEFAULT false,
  flag_reason     TEXT,
  PRIMARY KEY (user_id, achievement_id)
);

CREATE INDEX idx_user_achievements_unlocked
  ON user_achievements(user_id, unlocked_at DESC);

CREATE TABLE achievements_progress (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  achievement_id  TEXT NOT NULL REFERENCES achievements_catalog(id),
  current_value   FLOAT NOT NULL,
  target_value    FLOAT NOT NULL,
  last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, achievement_id)
);
```

### 7.6 Streaks schema

```sql
CREATE TABLE user_streaks (
  user_id             UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  current_streak      INT NOT NULL DEFAULT 0,
  longest_streak      INT NOT NULL DEFAULT 0,
  last_active_date    DATE,                       -- en TZ del user
  freezes_available   INT NOT NULL DEFAULT 0 CHECK (freezes_available BETWEEN 0 AND 3),
  total_freezes_used  INT NOT NULL DEFAULT 0,
  vacation_until      DATE,                       -- pause si setteado
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 7.7 Daily goals schema

```sql
CREATE TABLE user_daily_goals (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  goal_date       DATE NOT NULL,
  target_minutes  INT NOT NULL,
  target_focus    TEXT,                           -- 'pronunciation /θ/'
  achieved_minutes INT NOT NULL DEFAULT 0,
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending', 'achieved', 'missed', 'paused')),
  achieved_at     TIMESTAMPTZ,
  PRIMARY KEY (user_id, goal_date)
);

CREATE INDEX idx_daily_goals_pending
  ON user_daily_goals(user_id, goal_date)
  WHERE status = 'pending';
```

---

## 8. Capa 4: Social y comunidad

### 8.1 Sistema de referidos [Fase 1]

#### Mecánica

Cada user tiene código único. Comparte por WhatsApp, redes, etc.

| Trigger | Recompensa al referrer | Recompensa al referee |
|---------|----------------------:|----------------------:|
| Referee se registra | 10 Sparks | 10 Sparks bonus |
| Referee completa onboarding | +5 Sparks | — |
| Referee paga primer mes | +50 Sparks | — |
| Referidos subsecuentes que pagan | +30 Sparks cada uno | — |

#### Anti-abuso

- Cap 100 Sparks/mes en bonos para nuevos users (excepto pagos
  reales).
- Detección de cuentas duplicadas via fingerprint (`anti-fraud`).
- Verificación de email/teléfono para que cuente.
- Auditoría manual con > 50 referidos en 30 días.

### 8.2 Compartir externos [Fase 1]

#### Tipos

**Logro desbloqueado:**

```
[Imagen generada con el logro]
"¡Acabo de desbloquear 'Mes de hierro' en MyEnglishApp!
30 días seguidos practicando inglés 🔥
Mejor decisión de este año.
#aprendiendoinglés"
```

**Progreso semanal, certificado al completar track:** análogo.

#### Generación de imágenes

Server-side con templates predefinidos. Cada imagen:
- Branded con logo discreto.
- Datos personalizados.
- Diseño compartible (vertical para Stories, cuadrado para feed,
  horizontal para Twitter).
- QR code o link corto.

### 8.3 Friend System [Fase 2]

- Agregar amigos por código de referido o búsqueda.
- Ver perfil del amigo: streak, total practicado, logros recientes.
- Sentir presencia social sin competencia agresiva.

#### Activity feed

```
"Carlos completó 'First level'" - hace 2 horas.
"Ana mantiene su racha de 45 días 🔥" - hoy.
"Pedro desbloqueó 'Pronunciation impecable'" - ayer.
```

#### Reactions

Reacciones rápidas: 🔥 💪 👏 🚀.

Sin chat directo en MVP de friend system. Mantener simple, evitar
moderación compleja.

### 8.4 Friend Challenges [Fase 2]

Tipos:
- **Minutos:** quien practique más en 7 días.
- **Constancia:** quien mantenga racha más larga.
- **Mejora:** quien mejore más score de pronunciación.
- **Ejercicios:** quien complete más en 14 días.

Mecánica:
- Iniciador propone a un amigo.
- Amigo acepta o rechaza.
- Ambos ven progreso en tiempo real.
- Ganador recibe Sparks; ambos reciben participación.

### 8.5 Leagues semanales [Fase 2]

#### Concepto

Cada semana, user asignado a liga de 30 con perfil similar (mismo país,
CEFR similar, frecuencia similar).

#### Mecánica

Lunes a domingo, todos compiten por puntos:
- Minutos: 1 pto/min.
- Daily goals: 10 ptos/goal.
- Logros: variable según raridad.
- Ejercicios: 2 ptos/ejercicio.

Domingo 23:59 cierre:
- Top 3: suben división, +50/30/20 Sparks.
- Top 10: se mantienen, pequeño bonus.
- Bottom 5: bajan división.
- Resto: se mantienen.

#### Divisiones

Bronce 1-3, Plata 1-3, Oro 1-3, Platino, Diamante, Maestro (top 1%
global).

### 8.6 Eventos comunitarios [Fase 3]

- **Marathons temporales:** "Marathon de pronunciación: 1-3 mayo".
- **Themed weeks:** "Semana del Job Ready".
- **Country challenges:** "México vs Argentina: ¿quién practica más?"
- **Eventos por temporadas:** resoluciones de año nuevo, vuelta a
  clases, vacaciones.

---

## 9. Capa 5: Personalización IA

### 9.1 Detección automática del perfil motivacional

Algoritmo:

```python
def calculate_motivation_profile(user_id):
    behaviors = get_user_behaviors_last_30_days(user_id)

    scores = {
        'streak_driven':     calc_streak_response(behaviors),
        'achievement_hunter': calc_achievement_response(behaviors),
        'social':            calc_social_response(behaviors),
        'progress_driven':   calc_progress_response(behaviors),
        'purpose_driven':    calc_purpose_response(behaviors),
    }

    primary = max(scores, key=scores.get)
    secondary = sorted(scores.items(), key=lambda x: -x[1])[1][0]

    return {
        'primary_motivator': primary,
        'secondary_motivator': secondary,
        'all_scores': scores,
    }
```

Recalculado mensualmente. Mensajes y experiencia se ajustan.

### 9.2 Mensajes personalizados

| Tipo | Mensaje ejemplo |
|------|----------------|
| Streak-driven | "Tu racha de 23 días está en juego. ¿Te quedan 4 horas?" |
| Achievement-hunter | "Estás a 3 ejercicios de desbloquear 'Maratonista'. ¿Vamos?" |
| Social | "Carlos acaba de completar el nivel que vos estás haciendo. ¿Lo alcanzás?" |
| Progress-driven | "Tu pronunciación mejoró 15% en 4 semanas. Mira tu evolución →" |
| Purpose-driven | "Recordá: empezaste por la entrevista en TechCorp. Estás 60% del camino." |

---

## 10. Diferenciación cultural latinoamericana

### 10.1 Logros con sabor local

| ID | Nombre | Criterio | País |
|----|--------|----------|------|
| `cinco_de_mayo` | 5 de Mayo | Practicar el 5 mayo | México |
| `dia_del_amigo` | Día del amigo | Practicar el 30 julio | Argentina |
| `independencia_mx` | Mes de la patria | Practicar todo septiembre | México |
| `anniversary_ar` | 25 de Mayo | Practicar el 25 mayo | Argentina |
| `colombia_day` | Independencia | Practicar el 20 julio | Colombia |
| `ave_fenix` | Ave Fénix | Volver después de 30+ días | Todos |
| `aguante` | Aguante | Practicar todos los días de un mes lluvioso/de exámenes | Todos |

### 10.2 Eventos calendario por país

(Detalle en `cross-cutting/i18n.md` §5.4.)

### 10.3 Tono de comunicación

(Detalle en `cross-cutting/i18n.md` §2.2.)

**Bien:**
- "Vamos, tú puedes".
- "Tu meta te espera".
- "Un día más, un paso más".

**Evitar:**
- Modismos muy locales que excluyen.
- Excesiva familiaridad ("flaco", "compa").
- Tono gringo traducido ("¡Tú lo lograste, campeón!").

---

## 11. API contracts

### 11.1 `getDailyGoal`

```typescript
interface GetDailyGoalRequest {
  user_id: string;
  date?: string;                   // default today
}

interface GetDailyGoalResponse {
  goal: {
    target_minutes: number;
    target_focus: string;
    achieved_minutes: number;
    status: 'pending' | 'achieved' | 'missed' | 'paused';
  };
  streak: {
    current: number;
    icon: string;
    freezes_available: number;
    can_repair: boolean;
  };
}
```

### 11.2 `getAchievements`

```typescript
interface GetAchievementsRequest {
  user_id: string;
  filter_category?: string;
  filter_unlocked?: boolean;
}

interface GetAchievementsResponse {
  unlocked: Array<{
    id: string;
    name: string;
    rarity: string;
    unlocked_at: string;
    sparks_awarded: number;
    pct_users_unlocked: number;
  }>;
  in_progress: Array<{
    id: string;
    name: string;
    rarity: string;
    is_secret: boolean;
    progress_pct: number;
    current_value: number;
    target_value: number;
  }>;
  total_unlocked: number;
  total_available: number;
}
```

### 11.3 `useStreakFreeze`

Trigger interno cuando user pierde día con freeze available:

```typescript
async function useStreakFreeze(userId: string, missedDate: string): Promise<void>
```

Aplicado automáticamente por job diario, no llamable por cliente.

### 11.4 `repairStreak`

Llamado por cliente si user tiene Sparks:

```typescript
interface RepairStreakRequest {
  user_id: string;
}

interface RepairStreakResponse {
  cost_sparks: number;
  streak_restored_to: number;
  expires_in_minutes: number;       // 1440 = 24h
}
```

**Reglas:**
- Solo si streak roto en últimas 24h.
- Costo proporcional al streak (e.g., 30 Sparks por streak de 23
  días).
- Se cobra Sparks via `sparks-system`.

### 11.5 `setVacationMode`

```typescript
interface SetVacationModeRequest {
  user_id: string;
  duration_days: number;
}

interface SetVacationModeResponse {
  vacation_until: string;
  streak_paused_at: number;
}
```

### 11.6 `recordReferralSignup`

Llamado por auth-system cuando un user usa código de referido:

```typescript
interface RecordReferralSignupRequest {
  referrer_user_id: string;
  referee_user_id: string;
}
```

**Reglas:**
- Verificar anti-fraud (mismo fingerprint → no recompensa).
- Otorgar Sparks bonus a ambos via `sparks-system.awardBonus`.

### 11.7 Internal: `evaluateAchievementsForUser`

Job nocturno calls:

```typescript
async function evaluateAchievementsForUser(userId: string): Promise<{
  newly_unlocked: string[];
  progress_updated: string[];
}>
```

Iterates achievements_catalog, verifica criterios contra user state,
otorga si match.

---

## 12. Edge cases (tests obligatorios)

### 12.1 Streaks

1. **User practica a las 23:55 PM TZ y la sync llega después de
   medianoche:** streak debe contar para el día anterior (`completed_at`
   en TZ del user, no UTC).
2. **User cambia TZ entre dos sesiones:** mantener streak; usar TZ
   actual.
3. **User en vacation mode practica:** no aumenta streak (lo que es
   esperado), pero permite "regalo": +1 al streak final.
4. **User con streak 30 usa último freeze:** funciona; siguiente día
   sin practicar y sin freeze → streak roto.
5. **User con streak roto repair pero ya no tiene Sparks:**
   `INSUFFICIENT_SPARKS`; ofrecer comprar pack.
6. **Streak repair llega justo después de las 24h post-rotura:**
   rechazado; restart from scratch.

### 12.2 Daily goals

7. **User cumple daily goal con minutos exactos al límite:** otorga
   bonus.
8. **User practica con 10 min sobrantes después de cumplir:** no genera
   logro adicional. Cumplir es el objetivo.
9. **Daily goal generado con `target_minutes = 0` por bug:** validar
   antes de presentar; si bug, default 10 min.

### 12.3 Achievement engine

10. **User cumple criterios de logro durante restriction (ej:
    no_premium_features):** logro se otorga sin Sparks bonus, persistido
    con flag `awarded_during_restriction`. Cuando restriction se
    lift, opcional: otorgar Sparks retroactivamente (decisión de
    admin).
11. **Multiple events disparan mismo logro concurrentemente:**
    idempotency en `INSERT INTO user_achievements ON CONFLICT DO
    NOTHING`. Sparks awarded una sola vez.
12. **Logro retroactivo (cambio de criterio):** evaluación batch
    nocturna lo otorga al primer match. No retroactivo a fecha pasada.
13. **Logro country-specific en user que cambió de país:** evaluación
    al momento de unlock; si país en ese momento matchea, otorgar.

### 12.4 Referrals

14. **Referrer y referee comparten device:** no otorgar bonus (ver
    `anti-fraud`); ambos reciben mensaje "no podemos verificar el
    referido".
15. **Referee borra cuenta antes de otorgar bonus por pago:** clawback
    si bonus ya otorgado prematuramente.

### 12.5 Vacation mode

16. **User activa vacation mode con streak 50:** streak congelado.
    Después de vacation, sigue en 50.
17. **User en vacation pero practica voluntariamente:** opcional:
    permitir; streak puede aumentar pero no obligatorio.

### 12.6 Cheating

18. **User detectado con bot pattern (anti-fraud) y tiene logros
    sospechosos:** flagged_for_review = true. Admin decide retirar o
    mantener.
19. **User logra `legendary_first_year` (primeros 1000) por bug en
    counter:** verificación al unlocking; si race condition, primero
    en commit gana.

---

## 13. Eventos

### 13.1 Emitidos

(Detalle en `cross-cutting/data-and-events.md` §5.7.)

| Evento | Cuándo |
|--------|--------|
| `streak.extended` | Streak crece (cumplió daily goal) |
| `streak.broken` | Día sin practicar y sin freeze |
| `streak.at_risk` | 4h antes del corte sin práctica |
| `streak.freeze_used` | Auto-uso de freeze |
| `streak.repaired` | User pagó Sparks para repair |
| `daily_goal.met` | User cumplió goal del día |
| `daily_goal.missed` | Día terminó sin cumplir |
| `achievement.unlocked` | Nuevo logro desbloqueado |
| `vacation_mode.activated` | User activó vacation |
| `vacation_mode.ended` | Period terminó |
| `motivation_profile.updated` | Recálculo mensual del profile |
| `league.week_ended` | Cierre semanal de liga (post-MVP) |

### 13.2 Consumidos

| Evento | Acción |
|--------|--------|
| `exercise.attempt_completed` (pedagogical) | Update progress hacia logros, daily goal minutes |
| `block.completed` | Verificar logros de roadmap |
| `block.mastered` | Verificar logros de mastery |
| `subskill.level_up` | Verificar logros maestría específicos |
| `user.cefr_changed` | Verificar logros `level_up_*` |
| `level.completed` (ai-roadmap) | Verificar `first_level`, `track_complete` |
| `payment.succeeded` | Verificar logros de referral pagado |
| `user.signed_up` | Inicializar streak, daily_goal, motivation_profile |

---

## 14. Decisiones cerradas

### 14.1 Sparks o XP separado: **Solo Sparks** ✓

**Razón:** evitar fragmentar economía. Sparks tienen valor real (cuestan
dinero, se canjean por uso), darle función de XP los hace más
significativos. Una sola moneda mental para el user.

### 14.2 Borrar logros por cheating: **Sí, con flag** ✓

**Razón:** mantener integridad del sistema. Si user es banned por
bots, sus logros se marcan `flagged_for_review = true` y no aparecen
públicamente. No se borran (audit), pero se invalidan.

### 14.3 Mostrar % de users que tienen cada logro: **Sí** ✓

**Razón:** hace concreta la raridad. Aumenta valor percibido de los
épicos/legendarios. Riesgo de desmotivación es bajo si UI lo presenta
con tono de "logro raro" no de "casi nadie lo tiene".

### 14.4 Donar Sparks a amigos: **NO** ✓

**Razón:** vector grande de farming/laundering de Sparks. El beneficio
(viralidad) no compensa el riesgo. Reconsiderar año 2 si tenemos
sistema robusto anti-abuse.

### 14.5 Sistema de "rivales" (Strava-style): **NO en MVP** ✓

**Razón:** complejidad alta para MVP. Friend challenges + leagues
cubren competencia social sin requerir matchmaking dedicado.
Reconsiderar post-soft-launch si users solicitan.

---

## 15. Plan de implementación

### 15.1 Fase 1: MVP de motivación (meses 0–3)

**Crítico:**
- Daily goals con visualización clara.
- Streaks con freeze tokens.
- Anti-burnout básico (vacation mode, sugerencia bajar metas).
- Catálogo de 30 logros core (constancia, volumen, roadmap).
- Achievement engine.
- Pantalla de colección.
- Compartir logros externos.
- Sistema de referidos básico.

**Importante:**
- Visualizaciones de progreso (gráfico, heatmap).
- Insights semanales por email.
- Milestones celebrations.

### 15.2 Fase 2: Profundización (meses 3–6)

**Crítico:**
- "Tú vs tú hace 30 días" con audios.
- Leagues semanales por región/nivel.
- Mapa visual del roadmap.
- Catálogo expandido a 60 logros.

**Importante:**
- Friend system básico.
- Friend challenges.
- Comeback mechanics (repair streak).

### 15.3 Fase 3: Social profundo (meses 6–12)

**Crítico:**
- Eventos comunitarios temporales.
- Country challenges.
- Catálogo completo 80 logros.
- Detección automática de perfil motivacional.

**Importante:**
- Mensajes personalizados según perfil.
- Logros culturales por país.
- Activity feed con reactions.

### 15.4 Fase 4: Sofisticación (año 2+)

- Mentoría peer-to-peer.
- Foros temáticos.
- Marathons collaborativos globales.
- Sistema de "coaches" (users premium ayudan a principiantes).
- Logros co-creados con la comunidad.

---

## 16. Métricas

### 16.1 Salud del sistema motivacional

**Engagement:**
- DAU.
- Stickiness (DAU/MAU).
- Sesiones por user por semana.
- Streak average length.
- % users con streak > 7 días.

**Logros:**
- Logros desbloqueados promedio por user.
- Distribución por raridad (debería ser piramidal).
- % users que ven la pantalla de logros frecuentemente.

**Social:**
- % users con al menos 1 referido.
- Conversion rate de referidos.
- % users participando en leagues (post-MVP).

### 16.2 Predictores de retención

Patrones que indican alta probabilidad de retención a 90 días:
- Completó primer logro en primeros 3 días.
- Streak >= 7 en primera semana.
- Vio stats de progreso al menos 3 veces.
- Activó al menos una mecánica social.

Sistema usa estos para identificar users en riesgo de churn y aplicar
intervenciones.

### 16.3 Anti-señales

Patrones que indican burnout:
- Cumplir daily goal sin exceder por 14+ días seguidos (mínimo
  esfuerzo).
- Caer abajo del 25% en 3 leagues consecutivas.
- Reducir tiempo por sesión consistentemente.

Sistema detecta y sugiere ajustes.

---

## 17. Riesgos y mitigaciones

| Riesgo | Probabilidad | Mitigación |
|--------|-------------|-----------|
| Saturación de notificaciones de logros | Alta | Calibración cuidadosa, opt-out granular, rate limit |
| Cheating de logros (cuentas múltiples) | Media | Anti-fraud, flag retroactivo |
| Frustración por no llegar al top en leagues | Alta | Divisiones (compite con peers) |
| Streak addiction tóxica | Media | Vacation mode, freeze tokens, mensajes anti-burnout |
| Eventos comunitarios con baja participación | Media | Cap mínimo, recompensas garantizadas |

---

## 18. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) | Provee `observed_behavior` para perfil motivacional. |
| [`pedagogical-system.md`](pedagogical-system.md) | Eventos consumidos: `block.*`, `subskill.*`, `user.cefr_changed`. |
| [`ai-roadmap-system.md`](ai-roadmap-system.md) | Eventos consumidos: `level.*`, `roadmap.completed`. |
| [`../architecture/sparks-system.md`](../architecture/sparks-system.md) | `awardBonus` para recompensas. |
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) | Push de logros y streak alerts. |
| [`../architecture/anti-fraud-system.md`](../architecture/anti-fraud-system.md) | Validación de referidos, restricciones. |
| [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) | Tasks `generate_weekly_summary`, `generate_morning_message`. |
| [`../cross-cutting/i18n.md`](../cross-cutting/i18n.md) §5 | Logros y eventos por país. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.7 | Eventos. |

---

## 19. Referencias externas e inspiración

- Duolingo's gamification system (referencia, no copia).
- Strava's segments and leaderboards.
- GitHub contributions heatmap (visualización).
- Headspace's streak system (manejo saludable).
- Self-Determination Theory (Deci & Ryan).

---

*Documento vivo. Actualizar cuando se agreguen logros, mecánicas
sociales, o se observen patrones que sugieran ajustes.*

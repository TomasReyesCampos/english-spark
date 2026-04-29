# Motivation, Achievements and Social System

> Sistema integral de motivación, logros, gamificación y comunidad
> diseñado para sostener el engagement a largo plazo. Documenta el
> sistema completo, desde mecánicas individuales hasta dinámicas sociales.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Alcance:** Sistema completo (con fases marcadas)

---

## 1. Por qué la motivación es crítica

### 1.1 El problema fundamental

El 80% de las personas que empiezan a aprender un idioma abandonan en
menos de 60 días. La razón raramente es "el contenido era malo". Casi
siempre es "perdí la motivación".

El aprendizaje de idiomas tiene un problema único: la **curva de progreso
percibida es plana** durante semanas o meses, aunque objetivamente el
usuario esté mejorando. Sin sistema externo de motivación, el cerebro
abandona porque no detecta progreso.

### 1.2 Por qué las apps que sobreviven son las que motivan mejor

Duolingo no ganó por contenido superior (su contenido es académicamente
mediocre). Ganó por su sistema de motivación. Cientos de millones de
usuarios mantienen rachas diarias por mecánicas, no por mejor pedagogía.

La lección aplicable: **invertir en motivación es invertir en retención**,
y retención es el factor que determina si un producto de educación es
viable o no.

### 1.3 Marco teórico: Self-Determination Theory

La psicología tiene un marco bien establecido sobre qué motiva
sostenidamente a los humanos. Cuatro motivadores principales:

**Autonomía:** sentir que controlo qué y cómo aprendo.
- En el sistema: roadmap personalizado, opciones de orden, capacidad
  de saltar bloques que ya domino.

**Competencia:** sentir que estoy mejorando, que soy capaz.
- En el sistema: logros, progreso visualizado, feedback positivo, comparación
  con yo mismo en el pasado.

**Relatedness (conexión):** sentir que soy parte de algo, no estoy solo.
- En el sistema: leagues, eventos comunitarios, friend system, comparación
  con peers similares.

**Propósito:** conectar lo que hago hoy con un objetivo significativo.
- En el sistema: tracks orientados a metas (Job Ready, Travel), recordatorios
  del por qué empecé, certificados al completar tracks.

Un buen sistema motivacional ataca los 4 ejes simultáneamente. Apps que
solo atacan uno (típicamente competencia con XP/badges) fallan a mediano
plazo.

---

## 2. Arquitectura del sistema motivacional

### 2.1 Capas del sistema

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
│  Catálogo de achievements, raridad, Sparks bonus            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  CAPA 4: SOCIAL Y COMUNIDAD                 │
│  Referidos, leagues, friend system, eventos                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  CAPA 5: PERSONALIZACIÓN IA                 │
│  Detección de qué motiva a cada usuario, mensajes únicos    │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Principios de diseño

**No saturar al usuario:** todas las capas son opt-in (excepto las básicas).
El usuario no debe ser bombardeado con celebraciones, leaderboards y
notificaciones simultáneamente.

**Calibrar magnitud con logro:** una celebración de "completaste 1 ejercicio"
no debe sentirse igual que "completaste tu primer track". Si todo es épico,
nada lo es.

**El sistema es aliado, no acosador:** detecta burnout, sugiere descanso,
respeta vacaciones. Apps que se sienten como obligación se abandonan.

**Personalizar el motivador:** algunos usuarios responden a streaks, otros
a logros, otros a comunidad. El sistema aprende y enfatiza lo que funciona
para cada uno.

---

## 3. Capa 1: Hábito diario

### 3.1 Daily Goals

#### Concepto

Cada día tiene un objetivo concreto, alcanzable y específico al usuario.
"Hoy: 12 minutos de práctica con foco en pronunciación de /θ/".

No es un genérico "practica inglés". Es un objetivo claro con criterio
binario de completitud.

#### Fuente del daily goal

Generado por el job nocturno basado en:
- Tiempo declarado disponible (`daily_minutes_available`).
- Plan del roadmap actual.
- Performance reciente.
- Foco pedagógico de hoy según el roadmap.

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

Una vez completado:
```
┌─────────────────────────────────────┐
│  ✅ Objetivo cumplido                │
│  +5 Sparks bonus                    │
│  Racha: 8 días 🔥                   │
└─────────────────────────────────────┘
```

#### Mecánica de "objetivo cumplido"

- Cumplir el objetivo otorga 2-5 Sparks bonus.
- Cumplir 7 días seguidos: badge especial + 20 Sparks.
- Si el usuario excede el objetivo (practica más), no penaliza ni recompensa
  proporcionalmente. Cumplir es el objetivo.

### 3.2 Streaks (Rachas)

#### Concepto

Días consecutivos cumpliendo el daily goal. Se muestra prominentemente en
la app y notificaciones.

#### Por qué streaks funcionan tan bien

Combinan tres mecánicas psicológicas potentes:
- **Endowment effect:** lo que tengo, no quiero perder.
- **Loss aversion:** perder 23 días duele más que la satisfacción de
  ganar 1 más.
- **Visible progress:** un número que sube es satisfactorio.

Pero tienen un lado oscuro: cuando se rompe un streak largo, mucha gente
abandona definitivamente. Necesitan mitigación.

#### Mitigaciones del lado oscuro

**Streak Freeze:** el usuario puede ganar/comprar "freezes" que protegen
el streak por un día perdido.

```
🧊 Tienes 2 Streak Freezes
Estos te salvan automáticamente si te pierdes un día.
```

Reglas:
- Se otorga 1 freeze gratis al alcanzar 7 días de streak.
- Se otorga 1 freeze gratis cada 30 días de streak completado.
- Máximo 3 freezes acumulados por vez.
- Un freeze se aplica automáticamente si el usuario no practica un día.
- El usuario es notificado: "Usamos un Streak Freeze. Tu racha sigue intacta".

**Streak Repair:** si se rompe el streak (sin freezes disponibles), opción
de "comprarlo de vuelta" pagando Sparks dentro de las primeras 24 horas.

```
😔 Tu racha de 23 días se rompió ayer
Pero podés recuperarla pagando 30 Sparks
[Recuperar mi racha]
```

Esto es psicológicamente potente: convierte la pérdida en agencia.

**Pause modes:** ver sección 3.4.

### 3.3 Visualización de streaks

#### Fire icon evolutivo

El icono de la racha evoluciona según duración:
- 1-6 días: 🔥 (llama pequeña)
- 7-29 días: 🔥🔥 (dos llamas)
- 30-99 días: ⚡ (electricidad)
- 100-364 días: 💎 (diamante)
- 365+ días: 👑 (corona)

Esto crea anticipación visual: "quiero llegar a la corona".

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

### 3.4 Anti-burnout: el sistema como aliado

#### Detección de fatiga

Señales de fatiga monitoreadas:
- Scores cayendo en sesiones consecutivas.
- Tiempo por ejercicio aumentando significativamente.
- Tasa de abandono de ejercicios subiendo.
- Sesiones más cortas que el promedio del usuario.

Si se detectan 3+ señales en 2 días consecutivos, sistema sugiere:
```
Notamos que las últimas sesiones fueron más difíciles.
Tu cerebro consolida mejor con descanso.

¿Querés tomarte el día?
[Sí, descansar] [No, sigo]

Si descansás, no afecta tu racha.
```

#### Vacation Mode

Opción explícita en settings:
```
🌴 Modo Vacaciones
"No me molesten por X días"
- Pausa notificaciones
- Pausa cuenta de streak (no se rompe)
- Mantiene Sparks y progreso intactos

[Activar 7 días] [Activar 14 días] [Personalizar]
```

Sin culpa, sin penalización. El usuario vuelve cuando puede.

#### Sugerencia de bajar metas

Si el usuario consistentemente no cumple su daily goal:
```
Notamos que tu objetivo de 30 min/día puede estar muy alto.
¿Querés ajustar a 15 min/día? Es mejor cumplir 15 min que fallar 30.

[Sí, ajustar] [No, lo intento]
```

Filosofía: mejor lograr metas modestas con consistencia que fallar metas
ambiciosas.

---

## 4. Capa 2: Progreso visible

### 4.1 El problema de la progresión percibida

Como mencionamos, en idiomas el progreso se siente plano aunque sea real.
Esta capa hace **VISIBLE el progreso invisible**.

### 4.2 Visualizaciones core

#### Gráfico de evolución de habilidades

Línea temporal con 4 dimensiones (pronunciación, fluidez, gramática,
vocabulario). 4-12 semanas visualizadas.

```
Tu evolución últimas 8 semanas
                                    ●
Pronunciación  ●━━●━━●━━●━━●━━●━━●━●
              60     70     80    +12

Fluidez       ●━━━━●━━━●━━●━━●━━●━●
              55      62        68  +6

[ver detalles]
```

Esta visualización es de las más motivadoras porque hace tangible algo
que se siente intangible.

#### "Tú vs tú hace 30 días"

Comparación side-by-side con audios reales del usuario:

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

Audios reales del usuario son la prueba más potente de progreso. El
usuario se escucha mejor, no es opinión, es objetivo.

#### Heatmap anual de actividad

Estilo GitHub contributions, pero con interpretación:

```
Tu año en inglés - 2026

[grid 52 semanas x 7 días, celdas con intensidad de color]

🟩 Día épico (>30 min)
🟢 Día completado (objetivo cumplido)
🟡 Día parcial
⬛ Día sin práctica

Total: 184 días practicados
Mejor mes: Mayo (28 días)
Racha más larga: 47 días
```

Visualmente satisfactorio para usuarios consistentes. Orgullo concreto.

#### Mapa visual del roadmap

El roadmap completo del track activo, mostrado como camino con:
- Niveles ya completados (verde, con checkmark).
- Nivel actual (destacado, con animación).
- Niveles futuros (grises, con preview de qué se aprenderá).
- Estimación: "estás en el 35% del camino".

### 4.3 Insights semanales

Cada domingo, el usuario recibe (vía email + in-app):

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

Generado por el job nocturno con ayuda de IA para los insights cualitativos.

### 4.4 Milestones celebrations

Calibración por magnitud:

| Magnitud | Trigger | Tipo de celebración |
|----------|---------|---------------------|
| Pequeño | Completar 1 ejercicio | Mensaje breve, feedback positivo |
| Medio | Completar daily goal | Animación + Sparks + toast |
| Grande | Desbloquear logro | Pantalla completa, animación, mensaje |
| Épico | Completar nivel del roadmap | Pantalla, audio, certificado parcial |
| Legendario | Completar track completo | Experiencia completa, certificado, opción compartir |

Cada celebración debe sentirse genuina, no robótica. El mensaje debe ser
personalizado y acorde al esfuerzo.

---

## 5. Capa 3: Logros y recompensas

### 5.1 Estructura del sistema de logros

#### Categorías de logros

1. **Constancia** - rachas y consistencia.
2. **Volumen** - acumulación de actividad.
3. **Maestría** - logros cualitativos en habilidades.
4. **Variedad** - exploración del producto.
5. **Roadmap** - hitos del progreso estructurado.
6. **Social** - interacción con comunidad.
7. **Especiales** - logros únicos o secretos.

#### Niveles de raridad

| Raridad | Color | Sparks bonus | % usuarios que lo desbloquean |
|---------|-------|-------------|-------------------------------|
| Común | Gris | 5-10 | >50% |
| Raro | Azul | 20-30 | 20-50% |
| Épico | Morado | 50-100 | 5-20% |
| Legendario | Dorado | 200-500 | <5% |
| Único | Iridiscente | Variable | <1% |

### 5.2 Catálogo completo de logros

#### 5.2.1 Constancia (15 logros)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|--------|
| `streak_3` | Empezando | 3 días seguidos | Común | 5 |
| `streak_7` | Primera semana | 7 días seguidos | Común | 10 |
| `streak_14` | Hábito formándose | 14 días seguidos | Raro | 20 |
| `streak_30` | Mes de hierro | 30 días seguidos | Raro | 30 |
| `streak_60` | Imparable | 60 días seguidos | Épico | 75 |
| `streak_100` | Centenario | 100 días seguidos | Épico | 100 |
| `streak_180` | Medio año | 180 días seguidos | Legendario | 250 |
| `streak_365` | Año completo | 365 días seguidos | Legendario | 500 |
| `daily_goal_7` | Cumplidor | 7 daily goals cumplidos | Común | 10 |
| `daily_goal_30` | Disciplinado | 30 daily goals cumplidos | Raro | 30 |
| `daily_goal_100` | Constante | 100 daily goals cumplidos | Épico | 100 |
| `weekend_warrior` | Guerrero del finde | Practicar 4 fines de semana seguidos | Raro | 25 |
| `early_bird` | Madrugador | 10 sesiones antes de las 7am | Raro | 25 |
| `night_owl` | Búho nocturno | 10 sesiones después de las 11pm | Raro | 25 |
| `comeback_kid` | Resurrección | Volver después de 14+ días | Raro | 30 |

#### 5.2.2 Volumen (10 logros)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|--------|
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

#### 5.2.3 Maestría (15 logros)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|--------|
| `pronunciation_master_th` | TH dominado | Score > 9 en /θ/ por 5 sesiones | Raro | 40 |
| `pronunciation_master_v` | V perfecta | Score > 9 en /v/ por 5 sesiones | Raro | 40 |
| `pronunciation_master_all` | Pronunciación impecable | Score promedio > 8.5 mantenido por 30 días | Épico | 150 |
| `fluency_breakthrough` | Fluidez liberada | WPM > 120 por 5 sesiones | Raro | 50 |
| `fluency_master` | Fluido como nativo | WPM > 150 mantenido | Épico | 150 |
| `no_filler` | Sin muletillas | Reducir muletillas 80% en 4 semanas | Raro | 60 |
| `vocab_b1_complete` | Vocabulario B1 | Dominar vocabulario nivel B1 completo | Raro | 50 |
| `vocab_b2_complete` | Vocabulario B2 | Dominar vocabulario nivel B2 completo | Épico | 100 |
| `grammar_pro` | Gramática pro | <2% errores gramaticales en 4 semanas | Épico | 100 |
| `level_up_b1_to_b2` | De B1 a B2 | Subir nivel CEFR oficialmente | Épico | 200 |
| `level_up_b2_to_c1` | De B2 a C1 | Subir nivel CEFR oficialmente | Legendario | 400 |
| `listening_native_speed` | Comprensión nativa | Comprender audio nativo > 90% | Épico | 100 |
| `accent_match` | Acento auténtico | Pronunciación reconocida como nativa | Legendario | 300 |
| `improvement_50` | Gran progreso | Mejorar score general > 50% | Épico | 150 |
| `improvement_100` | Transformación | Duplicar score general | Legendario | 350 |

#### 5.2.4 Variedad (8 logros)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|--------|
| `try_5_types` | Curioso | Probar 5 tipos de ejercicio | Común | 10 |
| `try_all_types` | Explorador | Probar todos los tipos | Raro | 30 |
| `versatile` | Versátil | Ejercicios en 5 categorías | Raro | 25 |
| `multilevel` | Multinivel | Bloques en 3 niveles distintos | Raro | 30 |
| `roleplay_5` | Actor | 5 roleplays distintos | Común | 20 |
| `roleplay_20` | Camaleón | 20 roleplays distintos | Raro | 60 |
| `pronunciation_drill_master` | Pronunciación intensiva | 100 ejercicios de pronunciación | Raro | 50 |
| `assessment_dedicated` | Auto-conocedor | Completar 3 re-evaluaciones | Raro | 75 |

#### 5.2.5 Roadmap (10 logros)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|--------|
| `first_block` | Primera lección | Completar primer bloque | Común | 5 |
| `first_level` | Primer nivel | Completar primer nivel del track | Común | 25 |
| `halfway` | Medio camino | 50% del track completado | Raro | 75 |
| `track_complete` | Track completo | Completar un track entero | Épico | 200 |
| `track_perfect` | Track perfecto | Track completo con >85% mastery | Épico | 300 |
| `multi_track` | Polyglot path | Completar 2 tracks | Legendario | 400 |
| `all_tracks` | Maestro de inglés | Completar todos los tracks disponibles | Legendario | 1000 |
| `fast_track` | Vía rápida | Completar track antes del estimado | Raro | 100 |
| `goal_achieved` | Misión cumplida | Lograr el objetivo declarado en onboarding | Épico | 250 |
| `deadline_made` | A tiempo | Cumplir antes del deadline declarado | Épico | 200 |

#### 5.2.6 Social (10 logros) [Fase 2+]

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|--------|
| `referral_1` | Embajador | Referir 1 amigo que se registre | Común | 20 |
| `referral_5` | Influencer | Referir 5 amigos | Raro | 100 |
| `referral_10` | Líder de comunidad | Referir 10 amigos | Épico | 250 |
| `referral_paid_1` | Promotor | Primer referido paga suscripción | Raro | 50 |
| `referral_paid_5` | Súper promotor | 5 referidos pagan | Épico | 250 |
| `friend_first` | Primer amigo | Agregar primer amigo en la app | Común | 10 |
| `challenge_won` | Ganador | Ganar un friend challenge | Raro | 30 |
| `league_top3` | Podio | Top 3 en liga semanal | Raro | 50 |
| `league_first` | Campeón | Primer puesto en liga semanal | Épico | 100 |
| `event_winner` | Ganador de evento | Top 10 en evento comunitario | Épico | 150 |

#### 5.2.7 Especiales y secretos (12 logros)

| ID | Nombre | Criterio | Raridad | Sparks |
|----|--------|----------|---------|--------|
| `holiday_practitioner` | Espíritu navideño | Practicar el 25 diciembre | Raro | 30 |
| `new_year_starter` | Año nuevo, hábito nuevo | Practicar el 1 de enero | Raro | 30 |
| `birthday_bash` | Cumpleaños bilingüe | Practicar en tu cumpleaños | Raro | 50 |
| `marathon_session` | Maratón | Sesión de 90+ minutos | Épico | 100 |
| `lightning_fast` | Velocista | Completar daily goal en <5 min | Raro | 25 |
| `perfectionist` | Perfeccionista | 10 ejercicios consecutivos perfectos | Épico | 100 |
| `night_marathon` | Trasnochada productiva | Sesión de 60+ min después de medianoche | Raro | 40 |
| `weekend_double` | Doble fin de semana | Practicar +60 min sáb y dom | Raro | 30 |
| `early_assessor` | Adelantado | Hacer assessment antes del día 7 | Raro | 40 |
| `feedback_giver` | Voz crítica | Enviar feedback útil 5 veces | Raro | 50 |
| `bug_hunter` | Cazador de bugs | Reportar bug confirmado | Épico | 100 |
| `legendary_first_year` | Pionero | Usuario de los primeros 1000 | Único | 500 |

**Total: 80 logros** distribuidos en 7 categorías y 5 niveles de raridad.

### 5.3 Detección y otorgamiento de logros

#### Engine de logros

```typescript
interface AchievementEngine {
  // Verifica todos los logros pendientes para un usuario
  checkAchievements(userId: string, trigger: TriggerEvent): Promise<Achievement[]>;

  // Otorga un logro al usuario
  grantAchievement(userId: string, achievementId: string): Promise<void>;

  // Calcula progreso hacia logros aún no obtenidos
  getProgressTowardsAchievements(userId: string): Promise<AchievementProgress[]>;
}

interface TriggerEvent {
  type: 'session_completed' | 'streak_updated' | 'level_completed' |
        'assessment_done' | 'referral_signup' | 'manual_check';
  data: any;
}
```

#### Estrategia de evaluación

**Evaluación inmediata** (en respuesta a eventos):
- Logros que dependen de un evento puntual (completar primer ejercicio,
  ganar liga).
- Triggered desde la lógica de negocio que generó el evento.

**Evaluación batch** (job nocturno):
- Logros acumulativos (totales, promedios).
- Logros de mantenimiento (mantener score por X días).
- Logros temporales (eventos calendario).

#### Notificación al usuario

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

### 5.4 Pantalla de colección de logros

Sección dedicada en la app que muestra:

- Total: 23/80 logros desbloqueados (29%).
- Filtro por categoría.
- Filtro por raridad.
- Logros bloqueados muestran progreso (ej: "12/14 días" para `streak_14`).
- Logros secretos muestran "???" hasta desbloquearlos.

Funciona como museo personal de logros, fuente de orgullo.

### 5.5 Schema en Postgres

```sql
CREATE TABLE achievements_catalog (
  id              TEXT PRIMARY KEY,
  category        TEXT NOT NULL,
  name            TEXT NOT NULL,
  description     TEXT NOT NULL,
  rarity          TEXT NOT NULL CHECK (rarity IN ('common', 'rare', 'epic', 'legendary', 'unique')),
  sparks_reward   INT NOT NULL,
  icon_url        TEXT,
  criteria        JSONB NOT NULL,
  is_secret       BOOLEAN NOT NULL DEFAULT false,
  is_active       BOOLEAN NOT NULL DEFAULT true,
  total_unlocked  INT DEFAULT 0  -- denormalized counter
);

CREATE TABLE user_achievements (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  achievement_id  TEXT NOT NULL REFERENCES achievements_catalog(id),
  unlocked_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  shared_count    INT DEFAULT 0,
  PRIMARY KEY (user_id, achievement_id)
);

CREATE INDEX idx_user_achievements_unlocked ON user_achievements(user_id, unlocked_at DESC);

CREATE TABLE achievements_progress (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  achievement_id  TEXT NOT NULL REFERENCES achievements_catalog(id),
  current_value   FLOAT NOT NULL,
  target_value    FLOAT NOT NULL,
  last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, achievement_id)
);
```

---

## 6. Capa 4: Social y comunidad

### 6.1 Sistema de referidos [Fase 1]

#### Mecánica

Cada usuario tiene un código de referido único. Comparte por WhatsApp,
redes sociales, etc.

Recompensas:
- Referido se registra: referente gana 10 Sparks, referido gana 10 Sparks bonus.
- Referido completa onboarding: referente gana 5 Sparks adicionales.
- Referido paga primer mes: referente gana 50 Sparks bonus.
- Cada referido subsecuente que paga: 30 Sparks bonus al referente.

#### Interfaz

```
┌─────────────────────────────────────┐
│   🎁 Invitá amigos, ganá Sparks     │
│                                     │
│   Tu código: MARIA-LATAM-XK29       │
│                                     │
│   Por cada amigo que se registra:   │
│   • Vos ganás 10 Sparks ⚡          │
│   • Tu amigo gana 10 Sparks bonus   │
│                                     │
│   Si tu amigo paga su suscripción:  │
│   • Vos ganás 50 Sparks adicionales │
│                                     │
│   [Compartir por WhatsApp]          │
│   [Compartir por Instagram]          │
│   [Copiar link]                     │
│                                     │
│   Tus referidos: 3                  │
│   Ganados con referidos: 95 ⚡       │
└─────────────────────────────────────┘
```

#### Anti-abuso

- Limit de 100 Sparks por mes en bonos de referidos para nuevos usuarios.
- Detección de cuentas duplicadas (IP, device fingerprint, patrones).
- Verificación de email/teléfono para que cuente como referido válido.
- Auditoría manual de usuarios con > 50 referidos en 30 días.

### 6.2 Compartir externos [Fase 1]

#### Tipos de compartibles

**Compartir logro desbloqueado:**
```
[Imagen generada con el logro]
"¡Acabo de desbloquear 'Mes de hierro' en MyEnglishApp!
30 días seguidos practicando inglés 🔥
Mejor decisión de este año.
#aprendiendoinglés"
```

**Compartir progreso semanal:**
```
[Imagen con stats de la semana]
"Esta semana en inglés:
✅ 7 días con racha
🎯 124 minutos practicados
📈 Pronunciación: +12 puntos
[link de la app]"
```

**Compartir certificado al completar track:**
```
[Imagen del certificado]
"Acabo de completar 'Job Ready' en MyEnglishApp.
14 semanas, 68 lecciones. ¡Listo para mi próxima entrevista!"
```

#### Generación de imágenes

Imágenes generadas server-side con templates predefinidos. Cada imagen:
- Branded con logo discreto.
- Datos personalizados del usuario.
- Diseño compartible (formato vertical para Instagram Stories, cuadrado
  para feed, horizontal para Twitter).
- QR code o link corto para que viewers se registren.

### 6.3 Friend System [Fase 2]

#### Funcionalidad

- Agregar amigos por código de referido o búsqueda por username.
- Ver perfil del amigo: streak actual, total practicado, logros recientes.
- Sentir presencia social sin estar compitiendo agresivamente.

#### Activity feed

Sección "Amigos" muestra:
- "Carlos completó 'First level'" - hace 2 horas.
- "Ana mantiene su racha de 45 días 🔥" - hoy.
- "Pedro desbloqueó 'Pronunciation impecable'" - ayer.

Esto crea sensación de comunidad sin presión competitiva.

#### Reactions y aliento

Reacciones rápidas a logros de amigos:
- 🔥 (impresionante)
- 💪 (vamos)
- 👏 (felicitaciones)
- 🚀 (a la luna)

Sin chat directo en MVP de friend system. Mantener simple, evitar moderación
compleja.

### 6.4 Friend Challenges [Fase 2]

#### Tipos de desafíos

**Challenge de minutos:**
"Quien practique más minutos en 7 días gana"

**Challenge de constancia:**
"Quien mantenga su racha por más días gana"

**Challenge de mejora:**
"Quien mejore más su score de pronunciación gana"

**Challenge de ejercicios:**
"Quien complete más ejercicios en 14 días gana"

#### Mecánica

- Iniciador propone el challenge a un amigo.
- Amigo acepta o rechaza.
- Ambos ven progreso en tiempo real.
- Ganador recibe Sparks, ambos reciben participación.
- Empates: ambos reciben recompensa de ganador.

### 6.5 Leagues semanales [Fase 2]

#### Concepto

Cada semana, el usuario es asignado automáticamente a una liga de 30
usuarios con perfil similar (mismo país, nivel CEFR similar, frecuencia
de uso similar).

#### Mecánica

Lunes a domingo, todos compiten por puntos basados en:
- Minutos practicados (1 punto por minuto).
- Daily goals cumplidos (10 puntos por goal).
- Logros desbloqueados (variable según raridad).
- Ejercicios completados (2 puntos por ejercicio).

Domingo a las 23:59, se cierra la liga:
- Top 3: suben de división, reciben Sparks bonus (50, 30, 20).
- Top 10: se mantienen en su división, reciben pequeño bonus.
- Bottom 5: bajan de división.
- Resto: se mantienen.

#### Divisiones

- Bronce 1, 2, 3
- Plata 1, 2, 3
- Oro 1, 2, 3
- Platino
- Diamante
- Maestro (top 1% global)

Esta estructura mantiene la competencia interesante a largo plazo.

#### Por qué leagues funcionan

Investigación de Duolingo demostró que leagues:
- Aumentan retención significativamente.
- Crean compromiso semanal claro.
- Comparación con peers (no top globales) es motivante, no desmotivante.

### 6.6 Eventos comunitarios [Fase 3]

#### Tipos de eventos

**Marathons temporales:**
"Marathon de pronunciación: 1-3 mayo"
- Objetivo colectivo: alcanzar X minutos de pronunciación entre todos.
- Si se logra: todos los participantes reciben recompensa.
- Crea sensación de comunidad colaborativa.

**Themed weeks:**
"Semana del Job Ready: 15-21 abril"
- Doble Sparks por completar bloques del track Job Ready.
- Logros temporales especiales.

**Country challenges:**
"México vs Argentina: ¿quién practica más esta semana?"
- Competencia amistosa por país.
- Genera identidad y orgullo regional.

**Eventos por temporadas:**
- Resoluciones de año nuevo (1-7 enero).
- Vuelta a clases (febrero/marzo, agosto).
- Día del estudiante (varía por país).
- Vacaciones (julio, diciembre).

#### Implementación

Eventos como entidades configurables:

```sql
CREATE TABLE community_events (
  id              UUID PRIMARY KEY,
  name            TEXT NOT NULL,
  description     TEXT,
  event_type      TEXT NOT NULL,
  starts_at       TIMESTAMPTZ NOT NULL,
  ends_at         TIMESTAMPTZ NOT NULL,
  target_countries TEXT[],
  rules           JSONB NOT NULL,
  rewards         JSONB NOT NULL,
  status          TEXT NOT NULL CHECK (status IN ('upcoming', 'active', 'ended')),
  participants_count INT DEFAULT 0
);
```

---

## 7. Capa 5: Personalización IA

### 7.1 El sistema aprende qué motiva a cada usuario

Diferentes usuarios responden a diferentes mecánicas. El sistema detecta
patrones y enfatiza lo que funciona para cada uno.

#### Señales de motivación primaria

**Usuario "streak-driven":**
- Abre la app principalmente cuando le notifican que su racha está en peligro.
- Sus mejores días de actividad coinciden con momentos de "salvar racha".
- → Sistema enfatiza notificaciones de streak, freeze tokens, etc.

**Usuario "achievement-hunter":**
- Pasa tiempo viendo la sección de logros.
- Completa más ejercicios cuando está cerca de un logro.
- → Sistema le muestra logros próximos, personaliza notificaciones tipo
  "estás cerca de X".

**Usuario "social":**
- Activa friend system, comparte logros, mira activity feed.
- Mejor performance en leagues que en práctica solo.
- → Sistema le sugiere challenges, le notifica cuando amigos están
  practicando, fomenta más comunidad.

**Usuario "progress-driven":**
- Pasa tiempo viendo gráficos de progreso.
- Hace re-evaluaciones cuando están disponibles.
- → Sistema le da más insights, comparaciones temporales, predicciones.

**Usuario "purpose-driven":**
- Conectado fuertemente a su objetivo declarado.
- Pregunta frecuentemente sobre su roadmap.
- → Sistema le recuerda más su "por qué", muestra progreso hacia objetivo
  específico.

### 7.2 Mensajes personalizados generados por IA

El job nocturno genera mensajes motivacionales únicos según el perfil:

**Para streak-driven:**
"Tu racha de 23 días está en juego. ¿Te quedan 4 horas para mantenerla?"

**Para achievement-hunter:**
"Estás a 3 ejercicios de desbloquear 'Maratonista'. ¿Vamos por él?"

**Para social:**
"Carlos acaba de completar el nivel que vos estás haciendo. ¿Lo alcanzás?"

**Para progress-driven:**
"Tu pronunciación mejoró 15% en 4 semanas. Mira tu evolución →"

**Para purpose-driven:**
"Recordá: empezaste por la entrevista en TechCorp. Estás 60% del camino."

### 7.3 Detección automática del perfil motivacional

Algoritmo conceptual:

```python
def calculate_motivation_profile(user_id):
    behaviors = get_user_behaviors_last_30_days(user_id)

    scores = {
        'streak_driven': calc_streak_response(behaviors),
        'achievement_hunter': calc_achievement_response(behaviors),
        'social': calc_social_response(behaviors),
        'progress_driven': calc_progress_response(behaviors),
        'purpose_driven': calc_purpose_response(behaviors)
    }

    primary = max(scores, key=scores.get)
    secondary = sorted(scores.items(), key=lambda x: -x[1])[1][0]

    return {
        'primary_motivator': primary,
        'secondary_motivator': secondary,
        'all_scores': scores
    }
```

Recalculado mensualmente. Mensajes y experiencia se ajustan según resultado.

---

## 8. Diferenciación cultural latinoamericana

### 8.1 Logros con sabor local

Algunos logros con referencias culturales (sin caer en cliché):

| ID | Nombre | Criterio | País |
|----|--------|----------|------|
| `cinco_de_mayo` | 5 de Mayo | Practicar el 5 de mayo | México |
| `dia_del_amigo` | Día del amigo | Practicar el 30 de julio | Argentina |
| `independencia_mx` | Mes de la patria | Practicar todo septiembre | México |
| `anniversary_ar` | 25 de Mayo | Practicar el 25 mayo | Argentina |
| `colombia_day` | Independencia | Practicar el 20 julio | Colombia |
| `ave_fenix` | Ave Fénix | Volver después de 30+ días sin práctica | Todos |
| `aguante` | Aguante | Practicar todos los días de un mes lluvioso/de exámenes | Todos |

### 8.2 Eventos calendario por país

Sistema de eventos respeta calendarios locales:

**México:**
- Día del Estudiante (23 mayo): challenge especial.
- Vacaciones de verano (julio-agosto): "vacation track" con vocabulario
  de viaje.
- Día de Muertos (1-2 nov): evento especial.
- Posadas (16-24 dic): challenge de fin de año.

**Argentina:**
- Día del Estudiante (21 sept): primavera, evento especial.
- Vacaciones de invierno (julio): challenge de inverno.
- Mundial (cuando aplique): evento Football English.

**Colombia, Chile, Perú:** calendarios específicos respetados.

### 8.3 Tono de comunicación

El tono debe sentirse latinoamericano sin caer en estereotipos:

**Bien:**
- "Vamos, vos podés" (genérico y cálido).
- "Tu meta te espera" (motivador).
- "Un día más, un paso más" (filosófico).

**Evitar:**
- Modismos muy locales que excluyen a otros países.
- Excesiva familiaridad ("flaco", "compa").
- Tono gringo traducido literalmente ("¡Tú lo lograste, campeón!").

---

## 9. Plan de implementación por fases

### 9.1 Fase 1: MVP de motivación (meses 0-3)

**Crítico:**
- Daily goals con visualización clara.
- Streaks con freeze tokens.
- Anti-burnout básico (vacation mode, sugerencia bajar metas).
- Catálogo de 30 logros core (constancia, volumen, roadmap).
- Engine de detección de logros.
- Pantalla de colección de logros.
- Compartir logros externos.
- Sistema de referidos básico.

**Importante:**
- Visualizaciones de progreso (gráfico de evolución, heatmap).
- Insights semanales por email.
- Milestones celebrations.

### 9.2 Fase 2: Profundización (meses 3-6)

**Crítico:**
- "Tú vs tú hace 30 días" con audios.
- Leagues semanales por región/nivel.
- Mapa visual del roadmap.
- Catálogo expandido a 60 logros.

**Importante:**
- Friend system básico (agregar, ver actividad).
- Friend challenges.
- Comeback mechanics (recuperar streak por Sparks).

### 9.3 Fase 3: Social profundo (meses 6-12)

**Crítico:**
- Eventos comunitarios temporales.
- Country challenges.
- Catálogo completo de 80 logros.
- Detección automática de perfil motivacional.

**Importante:**
- Mensajes personalizados según perfil motivacional.
- Logros culturales por país.
- Activity feed con reactions.

### 9.4 Fase 4: Sofisticación (año 2+)

- Mentoría peer-to-peer.
- Foros temáticos.
- Marathon collaborativos globales.
- Sistema de "coaches" (usuarios premium ayudan a principiantes).
- Logros co-creados con la comunidad.

---

## 10. Métricas críticas

### 10.1 Salud del sistema motivacional

**Engagement:**
- Daily Active Users (DAU).
- Stickiness (DAU/MAU).
- Sesiones por usuario por semana.
- Streak average length.
- % de usuarios con streak activo > 7 días.

**Logros:**
- Logros desbloqueados promedio por usuario.
- Distribución por raridad (debería ser piramidal: muchos comunes, pocos legendarios).
- % de usuarios que ven la pantalla de logros frecuentemente.

**Social:**
- % de usuarios con al menos 1 referido.
- Conversion rate de referidos.
- % de usuarios participando en leagues.
- Engagement con friend system.

### 10.2 Predictores de retención

Patrones que indican alta probabilidad de retención a 90 días:
- Completó el primer logro en los primeros 3 días.
- Tiene streak >= 7 días en la primera semana.
- Vio sus stats de progreso al menos 3 veces.
- Activó al menos una mecánica social.

Sistema usa estos patrones para identificar usuarios en riesgo de churn
y aplicar intervenciones específicas.

### 10.3 Anti-señales

Patrones que indican burnout o desencanto:
- Cumplir daily goal pero no exceder por 14+ días seguidos (mínimo esfuerzo).
- Caer abajo del 25% en 3 leagues consecutivas (frustración competitiva).
- Reducir tiempo por sesión consistentemente.

Sistema detecta y sugiere ajustes (bajar metas, modo descanso, cambio de
foco).

---

## 11. Decisiones abiertas

- [ ] ¿Sparks o XP separado para gamificación? (Mi recomendación: solo
  Sparks para no fragmentar economía).
- [ ] ¿Logros borrables si el usuario hace cheating obvio? Política de
  fraude.
- [ ] ¿Mostrar % de usuarios que tienen cada logro o no? Pros: hace
  raridad concreta. Contras: puede desmotivar.
- [ ] ¿Permitir que usuarios "donen" Sparks a amigos? Mecánica social
  poderosa pero riesgo de abuso.
- [ ] ¿Sistema de "rivales" amistosos (siempre comparar con un usuario
  específico de nivel similar)? Inspirado en Strava.

---

## 12. Riesgos y mitigaciones

| Riesgo | Probabilidad | Mitigación |
|--------|-------------|-----------|
| Saturación de notificaciones de logros | Alta | Calibración cuidadosa, opt-out granular |
| Cheating de logros (cuentas múltiples) | Media | Anti-fraude, detección de patrones |
| Frustración por no llegar al top en leagues | Alta | Divisiones para que cada uno compita con peers |
| Streak addiction tóxica | Media | Vacation mode, freeze tokens, mensajes anti-burnout |
| Eventos comunitarios con baja participación | Media | Cap mínimo, recompensas garantizadas |

---

## 13. Referencias internas

- `docs/business/plan_de_negocio.docx` — Plan de negocio.
- `docs/product/student-profile-and-assessment.md` — Perfil de estudiante.
- `docs/product/ai-roadmap-system.md` — Sistema de roadmap.
- `docs/architecture/sparks-system.md` — Sistema de Sparks (recompensas).
- `docs/architecture/notifications-system.md` — Notificaciones de logros.
- `docs/architecture/ai-gateway-strategy.md` — Generación de mensajes personalizados.

---

## 14. Referencias externas e inspiración

- Duolingo's gamification system (referencia, no copia).
- Strava's segments and leaderboards.
- GitHub contributions heatmap (visualización).
- Headspace's streak system (manejo saludable).
- Self-Determination Theory (Deci & Ryan).

---

*Documento vivo. Actualizar cuando se agreguen logros, mecánicas
sociales, o se observen patrones que sugieran ajustes al sistema
motivacional.*

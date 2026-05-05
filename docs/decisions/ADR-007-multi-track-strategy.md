# ADR-007: Estrategia multi-track — un track activo a la vez en MVP

**Status:** Accepted
**Date:** 2026-05
**Author:** —
**Audiencia:** agente AI implementador.

---

## Contexto

El producto define **3 tracks pedagógicos** para el MVP, cada uno con
su propio plan completo de bloques B1→B2+:

| Track | Bloques MVP | Contexto |
|-------|------------:|----------|
| Job Ready | 74 | Trabajo profesional / entrevistas USA |
| Travel Confident | 39 | Viajes a destinos angloparlantes |
| Daily Conversation | 62 | Conversación social cotidiana |
| **Total MVP** | **175** | |

(Detalle por track en `track-job-ready-blocks.md`,
`track-travel-confident-blocks.md`,
`track-daily-conversation-blocks.md`.)

Surge naturalmente la pregunta:

> **¿Un usuario puede tener múltiples tracks activos simultáneamente?**

La respuesta no es trivial:

1. **Argumento pro-multi:** muchos usuarios tienen objetivos múltiples
   (un developer que quiere conseguir trabajo en USA y también va a
   viajar; un estudiante que quiere conversación casual y también
   prep para entrevistas).
2. **Argumento pro-single:** un solo track activo da claridad mental,
   reduce cognitive load del user, simplifica el roadmap engine,
   permite progress tangible y rápido.
3. **Argumento pedagógico:** Daily Conversation genera atomics
   universales que son foundation para los otros 2 tracks. Hacer DC
   primero acelera el progreso en JR y TC posteriormente.
4. **Argumento de modelo de datos:** sub-skill mastery (de
   `pedagogical-system.md` §2.3) es **track-orthogonal** — un user
   que mejora `pron_th_voiceless` lo mejora globalmente, no por
   track.

`ai-roadmap-system.md` §14.2 ya estableció "single track en MVP" como
decisión preliminar. Este ADR formaliza esa decisión y especifica los
detalles operativos que faltaban: selección inicial, switching,
transferencia de mastery, implicaciones del paywall.

## Decisión

**Un solo track activo a la vez en MVP. User puede switch tracks via
Settings con confirmación. Sub-skill mastery transfiere globalmente
entre tracks. Multi-track simultáneo se reconsidera año 2 con datos.**

### Estructura

| Componente | Comportamiento |
|------------|----------------|
| **Track activo** | Exactamente uno por user en cualquier momento |
| **Selección inicial** | Determinada en onboarding por `primary_goals` |
| **Switching** | Permitido vía Settings → "Cambiar track", con confirmación y reset de posición en el nuevo track (pero NO reset de mastery) |
| **Mastery transfer** | Sub-skills mastery se mantiene global (transfiere). Block completion es por-track (no transfiere) |
| **Roadmap engine** | Genera roadmap solo del track activo |

### Selección inicial del track (onboarding Day 0)

Lógica determinística basada en `student_profile.primary_goals`:

```typescript
function determineInitialTrack(profile: StudentProfile): TrackId {
  const goals = profile.primary_goals;

  // Job-related goals → Job Ready
  const jobGoals = ['job_interview', 'remote_work',
                    'business_communication', 'exam_prep'];
  if (goals.some(g => jobGoals.includes(g))) {
    return 'job_ready';
  }

  // Travel goal → Travel Confident
  if (goals.includes('travel')) {
    return 'travel_confident';
  }

  // Default fallback → Daily Conversation
  // Casos: 'studies', 'personal_growth',
  // 'communication_with_family', 'entertainment'
  return 'daily_conversation';
}
```

**Reglas de desempate:**
- Si user selecciona múltiples goals que apuntan a tracks distintos
  (ej: `job_interview` + `travel`): priorizar **el primer goal en el
  orden seleccionado** por user (UI captura este orden).
- Mostrar al final del onboarding: "Vamos a empezar con [track]. Más
  adelante puedes cambiar o agregar otros desde Settings."

### Switching de track

User puede cambiar track desde Settings → "Mi plan" → "Cambiar track":

1. UI muestra confirmación con copy claro:
   > "Vas a cambiar de [track actual] a [track nuevo]. Tu progreso de
   > sub-skills se mantiene, pero empezarás desde el principio del
   > nuevo track. ¿Continuar?"
2. Si confirma:
   - Archive `roadmap` actual con flag `archived_reason = 'track_switched'`.
   - Llamar `ai-roadmap.generateInitialRoadmap` con nuevo track.
   - **Mantener** `user_subskill_mastery` (sub-skills no se resetean).
   - Emit event `user.track_switched` con `from`, `to`, `at`.
3. UI muestra nuevo roadmap inmediato.

**Rate limit:** max 1 switch cada 14 días (evita abuse / decisión
errática). Si user intenta switch antes: prompt "Ya cambiaste track
hace X días, ¿seguro?". Hard block después de 3 switches en 60 días.

### Transferencia de mastery entre tracks

```
┌──────────────────────────────────────────────────────────────┐
│  Lo que TRANSFIERE entre tracks:                             │
│  ├── user_subskill_mastery (todas las 50 sub-skills)         │
│  ├── observed_behavior (preferencias, patrones)              │
│  ├── CEFR overall del user                                   │
│  ├── streaks, daily goals, achievements                      │
│  └── Sparks balance                                          │
│                                                              │
│  Lo que NO transfiere:                                        │
│  ├── Posición en bloques (start de cero en track nuevo)      │
│  ├── Bloques marcados completed (cada track tiene los suyos) │
│  └── Capstone completions (cada track tiene los suyos)       │
└──────────────────────────────────────────────────────────────┘
```

**Implicación:** un user que viene de Job Ready y switch a Travel
Confident va a saltar muchos bloques iniciales de TC vía test-out
(porque sus sub-skills ya están en `mastered`). Esto es **deseable**
— evita aburrimiento y respeta el progreso ya logrado.

### Implicaciones del paywall y trial

- **Trial (Day 1-7):** track inicial determinado en onboarding. No
  switch durante trial (mantener foco).
- **Day 7+ (post-assessment):** assessment puede recomendar switch si
  measured CEFR + observed_behavior sugieren mismatch con track
  actual. UI prompt "Considera cambiar a [track]" pero user decide.
- **Plan pago:** todos los planes (Básico, Pro, Premium) incluyen
  acceso a switch sin restricciones adicionales.
- **NO hay "premium track"**: ningún track es exclusivo de Premium.
  Todos los tracks accesibles desde Day 0.

### Roadmap engine implications

`ai-roadmap-system.md` recibe el track activo como parámetro y genera
un roadmap **solo** de bloques de ese track:

```python
def generate_roadmap(profile, active_track):
    available_blocks = filter_blocks_by_track(active_track)
    available_blocks = filter_by_cefr(available_blocks, profile.cefr)
    available_blocks = filter_by_prerequisites(available_blocks, profile)
    available_blocks = filter_by_subskill_mastery(available_blocks, profile)
    # ... genera secuencia
    return roadmap
```

**Excepción cross-track:** algunos bloques de Daily Conversation B1
son recomendados como "exposure" en roadmaps de Job Ready /
Travel cuando el user muestra gaps en sub-skills foundational
(ej: `flu_turn_taking` aparece en JR y DC; si user en JR no domina
turn-taking, el roadmap puede incluir `dc_b1_phone_call_personal` como
exposure block aunque no sea su track activo).

**Esto NO es "track múltiple"** — es exposure ocasional dentro del
roadmap del track activo. Max 10% de los bloques del roadmap pueden
ser cross-track exposure.

## Alternativas consideradas

### A. Múltiples tracks simultáneos (full multi-track)

**Rechazada para MVP.**

- Cognitive overload para el user (¿cuál bloque hago hoy?).
- Roadmap engine 3x más complejo (interleaving inteligente entre
  tracks).
- Métrica de progreso confusa ("estás 30% en Job Ready, 50% en
  Travel" → ninguno se siente cerca).
- Decisión de UI: ¿qué track mostrar como "primary" en home? Sin
  default obvio.

**Reconsiderar año 2** si data muestra que users completan track 1 y
quieren entrar a track 2 sin perder momentum. En ese caso, multi-track
podría ser feature de Premium.

### B. Track híbrido "Foundation + Especializado"

**Considerada, rechazada.**

Idea: Daily Conversation siempre activo como "foundation" + un track
especializado (JR o TC) corriendo en paralelo. Argumento: DC es
foundation universal, ya generamos atomics para él, ¿por qué no
entregarlo al user en paralelo?

**Razones de rechazo:**
- Dilution de progreso: 2 tracks simultáneos = avance lento en ambos
  → frustración.
- Increased Sparks consumption: more blocks practiced per day.
- Mejor pedagógicamente que el user **complete DC primero** si es
  débil en conversational fluency, y **después** especialice. Esto se
  logra via switch (de DC a JR cuando complete DC), no
  paralelizando.

### C. Track único inmutable (no switching)

**Rechazada.**

Sin switching, el user que descubre mid-trial que eligió mal queda
atrapado. Genera support tickets y churn.

### D. Free switching ilimitado

**Rechazada.**

Sin rate limit, users indecisos rotan entre tracks sin completar
ninguno. Se pierde el momentum del track. Rate limit de 14 días es
suave pero da peso a la decisión.

### E. Switch via re-onboarding completo

**Rechazada.**

Forzar re-onboarding para cambiar track agrega 5-7 minutos de
fricción innecesaria. Sub-skill mastery + observed_behavior ya son
suficiente para regenerar roadmap del nuevo track sin re-preguntar.

## Consecuencias

### Positivas

- **Foco mental:** user sabe siempre qué track está practicando.
- **UX simple:** home screen muestra "tu plan" sin ambigüedad.
- **Roadmap engine simple:** filtra blocks por un único track activo.
- **Métrica de progreso clara:** "Estás 45% en Job Ready" es legible.
- **Cross-track atomic reuse aprovechado** sin requiring multi-track
  UI complexity.
- **Mastery transfer respeta el progreso del user** al cambiar track.

### Negativas

- Users con goals múltiples genuinos sienten que "deben elegir".
  Mitigación: UI explícita "Más adelante puedes cambiar". Switching
  fácil reduce el peso de la decisión inicial.
- Logo `roadmap.completed` requiere completar todo un track antes de
  iniciar otro (si user no switch antes). Para users que solo quieren
  el "highlight reel" de cada track esto es lento. Mitigación
  post-MVP: feature "playlist mode" para users avanzados.
- AI Roadmap engine requiere lógica de cross-track exposure (max 10%)
  para gaps foundational, agregando complejidad menor al roadmap
  generation.

### Riesgos a monitorear

- **Switch rate excesivo (> 20% de users en 30 días):** señal de mala
  selección inicial → revisar onboarding question 3 (objetivo) o
  agregar question específica de track preference.
- **Switch rate muy bajo (< 2% en 6 meses):** señal de que users no
  saben que pueden switchear → mejorar UI Settings.
- **Track stickiness:** si users completan al 30% y abandonan en lugar
  de switchear, el problema NO es track sino dificultad / engagement.
  No mover a multi-track como solución.

### A/B tests prioritarios (post-MVP)

1. **Track recommendation post-assessment:** ¿el assessment debe
   recomendar switch si detecta mismatch?
2. **Switch friction:** ¿confirmation simple vs confirmation con
   "are you sure" + "tip de cuándo cambiar"?
3. **Cross-track exposure ratio:** 0% vs 10% vs 20% — ¿user retention
   mejor con más o menos exposure cross-track?
4. **Multi-track simultaneous (año 2):** experiment activo con
   subset de users, medir completion rate, retention, NPS.

## Métricas de éxito

| Métrica | Target MVP |
|---------|-----------|
| % users que mantienen track inicial 90+ días | ≥ 70% |
| Switch rate primer 30 días | 5-15% (sweet spot) |
| Track completion rate (B1 → B2+) | ≥ 8% in 12 months |
| Cross-track exposure success: % de exposure blocks completados | ≥ 60% |
| Retention día 30 difference: ¿single-track users vs hipotético multi? | (a medir post-MVP) |

## Implementación

### Schema implications

`student_profiles` (de `student-profile-and-assessment.md` §3.1) ya
tiene `primary_goals`. Agregar:

```sql
ALTER TABLE student_profiles
  ADD COLUMN active_track TEXT NOT NULL DEFAULT 'daily_conversation'
    CHECK (active_track IN ('job_ready', 'travel_confident', 'daily_conversation')),
  ADD COLUMN track_switched_count INT NOT NULL DEFAULT 0,
  ADD COLUMN last_track_switch_at TIMESTAMPTZ;

CREATE INDEX idx_profiles_active_track ON student_profiles(active_track);
```

### API contracts

**`GET /tracks/available`** — lista de tracks disponibles + bloques
totales + estimación de tiempo restante para user actual.

**`POST /tracks/switch`** — switch del active track:

```typescript
interface SwitchTrackRequest {
  user_id: string;
  to_track: 'job_ready' | 'travel_confident' | 'daily_conversation';
  confirmation: true;       // explicit
}

interface SwitchTrackResponse {
  new_active_track: string;
  new_roadmap_id: string;
  message: string;          // mensaje al user con summary
}
```

**Reglas:**
- Verificar rate limit (1 switch/14 días).
- Archive roadmap actual.
- Generate nuevo roadmap.
- Preserve sub-skill mastery + observed_behavior.
- Emit `user.track_switched` event.

### Eventos emitidos

(De `cross-cutting/data-and-events.md` §5.X — agregar).

| Evento | Cuándo |
|--------|--------|
| `user.track_selected` | En onboarding completion (initial track) |
| `user.track_switched` | User cambia track via Settings |
| `roadmap.regenerated_for_track_switch` | Cuando se regenera roadmap por switch |

## Referencias

- [`docs/product/ai-roadmap-system.md`](../product/ai-roadmap-system.md)
  §14.2 — decisión cerrada de single-track en MVP.
- [`docs/product/curriculum-by-cefr.md`](../product/curriculum-by-cefr.md)
  §11 — matriz tracks × CEFR.
- [`docs/product/track-job-ready-blocks.md`](../product/track-job-ready-blocks.md)
  — desglose de Job Ready (74 bloques).
- [`docs/product/track-travel-confident-blocks.md`](../product/track-travel-confident-blocks.md)
  — desglose de Travel Confident (39 bloques).
- [`docs/product/track-daily-conversation-blocks.md`](../product/track-daily-conversation-blocks.md)
  — desglose de Daily Conversation (62 bloques).
- [`docs/product/student-profile-and-assessment.md`](../product/student-profile-and-assessment.md)
  §3.1 — `student_profiles.primary_goals` que determina track inicial.
- [`docs/product/pedagogical-system.md`](../product/pedagogical-system.md)
  §2.3 — modelo de mastery por sub-skill (track-orthogonal).
- [`docs/decisions/ADR-006-trial-assessment.md`](ADR-006-trial-assessment.md)
  — trial 7 días donde el track es elegido.

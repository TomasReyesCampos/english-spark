# US-012: Pregunta objetivo + deadline (Pantallas 4-5)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Pantallas 4 y 5 del onboarding capturan los 2 inputs más
importantes para personalización:

1. **Objetivo principal** (multi-select de 9 opciones) → determina
   el **track inicial** del producto (ADR-007).
2. **Deadline** (4 opciones) → influye en agresividad del roadmap
   y prioritization de bloques.

Combiné ambas pantallas en una story porque son secuenciales,
comparten patrones de UI (multi-select con opciones predefinidas) y
juntas determinan el setup pedagógico del user.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §5
(objetivo) y §6 (deadline).

## Scope

### In

- **Pantalla 4: Objetivo principal**
  - Pregunta: "¿Qué quieres lograr?"
  - 9 opciones multi-select (max 3 selecciones):
    - Conseguir trabajo
    - Trabajo remoto / freelance
    - Mejor comunicación en mi trabajo actual
    - Viajar
    - Estudios (universidad, posgrado)
    - Crecimiento personal
    - Examen oficial (TOEFL, IELTS, etc.)
    - Comunicación con familia
    - Entretenimiento (películas, música)
  - Orden visual (ranking) determina el primary goal.
  - CTA disabled hasta seleccionar al menos 1.
  - Persiste en `student_profiles.primary_goals` (TEXT[]) y el
    primero del array en `active_track` (de ADR-007).
- **Pantalla 5: Deadline**
  - Pregunta: "¿Para cuándo quieres lograrlo?"
  - 4 opciones single-select:
    - Sí, en menos de 1 mes
    - En 1 a 3 meses
    - En 3 a 6 meses
    - No tengo apuro
  - Persiste en `student_profiles.has_deadline` y `deadline_date`
    (calculado a partir de la opción).
- Track inicial determinístico (ADR-007 §"Selección inicial"):
  - `job_interview`, `remote_work`, `business_communication`,
    `exam_prep` → `job_ready`.
  - `travel` → `travel_confident`.
  - Default → `daily_conversation`.
- Telemetry: `onboarding.goal_seen`, `.goal_selected`,
  `.deadline_seen`, `.deadline_selected`,
  `.track_initial_assigned`.

### Out

- Cambiar track desde Settings (lógica de switch) — story propia.
- Examen target específico (TOEFL vs IELTS vs ...) — captura
  inicial; spec específico de exam_prep es post-MVP feature.

## Acceptance criteria

- **Given** user en pantalla 4 sin selección, **When** se renderiza,
  **Then** CTA "Continuar" está disabled.
- **Given** user selecciona "Conseguir trabajo" + "Viajar" + "Crecimiento
  personal" en ese orden, **When** tapea Continuar, **Then**
  `student_profiles.primary_goals = ['job_interview', 'travel',
  'personal_growth']` y `active_track = 'job_ready'` (primer goal
  determina track).
- **Given** user intenta seleccionar 4ta opción, **When** ya tiene
  3 seleccionadas, **Then** UI lo previene con mensaje "Máximo 3
  objetivos" y la 4ta no se selecciona.
- **Given** user en pantalla 5, **When** selecciona "En 1 a 3
  meses", **Then** `student_profiles.has_deadline = true` y
  `deadline_date = now() + 60 días` (mediano de 1-3m).
- **Given** user selecciona "No tengo apuro", **When** confirma,
  **Then** `has_deadline = false` y `deadline_date = null`.
- **Given** user con goal `travel` único, **When** completa,
  **Then** `active_track = 'travel_confident'` y se emite
  `onboarding.track_initial_assigned` con
  `track: 'travel_confident'`.
- **Given** user con goals que no encajan en JR ni TC (ej:
  `entertainment` solo), **When** completa, **Then** `active_track
  = 'daily_conversation'` (default fallback).
- **Given** request a `/profile/update` falla mid-flow, **When**
  user tapea Continuar, **Then** ve toast error y permanece en la
  pantalla con selección preservada.

## Developer details

### Owning service

Mobile app + backend `/profile/update`.

### Dependencies

- US-011: precede en el flow.
- US-008: endpoint para persistir.
- ADR-007: lógica de selección de track inicial.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §5, §6.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §3.1 — campos `primary_goals`, `has_deadline`, `deadline_date`.
- [`decisions/ADR-007-multi-track-strategy.md`](../../decisions/ADR-007-multi-track-strategy.md)
  — `determineInitialTrack()`.

### Implementación esperada

```typescript
// app/lib/track-selection.ts
const JOB_GOALS = ['job_interview', 'remote_work',
                   'business_communication', 'exam_prep'];
const TRAVEL_GOALS = ['travel'];

export function determineInitialTrack(goals: string[]): TrackId {
  for (const goal of goals) { // priority by order
    if (JOB_GOALS.includes(goal)) return 'job_ready';
    if (TRAVEL_GOALS.includes(goal)) return 'travel_confident';
  }
  return 'daily_conversation';
}

// app/lib/deadline-calc.ts
export function deadlineFromOption(option: string): Date | null {
  const days: Record<string, number> = {
    'less_than_1m': 20,    // ~3 weeks
    '1_to_3m': 60,         // 2 months
    '3_to_6m': 135,        // 4.5 months
    'no_deadline': 0,      // null
  };
  if (option === 'no_deadline') return null;
  const date = new Date();
  date.setDate(date.getDate() + days[option]);
  return date;
}
```

### Integration points

- `/profile/update` para persistir.
- Telemetry.
- Roadmap engine (consumirá `active_track` en US-021).

### Notas técnicas

- `primary_goals` se persiste como TEXT[] preservando el orden de
  selección (importa para tie-breaking en track selection).
- `deadline_date` calculado client-side con valores sugeridos del
  rango medio.
- Cambio post-onboarding de `primary_goals` puede triggerar
  regenerar roadmap (`significant_change` flag, ver
  `student-profile-and-assessment.md` §13.2).

## Definition of Done

- [ ] Pantallas 4 y 5 implementadas.
- [ ] Multi-select con max 3 funcional.
- [ ] Track selection determinístico (test con 5+ combinaciones).
- [ ] Persistencia en BD verificada.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit de `determineInitialTrack` y `deadlineFromOption`.
- [ ] Test integration con todos los tracks possible.
- [ ] QA manual.
- [ ] Screenshots.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-008, US-011. Precede US-013.*

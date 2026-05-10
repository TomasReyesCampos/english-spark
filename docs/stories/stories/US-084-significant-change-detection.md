# US-084: significant_change detection on profile update

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3-4
**Story points:** 2
**Persona:** Admin
**Owner:** —

---

## Contexto

Cuando user updates su profile (Settings), debemos detectar si el
cambio es **significant** (justifica regenerar roadmap) o **minor**
(solo persist).

Significant changes:
- Cambio mayor de `primary_goals` (track-implication).
- Cambio de `target_english_variant` (american ↔ british).
- Cambio de `deadline_date` > 50% delta.
- Cambio de `professional_field` (afecta variantes).

Backend en
[`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
§10.6.

## Scope

### In

- Extender US-015 (`PATCH /profile/update`) con detection logic:
  - Compara new value vs current row.
  - Aplica reglas de significant change.
  - Si significant: setea response flag `significant_change:
    true`.
  - Emit `profile.updated` event con `fields_changed +
    significant_change`.
- Reglas:
  - `primary_goals`: significant si nuevo array NO comparte 1ro
    con anterior (track-changing).
  - `target_english_variant`: significant siempre.
  - `deadline_date`: significant si delta > 50% del rango.
  - `professional_field`: significant siempre.
- Telemetry: `profile.significant_change_detected`.

### Out

- Roadmap regeneration en sí (US-085).
- UI prompt (responsibility del cliente leyendo flag).

## Acceptance criteria

- **Given** user con primary_goals=['job_interview'], **When**
  PATCH a ['travel'], **Then** significant_change=true.
- **Given** primary_goals=['job_interview','travel'], **When**
  reordenan a ['travel','job_interview'], **Then**
  significant_change=true (first goal changed → track changes).
- **Given** target_english_variant='american', **When** PATCH a
  'british', **Then** significant_change=true.
- **Given** deadline_date='2026-06-01', **When** PATCH a
  '2026-06-08' (1 semana diff vs 4-week target = ~25% delta),
  **Then** significant_change=false.
- **Given** deadline_date='2026-06-01', **When** PATCH a
  '2026-09-01' (3 meses = 100%+ delta), **Then**
  significant_change=true.
- **Given** PATCH solo cambia `daily_minutes_available`, **When**
  procesa, **Then** significant_change=false (minor).
- **Given** PATCH con primary_goals reorder sin track change
  (ambos JR), **When** procesa, **Then** significant_change=false.
- **Given** significant_change=true detected, **When** completes,
  **Then** event emitido con flag, US-085 consumer triggers
  regeneration.

## Developer details

### Owning service

`apps/workers/api/handlers/profile.ts` (extiende US-015).

### Dependencies

- US-015 (base endpoint).

### Specs referenciados

- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §10.6.
- [`decisions/ADR-007-multi-track-strategy.md`](../../decisions/ADR-007-multi-track-strategy.md)
  — track derivation.

### Implementación esperada

```typescript
// apps/workers/api/handlers/profile.ts (extension)

const JOB_GOALS = ['job_interview', 'remote_work', 'business_communication', 'exam_prep'];
const TRAVEL_GOALS = ['travel'];

function deriveTrack(goals: string[]): string {
  for (const g of goals) {
    if (JOB_GOALS.includes(g)) return 'job_ready';
    if (TRAVEL_GOALS.includes(g)) return 'travel_confident';
  }
  return 'daily_conversation';
}

function detectSignificantChange(current: any, updates: any): boolean {
  if (updates.primary_goals !== undefined) {
    const oldTrack = deriveTrack(current.primary_goals ?? []);
    const newTrack = deriveTrack(updates.primary_goals);
    if (oldTrack !== newTrack) return true;
  }

  if (updates.target_english_variant !== undefined
      && updates.target_english_variant !== current.target_english_variant) {
    return true;
  }

  if (updates.professional_field !== undefined
      && updates.professional_field !== current.professional_field) {
    return true;
  }

  if (updates.deadline_date !== undefined && current.deadline_date) {
    const oldTs = new Date(current.deadline_date).getTime();
    const newTs = new Date(updates.deadline_date).getTime();
    const now = Date.now();
    const oldDelta = oldTs - now;
    const newDelta = newTs - now;
    if (Math.abs(newDelta - oldDelta) / Math.abs(oldDelta) > 0.5) return true;
  }

  return false;
}

// In handleUpdateProfile, after fetching current and parsing updates:
const significantChange = detectSignificantChange(current, parsed.data);
// ... persist update ...
await emitEvent('profile.updated', {
  user_id: userId,
  fields_changed: fields,
  significant_change: significantChange,
});

if (significantChange) {
  track('profile.significant_change_detected', { fields });
}

return jsonResponse({
  ...updated,
  significant_change: significantChange,
});
```

### Integration points

- US-015 base.
- US-085 consumer.
- Mobile app reads flag → prompt user "¿Quieres regenerar tu plan?".

### Notas técnicas

- Detection es **server-side**: no confiar en cliente.
- 50% delta threshold para deadline es heurístico — puede ajustarse
  post-data.
- Reorder de primary_goals NO trigger si track no cambia (avoid
  false positives).

## Definition of Done

- [ ] Detection logic implementada.
- [ ] Event payload extendido.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit con muchos scenarios.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-015. Bloqueante para US-085.*

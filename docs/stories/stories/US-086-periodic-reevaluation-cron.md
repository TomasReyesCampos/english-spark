# US-086: Re-evaluación periódica cron (cada 4-8 semanas)

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 4
**Story points:** 2
**Persona:** Estudiante (recibe prompt para re-evaluar)
**Owner:** —

---

## Contexto

Cada 4-8 semanas (según plan), invitar al user a hacer mini
re-evaluation. Detecta si CEFR cambió, si focus areas updates, si
roadmap necesita ajuste.

Backend en
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §14.5
+ `student-profile-and-assessment.md` §8.

## Scope

### In

- Cron Worker `periodic-reevaluation-prompt.ts`:
  - Trigger: diario 06:00 UTC.
  - Por plan, diferente frecuencia:
    - `basic`: cada 8 semanas.
    - `pro`: cada 4 semanas.
    - `premium`: cada 4 semanas.
  - Detect users con
    `assessment_completed_at < now() - {N} weeks`:
    - No paid plan with active_track set.
    - Has roadmap activo.
    - assessments_periodic_count < threshold (max 4/year).
  - Emit event `reevaluation.due` con user_id + plan_id.
- Consumer del event: notification system (US-066 patterns)
  envía push "Tiempo de re-evaluación".
- User accepts → US-080 `POST /assessment/start` con
  `reason='periodic_reassessment'`.
- Schema `assessment_sessions` campo `reason` ya existe (US-080).
- Schema field `last_periodic_reevaluation_at` en
  `student_profiles`.
- Telemetry: `reevaluation.prompt_sent`,
  `.accepted`, `.declined`.

### Out

- Mini-test específico (post-MVP — MVP reusa full assessment con
  modo abbreviated).
- UI del prompt (responsibility de notifications + mobile screen).

## Acceptance criteria

- **Given** user plan Pro con
  assessment_completed_at hace 4+ semanas, **When** cron, **Then**
  emit `reevaluation.due` event.
- **Given** user plan Basic con
  assessment hace 6 semanas, **When** cron, **Then** NO emit
  (necesita 8 semanas).
- **Given** user con
  last_periodic_reevaluation_at hace 1 semana, **When** cron,
  **Then** skip (recently re-evaluated).
- **Given** trial user (no plan), **When** cron, **Then** skip
  (re-evaluation solo para paid).
- **Given** user ya recibió 4 prompts en este año, **When** cron,
  **Then** skip (annual cap).
- **Given** event `reevaluation.due`, **When** notifications
  consume, **Then** push enviado "Tiempo de re-evaluación".
- **Given** user accepts y completa nuevo assessment, **When**
  finalize, **Then** new measured_cefr update + new roadmap +
  set `last_periodic_reevaluation_at = now()`.
- **Given** user dismisses prompt 3 veces consecutivas, **When**
  cron detecta, **Then** apply cooldown extended (2x normal
  schedule) para evitar fatigue.

## Developer details

### Owning service

`apps/workers/crons/periodic-reevaluation-prompt.ts` +
schema extension.

### Dependencies

- US-080 (consumer del 'periodic_reassessment' reason).
- US-066 patterns (notification handler).
- US-055 (plan tracking).

### Specs referenciados

- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md)
  §14.5.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §8.1.

### Schema extension

```sql
ALTER TABLE student_profiles
  ADD COLUMN last_periodic_reevaluation_at TIMESTAMPTZ,
  ADD COLUMN periodic_reevaluation_count INT NOT NULL DEFAULT 0,
  ADD COLUMN periodic_reevaluation_declined_count INT NOT NULL DEFAULT 0;
```

### Implementación esperada

```typescript
// apps/workers/crons/periodic-reevaluation-prompt.ts
const FREQ_WEEKS: Record<string, number> = {
  basic: 8, pro: 4, premium: 4,
};
const ANNUAL_CAP = 4;

export async function runPeriodicReevaluationCron(env: Env) {
  const eligible = await env.DB.query(`
    SELECT sp.user_id, sub.plan_id, sp.assessment_completed_at,
           sp.last_periodic_reevaluation_at,
           sp.periodic_reevaluation_count,
           sp.periodic_reevaluation_declined_count
    FROM student_profiles sp
    JOIN user_subscriptions sub ON sub.user_id = sp.user_id AND sub.status = 'active'
    WHERE sp.assessment_completed_at IS NOT NULL
      AND sp.periodic_reevaluation_count < $1
      AND (
        sp.last_periodic_reevaluation_at IS NULL
        OR sp.last_periodic_reevaluation_at < now() - (CASE
          WHEN sub.plan_id = 'basic' THEN interval '8 weeks'
          ELSE interval '4 weeks'
        END * (1 + sp.periodic_reevaluation_declined_count / 3))  -- extended cooldown if declined
      )
  `, [ANNUAL_CAP]);

  let promptsSent = 0;
  for (const user of eligible) {
    await emitEvent('reevaluation.due', {
      user_id: user.user_id,
      plan_id: user.plan_id,
      weeks_since_last: user.assessment_completed_at
        ? Math.floor((Date.now() - new Date(user.assessment_completed_at).getTime()) / (7 * 24 * 3600 * 1000))
        : null,
    });
    promptsSent++;
  }

  track('reevaluation.prompt_sent', { count: promptsSent });
  return { sent: promptsSent };
}
```

### Cron trigger

```toml
[triggers]
crons = ["0 6 * * *"]  # Diario 06:00 UTC
```

### Integration points

- US-080 consumer (accepting prompt starts new assessment).
- Notifications system (consume event).
- US-082 finalize updates `last_periodic_reevaluation_at`.

### Notas técnicas

- Frequency post-MVP puede personalizarse (smart timing).
- Declined cooldown extension (×1, ×2, ×3...) previene fatigue.

## Definition of Done

- [ ] Schema extension.
- [ ] Cron implementado.
- [ ] Plan-aware frequency.
- [ ] Annual cap + declined cooldown.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] PR aprobada y mergeada.

---

*Depende de US-080/055. Long-term retention driver.*

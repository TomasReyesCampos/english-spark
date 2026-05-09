# US-014: Pregunta autoevaluación + tiempo (Pantalla 7)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Pantalla 7 captura los últimos 2 inputs declarativos:
1. **Autoevaluación de speaking confidence** (1-5 escala
   visual con emojis).
2. **Tiempo diario disponible** (5/10/15/20/30/45/60 min).

Ambos son inputs críticos para el algoritmo de roadmap:
- Confidence baja triggerea bloques más fáciles primero (ver
  `pedagogical-system.md` §6 anxiety handling).
- Tiempo disponible determina cuántos bloques caben en el
  estimated_completion_weeks del roadmap.

Combiné en una story porque visualmente caben en una pantalla
(spec UI lo confirma) y son secuenciales rápidas.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §8.

## Scope

### In

- **Pregunta 1: ¿Cómo te sientes hablando inglés?** (escala 1-5)
  - 5 opciones visuales con emoji + label:
    - 😰 "Me bloqueo completamente" (1)
    - 😟 "Muy nervioso, evito hablar" (2)
    - 😐 "Nervioso pero hablo" (3)
    - 😊 "Hablo con errores pero fluyo" (4)
    - 😎 "Hablo bien pero quiero pulir" (5)
  - Single-select obligatorio.
  - Persiste en `student_profiles.speaking_confidence` (1-5).
  - También persiste `language_anxiety` derivado: 1-2 → 5
    (high anxiety), 3 → 3, 4-5 → 1-2 (low anxiety).
- **Pregunta 2: ¿Cuánto tiempo puedes dedicar al día?**
  - 6 opciones: 5, 10, 15, 20, 30, 45 minutos.
  - Single-select default 15.
  - Persiste en
    `student_profiles.daily_minutes_available`.
- Self-perceived CEFR derivado heurísticamente:
  - confidence 1-2 → A2 estimate.
  - confidence 3 → B1.
  - confidence 4 → B1+.
  - confidence 5 → B2.
  - Persiste en `self_perceived_level`.
- CTA disabled hasta ambas preguntas respondidas.
- Telemetry: `onboarding.confidence_seen`, `.confidence_selected`,
  `.time_seen`, `.time_selected`,
  `.self_perceived_cefr_assigned`.

### Out

- Frequency (días/semana): default 5, captura post-MVP.
- Preferred hour: captura en US-023 (push permission).
- Detalles de horario o reminders: en stories posteriores.

## Acceptance criteria

- **Given** user en pantalla 7, **When** se renderiza, **Then** ve
  pregunta 1 con 5 opciones emoji y pregunta 2 con 6 opciones de
  tiempo. CTA disabled.
- **Given** user selecciona confidence "😊 Hablo con errores pero
  fluyo" (4) y tiempo "20 min", **When** confirma, **Then**:
  - `speaking_confidence = 4`
  - `language_anxiety = 2` (derivado low)
  - `daily_minutes_available = 20`
  - `self_perceived_level = 'B1+'`
- **Given** user selecciona confidence "😰 Me bloqueo" (1), **When**
  confirma, **Then** `language_anxiety = 5` y `self_perceived_level
  = 'A2'`.
- **Given** user solo responde 1 pregunta, **When** intenta tapear
  CTA, **Then** botón está disabled hasta responder ambas.
- **Given** user con tiempo "5 min" seleccionado, **When**
  confirma, **Then** se persiste y `estimated_completion_weeks` del
  roadmap se ajustará multiplicado por 3x (slow track, ver spec).
- **Given** request falla, **When** confirma, **Then** ve toast
  error con selección preservada.

## Developer details

### Owning service

Mobile app + `/profile/update`.

### Dependencies

- US-008: endpoint.
- US-013: precede.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §8.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §3.1 — campos `speaking_confidence`, `language_anxiety`,
  `daily_minutes_available`, `self_perceived_level`.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §6 — anxiety handling pedagógico.

### Implementación esperada

```typescript
// app/lib/cefr-from-confidence.ts
export function selfPerceivedCefrFromConfidence(c: 1|2|3|4|5): CefrLevel {
  return c === 1 ? 'A2'
       : c === 2 ? 'A2'
       : c === 3 ? 'B1'
       : c === 4 ? 'B1+'
       : 'B2';
}

export function anxietyFromConfidence(c: 1|2|3|4|5): 1|2|3|4|5 {
  // Inverse mapping (simplified)
  return (6 - c) as 1|2|3|4|5;
}
```

### Integration points

- `/profile/update`.
- Telemetry.
- Roadmap engine (US-021 usa todos estos campos).

### Notas técnicas

- Self-perceived CEFR es **estimación gruesa** y se valida con
  mini-test (US-017 a US-019) y assessment Day 7 (post-MVP).
- Hybrid CEFR sampling de
  `student-profile-and-assessment.md` §13.2 compara
  `self_perceived_level` con `initial_test_results.cefr_estimate`
  para decidir el assessment level.

## Definition of Done

- [ ] Pantalla con 2 preguntas + UI emoji.
- [ ] Validación de ambas respondidas antes de CTA.
- [ ] Cálculos de derived fields correctos.
- [ ] Persistencia en BD.
- [ ] 6 acceptance criteria pasan.
- [ ] Tests unit de helpers (`selfPerceivedCefrFromConfidence`,
  `anxietyFromConfidence`).
- [ ] QA manual.
- [ ] Screenshots.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-008, US-013.*

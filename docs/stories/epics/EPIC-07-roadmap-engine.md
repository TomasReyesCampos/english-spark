# EPIC-07: Roadmap engine completo

**Status:** Draft
**Sprint target:** Sprint 2-4
**Owner:** —
**Estimación total:** ~45 puntos (17 stories aprox.)

---

## Resumen

El roadmap engine es **el cerebro pedagógico del producto**: decide
qué bloque servir al user en cada momento, basado en su progreso,
mastery, y track. Componentes:

1. Persistencia de roadmaps + roadmap_blocks (US-021 ya generó el
   initial; esta epic completa el lifecycle).
2. Track switching con confirmación + rate limit (per ADR-007).
3. Block progression tracking (start → in_progress → completed).
4. Test-out flow (skip bloques cuya sub-skill ya está mastered).
5. Assessment Day 7 completo (4 partes, 12 ejercicios, genera
   roadmap definitivo).
6. Significant_change detection → roadmap regeneration.
7. Re-evaluación periódica (cada 4-8 semanas).
8. Sporadic questions integration.

Este epic **bloquea EPIC-03 (Pedagogical engine)** porque sin
saber qué bloque/asset servir, el pedagogical engine no tiene
input.

---

## Por qué este epic

EPIC-01 generó el roadmap initial. EPIC-07 implementa el resto del
lifecycle:
- **Sin progression tracking:** user "termina" bloques sin que el
  sistema se entere → mastery nunca avanza.
- **Sin assessment Day 7:** trial no produce el roadmap definitivo
  → el producto no entrega su promesa core.
- **Sin track switching:** users pegados con primary_goals
  iniciales.
- **Sin regeneration:** profile changes (objetivo cambia, etc.) no
  se reflejan en roadmap.

---

## Specs autoritativos referenciados

| Spec | Cubre |
|------|-------|
| [`product/ai-roadmap-system.md`](../../product/ai-roadmap-system.md) | Sistema completo (initial + definitivo + regeneration) |
| [`product/student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md) §6 | Assessment Day 7 estructura |
| [`product/post-assessment-flow.md`](../../product/post-assessment-flow.md) | 8 pantallas reveal post-assessment |
| [`decisions/ADR-007-multi-track-strategy.md`](../../decisions/ADR-007-multi-track-strategy.md) | Track switching reglas |
| [`product/track-*-blocks.md`](../../product/track-job-ready-blocks.md) | 175 bloques con prerequisites |

---

## Stories del epic

### Slice 1: Schemas + reads (~6 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-072 | Schema roadmaps + roadmap_blocks + block_completions | 3 |
| US-073 | GET /roadmap/active + listing endpoints | 2 |
| US-074 | Block detail + asset references endpoint | 1 |

### Slice 2: Track switching (~7 pts, 2 stories)

| US | Título | Pts |
|----|--------|----:|
| US-075 | POST /tracks/switch endpoint con rate limit | 5 |
| US-076 | Track switch UI + confirmation flow | 2 |

### Slice 3: Block progression (~8 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-077 | Block start + status tracking | 2 |
| US-078 | Block completion + mastery propagation | 3 |
| US-079 | Test-out flow (skip via mini-assessment) | 3 |

### Slice 4: Assessment Day 7 (~12 pts, 4 stories)

| US | Título | Pts |
|----|--------|----:|
| US-080 | Assessment session start + orchestration 4 partes | 3 |
| US-081 | Submit assessment exercise + per-exercise scoring | 3 |
| US-082 | Finalize assessment + roadmap definitivo generation | 5 |
| US-083 | Assessment results 8-screen reveal flow | 3 |

### Slice 5: Re-evaluation + regeneration (~7 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-084 | significant_change detection on profile update | 2 |
| US-085 | Roadmap regeneration on significant_change | 3 |
| US-086 | Re-evaluación periódica cron (cada 4-8 semanas) | 2 |

### Slice 6: Sporadic questions (~5 pts, 2 stories)

| US | Título | Pts |
|----|--------|----:|
| US-087 | Sporadic questions scheduling + presentation | 3 |
| US-088 | Sporadic responses scoring + roadmap reorder | 2 |

---

## Total estimado

~45 puntos en 17 stories. Asumiendo velocity 20-25 pts/sprint:
**~2-3 sprints**.

---

## Dependencies externas

- US-021 (initial roadmap generation) ya cubierto en EPIC-01.
- AI Gateway tasks: `generate_definitive_roadmap` (US-032 covers
  pattern; podría requerir story propia o reuso del initial task).

---

## Dependencies inter-EPIC

- **EPIC-01:** US-021 inicial roadmap; US-015 profile updates.
- **EPIC-05:** US-032 (generate_initial_roadmap pattern reusable
  para definitivo).
- **EPIC-03 (futuro):** consume `block.completed` events para
  scoring + mastery propagation desde esta epic.
- **EPIC-04:** assessment Day 7 NO cobra Sparks (es free); regular
  exercises del roadmap SÍ.

---

## Definition of Done del epic

- [ ] User puede ver su roadmap activo en cualquier momento.
- [ ] User puede switch track con confirmación y rate limit.
- [ ] Block start/complete tracking funcional.
- [ ] Test-out permite skip bloques con sub-skill mastered.
- [ ] Assessment Day 7 completo (4 partes) ejecuta + genera
  roadmap definitivo.
- [ ] 8-screen reveal flow post-assessment.
- [ ] Roadmap regenera al detectar significant_change.
- [ ] Re-evaluación periódica programada según plan.
- [ ] Sporadic questions integradas en el flow Day 1-7.

---

## Riesgos identificados

| Riesgo | Mitigación |
|--------|------------|
| Roadmap regeneration mid-block | Lock current block hasta complete o cancel; regenerate solo del próximo en adelante |
| Test-out abusable (skipear todo) | Cap diario + scoring estricto del mini-assessment |
| Assessment Day 7 muy largo → drop-off | Allow pause + resume; save partial results |
| Track switch antes de terminar | Rate limit 14d + sub-skill mastery transfiere (no resets) |
| Significant_change overactive | Define umbrales estrictos para evitar regeneration constante |

---

## A/B tests post-MVP

1. Assessment duration: 4 partes vs 3 partes vs 5 partes.
2. Test-out threshold strictness.
3. Re-evaluación timing: 4 vs 6 vs 8 semanas.

---

*Documento vivo. Actualizar cuando se rebalancee la pedagogía o
se agreguen tracks.*

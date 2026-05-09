# EPIC-01: Auth + Onboarding (Sprint 0 foundational)

**Status:** Draft
**Sprint target:** Sprint 0 (semanas 1-4)
**Owner:** —
**Estimación total:** ~80 puntos (20 stories aprox.)

---

## Resumen

Establecer el flujo completo desde **descarga de la app** hasta
**primer ejercicio del producto**, cubriendo:

1. Authentication (Firebase + 4 providers).
2. Onboarding completo (16 pantallas).
3. Mini-test del Day 0 (3 ejercicios).
4. Generación del roadmap inicial provisional.
5. Setup de notifications + first exercise handoff.

Este epic es **bloqueante para todo el resto del producto** porque
sin onboarding completado el user no entra al flow normal.

---

## Por qué este epic primero

- **Sin auth, no hay user.** Todo lo demás depende de tener
  `student_profile` persistido.
- **Sin onboarding, no hay roadmap.** El roadmap inicial necesita
  data del onboarding para generarse.
- **Sin mini-test, no hay CEFR estimado.** El sistema pedagógico
  necesita un baseline para calibrar.
- **Sin first exercise success, churn Day 1 alto.** El primer
  contacto post-onboarding es crítico.

---

## Specs autoritativos referenciados

| Spec | Cubre |
|------|-------|
| [`architecture/authentication-system.md`](../../architecture/authentication-system.md) | Firebase Auth, providers, JWT validation |
| [`product/onboarding-flow.md`](../../product/onboarding-flow.md) | 16 pantallas con copy literal |
| [`product/student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md) | Schema `student_profiles`, mini-test |
| [`product/ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §5 | `generateInitialRoadmap` |
| [`architecture/notifications-system.md`](../../architecture/notifications-system.md) | Push permission setup |
| [`decisions/ADR-005-firebase-auth.md`](../../decisions/ADR-005-firebase-auth.md) | Decisión arquitectónica |
| [`decisions/ADR-008-spanish-locale-strategy.md`](../../decisions/ADR-008-spanish-locale-strategy.md) | Copy mexicano-tuteo |

---

## Stories del epic

### Slice 1: Auth foundation (~24 pts)

| US | Título | Pts |
|----|--------|----:|
| US-001 | Configurar Firebase + Auth providers | 3 |
| US-002 | Schema `users` + `student_profiles` | 3 |
| US-003 | Google Sign In flow | 5 |
| US-004 | Apple Sign In flow | 5 |
| US-005 | Email + password auth | 5 |
| US-006 | Anonymous mode (trial sin registro) | 3 |

### Slice 2: JWT + sync (~10 pts)

| US | Título | Pts |
|----|--------|----:|
| US-007 | JWT validation middleware en Workers | 3 |
| US-008 | Endpoint `/auth/sync` Firebase ↔ Postgres | 5 |
| US-009 | Sign out + account deletion | 2 |

### Slice 3: Onboarding pre-questions (~15 pts)

| US | Título | Pts |
|----|--------|----:|
| US-010 | Welcome + saludo screen (Pantalla 2) | 2 |
| US-011 | Pregunta ubicación (Pantalla 3) | 3 |
| US-012 | Pregunta objetivo + deadline (Pantallas 4-5) | 3 |
| US-013 | Pregunta contexto profesional (Pantalla 6) | 2 |
| US-014 | Pregunta autoevaluación + tiempo (Pantalla 7) | 3 |
| US-015 | Persistir respuestas en student_profile | 2 |

### Slice 4: Mic permission + mini-test (~15 pts)

| US | Título | Pts |
|----|--------|----:|
| US-016 | Mic permission flow + recovery | 3 |
| US-017 | Mini-test ejercicio 1 (repetir frase) | 3 |
| US-018 | Mini-test ejercicio 2 (responder pregunta) | 3 |
| US-019 | Mini-test ejercicio 3 (nombrar imágenes) | 3 |
| US-020 | Procesamiento screen + initial_test_results | 3 |

### Slice 5: Roadmap reveal + handoff (~10 pts)

| US | Título | Pts |
|----|--------|----:|
| US-021 | Generate initial roadmap (AI Gateway integration) | 5 |
| US-022 | Roadmap initial reveal screen | 3 |
| US-023 | Push permission flow | 2 |
| US-024 | First exercise intro screen | 2 |

### Slice 6: Cross-cutting (~5 pts)

| US | Título | Pts |
|----|--------|----:|
| US-025 | Telemetry events del onboarding (17 events) | 3 |
| US-026 | Edge cases handling (resume session, fail mic, etc.) | 5 |

---

## Total estimado

~85 puntos en 26 stories. Asumiendo velocity de 20-25 pts/sprint:
**~4 sprints (semanas 1-4)** para completar el epic.

---

## Dependencies externas a este epic

- Cuenta Firebase + Apple Developer + Google Cloud Console
  configuradas (Sprint 0 setup).
- AI Gateway con task `generate_initial_roadmap` registrada
  (US-021 depende).
- Atomics seed mínimo (al menos los 3 ejercicios del mini-test) en
  `media_atomics`.

---

## Definition of Done del epic

- [ ] User puede registrarse con cualquier provider.
- [ ] User completa onboarding 5-7 min.
- [ ] User completa mini-test 3 ejercicios.
- [ ] `student_profile` persiste con todos los campos requeridos.
- [ ] `initial_test_results` calculado y persistido.
- [ ] Roadmap inicial generado y mostrado.
- [ ] Push permission solicitado (con preferences guardadas si se
  niega).
- [ ] User llega a "primer ejercicio intro" listo para entrar al
  producto.
- [ ] Edge cases del §5.X de cada doc cubiertos.
- [ ] Telemetry events emitidos correctamente.
- [ ] QA manual end-to-end completo (anonymous + email + Google
  paths).

---

## Riesgos identificados

| Riesgo | Mitigación |
|--------|------------|
| Apple Sign In requires setup específico iOS | US-004 incluye dev environment setup explícito |
| AI Gateway no listo cuando llegue US-021 | Fallback hardcoded roadmap genérico mientras se setup |
| Atomics del mini-test no listos | Buffer en sprint planning + content team primer sprint |
| Recovery de mic permission complejo en iOS | US-016 agrega 1 pt de buffer para edge cases iOS |

---

## A/B tests post-MVP planeados

(Coordinar con `onboarding-flow.md` §19.)

1. Onboarding length: 5 vs 7 vs 3 preguntas.
2. Anonymous vs forced auth signup conversion.
3. Mini-test 3 vs 5 ejercicios.

---

*Documento vivo. Actualizar cuando se agreguen/quiten stories o
cambien estimaciones tras spike.*

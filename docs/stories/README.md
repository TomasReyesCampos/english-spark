# User Stories — english-spark

> Índice y convenciones para las user stories del producto. Cada
> story es un ticket implementable independiente con acceptance
> criteria testables.

**Última actualización:** 2026-05

---

## Estructura

```
docs/stories/
├── README.md                    ← este archivo
├── epics/                       ← agrupaciones de alto nivel
│   ├── EPIC-01-auth-onboarding.md
│   └── ...
└── stories/                     ← user stories individuales
    ├── US-001-firebase-auth-setup.md
    ├── US-002-users-profile-schema.md
    └── ...
```

---

## Personas

| Persona | Descripción |
|---------|-------------|
| **Estudiante** | Hispanohablante LatAm aprendiendo inglés. Persona principal del producto. |
| **Estudiante anonymous** | Pre-registro, modo trial sin auth fuerte. Sub-set del Estudiante. |
| **Estudiante pago** | Con Plan Básico/Pro/Premium activo. Sub-set del Estudiante. |
| **Content reviewer** | TESOL / native speaker que valida atomics y composites antes de aprobar para producción. |
| **Admin** | Personal interno operando la app (soporte, métricas, content management). |
| **API consumer** | Sistema externo integrando vía APIs públicas (post-MVP). |

---

## Convenciones

### Naming

- **Stories:** `US-XXX-<slug-kebab-case>.md`. ID secuencial 3 dígitos.
- **Epics:** `EPIC-XX-<slug-kebab-case>.md`. ID secuencial 2 dígitos.

### Story sizing (Fibonacci)

| Puntos | Guía |
|-------:|------|
| **1** | Trivial: config update, copy change, single-line fix |
| **2** | Small, well-understood: single endpoint, single component |
| **3** | Moderate: multiple files, clear requirements |
| **5** | Significant: new endpoint completo, new component con state |
| **8** | Large: multiple integration points, cross-service |
| **13** | Maximum. Si excede, split. |

**Spikes** (research, exploración): max **3 puntos**, deben producir
decisión documentada.

### Title rules

- Max **60 chars**.
- Action-oriented (verbo o sustantivo describiendo lo que se construye).
- **NO** usar formato "As a X, I want..." en el título.
- La persona va en el campo `Persona` del body, no en el título.

### Acceptance criteria

- Mínimo **2**, máximo **8** por story.
- Formato **Given / When / Then**.
- Al menos uno cubre **error/negative path**.
- Cada criterio debe ser **independientemente testable**.
- Sin detalles de implementación (eso va en Developer details).

### Definition of Done estándar

Cada story incluye DoD baseline:
- [ ] Tests unit + integration pasan.
- [ ] Validation contra spec referenciado.
- [ ] PR aprobada y mergeada.

Stories con UI agregan:
- [ ] QA manual del flow afectado.
- [ ] Screenshots o video de la UI.

Stories con cambios de schema agregan:
- [ ] Migration SQL revisada.
- [ ] Rollback documented.

---

## Epics activos

| Epic | Título | Status | Stories estimadas |
|------|--------|--------|------------------:|
| [EPIC-01](epics/EPIC-01-auth-onboarding.md) | Auth + Onboarding (Sprint 0 foundational) | Draft (26 stories ready) | 26 |
| [EPIC-04](epics/EPIC-04-sparks-system.md) | Sparks system (paralelo Sprint 1-2) | Draft (6/15 stories ready) | 15 |
| [EPIC-05](epics/EPIC-05-ai-gateway-foundation.md) | AI Gateway foundation (paralelo Sprint 0-1) | Draft (17 stories ready) | 17 |

(Más epics se agregan según se planifiquen los siguientes sprints.)

---

## Index de stories

| ID | Título | Epic | Status | Pts |
|----|--------|------|--------|----:|
| [US-001](stories/US-001-firebase-auth-setup.md) | Configurar Firebase + Auth providers | EPIC-01 | Draft | 3 |
| [US-002](stories/US-002-users-profile-schema.md) | Schema `users` + `student_profiles` | EPIC-01 | Draft | 3 |
| [US-003](stories/US-003-google-signin.md) | Google Sign In flow | EPIC-01 | Draft | 5 |
| [US-004](stories/US-004-apple-signin.md) | Apple Sign In flow | EPIC-01 | Draft | 5 |
| [US-005](stories/US-005-email-password-auth.md) | Email + password auth | EPIC-01 | Draft | 5 |
| [US-006](stories/US-006-anonymous-mode.md) | Anonymous mode (trial sin registro) | EPIC-01 | Draft | 3 |
| [US-007](stories/US-007-jwt-validation-middleware.md) | JWT validation middleware en Workers | EPIC-01 | Draft | 3 |
| [US-008](stories/US-008-auth-sync-endpoint.md) | Endpoint `/auth/sync` Firebase ↔ Postgres | EPIC-01 | Draft | 5 |
| [US-009](stories/US-009-signout-account-deletion.md) | Sign out + account deletion | EPIC-01 | Draft | 2 |
| [US-010](stories/US-010-welcome-screen.md) | Welcome + saludo screen (Pantalla 2) | EPIC-01 | Draft | 2 |
| [US-011](stories/US-011-location-question.md) | Pregunta ubicación (Pantalla 3) | EPIC-01 | Draft | 3 |
| [US-012](stories/US-012-goal-deadline-questions.md) | Pregunta objetivo + deadline (Pantallas 4-5) | EPIC-01 | Draft | 3 |
| [US-013](stories/US-013-professional-context-question.md) | Pregunta contexto profesional (Pantalla 6) | EPIC-01 | Draft | 2 |
| [US-014](stories/US-014-self-assessment-time-questions.md) | Pregunta autoevaluación + tiempo (Pantalla 7) | EPIC-01 | Draft | 3 |
| [US-015](stories/US-015-persist-onboarding-answers.md) | Persistir respuestas en student_profile | EPIC-01 | Draft | 2 |
| [US-016](stories/US-016-mic-permission-flow.md) | Mic permission flow + recovery | EPIC-01 | Draft | 3 |
| [US-017](stories/US-017-mini-test-repeat-phrase.md) | Mini-test ejercicio 1 (repetir frase) | EPIC-01 | Draft | 3 |
| [US-018](stories/US-018-mini-test-answer-question.md) | Mini-test ejercicio 2 (responder pregunta) | EPIC-01 | Draft | 3 |
| [US-019](stories/US-019-mini-test-name-images.md) | Mini-test ejercicio 3 (nombrar imágenes) | EPIC-01 | Draft | 3 |
| [US-020](stories/US-020-processing-screen-test-results.md) | Procesamiento screen + initial_test_results | EPIC-01 | Draft | 3 |
| [US-021](stories/US-021-generate-initial-roadmap.md) | Generate initial roadmap (AI Gateway) | EPIC-01 | Draft | 5 |
| [US-022](stories/US-022-roadmap-reveal-screen.md) | Roadmap initial reveal screen | EPIC-01 | Draft | 3 |
| [US-023](stories/US-023-push-permission-flow.md) | Push permission flow | EPIC-01 | Draft | 2 |
| [US-024](stories/US-024-first-exercise-intro.md) | First exercise intro screen | EPIC-01 | Draft | 2 |
| [US-025](stories/US-025-onboarding-telemetry-events.md) | Telemetry events del onboarding (17 events) | EPIC-01 | Draft | 3 |
| [US-026](stories/US-026-onboarding-edge-cases.md) | Edge cases handling del onboarding | EPIC-01 | Draft | 5 |
| [US-027](stories/US-027-ai-gateway-worker-skeleton.md) | AI Gateway Worker skeleton + routing | EPIC-05 | Draft | 3 |
| [US-028](stories/US-028-task-registry-schema.md) | Task Registry schema + persistence | EPIC-05 | Draft | 3 |
| [US-029](stories/US-029-provider-adapters.md) | Provider adapter interface + Anthropic adapter | EPIC-05 | Draft | 5 |
| [US-030](stories/US-030-multi-provider-fallback.md) | Multi-provider fallback chain | EPIC-05 | Draft | 3 |
| [US-031](stories/US-031-cost-tracking-budget.md) | Cost tracking + budget enforcement | EPIC-05 | Draft | 3 |
| [US-032](stories/US-032-task-generate-initial-roadmap.md) | Task `generate_initial_roadmap` | EPIC-05 | Draft | 5 |
| [US-033](stories/US-033-task-score-pronunciation.md) | Task `score_pronunciation` (Azure) | EPIC-05 | Draft | 3 |
| [US-034](stories/US-034-task-transcribe-audio.md) | Task `transcribe_user_audio` (Whisper) | EPIC-05 | Draft | 2 |
| [US-035](stories/US-035-task-score-fluency.md) | Task `score_fluency` | EPIC-05 | Draft | 3 |
| [US-036](stories/US-036-task-generate-notification-content.md) | Task `generate_notification_content` | EPIC-05 | Draft | 2 |
| [US-037](stories/US-037-task-generate-audio-tts.md) | Task `generate_audio_tts` (ElevenLabs) | EPIC-05 | Draft | 3 |
| [US-038](stories/US-038-task-generate-image.md) | Task `generate_image` (DALL-E 3) | EPIC-05 | Draft | 3 |
| [US-039](stories/US-039-task-validate-image-no-text.md) | Task `validate_image_no_text` (OCR) | EPIC-05 | Draft | 2 |
| [US-040](stories/US-040-task-detect-grammar-errors.md) | Task `detect_grammar_errors` | EPIC-05 | Draft | 3 |
| [US-041](stories/US-041-logging-observability-dashboard.md) | Logging + observability dashboard | EPIC-05 | Draft | 3 |
| [US-042](stories/US-042-prompt-versioning-audit.md) | Prompt versioning + audit trail | EPIC-05 | Draft | 3 |
| [US-043](stories/US-043-admin-tasks-inspect-endpoint.md) | Admin endpoint para inspeccionar tasks | EPIC-05 | Draft | 2 |
| [US-044](stories/US-044-sparks-balances-transactions-schema.md) | Schema `sparks_balances` + `sparks_transactions` | EPIC-04 | Draft | 3 |
| [US-045](stories/US-045-sparks-balance-read-endpoint.md) | Sparks balance read endpoint + cache | EPIC-04 | Draft | 2 |
| [US-046](stories/US-046-sparks-trial-award.md) | Sparks initial trial award (50 al signup) | EPIC-04 | Draft | 2 |
| [US-047](stories/US-047-sparks-charge-endpoint.md) | Sparks charge endpoint (cobro previo idempotente) | EPIC-04 | Draft | 5 |
| [US-048](stories/US-048-sparks-refund.md) | Sparks refund (en caso de fallo de operación) | EPIC-04 | Draft | 3 |
| [US-049](stories/US-049-audit-log-validation-cron.md) | Audit log inmutable validation cron | EPIC-04 | Draft | 2 |

**Total acumulado: 150 puntos en 49 stories.**
- EPIC-01: 26 stories (80 pts) — completo a nivel spec.
- EPIC-04: 6/15 stories (17 pts) — Slices 1-2 listos.
- EPIC-05: 17 stories (53 pts) — completo a nivel spec.

(Más stories se agregan a medida que se generen.)

---

## Workflow

1. **Draft:** story creada, en revisión.
2. **Ready:** acceptance criteria validados, dependencies claras,
   estimación confirmada.
3. **In Progress:** alguien la está implementando.
4. **Done:** mergeada y validada en QA.

Para mover a Ready: la story debe tener owner asignado y todas las
dependencies (otras stories) en al menos `In Progress`.

---

## Cómo crear una story nueva

1. Identificar el epic (o crear uno nuevo si no existe).
2. Asignar `US-<next-id>`.
3. Copiar template de cualquier story existente.
4. Completar contexto, scope, AC, dev details.
5. Agregar entry en este README + en el epic correspondiente.
6. Submit PR para review.

---

*Documento vivo. Actualizar el índice cuando se agreguen stories o
epics, y el workflow si se descubren patterns.*

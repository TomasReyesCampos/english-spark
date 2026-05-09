# US-013: Pregunta contexto profesional (Pantalla 6)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 2
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Pantalla 6 captura el campo profesional del user. Esto refina el
contenido pedagógico:
- Job Ready track usa esto para variantes
  (`jr_b1_industry_vocab_intro` tiene 5 variantes por
  `professional_field`).
- Roleplays de business contextualizan al field.
- Recommendation engine ajusta vocabulary intros.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §7.

## Scope

### In

- Pantalla con pregunta "¿En qué área trabajas o estudias?"
- 11 opciones single-select (de spec):
  - Tecnología / Software
  - Salud / Medicina
  - Finanzas / Banca
  - Ventas
  - Marketing
  - Educación
  - Legal
  - Creativo (diseño, arte, contenido)
  - Servicios (atención al cliente, retail)
  - Estudiante (no trabajo aún)
  - Otro
- Iconos por opción (sutil pero visualmente distinto).
- Persiste en `student_profiles.professional_field` (enum/check
  constraint).
- Skip opcional para users sin contexto laboral claro
  (`professional_field = 'other'` o null).
- Telemetry: `onboarding.professional_field_seen`,
  `.professional_field_selected`.

### Out

- Sub-categorías por field (ej: tech → web/mobile/data) — post-MVP.
- Job title específico (free text input) — post-MVP.

## Acceptance criteria

- **Given** user en pantalla 6, **When** se renderiza, **Then** ve
  los 11 botones con iconos y la pregunta.
- **Given** user selecciona "Tecnología / Software", **When** tapea
  Continuar, **Then** `student_profiles.professional_field = 'tech'`
  y navega a US-014.
- **Given** user selecciona "Estudiante (no trabajo aún)", **When**
  confirma, **Then** `professional_field = 'student'` se persiste.
- **Given** user no quiere responder, **When** tapea "Saltar
  pregunta" (link discreto), **Then** `professional_field = null`
  y navega a US-014.
- **Given** request `/profile/update` falla, **When** user
  confirma, **Then** ve toast error y permanece en pantalla con
  selección preservada.
- **Given** user de track 'job_ready' (US-012 lo asignó), **When**
  selecciona field, **Then** este valor se usará para asignar
  variantes de bloques (ej: `jr_b1_industry_vocab_intro_tech`).

## Developer details

### Owning service

Mobile app + backend `/profile/update`.

### Dependencies

- US-008: endpoint persist.
- US-012: precede en el flow.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §7.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §3.1 — campo `professional_field`.
- [`track-job-ready-blocks.md`](../../product/track-job-ready-blocks.md)
  §8.3 — variantes por `professional_field`.

### Mapping de opciones a IDs

```typescript
const PROFESSIONAL_FIELDS = [
  { id: 'tech', label: 'Tecnología / Software', icon: '💻' },
  { id: 'health', label: 'Salud / Medicina', icon: '🏥' },
  { id: 'finance', label: 'Finanzas / Banca', icon: '🏦' },
  { id: 'sales', label: 'Ventas', icon: '📈' },
  { id: 'marketing', label: 'Marketing', icon: '📣' },
  { id: 'education', label: 'Educación', icon: '🎓' },
  { id: 'legal', label: 'Legal', icon: '⚖️' },
  { id: 'creative', label: 'Creativo (diseño, arte, contenido)', icon: '🎨' },
  { id: 'service', label: 'Servicios (atención, retail)', icon: '🤝' },
  { id: 'student', label: 'Estudiante (no trabajo aún)', icon: '📚' },
  { id: 'other', label: 'Otro', icon: '✨' },
];
```

### Integration points

- `/profile/update`.
- Telemetry.
- Roadmap engine (US-021 usa esto para asignar variantes).

## Definition of Done

- [ ] Pantalla con 11 opciones + iconos.
- [ ] Skip link funcional.
- [ ] Persistencia en BD.
- [ ] 6 acceptance criteria pasan.
- [ ] Tests unit del componente.
- [ ] QA manual.
- [ ] Screenshot.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-008, US-012.*

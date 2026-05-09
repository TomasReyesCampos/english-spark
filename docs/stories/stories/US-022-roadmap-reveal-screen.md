# US-022: Roadmap initial reveal screen

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Pantalla 15 del onboarding: el "wow moment". User ve su roadmap
personalizado por primera vez. Diseñado para sentir:
- "Esto es para mí" (personal name + sus goals).
- "Tiene un plan claro" (semanas + bloques + level structure).
- "Es alcanzable" (estimación por hora diaria).
- Expectativa positiva del Day 7 ("se actualizará al hacer el
  assessment").

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §15 con
copy literal mexicano-tuteo.

## Scope

### In

- Pantalla con copy literal:
  - Hero: "✨ ¡Listo, {name}!"
  - Body: "Detecté que tu nivel está alrededor de **{cefr}** y tu
    mayor oportunidad es **{weakness}**. Armamos un plan inicial
    de **{weeks} semanas**."
  - 3 stats cards:
    - Nivel inicial: {cefr}
    - Foco principal: {weakness}
    - Duración: {weeks} semanas
  - Sección "Tu plan se ve así:" con primeros 2 niveles del
    roadmap visibles + el resto como preview borrosa (decisión
    cerrada en `student-profile-and-assessment.md` §13.4).
  - Disclaimer breve: "Tu plan se actualizará el día 7 cuando
    hagas el assessment completo."
  - CTA primary: "Empezar mi primer ejercicio"
- Datos consumidos del roadmap generado en US-021.
- Telemetry: `onboarding.roadmap_revealed`,
  `.roadmap_cta_tapped`.

### Out

- Edición / drag-and-drop del roadmap (post-MVP).
- Detalle de cada bloque (post-MVP, accesible después en home).
- Voice-over del reveal (post-MVP).

## Acceptance criteria

- **Given** user con roadmap generado (US-021 completed) y
  `name = 'María'`, `cefr_estimate = 'B1+'`,
  `weakness = 'fluidez al hablar'`,
  `estimated_completion_weeks = 4`, **When** llega a la pantalla,
  **Then** ve el hero "✨ ¡Listo, María!" y el body con esos
  valores.
- **Given** user con `name = null` o "Invitado", **When** se
  renderiza, **Then** ve "✨ ¡Listo!" sin nombre (fallback).
- **Given** roadmap con 4 niveles temáticos, **When** se muestra
  "Tu plan se ve así:", **Then** los primeros 2 niveles se
  visualizan con sus bloques (titles), niveles 3 y 4 aparecen
  borrosos / locked.
- **Given** la pantalla render, **When** completa la animation,
  **Then** `onboarding.roadmap_revealed` se emite con
  `{ roadmap_id, cefr, weeks }`.
- **Given** user tapea "Empezar mi primer ejercicio", **When**
  navega, **Then** va a US-023 (push permission) y emite
  `roadmap_cta_tapped`.
- **Given** user en proceso de generación lenta, **When** roadmap
  todavía no está listo en BD, **Then** muestra spinner con
  mensaje "Estamos terminando tu plan..." hasta que esté.
- **Given** roadmap is_active = false (caso edge raro), **When** se
  intenta cargar, **Then** muestra error "No pudimos cargar tu
  plan. Reintenta." con botón retry.

## Developer details

### Owning service

Mobile app + endpoint `GET /roadmap/active`.

### Dependencies

- US-021: roadmap generado.
- US-020: profile actualizado con cefr_estimate.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §15.
- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §10
  — endpoint contracts.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §13.4 — decisión de mostrar solo primeros 2 niveles + preview
  borrosa.

### Implementación esperada

```tsx
// app/screens/onboarding/RoadmapReveal.tsx
export function RoadmapRevealScreen() {
  const { user, profile } = useAuth();
  const { data: roadmap, isLoading } = useSWR('/roadmap/active');

  useEffect(() => {
    if (roadmap) {
      track('onboarding.roadmap_revealed', {
        roadmap_id: roadmap.id,
        cefr: roadmap.cefr_at_generation,
        weeks: roadmap.estimated_completion_weeks,
      });
    }
  }, [roadmap]);

  if (isLoading) return <LoadingScreen message="Estamos terminando tu plan..." />;
  if (!roadmap) return <ErrorRetryScreen onRetry={() => mutate('/roadmap/active')} />;

  const firstName = extractFirstName(user.display_name);
  const heroText = firstName ? `✨ ¡Listo, ${firstName}!` : '✨ ¡Listo!';
  const weakness = profile.initial_test_results?.detected_error_patterns?.[0]
    ?? 'tu fluidez al hablar';

  const visibleLevels = roadmap.levels.slice(0, 2);
  const lockedLevels = roadmap.levels.slice(2);

  return (
    <Screen>
      <Hero>{heroText}</Hero>
      <Body>
        Detecté que tu nivel está alrededor de <Bold>{roadmap.cefr_at_generation}</Bold> y
        tu mayor oportunidad es <Bold>{weakness}</Bold>. Armamos un plan
        inicial de <Bold>{roadmap.estimated_completion_weeks} semanas</Bold>.
      </Body>
      <StatsRow>
        <StatCard label="Nivel inicial" value={roadmap.cefr_at_generation} />
        <StatCard label="Foco principal" value={weakness} />
        <StatCard label="Duración" value={`${roadmap.estimated_completion_weeks} semanas`} />
      </StatsRow>

      <SectionTitle>Tu plan se ve así:</SectionTitle>
      {visibleLevels.map(level => (
        <LevelCard key={level.index} level={level} />
      ))}
      {lockedLevels.length > 0 && (
        <LockedLevelsPreview count={lockedLevels.length} />
      )}

      <Disclaimer>
        Tu plan se actualizará el día 7 cuando hagas el assessment completo.
      </Disclaimer>

      <PrimaryButton onPress={handleNext}>
        Empezar mi primer ejercicio
      </PrimaryButton>
    </Screen>
  );
}
```

### Endpoint

```typescript
// apps/workers/api/handlers/roadmap-active.ts
export async function handleGetActiveRoadmap(
  request: AuthedRequest, env: Env
) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const roadmap = await db.getActiveRoadmap(request.user!.firebase_uid);
  if (!roadmap) return jsonResponse({ error: 'no_active_roadmap' }, 404);

  return jsonResponse(roadmap);
}
```

### Integration points

- US-021 (generación).
- Telemetry.
- US-023 (push permission, siguiente).

## Definition of Done

- [ ] Pantalla con copy literal correcto.
- [ ] Niveles 1-2 visibles, 3+ borrosos.
- [ ] Polling/SWR para esperar roadmap si todavía no está.
- [ ] 7 acceptance criteria pasan.
- [ ] Tests unit del componente.
- [ ] Test integration end-to-end (mock roadmap → render).
- [ ] QA manual.
- [ ] Screenshot/video.
- [ ] Validation contra spec `onboarding-flow.md` §15.
- [ ] PR aprobada y mergeada.

---

*Depende de US-021. Precede US-023.*

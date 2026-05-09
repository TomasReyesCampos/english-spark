# US-024: First exercise intro screen

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 2
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Pantalla 16 (final) del onboarding: bridge al producto normal. User
ve qué viene (primer bloque del roadmap) y tapea "Empezar" para
entrar al flow regular.

Esta es **distinta** del welcome flow post-paywall (US separada,
después del Day 7 trial). Esta es el handoff inicial del trial Day
1.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §17.

## Scope

### In

- Pantalla con copy literal:
  - Hero: "Tu primer ejercicio te espera"
  - Body: "Vamos con: **{first_block_title}**\n\n
    {first_block_description_short}\n\n⏱️ ~{minutes} minutos"
  - 50 Sparks badge visible: "Tienes ⚡50 Sparks gratis para tu
    trial".
  - CTA primary: "Empezar"
- Datos consumidos:
  - First block del roadmap (US-021/022).
  - Sparks balance del user (50 acreditados al user.signed_up,
    fuera de scope esta story; asumir balance disponible).
- Tap "Empezar":
  - Navega al flow normal del producto: home con el primer bloque
    activo.
  - Emit evento `onboarding.completed`.
- Telemetry: `onboarding.first_exercise_intro_seen`,
  `onboarding.completed`.

### Out

- Flow del producto en sí (post-onboarding, otros epics).
- Sparks acreditación (story propia, parte de Sparks system MVP).
- Welcome flow post-paywall (US separada, post-Day 7 conversion).

## Acceptance criteria

- **Given** user con roadmap activo y first_block
  `jr_b1_intro_self_professional`, **When** llega a la pantalla,
  **Then** ve hero, descripción de ese bloque y duration estimate.
- **Given** user con `consent_audio_processing = false` (skipped
  mini-test), **When** ve la pantalla, **Then** experiencia es la
  misma (no hay flag visible al user).
- **Given** user tapea "Empezar", **When** procesa, **Then**:
  - `onboarding.completed` event emitido con todos los profile
    fields finales.
  - `student_profiles.trial_started_at` se confirma con `now()`
    (ya estaba en US-008 pero re-confirma timestamp final).
  - Navega al home del producto con el primer bloque activo.
- **Given** roadmap no tiene bloques aún (caso edge improbable),
  **When** se intenta cargar, **Then** muestra fallback: "Tu plan
  está siendo finalizado, vuelve en un momento" con retry.
- **Given** Sparks balance < 50 por algún error, **When** se
  renderiza, **Then** muestra el balance real (no 50 hardcoded) y
  log warning.
- **Given** user cierra app sin tapear "Empezar", **When** vuelve
  a abrir, **Then** retoma esta pantalla (no re-hace onboarding).

## Developer details

### Owning service

Mobile app + handler de event `onboarding.completed`.

### Dependencies

- US-021/022: roadmap.
- US-023: precede.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §17.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §5.1 — Sparks 50 trial.

### Implementación esperada

```tsx
// app/screens/onboarding/FirstExerciseIntro.tsx
export function FirstExerciseIntroScreen() {
  const { data: roadmap } = useSWR('/roadmap/active');
  const { data: sparks } = useSWR('/sparks/balance');

  useEffect(() => {
    track('onboarding.first_exercise_intro_seen');
  }, []);

  if (!roadmap || !roadmap.levels?.[0]?.blocks?.[0]) {
    return <FallbackLoadingScreen />;
  }

  const firstBlock = roadmap.levels[0].blocks[0];

  const handleStart = async () => {
    await api.post('/onboarding/complete', {});
    track('onboarding.completed');
    navigation.reset({ index: 0, routes: [{ name: 'Home' }] });
  };

  return (
    <Screen>
      <Hero>Tu primer ejercicio te espera</Hero>
      <BlockCard>
        <Title>{firstBlock.title}</Title>
        <Description>{firstBlock.description_short}</Description>
        <Duration>⏱️ ~{firstBlock.estimated_minutes} minutos</Duration>
      </BlockCard>
      <SparksBadge>
        Tienes ⚡{sparks?.current ?? 50} Sparks gratis para tu trial
      </SparksBadge>
      <PrimaryButton onPress={handleStart}>Empezar</PrimaryButton>
    </Screen>
  );
}
```

### Endpoint

```typescript
// apps/workers/api/handlers/onboarding-complete.ts
export async function handleOnboardingComplete(
  request: AuthedRequest, env: Env
) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const profile = await db.getProfile(request.user!.firebase_uid);

  await emitEvent('onboarding.completed', {
    user_id: profile.user_id,
    active_track: profile.active_track,
    cefr_estimate: profile.initial_test_results?.cefr_estimate,
    primary_goals: profile.primary_goals,
  });

  return jsonResponse({ ok: true });
}
```

### Integration points

- Roadmap endpoint.
- Sparks balance endpoint.
- Event bus (`onboarding.completed` consumed by analytics, etc.).
- Navigation reset al home.

## Definition of Done

- [ ] Pantalla con primer bloque y Sparks badge.
- [ ] Tap CTA navega correctamente.
- [ ] Event `onboarding.completed` emitido con payload completo.
- [ ] 6 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] QA manual end-to-end (signup → onboarding completo →
  home).
- [ ] Validation contra spec `onboarding-flow.md` §17.
- [ ] PR aprobada y mergeada.

---

*Depende de US-022, US-023. Cierra el onboarding flow MVP.*

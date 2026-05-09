# US-026: Edge cases handling del onboarding

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 5
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Las stories US-010 a US-024 cubren el **happy path** del onboarding.
Esta story consolida los **edge cases** identificados en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §18 que
deben funcionar antes del release:

1. User cierra app mid-onboarding → resume desde donde quedó.
2. User cambia de provider mid-flow (improbable pero posible).
3. Backend falla en algún step del flow.
4. User vuelve días después (offline o sleep prolongado).
5. Inconsistencia de state local vs server.

Sin esta story consolidada, los edge cases quedan diluidos y
algunos no se testean.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §18 (lista
exhaustiva de edge cases).

## Scope

### In

- **Resume de session:**
  - Al abrir la app, llamar `GET /profile`.
  - Determinar el último step completado:
    - sin profile → US-010 (welcome).
    - sin country → US-011.
    - sin primary_goals → US-012.
    - ... etc, mapping de campo → screen.
  - Navegar al step correcto sin re-mostrar pantallas previas.
  - Toast info: "Bienvenido de vuelta, sigamos donde quedamos."
- **Network failures retry pattern:**
  - Cualquier request a `/profile/update`,
    `/pedagogical/submit-minitest-attempt`,
    `/pedagogical/finalize-minitest`,
    `/onboarding/complete`:
    - Retry automático 1x con exponential backoff (1s).
    - Si falla 2x: toast con retry manual.
  - Implementación centralizada en wrapper de
    `api.ts`.
- **Inconsistency local vs server:**
  - Si client tiene state local pero server no tiene profile
    (raro): re-llamar `/auth/sync` para corregir.
  - Si server tiene fields que client no tiene cached: server is
    source of truth, sobrescribir local.
- **App background / foreground transitions:**
  - Si user cierra app mid-grabación de mini-test: descartar el
    audio parcial.
  - Si user cierra y vuelve después de >24h: re-emit
    `onboarding.welcome_seen` con flag `resumed: true`.
- **Multiple devices:**
  - User instala en device A, abre en device B con misma cuenta:
    sync de profile vía `/auth/sync` propaga state.
  - Onboarding incompleto en A se ve incompleto en B.
  - Nota: no se garantiza realtime sync (no websockets MVP).
- **Edge cases del consent:**
  - Si user revoca consent_audio_processing en Settings (Settings
    post-MVP, mock por ahora): mini-test results se mantienen pero
    NO se generan más audios.
- Test plan documentado en `docs/qa/onboarding-edge-cases.md` (a
  crear como parte de esta story).

### Out

- Settings UI para editar profile post-onboarding (story propia
  post-MVP).
- Multi-device realtime sync (post-MVP, requires websockets).
- Long-term offline mode (más allá de 24h, post-MVP).

## Acceptance criteria

- **Given** user cerró app después de US-011 (location), **When**
  abre la app, **Then** ve toast "Bienvenido de vuelta..." y
  navega directo a US-012.
- **Given** user cerró después de US-014 (autoeval), **When**
  abre, **Then** navega a US-016 (mic permission).
- **Given** user cerró mid-grabación US-018, **When** abre, **Then**
  navega a US-018 con grabación descartada (estado clean).
- **Given** request `/profile/update` falla 1x, **When** retry
  automático, **Then** segunda llamada se hace con backoff de 1s.
- **Given** request falla 2x, **When** display al user, **Then** ve
  toast "No pudimos guardar. Reintenta." y botón retry manual.
- **Given** user en US-021 con backend AI Gateway down, **When**
  fallback path ejecuta, **Then** roadmap genérico se persiste y
  user no se bloquea.
- **Given** user re-instala la app después de signup pero
  pre-onboarding, **When** sign in con Google de nuevo, **Then**
  `is_first_signin = false` y onboarding resume desde welcome
  (porque profile existe pero está vacío).
- **Given** user tiene state local stale (logout en otro device),
  **When** llama API, **Then** receive 401 y forced re-signin
  flow.
- **Given** user sin internet en US-021 (genera roadmap),
  **When** intenta, **Then** ve "Sin conexión. Conéctate para
  generar tu plan." con retry; profile data ya está guardado
  local.

## Developer details

### Owning service

Mobile app (state management + retry logic) +
backend (idempotency).

### Dependencies

- Todas las US-010 a US-024.
- API wrapper centralizado.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §18 —
  edge cases listados.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §11 — edge cases del profile.

### Implementación esperada

#### Resume logic

```typescript
// app/lib/onboarding-resume.ts
export function determineResumeScreen(profile: StudentProfile | null): string {
  if (!profile) return 'Welcome';
  if (!profile.country) return 'Location';
  if (!profile.primary_goals?.length) return 'Goal';
  if (!profile.has_deadline && !profile.deadline_date) return 'Deadline';
  if (!profile.professional_field && profile.professional_field !== null) return 'Professional';
  if (!profile.speaking_confidence) return 'SelfAssessment';
  if (profile.consent_audio_processing === undefined) return 'MicPermission';
  if (!profile.initial_test_results?.exercise_1) return 'MiniTest1';
  if (!profile.initial_test_results?.exercise_2) return 'MiniTest2';
  if (!profile.initial_test_results?.exercise_3) return 'MiniTest3';
  if (!profile.initial_test_results?.cefr_estimate) return 'Processing';
  // Roadmap activo? → next is push permission
  // Push answered? → next is FirstExerciseIntro
  return 'FirstExerciseIntro';
}
```

#### API retry wrapper

```typescript
// app/lib/api.ts
export async function apiCall<T>(
  path: string,
  options: RequestInit = {},
  retries: number = 1
): Promise<T> {
  let lastError: any;
  for (let i = 0; i <= retries; i++) {
    try {
      const response = await fetch(`${API_BASE}${path}`, {
        ...options,
        headers: {
          'Content-Type': 'application/json',
          ...await getAuthHeaders(),
          ...options.headers,
        },
      });
      if (response.status === 401) {
        await handleForcedSignout();
        throw new AuthError('forced_signout');
      }
      if (!response.ok) throw new ApiError(response.status);
      return await response.json();
    } catch (error) {
      lastError = error;
      if (i < retries && shouldRetry(error)) {
        await delay(Math.pow(2, i) * 1000);
        continue;
      }
      throw error;
    }
  }
  throw lastError;
}
```

### Test plan documentation

`docs/qa/onboarding-edge-cases.md` debe incluir:
- 9+ scenarios de manual testing.
- Steps para reproducir cada edge case.
- Expected behavior por scenario.
- Si automatizables: link al test integration correspondiente.

### Integration points

- Todas las pantallas del onboarding (resume logic).
- API wrapper centralizado.
- Local state management (Zustand persistence).
- Sentry para logging de edge cases.

### Notas técnicas

- Resume logic se ejecuta **una sola vez** al app start, no en cada
  navegación.
- Idempotency en backend ya cubierto en US-008/015 (ON CONFLICT).
- Local persistence: usar `expo-secure-store` para auth token,
  `AsyncStorage` para draft profile fields.

## Definition of Done

- [ ] Resume logic implementada y testeada en 9 scenarios.
- [ ] API retry wrapper centralizado.
- [ ] Forced signout flow al recibir 401.
- [ ] Test plan doc creado.
- [ ] 9 acceptance criteria pasan.
- [ ] Tests unit de `determineResumeScreen` con todos los profile
  states posibles.
- [ ] Test integration: simular crash mid-flow, verificar resume.
- [ ] QA manual con device en airplane mode mid-flow.
- [ ] Validation contra spec `onboarding-flow.md` §18.
- [ ] PR aprobada y mergeada.

---

*Cross-cutting. Cierra EPIC-01.*

# US-010: Welcome + saludo screen (Pantalla 2)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 2
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Después de signup exitoso, primer touchpoint del onboarding: una
pantalla de bienvenida cálida que reconoce al user por nombre y
explica qué viene. Es **breve** (2-3 segundos visualmente) pero
emocionalmente importante.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §3
(Pantalla 2: Welcome + saludo).

## Scope

### In

- Pantalla welcome con copy literal mexicano-tuteo (de spec §3.3):
  - Hero: "¡Hola, {name}!"
  - Body: "En 5 minutos te conozco mejor para armar tu plan.
    Después: 3 ejercicios cortos para medir tu nivel actual."
  - Tiempo estimate: "⏱️ 7-10 minutos en total"
  - CTA: "Empecemos"
- `{name}` resuelto a primer nombre del display_name. Si null o
  "Invitado": fallback a "" (omite saludo personal: "¡Hola!").
- Animación sutil de entrada (fade in del logo + texto staggered).
- Tap CTA → US-011 (Pantalla 3: Ubicación).
- Telemetry: `onboarding.welcome_seen`, `onboarding.welcome_cta_tapped`.

### Out

- Animation system completo (post-MVP, lottie integration).
- Voice intro (post-MVP).
- Pre-questions (van en US-011 a US-014).

## Acceptance criteria

- **Given** user viene de signup exitoso (cualquier provider) con
  display_name = "María García", **When** llega a la pantalla
  welcome, **Then** ve "¡Hola, María!" como hero (primer nombre
  solo).
- **Given** user anonymous con display_name = "Invitado", **When**
  ve la pantalla welcome, **Then** ve "¡Hola!" sin nombre (fallback).
- **Given** user con display_name = null (Apple sin captura), **When**
  ve la pantalla, **Then** ve "¡Hola!" sin nombre.
- **Given** user en la pantalla welcome, **When** tapea "Empecemos",
  **Then** navega a US-011 (Pantalla ubicación) y `onboarding.welcome_cta_tapped` se emite.
- **Given** user vuelve después de cerrar la app mid-onboarding,
  **When** abre la app, **Then** vuelve a la pantalla donde estaba
  (no re-muestra welcome).
- **Given** la pantalla se renderiza, **When** se completa la
  animation (~1s), **Then** `onboarding.welcome_seen` se emite.

## Developer details

### Owning service

Mobile app (React Native).

### Dependencies

- US-008: `/auth/sync` exitoso (display_name disponible en local
  state).

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §3 —
  Pantalla 2 mockup + copy literal.

### Implementación esperada

```tsx
// app/screens/onboarding/Welcome.tsx
export function WelcomeScreen() {
  const { user } = useAuth();
  const firstName = extractFirstName(user.display_name);

  useEffect(() => {
    track('onboarding.welcome_seen');
  }, []);

  const heroText = firstName ? `¡Hola, ${firstName}!` : '¡Hola!';

  return (
    <Screen>
      <FadeInView>
        <Logo />
        <Hero>{heroText}</Hero>
        <Body>
          En 5 minutos te conozco mejor para armar tu plan. Después:
          3 ejercicios cortos para medir tu nivel actual.
        </Body>
        <TimeEstimate>⏱️ 7-10 minutos en total</TimeEstimate>
      </FadeInView>
      <PrimaryButton onPress={handleNext}>Empecemos</PrimaryButton>
    </Screen>
  );
}

function extractFirstName(displayName: string | null): string | null {
  if (!displayName || displayName === 'Invitado') return null;
  return displayName.trim().split(' ')[0] ?? null;
}

function handleNext() {
  track('onboarding.welcome_cta_tapped');
  navigation.navigate('Location');
}
```

### Integration points

- Auth state (de US-003/004/005/006).
- Navigation stack del onboarding.
- Telemetry.

## Definition of Done

- [ ] Pantalla renderiza con copy literal correcto.
- [ ] 3 casos de display_name cubiertos (full name, "Invitado",
  null).
- [ ] Animación sutil implementada.
- [ ] Tap CTA navega correctamente.
- [ ] Telemetry events emitidos.
- [ ] Tests unit del componente.
- [ ] QA manual.
- [ ] Screenshot.
- [ ] Validation contra spec `onboarding-flow.md` §3.
- [ ] PR aprobada y mergeada.

---

*Depende de US-008. Bloqueante para US-011.*

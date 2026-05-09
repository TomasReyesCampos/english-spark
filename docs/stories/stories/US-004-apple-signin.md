# US-004: Apple Sign In flow

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 5
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Apple Sign In es **mandatorio en iOS** según las App Store Review
Guidelines (sección 4.8): si una app ofrece auth con Google /
Facebook / etc., debe ofrecer Apple Sign In en iOS.

UX está en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §2. Backend
en
[`authentication-system.md`](../../architecture/authentication-system.md)
§3.2.

Esta story es paralela a US-003 (Google) y comparte mucho del
backend, pero el SDK iOS es distinto + tiene caveats específicos
(privacy email, no display_name en sign-ins subsecuentes).

## Scope

### In

- Botón "Continuar con Apple" visible **solo en iOS** (por App
  Store guidelines; en Android se oculta).
- Integración `@invertase/react-native-apple-authentication`
  package.
- Flow completo: tap botón → Apple native sheet → user autoriza →
  Firebase `signInWithCredential` → `/auth/sync` → user creado /
  loggeado.
- Manejo de **privacy email** (Apple ofrece "ocultar mi email" que
  retorna un relay):
  - `users.email` se persiste tal como Apple lo retorna (relay o
    real).
  - No reject del relay email (es válido).
- Manejo de display_name solo en **first sign in** (Apple NO
  retorna display_name en sign-ins subsecuentes — debemos
  capturarlo y persistirlo en first sign in).
- Errors: cancel, no internet, JWT inválido.
- Telemetry: `auth.apple.started`, `.success`, `.cancelled`,
  `.failed`.

### Out

- Apple Sign In en Android (no soportado por Apple en non-Apple
  devices; opción "Sign in with Apple" via web es post-MVP).
- Otros providers (US-003 Google, US-005 Email, US-006 Anonymous).
- `/auth/sync` endpoint (US-008) — esta story consume.

## Acceptance criteria

- **Given** user en iOS con la app abierta sin signed in, **When**
  tapea "Continuar con Apple" en pantalla auth options, **Then**
  ve el sheet nativo de Apple con su Apple ID actual.
- **Given** user nuevo, **When** completa Apple Sign In sin elegir
  "Hide email", **Then** se crea row en `users` con email real y
  `signup_provider = 'apple'`.
- **Given** user nuevo, **When** completa Apple Sign In eligiendo
  "Hide my email", **Then** se crea row en `users` con email relay
  (`xxx@privaterelay.appleid.com`) sin error.
- **Given** user existente vuelve a sign in con Apple, **When**
  completa el flow, **Then** Apple NO retorna display_name (es
  esperado) pero el user no pierde su display_name persistido en
  BD del first sign in.
- **Given** user en Android, **When** ve la pantalla auth options,
  **Then** el botón "Continuar con Apple" NO está visible (lugar
  reservado en UI).
- **Given** user tapea "Cancelar" en el sheet de Apple, **When**
  el sheet se cierra, **Then** vuelve a auth options sin error
  visible (silent cancel).
- **Given** Apple retorna error de JWT inválido, **When** Firebase
  lo rechaza, **Then** user ve "No pudimos iniciar sesión. Intenta
  de nuevo." y puede retry.
- **Given** la app no está enrolled en Sign In with Apple
  capability, **When** se intenta el flow, **Then** error build-time
  o explícito en runtime ("Apple Sign In no configurado").

## Developer details

### Owning service

Mobile app (React Native client, iOS-specific).

### Dependencies

- US-001: Firebase + Apple provider habilitado + `.p8` key
  uploaded.
- US-002: Schema users + student_profiles.
- US-008: `/auth/sync` endpoint (puede mockearse).
- Apple Developer Program enrollment activo.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §2.5 —
  variantes auth (Apple visible solo iOS).
- [`authentication-system.md`](../../architecture/authentication-system.md)
  §3.2 — Apple provider configuration.
- [`decisions/ADR-005-firebase-auth.md`](../../decisions/ADR-005-firebase-auth.md)
  — decisión.

### Implementación esperada

```typescript
// app/auth/apple-signin.ts
import { appleAuth } from '@invertase/react-native-apple-authentication';
import auth from '@react-native-firebase/auth';
import { Platform } from 'react-native';

export async function signInWithApple() {
  if (Platform.OS !== 'ios') {
    throw new Error('Apple Sign In solo está disponible en iOS');
  }
  emit('auth.apple.started');
  try {
    const appleAuthRequestResponse = await appleAuth.performRequest({
      requestedOperation: appleAuth.Operation.LOGIN,
      requestedScopes: [appleAuth.Scope.EMAIL, appleAuth.Scope.FULL_NAME],
    });

    const { identityToken, nonce, fullName } = appleAuthRequestResponse;
    if (!identityToken) {
      throw new Error('Apple Sign In failed: no identity token');
    }

    const appleCredential = auth.AppleAuthProvider.credential(
      identityToken, nonce
    );
    const userCredential = await auth().signInWithCredential(appleCredential);

    // Capturar display_name solo en first sign in (Apple no lo retorna después)
    const displayName = fullName?.givenName
      ? `${fullName.givenName} ${fullName.familyName ?? ''}`.trim()
      : undefined;

    const firebaseToken = await userCredential.user.getIdToken();
    await api.post('/auth/sync', {
      firebase_uid: userCredential.user.uid,
      provider: 'apple',
      display_name_first_signin: displayName,  // si existe
    }, { headers: { Authorization: `Bearer ${firebaseToken}` } });

    emit('auth.apple.success');
    return userCredential.user;
  } catch (error) {
    if (error.code === appleAuth.Error.CANCELED) {
      emit('auth.apple.cancelled');
      return null;
    }
    emit('auth.apple.failed', { reason: error.code });
    throw error;
  }
}
```

### Caveats Apple-specific

1. **display_name solo en first sign in.** Apple regresa
   `fullName.givenName` y `familyName` SOLO en el primer sign in. En
   sign-ins posteriores son `null`. **Por eso el endpoint
   `/auth/sync` debe usar lógica "set if null"** (no overwrite).
2. **Privacy email relay** es válido. NO rechazar emails que
   matchean `*@privaterelay.appleid.com`.
3. **Capability iOS:** `Sign In with Apple` debe estar habilitado en
   Xcode → Capabilities. Si falta: build error.
4. **Test en simulator:** Apple Sign In NO funciona en simulator.
   Test obligatorio en device físico.

### Copy de errors (mexicano-tuteo)

| Caso | Copy |
|------|------|
| Sin internet | "Sin conexión. Verifica tu internet e intenta de nuevo." |
| Cancelación | (silent, no copy) |
| Error genérico | "No pudimos iniciar sesión con Apple. Intenta de nuevo." |
| Capability missing (build error) | (handled at build time, no runtime copy) |

### Integration points

- Mobile app iOS → Firebase Auth → backend (`/auth/sync`).
- Telemetry: PostHog.
- Settings screen post-MVP: mostrará "Iniciado con Apple".

## Definition of Done

- [ ] Botón "Continuar con Apple" funciona en iOS device físico.
- [ ] Botón NO visible en Android.
- [ ] Tests unit de `signInWithApple` (mocked).
- [ ] Test integration end-to-end con user nuevo (con email real Y
  con relay email).
- [ ] Test integration end-to-end con user existente (verifica que
  display_name no se sobreescribe en re-sign in).
- [ ] 4 errors del AC manejados.
- [ ] Telemetry events emitidos.
- [ ] QA manual en iOS device físico (no simulator).
- [ ] Screenshot/video del flow.
- [ ] Validation contra spec `onboarding-flow.md` §2.
- [ ] PR aprobada y mergeada.

---

*Depende de US-001 + US-002 + US-008 (mockable). iOS-only.*

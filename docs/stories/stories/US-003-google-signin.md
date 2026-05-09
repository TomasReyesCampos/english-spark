# US-003: Google Sign In flow

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 5
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Google Sign In es el provider preferido en Latam (Android es el OS
mayoritario). Esta story implementa el **flow completo** desde
botón "Continuar con Google" en la pantalla de auth options
(onboarding §2) hasta tener un Firebase user válido y un
`student_profile` creado.

UX está completamente especificada en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §2 (Auth
options screen). Backend de auth en
[`authentication-system.md`](../../architecture/authentication-system.md)
§3.1.

## Scope

### In

- Botón "Continuar con Google" en pantalla de auth options
  (onboarding §2.2 mockup).
- Integración `@react-native-google-signin/google-signin` package.
- Flow completo: tap botón → Google native picker → user selecciona
  cuenta → Firebase `signInWithCredential` → backend recibe JWT →
  user creado / loggeado.
- Llamada a `POST /auth/sync` (US-008) post-success para crear /
  actualizar user en Postgres.
- Manejo de errors: cancelación user, no internet, account picker
  cerrado, JWT inválido.
- Loading state visible durante el flow (spinner sin bloqueo total
  de UI).
- Telemetry events: `auth.google.started`, `auth.google.success`,
  `auth.google.cancelled`, `auth.google.failed`.

### Out

- Apple Sign In (US-004).
- Email/password auth (US-005).
- Anonymous mode (US-006).
- `POST /auth/sync` endpoint en sí (US-008) — esta story consume
  el endpoint asumiendo que existe.
- JWT validation en Workers (US-007).

## Acceptance criteria

- **Given** user en pantalla de auth options sin signed in,
  **When** tapea "Continuar con Google" y selecciona una cuenta
  válida, **Then** ve el welcome screen (onboarding §3) con su
  display_name visible.
- **Given** user es nuevo, **When** completa Google Sign In,
  **Then** se crea un row en `users` con `signup_provider =
  'google'` y un row en `student_profiles`.
- **Given** user existente que firma de nuevo, **When** completa
  Google Sign In, **Then** se actualiza `users.last_signin_at` y
  user va directo al home (skip onboarding si `student_profile`
  completed).
- **Given** user en mid-flow Google, **When** tapea "atrás" en el
  account picker, **Then** vuelve a la pantalla auth options sin
  error visible (silent cancel).
- **Given** user sin internet, **When** tapea "Continuar con
  Google", **Then** ve mensaje "Sin conexión. Verifica tu internet
  e intenta de nuevo." y botón retry.
- **Given** Google retorna un id_token inválido o expirado, **When**
  Firebase rechaza el credential, **Then** user ve error genérico
  "No pudimos iniciar sesión. Intenta de nuevo." y se puede
  retry.
- **Given** un user crea cuenta con Google email X, **When** vuelve
  a sign in con el mismo email, **Then** Firebase reconoce el user
  existente y NO duplica en BD.
- **Given** un user firma con Google sobre un device donde había
  estado anonymous, **When** completa el flow, **Then** se ofrece
  "linkear" la cuenta anonymous al Google account preserving
  trial Sparks (lógica detallada en US-006 + US-008).

## Developer details

### Owning service

Mobile app (React Native client).

### Dependencies

- US-001: Firebase + Google provider habilitado.
- US-002: Schema users + student_profiles.
- US-008: `/auth/sync` endpoint (puede mockearse mientras US-008
  se implementa en paralelo).

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §2 —
  Auth options screen mockup + copy literal.
- [`authentication-system.md`](../../architecture/authentication-system.md)
  §3.1 — Google provider configuration.
- [`authentication-system.md`](../../architecture/authentication-system.md)
  §6 — flow de auth + sync con backend.
- [`decisions/ADR-005-firebase-auth.md`](../../decisions/ADR-005-firebase-auth.md)
  — decisión arquitectónica.

### Implementación esperada

```typescript
// app/auth/google-signin.ts
import { GoogleSignin } from '@react-native-google-signin/google-signin';
import auth from '@react-native-firebase/auth';

GoogleSignin.configure({
  webClientId: Config.GOOGLE_WEB_CLIENT_ID,  // de Firebase Console
});

export async function signInWithGoogle() {
  emit('auth.google.started');
  try {
    await GoogleSignin.hasPlayServices();
    const { idToken } = await GoogleSignin.signIn();
    const googleCredential = auth.GoogleAuthProvider.credential(idToken);
    const userCredential = await auth().signInWithCredential(googleCredential);
    const firebaseToken = await userCredential.user.getIdToken();

    await api.post('/auth/sync', {
      firebase_uid: userCredential.user.uid,
      provider: 'google',
    }, { headers: { Authorization: `Bearer ${firebaseToken}` } });

    emit('auth.google.success');
    return userCredential.user;
  } catch (error) {
    if (error.code === statusCodes.SIGN_IN_CANCELLED) {
      emit('auth.google.cancelled');
      return null;
    }
    emit('auth.google.failed', { reason: error.code });
    throw error;
  }
}
```

### Copy de errors (mexicano-tuteo)

| Caso | Copy |
|------|------|
| Sin internet | "Sin conexión. Verifica tu internet e intenta de nuevo." |
| Cancelación | (silent, no copy) |
| Error genérico | "No pudimos iniciar sesión. Intenta de nuevo." |
| Account picker no abre (Play Services missing) | "Necesitas tener Google Play Services actualizado para usar esta opción." |

### Integration points

- Mobile app → Firebase Auth → backend (`/auth/sync`).
- Telemetry: PostHog events.
- Settings screen (post-MVP): mostrará el provider con el que se
  registró.

### Notas técnicas

- `webClientId` viene del OAuth client AUTO de Firebase (no del
  Android client). Confusing en docs pero correcto.
- En iOS, NO necesita configuración extra del Sign In package
  (Firebase lo maneja).
- En Android, `SHA-1` debe estar en Firebase project settings o el
  flow falla silenciosamente.
- Test en emulator: usar cuenta Google de pruebas, no la personal.

## Definition of Done

- [ ] Botón "Continuar con Google" funciona en iOS y Android.
- [ ] Tests unit de la función `signInWithGoogle` (mocked Firebase).
- [ ] Test integration: flow end-to-end con user nuevo + user
  existente.
- [ ] Errors handling cubre los 4 casos del AC.
- [ ] Telemetry events emitidos correctamente.
- [ ] QA manual del flow en device real (iOS + Android).
- [ ] Screenshot/video del flow agregado al PR.
- [ ] Validation contra spec `onboarding-flow.md` §2.
- [ ] PR aprobada y mergeada.

---

*Depende de US-001 + US-002 + US-008 (mockable).*

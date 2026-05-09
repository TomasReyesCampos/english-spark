# US-005: Email + password auth

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 5
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Email + password es el provider **fallback universal** para
usuarios sin Google account o que prefieren no usar federated
auth. Aunque no es el preferido (Google + Apple cubren la mayoría),
es necesario:

- Users en países sin Google penetrant (raro en Latam, pero
  posible).
- Users corporativos con políticas estrictas.
- Reducir fricción ("dame opción de email").

Esta story implementa **3 flows** dentro de un mismo provider:
sign up, sign in, password reset.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §2.
Backend en
[`authentication-system.md`](../../architecture/authentication-system.md)
§3.3.

## Scope

### In

- Botón "Continuar con email" en pantalla auth options.
- Pantalla de **sign up email + password**:
  - Campo email (validation client-side regex + Firebase server-side).
  - Campo password (min 8 chars, indicador de fuerza).
  - Pantalla de email verification (Firebase envía link).
  - User no puede continuar onboarding hasta verificar email.
- Pantalla de **sign in** (existing user):
  - Campos email + password.
  - Link "¿Olvidaste tu password?" → flow de reset.
- Flow de **password reset**:
  - Pantalla con campo email.
  - Confirmación "Te enviamos un link a [email]".
  - User abre link en email → Firebase UI default para resetear.
- Errors comunes: email ya en uso, password débil, password
  incorrecto, user no encontrado, email no verificado.
- Telemetry: `auth.email.signup_started`, `.signup_completed`,
  `.signin_started`, `.signin_completed`, `.password_reset_requested`.

### Out

- Magic link login (sin password) — post-MVP, decisión cerrada en
  `pendientes.md`.
- MFA — post-MVP.
- Cambio de email (Settings) — story propia post-MVP.
- Cambio de password (Settings) — story propia post-MVP.

## Acceptance criteria

- **Given** user en pantalla auth options sin signed in, **When**
  tapea "Continuar con email" y completa email + password válidos
  para sign up, **Then** ve pantalla "Te enviamos un link a [email]
  para verificar tu cuenta" y un row se crea en `users` con
  `signup_provider = 'email'` (estado pre-verificación marcado en
  Firebase).
- **Given** user no verificó email todavía, **When** intenta
  continuar al onboarding, **Then** ve banner "Verifica tu email
  para continuar" con botón "Reenviar email".
- **Given** user nuevo intenta sign up con email ya en uso, **When**
  Firebase rechaza, **Then** ve "Este email ya tiene cuenta.
  ¿Quieres iniciar sesión?" con botón "Iniciar sesión".
- **Given** user con cuenta verificada, **When** completa sign in
  email + password correctos, **Then** entra a la app (al
  onboarding o home según su estado).
- **Given** user intenta sign in con password incorrecto, **When**
  Firebase rechaza, **Then** ve "Email o password incorrecto.
  Intenta de nuevo." sin diferenciar cuál falló (security best
  practice).
- **Given** user tapea "¿Olvidaste tu password?", **When** ingresa
  email válido, **Then** ve "Te enviamos un link a [email] para
  resetear tu password" y Firebase envía el email.
- **Given** user ingresa password débil (< 8 chars o solo dígitos),
  **When** intenta sign up, **Then** ve "Tu password debe tener al
  menos 8 caracteres con letras y números" y NO se crea cuenta.
- **Given** user en pantalla de sign in que tapea botón "atrás" en
  device, **When** sale, **Then** vuelve a auth options sin error.

## Developer details

### Owning service

Mobile app (React Native).

### Dependencies

- US-001: Firebase + Email provider habilitado.
- US-002: Schema users + student_profiles.
- US-008: `/auth/sync` endpoint.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §2.3 —
  Auth options copy.
- [`authentication-system.md`](../../architecture/authentication-system.md)
  §3.3 — Email/Password provider.

### Implementación esperada

```typescript
// app/auth/email-signin.ts
import auth from '@react-native-firebase/auth';

export async function signUpWithEmail(email: string, password: string) {
  emit('auth.email.signup_started');
  try {
    const userCredential = await auth()
      .createUserWithEmailAndPassword(email, password);
    await userCredential.user.sendEmailVerification();
    // NO sync to backend yet — wait until email verified
    emit('auth.email.signup_completed');
    return userCredential.user;
  } catch (error) {
    handleFirebaseAuthError(error);
    throw error;
  }
}

export async function signInWithEmail(email: string, password: string) {
  emit('auth.email.signin_started');
  try {
    const userCredential = await auth()
      .signInWithEmailAndPassword(email, password);
    if (!userCredential.user.emailVerified) {
      throw new Error('email_not_verified');
    }
    const firebaseToken = await userCredential.user.getIdToken();
    await api.post('/auth/sync', {
      firebase_uid: userCredential.user.uid,
      provider: 'email',
    }, { headers: { Authorization: `Bearer ${firebaseToken}` } });
    emit('auth.email.signin_completed');
    return userCredential.user;
  } catch (error) {
    handleFirebaseAuthError(error);
    throw error;
  }
}

export async function resetPassword(email: string) {
  emit('auth.email.password_reset_requested');
  await auth().sendPasswordResetEmail(email);
}
```

### Mapping de errors Firebase

| Firebase code | Copy mexicano-tuteo |
|---------------|---------------------|
| `auth/email-already-in-use` | "Este email ya tiene cuenta. ¿Quieres iniciar sesión?" |
| `auth/invalid-email` | "El email no parece válido. Revisa que esté bien escrito." |
| `auth/weak-password` | "Tu password debe tener al menos 8 caracteres con letras y números." |
| `auth/user-not-found` | "Email o password incorrecto. Intenta de nuevo." |
| `auth/wrong-password` | "Email o password incorrecto. Intenta de nuevo." |
| `auth/too-many-requests` | "Muchos intentos. Espera unos minutos antes de intentar de nuevo." |
| `email_not_verified` (custom) | "Verifica tu email antes de continuar. Te enviamos un link." |
| Otros | "No pudimos completar la acción. Intenta de nuevo." |

### Validación de password (client-side)

- Min 8 caracteres.
- Al menos 1 letra y 1 número (regex
  `/^(?=.*[A-Za-z])(?=.*\d).{8,}$/`).
- Mostrar indicador de fuerza visual: débil / aceptable / fuerte.

### Integration points

- Firebase Auth (createUserWithEmailAndPassword, signInWithEmailAndPassword,
  sendEmailVerification, sendPasswordResetEmail).
- Backend `/auth/sync` (US-008).
- Email transaccional: enviado por Firebase (no controlamos
  template; es default de Firebase). Post-MVP: customizar template
  via Firebase Email Templates.

### Notas técnicas

- Email verification es **obligatorio**: user no puede entrar al
  onboarding sin verificar.
- Firebase email default está en inglés — para MVP es aceptable.
  Customización del template (a español + branded) post-MVP.
- Re-send verification email: máximo 1 cada 60s (rate limit
  Firebase).
- Password reset email también default Firebase, mismo plan
  post-MVP.

## Definition of Done

- [ ] 3 flows funcionan: sign up, sign in, password reset.
- [ ] Email verification gate antes de continuar onboarding.
- [ ] 7 errors del AC mapeados a copy mexicano-tuteo.
- [ ] Tests unit de las 3 funciones (mocked Firebase).
- [ ] Tests integration end-to-end de los 3 flows.
- [ ] QA manual: sign up con email gmail real, verificar email,
  sign in.
- [ ] QA manual: password reset → recibir email → resetear → sign
  in.
- [ ] Screenshots/video.
- [ ] Validation contra spec `onboarding-flow.md` §2.
- [ ] PR aprobada y mergeada.

---

*Depende de US-001 + US-002 + US-008.*

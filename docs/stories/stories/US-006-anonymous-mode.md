# US-006: Anonymous mode (trial sin registro)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante anonymous
**Owner:** —

---

## Contexto

**Reducir fricción de signup** es crítico para conversion del
funnel. Anonymous mode permite al user **probar el producto sin
crear cuenta**. Más adelante (al primer signal de commitment:
completar Day 1, query del assessment, etc.) ofrecemos upgrade a
cuenta real.

Esta story implementa el path "Empezar sin registrarme" del
onboarding §2.5 (variantes auth).

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §2.5
(variante anonymous). Backend en
[`authentication-system.md`](../../architecture/authentication-system.md)
§3.4.

## Scope

### In

- Botón terciario "Empezar sin registrarme" en pantalla auth
  options (visible debajo de los botones primarios).
- Flow: tap → Firebase `signInAnonymously()` → user UID generado →
  `/auth/sync` con `provider = 'anonymous'` → onboarding normal.
- `users.is_anonymous = true` y `users.email = null` (anonymous
  no tiene email).
- `users.display_name = 'Invitado'` por default (editable en
  Settings post-MVP, fuera de scope esta story).
- Anonymous user puede:
  - Completar onboarding completo.
  - Hacer mini-test.
  - Recibir roadmap inicial.
  - Practicar bloques.
  - Ganar Sparks del trial.
- Anonymous user NO puede:
  - Acceder desde otro device (no hay sync entre devices).
  - Recuperar cuenta si desinstala app.
- **Upgrade prompt** triggered en momentos clave:
  - Al completar Day 1 ("¿Quieres guardar tu progreso?").
  - Al pedir el assessment (Day 5+).
  - Al fin del trial (Day 7).
- Telemetry: `auth.anonymous.started`, `.success`,
  `.upgrade_prompted`, `.upgrade_completed`.

### Out

- **Linkear** anonymous user a Google/Apple/Email account preserving
  data — eso es US-008 (account linking lógica de backend).
- Settings screen para editar display_name — post-MVP.
- Multi-device sync para anonymous — explícitamente NO soportado
  (limitación documented al user).

## Acceptance criteria

- **Given** user en pantalla auth options, **When** tapea "Empezar
  sin registrarme", **Then** ve un disclaimer breve "Tu progreso
  vive solo en este teléfono. Te recomendamos crear cuenta cuando
  quieras." y un botón "Continuar".
- **Given** user confirma anonymous, **When** Firebase crea user
  anonymous, **Then** se persiste row en `users` con
  `is_anonymous = true`, `email = null`, `display_name = 'Invitado'`
  y `student_profile` se crea automáticamente.
- **Given** anonymous user mid-onboarding, **When** desinstala app y
  vuelve a instalar, **Then** queda con cuenta nueva (anterior se
  pierde — limitación esperada).
- **Given** anonymous user que completó Day 1, **When** abre la app
  Day 2, **Then** ve modal "¿Quieres guardar tu progreso? Crea cuenta
  para no perderlo." con CTA "Sí, crear cuenta" y secondary "Después".
- **Given** anonymous user en Day 7, **When** ve la pantalla del
  assessment, **Then** ve modal "Para hacer el assessment necesitas
  cuenta. Crea una en 30s." con upgrade flow obligatorio.
- **Given** anonymous user tapea "Sí, crear cuenta", **When**
  selecciona Google/Apple/Email, **Then** se hace **link** del
  Firebase user anonymous al provider elegido (preservando UID y
  todo el progreso).
- **Given** anonymous user que tapea "Después" 3 veces consecutivas,
  **When** próximo upgrade prompt, **Then** se aplica back-off (no
  prompt durante 24h).

## Developer details

### Owning service

Mobile app + backend (`/auth/sync` para anonymous).

### Dependencies

- US-001: Firebase + Anonymous provider habilitado.
- US-002: Schema users con `is_anonymous` column.
- US-008: `/auth/sync` y account linking logic.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §2.5 —
  variante anonymous.
- [`authentication-system.md`](../../architecture/authentication-system.md)
  §3.4 — Anonymous provider.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §6 — assessment requires real account (Day 7+).

### Implementación esperada

```typescript
// app/auth/anonymous-signin.ts
import auth from '@react-native-firebase/auth';

export async function signInAnonymously() {
  emit('auth.anonymous.started');
  try {
    const userCredential = await auth().signInAnonymously();
    const firebaseToken = await userCredential.user.getIdToken();
    await api.post('/auth/sync', {
      firebase_uid: userCredential.user.uid,
      provider: 'anonymous',
    }, { headers: { Authorization: `Bearer ${firebaseToken}` } });
    emit('auth.anonymous.success');
    return userCredential.user;
  } catch (error) {
    emit('auth.anonymous.failed', { reason: error.code });
    throw error;
  }
}

// Upgrade flow: link anonymous to permanent provider
export async function linkAnonymousToProvider(
  provider: 'google' | 'apple' | 'email',
  credentials: any
) {
  const currentUser = auth().currentUser;
  if (!currentUser?.isAnonymous) {
    throw new Error('Not an anonymous user');
  }

  emit('auth.anonymous.upgrade_started', { provider });

  let credential;
  if (provider === 'google') {
    credential = auth.GoogleAuthProvider.credential(credentials.idToken);
  } else if (provider === 'apple') {
    credential = auth.AppleAuthProvider.credential(
      credentials.identityToken, credentials.nonce
    );
  } else if (provider === 'email') {
    credential = auth.EmailAuthProvider.credential(
      credentials.email, credentials.password
    );
  }

  const linkedCredential = await currentUser.linkWithCredential(credential);
  // UID stays the same — all data preserved!
  await api.post('/auth/upgrade-anonymous', {
    firebase_uid: linkedCredential.user.uid,
    new_provider: provider,
  });
  emit('auth.anonymous.upgrade_completed', { provider });
  return linkedCredential.user;
}
```

### Disclaimer copy (mexicano-tuteo)

Pre-anonymous confirmation:

> "Empezar sin registrarte"
>
> Tu progreso vive solo en este teléfono. Si lo pierdes o cambias de
> dispositivo, no podemos recuperarlo.
>
> Te recomendamos crear cuenta cuando quieras (es rápido, podés
> elegir Google, Apple o email).
>
> [Continuar sin cuenta]    [Mejor crear cuenta]

### Upgrade prompt copy

| Momento | Copy |
|---------|------|
| Completed Day 1 | "¡Felicidades por tu primer día!  ¿Quieres guardar tu progreso? Te toma 30 segundos crear cuenta." |
| Sparks runout pre-Day 5 | "Estás avanzando rápido. Crea cuenta para no perder tu progreso." |
| Day 7 / Assessment | "Para hacer tu assessment necesitas cuenta. Crea una en 30s y desbloquea tu plan completo." |

### Back-off de upgrade prompts

```typescript
const UPGRADE_PROMPT_BACKOFF = {
  after_dismiss_count_1: 24 * 60 * 60 * 1000,    // 1 día
  after_dismiss_count_2: 3 * 24 * 60 * 60 * 1000, // 3 días
  after_dismiss_count_3: 7 * 24 * 60 * 60 * 1000, // 7 días, después no más prompt soft
};
```

Excepción: Day 7 / Assessment es **obligatorio** sin back-off.

### Integration points

- Firebase Auth (signInAnonymously, linkWithCredential).
- Backend `/auth/sync` y `/auth/upgrade-anonymous` (US-008).
- Telemetry events.
- Modal system del onboarding (compartido con otros prompts).

### Notas técnicas

- Anonymous Firebase users tienen UID válido como cualquier user.
  La diferencia es solo `provider = 'anonymous'`.
- Cuando se hace `linkWithCredential`, **el UID se preserva** y
  todo el data en Postgres (que está keyed por UID) automáticamente
  queda asociado al provider permanente. **NO hay migración de
  data.**
- Apple Sign In de un anonymous user requiere pedir email +
  fullName explícitamente (mismo caveat que US-004).

## Definition of Done

- [ ] Botón "Empezar sin registrarme" funciona iOS + Android.
- [ ] Disclaimer pre-confirm visible.
- [ ] Anonymous user puede completar onboarding 100%.
- [ ] Upgrade prompt triggered correctamente en los 3 momentos clave.
- [ ] Account linking preserva data (UID se mantiene).
- [ ] Back-off de prompts funciona.
- [ ] Tests unit de `signInAnonymously` + `linkAnonymousToProvider`.
- [ ] Test integration end-to-end: anonymous → onboarding → upgrade.
- [ ] QA manual del flow + 3 upgrade prompts.
- [ ] Screenshots/video.
- [ ] Validation contra spec `onboarding-flow.md` §2.5.
- [ ] PR aprobada y mergeada.

---

*Depende de US-001 + US-002 + US-008.*

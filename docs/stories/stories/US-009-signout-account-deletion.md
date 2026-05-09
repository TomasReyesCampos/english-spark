# US-009: Sign out + account deletion

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 2
**Persona:** Estudiante
**Owner:** —

---

## Contexto

User debe poder cerrar sesión y eliminar su cuenta desde la app.
Aunque MVP no expone Settings completos, el **delete account** es
**legalmente obligatorio** (GDPR, LFPDPPP, App Store
guidelines 5.1.1(v)).

UX simple desde Settings (Settings completo es post-MVP, pero la
opción "Cerrar sesión" + "Eliminar cuenta" debe existir desde MVP
1.0).

Backend en
[`authentication-system.md`](../../architecture/authentication-system.md)
§7. Compliance en
[`legal-compliance.md`](../../business/legal-compliance.md).

## Scope

### In

- Pantalla mínima de Settings con 2 opciones:
  - "Cerrar sesión" (destructiva-light, secondary).
  - "Eliminar cuenta" (destructive, requires confirmation).
- Sign out flow:
  - Tap "Cerrar sesión" → confirm modal → Firebase signOut() →
    clear local state → vuelve a pantalla auth options.
  - Anonymous users: warning "Vas a perder todo tu progreso porque
    no tienes cuenta. ¿Estás seguro?".
- Delete account flow:
  - Tap "Eliminar cuenta" → modal con disclaimer largo:
    "Esto borra TU PROGRESO, TUS Sparks y TU CUENTA. No se puede
    recuperar. ¿Estás seguro?".
  - Doble confirmation: typear "ELIMINAR" para activar botón.
  - DELETE `/auth/account` (US-008) → soft-delete.
  - Firebase user también deleted (con `auth.currentUser.delete()`).
  - Clear local state.
  - Show "Tu cuenta fue eliminada. Hasta pronto."
  - Vuelve a pantalla auth options.
- Telemetry: `auth.signout`, `auth.account_deletion_requested`,
  `auth.account_deletion_completed`.

### Out

- Settings completo con preferencias, notifications, etc. — story
  propia post-MVP.
- Recovery de cuenta deleted dentro de 30 días — post-MVP.
- Cron de hard-delete (cross-cutting cleanup) — story propia.
- Export de datos pre-deletion (GDPR data portability) — post-MVP.

## Acceptance criteria

- **Given** user logged in (cualquier provider), **When** entra a
  Settings y tapea "Cerrar sesión" y confirma, **Then** Firebase
  signOut() ejecuta, local state se limpia y user vuelve a auth
  options.
- **Given** user anonymous, **When** tapea "Cerrar sesión", **Then**
  ve warning "Vas a perder TODO tu progreso. ¿Estás seguro?" y
  doble confirmación.
- **Given** user logged in, **When** tapea "Eliminar cuenta",
  **Then** ve modal con disclaimer y campo de texto para typear
  "ELIMINAR".
- **Given** user typea "ELIMINAR" exacto, **When** tapea "Eliminar
  mi cuenta", **Then** se ejecuta DELETE `/auth/account` + Firebase
  delete user + local clear.
- **Given** user typea cualquier otra cosa que no sea "ELIMINAR",
  **When** intenta tapear el botón, **Then** botón está disabled.
- **Given** delete account success, **When** user vuelve a abrir la
  app, **Then** ve auth options (no logged in, su cuenta vieja no
  retorna).
- **Given** un user que se equivoca y quiere recuperar (dentro de
  30 días), **When** intenta sign in con mismo email, **Then** ve
  "Esta cuenta fue eliminada. Crea una nueva si quieres volver."
  (recovery NO es MVP — post-MVP).
- **Given** error de red durante delete, **When** request falla,
  **Then** user ve "No pudimos eliminar tu cuenta. Intenta de
  nuevo." y reintento permitido.

## Developer details

### Owning service

Mobile app + `apps/workers/api`.

### Dependencies

- US-001: Firebase Auth.
- US-007: JWT middleware.
- US-008: DELETE `/auth/account` endpoint.

### Specs referenciados

- [`authentication-system.md`](../../architecture/authentication-system.md)
  §7 — sign out + delete flows.
- [`legal-compliance.md`](../../business/legal-compliance.md) §6 —
  data deletion compliance.
- [`reglas.md`](../../reglas.md) — soft-delete con 30 días retention.

### Implementación esperada

```typescript
// app/auth/signout.ts
import auth from '@react-native-firebase/auth';

export async function signOut() {
  emit('auth.signout');
  await auth().signOut();
  await clearLocalState();
  navigation.reset({ index: 0, routes: [{ name: 'AuthOptions' }] });
}

export async function deleteAccount() {
  emit('auth.account_deletion_requested');

  const firebaseToken = await auth().currentUser!.getIdToken();
  await api.delete('/auth/account', {
    headers: { Authorization: `Bearer ${firebaseToken}` },
  });

  // Firebase delete (después de backend success)
  await auth().currentUser!.delete();
  await clearLocalState();

  emit('auth.account_deletion_completed');
  navigation.reset({ index: 0, routes: [{ name: 'AuthOptions' }] });
}
```

### Modal de delete (mexicano-tuteo)

```
┌─────────────────────────────────────┐
│  Eliminar tu cuenta                 │
│                                     │
│  Esto borra para siempre:           │
│  • Tu progreso de aprendizaje       │
│  • Tus {sparks} Sparks              │
│  • Tu plan personalizado            │
│  • Toda tu historia                 │
│                                     │
│  No se puede recuperar.             │
│                                     │
│  Para confirmar, escribí:           │
│  ┌─────────────────────────────┐    │
│  │ ELIMINAR                    │    │
│  └─────────────────────────────┘    │
│                                     │
│  [Cancelar]   [Eliminar mi cuenta]  │
│              (disabled hasta typear  │
│               "ELIMINAR" exacto)     │
└─────────────────────────────────────┘
```

### Integration points

- Firebase Auth (`signOut`, `currentUser.delete()`).
- Backend `/auth/account` (US-008).
- Local storage / state management (Zustand).
- Navigation (resetear stack a AuthOptions).
- Telemetry.

### Notas técnicas

- **Orden importa:** primero DELETE en backend (soft-delete), DESPUÉS
  Firebase delete. Si Firebase falla pero backend OK, user queda
  soft-deleted (correcto). Si backend falla, no eliminamos
  Firebase (correcto, retry posible).
- Anonymous user delete: mismo flow pero con warning extra "Tu
  progreso no es recuperable" (es lo mismo que cualquier delete
  pero refuerza expectativas).
- Firebase requiere recent sign-in para `delete()`. Si user no
  signed in recientemente: re-authentication prompt (Firebase
  estándar).

## Definition of Done

- [ ] Pantalla Settings minimal con sign out + delete cuenta.
- [ ] Sign out flow funciona con confirm.
- [ ] Delete account flow funciona con doble confirm.
- [ ] Anonymous warning extra implementado.
- [ ] 8 acceptance criteria cumplidos.
- [ ] Tests unit de las funciones.
- [ ] Test integration end-to-end (sign out + sign in de nuevo;
  delete + intent sign in fail).
- [ ] QA manual del flow.
- [ ] Screenshots/video.
- [ ] Validation contra spec `authentication-system.md` §7.
- [ ] PR aprobada y mergeada.

---

*Depende de US-001, US-007, US-008.*

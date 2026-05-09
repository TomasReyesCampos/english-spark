# US-008: Endpoint `/auth/sync` Firebase ↔ Postgres

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 5
**Persona:** Admin (consumido por Estudiante)
**Owner:** —

---

## Contexto

Firebase Auth solo conoce identidad (UID, email, provider). El
producto necesita un mirror del user en Postgres con campos
adicionales (perfil, trial, observed_behavior). El endpoint
`/auth/sync` es el **puente** que se llama después de cada sign-in
exitoso para crear/actualizar el user en BD.

Este endpoint también maneja el caso especial de **account
linking** (anonymous → permanent provider) preservando todo el
data del user.

Backend en
[`authentication-system.md`](../../architecture/authentication-system.md)
§6 (sync flow).

## Scope

### In

- Endpoint `POST /auth/sync` que:
  - Valida JWT (US-007 middleware).
  - Si el `firebase_uid` no existe en `users`: crea row + crea
    `student_profile` asociado con defaults.
  - Si existe: actualiza `last_signin_at` (no overwrite de otros
    campos por default).
  - Retorna `{ user, profile, is_first_signin }`.
- Endpoint `POST /auth/upgrade-anonymous` que:
  - Valida JWT.
  - Verifica que el `firebase_uid` existía como anonymous.
  - Actualiza `users.is_anonymous = false`,
    `users.signup_provider = <new>`, persiste email si Apple/Email
    lo retorna.
  - Preserva todo el `student_profile` (data del trial).
- Endpoint `DELETE /auth/account` que:
  - Valida JWT.
  - Soft-delete del user (`users.deleted_at = now()`).
  - Hard-delete cron mensual barre soft-deleted con > 30 días.
- Lógica de "set if null" para `display_name` (Apple caveat de
  US-004).
- Idempotency: re-llamadas con mismo body no producen duplicados.
- Telemetry events: `user.signed_up`, `user.signed_in`,
  `user.upgraded_from_anonymous`, `user.account_deleted`.

### Out

- Endpoints CRUD de profile (`/profile/*`) — story propia.
- UI de delete account (Settings) — story propia.
- Cron de hard-delete — story propia (cross-cutting cleanup).

## Acceptance criteria

- **Given** un Firebase user nuevo (firebase_uid no en BD), **When**
  POST `/auth/sync`, **Then** se crea row en `users` + row en
  `student_profiles` con defaults, retorna 201 con
  `is_first_signin: true`.
- **Given** un user existente, **When** POST `/auth/sync` con el
  mismo firebase_uid, **Then** retorna 200 con
  `is_first_signin: false` y `last_signin_at` actualizado.
- **Given** un anonymous user activo, **When** POST
  `/auth/upgrade-anonymous` con `new_provider: 'google'`, **Then**
  `users.is_anonymous = false`, `users.signup_provider = 'google'`,
  email se persiste, `student_profile` queda intacto (todos los
  trial data preservados).
- **Given** un user firma con Apple primer time con display_name
  "María Pérez", **When** vuelve a sign in (Apple no retorna
  display_name segunda vez), **Then** `users.display_name` sigue
  siendo "María Pérez" (no overwrite con null).
- **Given** un user que llama DELETE `/auth/account`, **When** se
  ejecuta, **Then** `users.deleted_at = now()`, queda
  inaccessible vía API pero no se hard-delete inmediatamente.
- **Given** sin JWT, **When** POST `/auth/sync`, **Then** retorna
  401.
- **Given** un user con JWT válido pero firebase_uid revocado o
  banned, **When** POST `/auth/sync`, **Then** retorna 403
  `{ error: "user_blocked" }` (caso anti-fraud).
- **Given** 2 requests POST `/auth/sync` simultáneas con el mismo
  user nuevo, **When** se procesan, **Then** ambas terminan con
  éxito y solo se crea 1 row en BD (idempotency vía
  `ON CONFLICT (firebase_uid) DO UPDATE`).

## Developer details

### Owning service

`apps/workers/api` (auth Worker).

### Dependencies

- US-001: Firebase project.
- US-002: Schema users + student_profiles aplicado.
- US-007: JWT validation middleware.

### Specs referenciados

- [`authentication-system.md`](../../architecture/authentication-system.md)
  §6 — sync flow.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §3.2 — defaults de student_profile.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §10 — API contracts.
- [`anti-fraud-system.md`](../../architecture/anti-fraud-system.md)
  — chequeo de user_restrictions.

### Implementación esperada

```typescript
// apps/workers/api/handlers/auth-sync.ts
import { validateFirebaseJwt } from '../shared/auth-middleware';

interface SyncRequestBody {
  firebase_uid: string;
  provider: 'google' | 'apple' | 'email' | 'anonymous';
  display_name_first_signin?: string;  // solo Apple first signin
}

export async function handleAuthSync(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const body: SyncRequestBody = await request.json();
  const { firebase_uid, provider, display_name_first_signin } = body;

  // Verify the firebase_uid in JWT matches the body
  if (request.user!.firebase_uid !== firebase_uid) {
    return jsonResponse({ error: 'uid_mismatch' }, 403);
  }

  // Anti-fraud check
  const restriction = await checkRestriction(firebase_uid, env);
  if (restriction?.type === 'banned') {
    return jsonResponse({ error: 'user_blocked' }, 403);
  }

  // UPSERT user + create profile if first
  const result = await env.DB.transaction(async (tx) => {
    const userResult = await tx.execute(`
      INSERT INTO users (
        firebase_uid, email, display_name, is_anonymous,
        signup_provider, signup_at, last_signin_at
      ) VALUES (?, ?, ?, ?, ?, now(), now())
      ON CONFLICT (firebase_uid) DO UPDATE
        SET last_signin_at = now(),
            display_name = COALESCE(users.display_name, EXCLUDED.display_name),
            email = COALESCE(EXCLUDED.email, users.email)
      RETURNING id, (xmax = 0) AS is_inserted
    `, [
      firebase_uid,
      request.user!.email,
      display_name_first_signin ?? (provider === 'anonymous' ? 'Invitado' : null),
      provider === 'anonymous',
      provider,
    ]);

    const userId = userResult[0].id;
    const isFirstSignin = userResult[0].is_inserted;

    if (isFirstSignin) {
      await tx.execute(`
        INSERT INTO student_profiles (
          user_id, country, timezone, native_language,
          target_english_variant
        ) VALUES (?, 'OT', 'America/Mexico_City', 'es-MX', 'neutral')
      `, [userId]);
    }

    return { userId, isFirstSignin };
  });

  // Emit event
  await emitEvent(result.isFirstSignin ? 'user.signed_up' : 'user.signed_in', {
    user_id: result.userId,
    firebase_uid,
    provider,
  });

  return jsonResponse({
    user: { id: result.userId, firebase_uid, provider },
    is_first_signin: result.isFirstSignin,
  }, result.isFirstSignin ? 201 : 200);
}
```

### Endpoint upgrade-anonymous

```typescript
export async function handleUpgradeAnonymous(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const { firebase_uid, new_provider } = await request.json();

  // Verify user was anonymous
  const user = await db.getUserByFirebaseUid(firebase_uid);
  if (!user || !user.is_anonymous) {
    return jsonResponse({ error: 'not_anonymous' }, 400);
  }

  await env.DB.execute(`
    UPDATE users SET
      is_anonymous = false,
      signup_provider = ?,
      email = ?,
      display_name = COALESCE(display_name, ?),
      updated_at = now()
    WHERE firebase_uid = ?
  `, [new_provider, request.user!.email, request.user!.email, firebase_uid]);

  await emitEvent('user.upgraded_from_anonymous', {
    user_id: user.id, new_provider,
  });

  return jsonResponse({ success: true });
}
```

### Integration points

- Postgres (Supabase) vía connection pool.
- Anti-fraud system (`checkRestriction`).
- Event bus (Inngest) para `user.*` events.
- Sparks system (consumer de `user.signed_up` event para conceder
  trial Sparks).

### Notas técnicas

- `ON CONFLICT (firebase_uid) DO UPDATE` garantiza idempotency.
- `(xmax = 0)` postgres trick para detectar si fue insert vs update
  en upsert.
- Soft-delete: queries futuras deben filtrar
  `WHERE deleted_at IS NULL`.
- Anti-fraud check ejecuta antes de cualquier escritura.

## Definition of Done

- [ ] 3 endpoints implementados (sync, upgrade-anonymous, delete).
- [ ] 8 acceptance criteria pasan en tests integration.
- [ ] Tests unit de cada handler con mocks.
- [ ] Idempotency verificada con concurrent requests.
- [ ] Telemetry events emitidos.
- [ ] Docs API en `docs/architecture/auth-api.md` (a crear).
- [ ] Validation contra spec `authentication-system.md` §6.
- [ ] PR aprobada y mergeada.

---

*Depende de US-001 + US-002 + US-007. Bloqueante para US-003, US-004,
US-005, US-006 (todos llaman este endpoint).*

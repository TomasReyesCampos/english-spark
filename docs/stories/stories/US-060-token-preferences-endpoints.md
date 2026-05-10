# US-060: Token registration + preferences endpoints

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 2
**Story points:** 3
**Persona:** Estudiante (consume), Admin (mantiene)
**Owner:** —

---

## Contexto

Endpoints HTTP que el cliente mobile usa para:
- Registrar el token FCM al iniciar app.
- Leer/actualizar preferences (opt-out por categoría, hora
  preferida, TZ).
- Trackear `notification.opened` cuando user abre app desde notif.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§10.

## Scope

### In

- `POST /notifications/register-token`:
  - Valida JWT.
  - Body: `{ token, platform, device_id?, app_version? }`.
  - UPSERT en `user_fcm_tokens` (ON CONFLICT token DO UPDATE
    last_used_at).
  - Si mismo token existe en otro user_id (cambio de cuenta en
    device compartido): mark inactive en viejo, associate al
    nuevo.
  - Si user ya tiene este token: refresh last_used_at.
- `PATCH /notifications/preferences`:
  - Valida JWT.
  - Body: any subset of:
    - `push_enabled`, `reminders_enabled`,
      `achievements_enabled`, `onboarding_enabled`,
      `reengagement_enabled`.
    - `preferred_reminder_hour` (0-23).
    - `timezone` (validar contra Intl supported list).
  - UPSERT (crea row si no existe; user_notification_preferences
    autocreado en signup post-MVP).
- `POST /notifications/track-opened`:
  - Valida JWT.
  - Body: `{ notification_log_id, opened_at }`.
  - Update `notifications_log.opened_at` para esa row.
  - Solo permite update si el log es del user actual.
- `GET /notifications/preferences`:
  - Retorna preferences actuales (con defaults si no existe row).
- Telemetry: `notif.token_registered`,
  `notif.preferences_updated`, `notif.opened_tracked`.

### Out

- UI Settings completo de preferences (post-MVP, MVP es solo
  endpoints).
- Email notifications opt-out (post-MVP).

## Acceptance criteria

- **Given** user logged in registra token, **When** POST
  /notifications/register-token, **Then** row insertada en
  `user_fcm_tokens` con `is_active=true`.
- **Given** mismo token re-registrado, **When** segunda POST,
  **Then** `last_used_at` se actualiza, NO crea row duplicada.
- **Given** user A registró token, ahora user B registra mismo
  token (device compartido), **When** procesa, **Then** token de
  user A queda `is_active=false`, user B obtiene row activo.
- **Given** user con preferences default, **When** PATCH con
  `{ reminders_enabled: false, preferred_reminder_hour: 21 }`,
  **Then** se persiste y retorna preferences completas.
- **Given** PATCH con `preferred_reminder_hour: 25`, **When**
  valida, **Then** 400 `{ error: 'invalid_hour' }`.
- **Given** PATCH con `timezone: 'Invalid/Tz'`, **When** valida,
  **Then** 400 `{ error: 'invalid_timezone' }`.
- **Given** notification log_id de otro user, **When** user X
  intenta `track-opened`, **Then** 403 (no permite mark opened
  notif ajena).
- **Given** GET preferences user sin row, **When** se llama,
  **Then** retorna defaults (push_enabled=true, hora=19, tz=MX).

## Developer details

### Owning service

`apps/workers/api/handlers/notifications-*`.

### Dependencies

- US-059: schema.
- US-007: JWT middleware.

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §10.

### Implementación esperada

```typescript
// apps/workers/api/handlers/notifications-token.ts
const RegisterTokenSchema = z.object({
  token: z.string().min(1),
  platform: z.enum(['ios', 'android']),
  device_id: z.string().optional(),
  app_version: z.string().optional(),
});

export async function handleRegisterToken(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const body = await request.json();
  const parsed = RegisterTokenSchema.safeParse(body);
  if (!parsed.success) {
    return jsonResponse({ error: parsed.error.issues[0].message }, 400);
  }

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  await env.DB.transaction(async (tx) => {
    // Mark token inactive in other users (device handoff)
    await tx.execute(`
      UPDATE user_fcm_tokens
      SET is_active = false, last_error = 'token_taken_by_other_user'
      WHERE token = $1 AND user_id != $2
    `, [parsed.data.token, userId]);

    // Upsert for current user
    await tx.execute(`
      INSERT INTO user_fcm_tokens
        (user_id, token, platform, device_id, app_version, is_active)
      VALUES ($1, $2, $3, $4, $5, true)
      ON CONFLICT (token) DO UPDATE
        SET user_id = EXCLUDED.user_id,
            platform = EXCLUDED.platform,
            device_id = EXCLUDED.device_id,
            app_version = EXCLUDED.app_version,
            last_used_at = now(),
            is_active = true,
            last_error = NULL
    `, [userId, parsed.data.token, parsed.data.platform,
        parsed.data.device_id ?? null, parsed.data.app_version ?? null]);
  });

  track('notif.token_registered', { platform: parsed.data.platform });
  return jsonResponse({ success: true });
}

// apps/workers/api/handlers/notifications-preferences.ts
const PreferencesSchema = z.object({
  push_enabled: z.boolean().optional(),
  reminders_enabled: z.boolean().optional(),
  achievements_enabled: z.boolean().optional(),
  onboarding_enabled: z.boolean().optional(),
  reengagement_enabled: z.boolean().optional(),
  preferred_reminder_hour: z.number().int().min(0).max(23).optional(),
  timezone: z.string().optional(),
});

const VALID_TIMEZONES = new Set(Intl.supportedValuesOf('timeZone'));

export async function handleUpdatePreferences(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const body = await request.json();
  const parsed = PreferencesSchema.safeParse(body);
  if (!parsed.success) {
    return jsonResponse({ error: parsed.error.issues[0].message }, 400);
  }

  if (parsed.data.timezone && !VALID_TIMEZONES.has(parsed.data.timezone)) {
    return jsonResponse({ error: 'invalid_timezone' }, 400);
  }

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  const fields = Object.keys(parsed.data);
  if (fields.length === 0) {
    return jsonResponse(await getPreferences(userId, env));
  }

  const setClause = fields.map((f, i) => `${f} = $${i + 2}`).join(', ');
  const values = [userId, ...fields.map(f => (parsed.data as any)[f])];

  await env.DB.execute(`
    INSERT INTO user_notification_preferences (user_id, ${fields.join(', ')})
    VALUES ($1, ${fields.map((_, i) => `$${i + 2}`).join(', ')})
    ON CONFLICT (user_id) DO UPDATE SET ${setClause}
  `, values);

  track('notif.preferences_updated', { fields });
  return jsonResponse(await getPreferences(userId, env));
}
```

### Integration points

- Mobile app (todos los endpoints).
- US-023 (push permission flow llama register-token).
- US-061 (FCM sender lee tokens active).
- US-062 (daily reminder cron lee preferences).

### Notas técnicas

- Token handoff cross-user: requirement de App Store guideline
  cuando devices se comparten. La invalidation manual hace que el
  old user deje de recibir push en ese device.
- Token register es **idempotent**: cliente puede llamarlo en cada
  app start sin issues.
- Default preferences en GET: usar valores defaults aún si row no
  existe (post-signup row se autocrea pero defensive).

## Definition of Done

- [ ] 4 endpoints implementados.
- [ ] Validation Zod completa.
- [ ] Device handoff logic.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-059, US-007.*

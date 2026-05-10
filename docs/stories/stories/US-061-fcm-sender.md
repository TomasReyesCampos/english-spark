# US-061: FCM HTTP v1 sender + OAuth token caching

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 2
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Cliente HTTP que envía notificaciones via FCM HTTP v1 API. FCM v1
**requiere OAuth 2.0 access tokens** (no API keys legacy). Esta
story implementa:
- Generación de access tokens firmando JWT con service account.
- Cache de access tokens (válidos 1h) en KV para evitar refirma
  en cada envío.
- Envío unicast a un device + manejo de tokens inválidos.

Función `sendFcmNotification` usada por todos los crons +
handlers downstream.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§7.4-§7.5.

## Scope

### In

- Función `generateGoogleAccessToken(env)`:
  - Lee `FIREBASE_SERVICE_ACCOUNT` (JSON con private_key,
    client_email).
  - Genera JWT con scope
    `https://www.googleapis.com/auth/firebase.messaging`.
  - Firma con RS256 (Web Crypto API).
  - POST a `https://oauth2.googleapis.com/token` con
    `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`.
  - Cachea response en KV (`fcm_access_token`) con TTL 50 min
    (5 min margen sobre 1h validity).
- Función `sendFcmNotification(job, env)`:
  - Input: `{ user_id, notification_id, category, title, body,
    data }`.
  - Fetch active tokens del user de `user_fcm_tokens`.
  - Para cada token: POST a FCM
    `messages:send` con payload.
  - Si FCM retorna `UNREGISTERED` o `INVALID_ARGUMENT`: mark token
    `is_active=false, last_error=<code>`.
  - Si FCM retorna 200: log success en `notifications_log` con
    `fcm_message_id`.
  - Si user tiene 0 active tokens: log warning + return early.
- Anti-fraud check: verifica `user_restrictions` antes de enviar
  (skip si user banned).
- Telemetry: `notif.fcm_sent`, `notif.fcm_failed`,
  `notif.token_marked_inactive`, `notif.user_blocked_skipped`.

### Out

- Multicast (post-MVP — MVP envía uno por uno).
- iOS-specific options (badge counter aggregation, etc.) — post-MVP.

## Acceptance criteria

- **Given** service account válido configurado, **When** generate
  access token primera vez, **Then** obtiene token válido y cachea
  en KV con TTL 3000s.
- **Given** access token cached (no expirado), **When** próxima
  llamada, **Then** sirve de cache sin hit a OAuth.
- **Given** access token expirado en KV, **When** llamada, **Then**
  regenera + actualiza cache.
- **Given** user con 2 tokens active (iOS + Android), **When**
  sendFcmNotification, **Then** FCM se llama 2x, log entry con
  ambos message_ids.
- **Given** uno de los tokens retorna UNREGISTERED, **When** mark,
  **Then** ese token queda `is_active=false`, otro sigue activo.
- **Given** user sin tokens activos, **When** se intenta enviar,
  **Then** log warning + retorna count={delivered:0} sin error.
- **Given** user con restriction='banned', **When** se intenta
  enviar (excepto transactional críticos), **Then** skip + log.
- **Given** FCM retorna 503, **When** sender procesa, **Then**
  retry 1x con backoff. Si segundo fail: log error + return
  count={failed:N}.

## Developer details

### Owning service

`apps/workers/notifications/src/senders/fcm-sender.ts` +
`utils/fcm-auth.ts`.

### Dependencies

- US-001: Firebase project configurado.
- US-059: schema notifications_log + user_fcm_tokens.
- Secret `FIREBASE_SERVICE_ACCOUNT` (JSON completo).

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §7.4 — envío FCM.
- [`notifications-system.md`](../../architecture/notifications-system.md)
  §7.5 — access token generation.

### Implementación esperada

```typescript
// apps/workers/notifications/src/utils/fcm-auth.ts
export async function generateGoogleAccessToken(env: Env): Promise<string> {
  const cached = await env.NOTIF_KV.get('fcm_access_token');
  if (cached) return cached;

  const serviceAccount = JSON.parse(env.FIREBASE_SERVICE_ACCOUNT);

  const header = btoa(JSON.stringify({ alg: 'RS256', typ: 'JWT' }))
    .replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_');
  const now = Math.floor(Date.now() / 1000);
  const payload = btoa(JSON.stringify({
    iss: serviceAccount.client_email,
    scope: 'https://www.googleapis.com/auth/firebase.messaging',
    aud: 'https://oauth2.googleapis.com/token',
    iat: now,
    exp: now + 3600,
  })).replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_');

  const signed = await signWithPrivateKey(
    `${header}.${payload}`, serviceAccount.private_key
  );
  const jwt = `${header}.${payload}.${signed}`;

  const response = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion=${jwt}`,
  });

  if (!response.ok) {
    throw new Error(`fcm_oauth_failed: ${response.status}`);
  }

  const { access_token } = await response.json();
  await env.NOTIF_KV.put('fcm_access_token', access_token, {
    expirationTtl: 3000, // 50 min
  });
  return access_token;
}

// apps/workers/notifications/src/senders/fcm-sender.ts
export interface NotificationJob {
  user_id: string;
  notification_id: string;
  category: 'reminder' | 'achievement' | 'onboarding' | 'reengagement' | 'transactional';
  title: string;
  body: string;
  data?: Record<string, string>;
  copy_id?: string;
}

export async function sendFcmNotification(
  job: NotificationJob, env: Env
): Promise<{ delivered: number; failed: number }> {
  // Anti-fraud check
  const restriction = await getUserRestriction(job.user_id, env);
  if (restriction === 'banned' && job.category !== 'transactional') {
    track('notif.user_blocked_skipped', { user_id: job.user_id, category: job.category });
    return { delivered: 0, failed: 0 };
  }

  const tokens = await env.DB.query(`
    SELECT id, token, platform FROM user_fcm_tokens
    WHERE user_id = $1 AND is_active = true
  `, [job.user_id]);

  if (tokens.length === 0) {
    console.warn({ event: 'notif.no_active_tokens', user_id: job.user_id });
    return { delivered: 0, failed: 0 };
  }

  const accessToken = await generateGoogleAccessToken(env);
  const projectId = env.FIREBASE_PROJECT_ID;
  const fcmUrl = `https://fcm.googleapis.com/v1/projects/${projectId}/messages:send`;

  let delivered = 0, failed = 0;
  const messageIds: string[] = [];

  for (const token of tokens) {
    try {
      const response = await fetch(fcmUrl, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          message: {
            token: token.token,
            notification: { title: job.title, body: job.body },
            data: { ...(job.data ?? {}), notification_id: job.notification_id },
            android: {
              priority: 'high',
              notification: { sound: 'default', channel_id: job.category },
            },
            apns: {
              payload: { aps: { sound: 'default', badge: 1 } },
            },
          },
        }),
      });

      if (response.ok) {
        const result = await response.json();
        messageIds.push(result.name);
        delivered++;
        track('notif.fcm_sent', { category: job.category });
      } else {
        const errorBody = await response.json().catch(() => ({}));
        const errorCode = errorBody?.error?.details?.[0]?.errorCode;
        if (errorCode === 'UNREGISTERED' || errorCode === 'INVALID_ARGUMENT') {
          await env.DB.execute(`
            UPDATE user_fcm_tokens
            SET is_active = false, last_error = $1, last_error_at = now()
            WHERE id = $2
          `, [errorCode, token.id]);
          track('notif.token_marked_inactive', { reason: errorCode });
        }
        failed++;
        track('notif.fcm_failed', { code: errorCode ?? response.status });
      }
    } catch (error: any) {
      failed++;
      console.error({ event: 'notif.fcm_exception', error: error.message });
    }
  }

  // Log
  await env.DB.execute(`
    INSERT INTO notifications_log
      (user_id, notification_id, category, title, body, data, copy_id, fcm_message_id)
    VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
  `, [
    job.user_id, job.notification_id, job.category,
    job.title, job.body, JSON.stringify(job.data ?? {}),
    job.copy_id ?? null, messageIds.join(',') || null,
  ]);

  return { delivered, failed };
}
```

### Integration points

- US-062 a US-068 (todos los crons + handlers usan sendFcmNotification).
- US-070 rate limiter (orquesta caller side).
- US-058 anti-fraud restrictions.

### Notas técnicas

- Web Crypto API en Workers permite RS256 signing nativamente.
- `signWithPrivateKey` requiere importar la private key como
  CryptoKey y sign con SubtleCrypto. PEM parsing + DER conversion
  necesario.
- FCM v1 quotas: 600,000 messages/min per project. MVP no se acerca.
- Failure handling: token inválido = `is_active=false` + log;
  retry no aplica (tokens malos no se arreglan).

## Definition of Done

- [ ] Funciones implementadas.
- [ ] Cache OAuth funcional.
- [ ] Token invalidation automática.
- [ ] Anti-fraud check.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit con mocked FCM + KV.
- [ ] Test integration: enviar push real a device de prueba.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-062 a US-068.*

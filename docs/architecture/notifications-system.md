# Notifications System

> Sistema de notificaciones del MVP basado en Firebase Cloud Messaging.
> Push a iOS + Android orquestado desde Cloudflare Workers. Daily
> reminders timezone-aware con contenido personalizado generado en
> batch nocturno. WhatsApp/SMS post-MVP.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** MVP (meses 0–6)

---

## 0. Cómo leer este documento

- §1 establece **objetivo** y scope.
- §2 cubre **boundaries**.
- §3 lista **stack** y por qué FCM directo (no Expo Push).
- §4 contiene el **catálogo** de notificaciones del MVP.
- §5 cubre el **modelo de datos**.
- §6 cubre el **flujo del cliente** (RN + Expo).
- §7 cubre el **orquestador** (Cloudflare Workers + Cron Triggers).
- §8 cubre **personalización con IA**.
- §9 cubre **rate limiting**.
- §10 contiene **API contracts**.
- §11 enumera **edge cases**.
- §12 cubre **eventos emitidos**.
- §13 cubre **observabilidad**.
- §14 cubre **roadmap post-MVP** (WhatsApp).
- §15 cubre **decisiones cerradas**.

---

## 1. Objetivo y scope

### 1.1 Objetivo

Sostener la retención del producto durante el MVP mediante push
notifications efectivas que mantengan al usuario practicando con
consistencia.

El aprendizaje de idiomas vive de la **consistencia diaria**. Sin
notificaciones, los usuarios olvidan practicar y abandonan en pocos
días.

### 1.2 Incluye en MVP

- Push notifications nativas a iOS y Android via FCM.
- Recordatorios diarios personalizados por hora preferida.
- Notificaciones de eventos clave (logros, streaks, sistema).
- Re-engagement de usuarios inactivos (D3, D7, D14).
- Preferencias granulares por tipo.
- Personalización del contenido con IA en batch nocturno.

### 1.3 NO incluye en MVP

- WhatsApp Business API (post-validación, §14).
- Email transaccional masivo (solo recibos via Stripe/RevenueCat).
- SMS.
- In-app messaging avanzado.

---

## 2. Boundaries

### 2.1 Es responsable de

- Recibir tokens FCM del cliente y persistirlos.
- Calcular qué usuarios deben recibir qué notificación cada día.
- Personalizar contenido (vía AI Gateway batch nocturno).
- Enviar push via FCM.
- Logging de delivery + opens.
- Rate limiting por categoría.
- Circuit breaker para usuarios que dejan de engagear.
- Cleanup de tokens inválidos.

### 2.2 NO es responsable de

- **Decidir qué logros existen** (eso es `motivation-and-achievements`;
  este sistema solo recibe `achievement.unlocked` y manda push).
- **Validar restricciones por fraude** (lee `user_restrictions` de
  `anti-fraud-system`).
- **Generar prompts de IA dinámicamente** (usa Task Registry de
  `ai-gateway-strategy`).
- **Notificaciones in-app** mientras el user está en la app (eso lo
  resuelve la UI).
- **Email transaccional** (Stripe/RevenueCat envían recibos directo).

### 2.3 Tensiones

| Tensión | Resolución |
|---------|-----------|
| User practicó hoy → no enviar daily reminder | Query verifica `exercise_attempts` últimas 24h en TZ del user antes de enviar |
| User tiene `restriction = banned` → no enviar push | Query lee `user_restrictions`; deniega envío |
| Logro desbloqueado mid-session → ¿push o notificación in-app? | Si app está abierta: in-app via UI; si cerrada: push (decisión del cliente FE) |
| Push falla por token inválido | Marcar token `is_active = false` y skip en próximo envío |

---

## 3. Stack

### 3.1 Componentes

| Componente | Servicio | Rol |
|-----------|----------|-----|
| Push delivery | FCM HTTP v1 API | Envío real a iOS y Android |
| SDK cliente | `@react-native-firebase/messaging` | Recepción y manejo en app |
| Orquestación | Cloudflare Workers | Lógica de cuándo y a quién enviar |
| Scheduling | Cloudflare Cron Triggers | Disparos programados |
| Estado de rate limits | Cloudflare Durable Objects | Anti-spam por usuario |
| Persistencia | Postgres (Supabase) | Tokens, preferencias, historial |
| Personalización | AI Gateway (existente) | Generación batch nocturna |

### 3.2 Por qué FCM directo y no Expo Push

(Ver ADR-004-notifications-fcm.)

- Control total sobre features de FCM.
- Sin intermediarios.
- Mejor analytics (Firebase Analytics integra automáticamente).
- Topics y conditions nativos.
- Costo cero (FCM gratis ilimitado).

### 3.3 Setup inicial necesario

- Proyecto Firebase con Cloud Messaging habilitado.
- iOS: certificado APNs (.p8) subido a Firebase.
- Android: `google-services.json` descargado.
- SDK en app: `@react-native-firebase/app` +
  `@react-native-firebase/messaging` (requiere Expo dev build).
- Service account JSON en Cloudflare Secret (`FIREBASE_SERVICE_ACCOUNT`).

---

## 4. Catálogo de notificaciones

> **Copy literal y plantillas completas** (variantes por contexto,
> banco de los 80 logros, validation rules, A/B tests priorizados)
> viven en [`../product/push-notifications-copy-bank.md`](../product/push-notifications-copy-bank.md).
> Esta sección documenta **trigger, frecuencia y categoría**; el doc
> referenciado es la fuente de verdad para los textos.

### 4.1 Tabla canónica

| `notification_id` | Categoría | Frecuencia | Trigger | Personalizada |
|-------------------|-----------|-----------|---------|:-:|
| `daily_reminder` | `reminder` | 1/día (hora preferida) | Cron horario | Sí |
| `streak_at_risk` | `reminder` | 1/día si aplica | Cron horario | Sí |
| `level_completed` | `achievement` | Por evento | Domain event | No |
| `achievement_unlocked` | `achievement` | Por evento | Domain event | No |
| `welcome_d1` | `onboarding` | 1 vez | 24h post-registro | No (template) |
| `welcome_d3` | `onboarding` | 1 vez | 72h post-registro | Sí |
| `welcome_d5` | `onboarding` | 1 vez | 5d post-registro | Sí (pre-assessment) |
| `welcome_d7` | `onboarding` | 1 vez | 7d post-registro | Sí (assessment day) |
| `inactivity_d3` | `reengagement` | 1 vez | 72h sin actividad | Sí |
| `inactivity_d7` | `reengagement` | 1 vez | 7d sin actividad | Sí |
| `inactivity_d14` | `reengagement` | 1 vez | 14d sin actividad | Sí |
| `low_sparks` | `transactional` | Por evento | Balance < 20% del plan | No |
| `pack_expiring` | `transactional` | Por evento | Pack expira en 7d | No |
| `payment_failed` | `transactional` | Por evento | Pago rechazado | No |
| `restriction_applied` | `transactional` | Por evento | Anti-fraud aplicó restricción | No |

### 4.2 Categorías y opt-out

```typescript
type NotificationCategory =
  | 'reminder'        // user puede opt-out
  | 'achievement'     // user puede opt-out
  | 'onboarding'      // user puede opt-out (afecta retention)
  | 'reengagement'    // user puede opt-out
  | 'transactional';  // SIEMPRE se envía si push está habilitado globalmente
```

Transactional **no** se puede desactivar individualmente sin desactivar
push global del SO.

### 4.3 Daily reminder (la más importante)

#### Características

- Hora preferida del usuario (default 19:00 hora local).
- En su timezone (cron horario por timezone).
- **NO se envía** si el usuario ya practicó hoy.
- Contenido personalizado al foco pedagógico de hoy.
- Menciona racha actual si > 3 días.

#### Ejemplos de contenido

```
Título: "Tu plan te espera, Juan"
Cuerpo: "Hoy practicamos pronunciación de /θ/. 12 minutos para
mantener tu racha de 8 días."
```

```
Título: "12 minutos de inglés, María"
Cuerpo: "Hoy: roleplay de entrevista técnica. Tu próxima entrevista
se acerca."
```

Generados por `ai-gateway` task `generate_notification_content` en
batch nocturno (§8).

### 4.4 Streak at risk

#### Reglas

- Solo si racha actual ≥ 5 días (sin urgencia para rachas cortas).
- Se envía 4 horas antes del corte (medianoche en TZ del user).
- Solo si user tiene `reminders_enabled = true`.
- Max 1/día (rate limit).

#### Ejemplos

```
Título: "🔥 Tu racha de 23 días en juego"
Cuerpo: "Quedan 4 horas. Solo necesitás 12 minutos."
```

### 4.5 Re-engagement (D3, D7, D14)

Tres niveles con tono distinto:

| Día | Tono | Contenido |
|-----|------|-----------|
| D3 | Suave, motivador | "Tu plan te está esperando" |
| D7 | Personal, mostrar progreso perdido | "Mejorabas X. Volvé." |
| D14 | Último intento, posible incentivo | "Volvé y te regalamos 20 Sparks bonus" |

Después de D14: no se envían más. User churned.

### 4.6 Transaccionales

Notificaciones críticas que se envían independiente de
"reminders_enabled" (siempre que push esté habilitado):

- `low_sparks`: balance < 20% del plan mensual.
- `pack_expiring`: 7 días antes de expirar.
- `payment_failed`: inmediato, con CTA para corregir.
- `restriction_applied`: anti-fraud aplicó restricción.

---

## 5. Modelo de datos

### 5.1 Tokens FCM por usuario

```sql
CREATE TABLE user_fcm_tokens (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token           TEXT NOT NULL UNIQUE,
  platform        TEXT NOT NULL CHECK (platform IN ('ios', 'android')),
  device_id       TEXT,
  app_version     TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_used_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_active       BOOLEAN NOT NULL DEFAULT true,
  last_error      TEXT,
  last_error_at   TIMESTAMPTZ
);

CREATE INDEX idx_fcm_tokens_user_active
  ON user_fcm_tokens(user_id) WHERE is_active = true;
CREATE INDEX idx_fcm_tokens_inactive_old
  ON user_fcm_tokens(last_used_at) WHERE is_active = false;
```

### 5.2 Preferencias

```sql
CREATE TABLE user_notification_preferences (
  user_id                 UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,

  push_enabled            BOOLEAN NOT NULL DEFAULT true,

  reminders_enabled       BOOLEAN NOT NULL DEFAULT true,
  achievements_enabled    BOOLEAN NOT NULL DEFAULT true,
  onboarding_enabled      BOOLEAN NOT NULL DEFAULT true,
  reengagement_enabled    BOOLEAN NOT NULL DEFAULT true,
  -- transactional siempre on cuando push_enabled

  preferred_reminder_hour INT NOT NULL DEFAULT 19 CHECK (preferred_reminder_hour BETWEEN 0 AND 23),
  timezone                TEXT NOT NULL DEFAULT 'America/Mexico_City',

  -- Circuit breaker
  paused_until            TIMESTAMPTZ,             -- pause auto si user no engaga
  pause_reason            TEXT,

  updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.3 Historial enviadas

```sql
CREATE TABLE notifications_log (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  notification_id TEXT NOT NULL,         -- ej: 'daily_reminder'
  category        TEXT NOT NULL,
  title           TEXT NOT NULL,
  body            TEXT NOT NULL,
  data            JSONB NOT NULL DEFAULT '{}',
  fcm_message_id  TEXT,                   -- response de FCM
  sent_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  delivered_at    TIMESTAMPTZ,
  opened_at       TIMESTAMPTZ,
  error           TEXT,
  CONSTRAINT valid_status CHECK (
    error IS NULL OR (delivered_at IS NULL AND opened_at IS NULL)
  )
);

CREATE INDEX idx_notif_log_user_date ON notifications_log(user_id, sent_at DESC);
CREATE INDEX idx_notif_log_type_date ON notifications_log(notification_id, sent_at DESC);
CREATE INDEX idx_notif_log_recent_opens
  ON notifications_log(user_id, opened_at DESC)
  WHERE opened_at IS NOT NULL;
```

Retention: 6 meses, después purge (cleanup cron).

### 5.4 Contenido pre-generado

```sql
CREATE TABLE notifications_scheduled (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  notification_id TEXT NOT NULL,
  scheduled_for   TIMESTAMPTZ NOT NULL,
  title           TEXT NOT NULL,
  body            TEXT NOT NULL,
  data            JSONB NOT NULL DEFAULT '{}',
  status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                    'pending', 'sent', 'cancelled', 'failed'
                  )),
  generated_by    TEXT NOT NULL DEFAULT 'system',  -- 'ai' | 'system'
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scheduled_pending ON notifications_scheduled(scheduled_for)
  WHERE status = 'pending';
```

El job nocturno popula con contenido del día siguiente. Cron cada 15
min lee y envía.

---

## 6. Flujo del cliente

### 6.1 Setup

```typescript
// app/notifications/setup.ts
import messaging from '@react-native-firebase/messaging';
import { Platform } from 'react-native';

export async function setupNotifications() {
  // 1. Solicitar permisos
  const authStatus = await messaging().requestPermission();
  const enabled =
    authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
    authStatus === messaging.AuthorizationStatus.PROVISIONAL;

  if (!enabled) {
    await api.post('/notifications/preferences', { push_enabled: false });
    return;
  }

  // 2. Obtener token FCM
  const token = await messaging().getToken();

  // 3. Registrar en backend
  await api.post('/notifications/register-token', {
    token,
    platform: Platform.OS,
    device_id: await getDeviceId(),
    app_version: Application.nativeApplicationVersion,
  });

  // 4. Listener para refresh de token
  messaging().onTokenRefresh(async (newToken) => {
    await api.post('/notifications/register-token', { token: newToken });
  });
}
```

### 6.2 Recepción

```typescript
// Notificación con app en background
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  // FCM ya muestra automáticamente
  await trackNotificationReceived(remoteMessage);
});

// Notificación con app abierta
messaging().onMessage(async (remoteMessage) => {
  showInAppNotification(remoteMessage);
});

// User abre app desde notificación
messaging().onNotificationOpenedApp((remoteMessage) => {
  if (remoteMessage.data?.deeplink) {
    navigate(remoteMessage.data.deeplink);
  }
  trackNotificationOpened(remoteMessage);
});
```

### 6.3 UI de preferencias

Settings con:
- Toggle global push on/off (links a Settings del SO si está
  desactivado a nivel sistema).
- Toggles por categoría (reminders, achievements, reengagement).
- Selector hora preferida (slider 0-23).
- Selector timezone (override del detectado).
- Indicación: "Las transaccionales siempre llegan."

---

## 7. Orquestador Workers

### 7.1 Estructura

```
apps/workers/notifications/
├── src/
│   ├── index.ts
│   ├── schedulers/
│   │   ├── daily-reminders.ts
│   │   ├── streak-checker.ts
│   │   ├── inactivity-detector.ts
│   │   ├── scheduled-dispatcher.ts
│   │   └── cleanup.ts
│   ├── senders/
│   │   └── fcm-sender.ts
│   ├── triggers/
│   │   ├── on-level-complete.ts
│   │   ├── on-achievement-unlocked.ts
│   │   ├── on-payment-failed.ts
│   │   └── on-low-sparks.ts
│   ├── utils/
│   │   ├── rate-limiter.ts        (Durable Object)
│   │   ├── timezone.ts
│   │   └── fcm-auth.ts
│   └── types/
└── wrangler.toml
```

### 7.2 wrangler.toml

```toml
name = "notifications-worker"
main = "src/index.ts"
compatibility_date = "2026-01-01"

[[d1_databases]]
binding = "DB"
database_name = "main"

[[kv_namespaces]]
binding = "RATE_LIMITS"

[[durable_objects.bindings]]
name = "USER_RATE_LIMITER"
class_name = "UserRateLimiter"

[triggers]
crons = [
  "*/15 * * * *",       # Cada 15 min: dispatch de scheduled
  "0 * * * *",          # Cada hora: chequeo de daily reminders por TZ
  "0 20 * * *",         # 20:00 UTC: detección de streaks en peligro
  "0 1 * * *",          # 01:00 UTC: detección de inactividad
  "0 2 * * *",          # 02:00 UTC: generar contenido del día siguiente
  "0 3 * * 0",          # Domingos 03:00 UTC: cleanup tokens viejos + logs
]

[vars]
FIREBASE_PROJECT_ID = "your-project-id"
```

### 7.3 Detección de daily reminders por timezone

```typescript
async function checkDailyReminders(env: Env) {
  const currentUtcHour = new Date().getUTCHours();

  // Query: usuarios cuya hora preferida coincide con UTC actual
  // según su TZ, que no han practicado hoy
  const users = await env.DB.prepare(`
    SELECT
      u.id, u.display_name, np.preferred_reminder_hour, np.timezone
    FROM users u
    JOIN user_notification_preferences np ON np.user_id = u.id
    WHERE np.push_enabled = true
      AND np.reminders_enabled = true
      AND (np.paused_until IS NULL OR np.paused_until < now())
      AND user_local_hour(?, np.timezone) = np.preferred_reminder_hour
      AND NOT EXISTS (
        SELECT 1 FROM exercise_attempts ea
        WHERE ea.user_id = u.id
          AND ea.completed_at > date_trunc('day', now() AT TIME ZONE np.timezone)
      )
      AND NOT EXISTS (
        SELECT 1 FROM user_restrictions ur
        WHERE ur.user_id = u.id
          AND ur.restriction_type IN ('suspended', 'banned')
          AND (ur.ends_at IS NULL OR ur.ends_at > now())
      )
  `).bind(currentUtcHour).all();

  for (const user of users.results) {
    await dispatchDailyReminder(user, env);
  }
}
```

### 7.4 Envío via FCM HTTP v1

```typescript
// src/senders/fcm-sender.ts
import { generateGoogleAccessToken } from '../utils/fcm-auth';

export async function sendFcmNotification(
  job: NotificationJob,
  env: Env,
): Promise<FcmResult> {
  const accessToken = await generateGoogleAccessToken(env);

  const fcmEndpoint =
    `https://fcm.googleapis.com/v1/projects/${env.FIREBASE_PROJECT_ID}/messages:send`;

  const tokens = await getActiveTokens(job.userId, env);
  if (tokens.length === 0) return { delivered: 0, failed: 0 };

  const results = await Promise.allSettled(tokens.map((token) =>
    fetch(fcmEndpoint, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        message: {
          token: token.token,
          notification: { title: job.title, body: job.body },
          data: job.data,
          android: {
            priority: 'high',
            notification: {
              sound: 'default',
              channel_id: job.category,
            },
          },
          apns: {
            payload: {
              aps: { sound: 'default', badge: 1 },
            },
          },
        },
      }),
    }).then(async (res) => ({
      token: token.token,
      success: res.ok,
      response: await res.json(),
    })),
  ));

  // Manejar tokens inválidos
  for (const result of results) {
    if (result.status === 'fulfilled' && !result.value.success) {
      const errorCode = result.value.response?.error?.details?.[0]?.errorCode;
      if (errorCode === 'UNREGISTERED' || errorCode === 'INVALID_ARGUMENT') {
        await markTokenInactive(result.value.token, env, errorCode);
      }
    }
  }

  await logNotification(job, results, env);

  return aggregateResults(results);
}
```

### 7.5 Generación del access token

FCM requiere OAuth 2.0 access tokens, no API keys. Se generan firmando
JWT con la service account.

```typescript
export async function generateGoogleAccessToken(env: Env): Promise<string> {
  // Cache (token válido 1h, cachear 50 min)
  const cached = await env.RATE_LIMITS.get('fcm_access_token');
  if (cached) return cached;

  const serviceAccount = JSON.parse(env.FIREBASE_SERVICE_ACCOUNT);

  const jwt = await createSignedJwt({
    iss: serviceAccount.client_email,
    scope: 'https://www.googleapis.com/auth/firebase.messaging',
    aud: 'https://oauth2.googleapis.com/token',
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + 3600,
  }, serviceAccount.private_key);

  const response = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion=${jwt}`,
  });

  const { access_token } = await response.json();

  await env.RATE_LIMITS.put('fcm_access_token', access_token, {
    expirationTtl: 3000, // 50 min
  });

  return access_token;
}
```

---

## 8. Personalización con IA

### 8.1 Generación batch nocturna

Cron 02:00 UTC (cuando es ~20:00 en MX, antes de daily reminders del
día siguiente):

```
1. Para cada user activo (logueado en últimos 7 días):
   a. Recopilar context: nombre, racha, foco pedagógico de mañana,
      logro reciente.
   b. Llamar AI Gateway task 'generate_notification_content'.
   c. Recibir contenido para daily_reminder y streak_at_risk si aplica.
   d. Persistir en notifications_scheduled con scheduled_for = mañana
      hora preferida.
2. Cron horario disparcha cuando scheduled_for matchea.
```

### 8.2 Prompt template

(Ver `ai-gateway-strategy.md` §4.2.7 para Task Registry config.)

```
Generá el contenido de notificaciones push para este usuario en español
neutro latinoamericano, tono cercano y motivador.

DATOS:
- Nombre: {{user.name}}
- Racha actual: {{user.streak}} días
- Foco pedagógico de mañana: {{tomorrow.focus}}
- Logro reciente: {{recent.highlight}}
- Plan: {{user.plan}}

NOTIFICACIONES A GENERAR:

1. daily_reminder (max 50 chars en título, 90 en cuerpo):
   - Personal y motivador
   - Mencionar racha si >= 3 días
   - Mencionar foco específico de mañana

2. streak_at_risk (solo si racha >= 5 días):
   - Tono urgente pero amigable
   - Énfasis en lo que están por perder

DEVOLVÉ JSON:
{
  "daily_reminder": { "title": "...", "body": "..." },
  "streak_at_risk": { "title": "...", "body": "..." }
}
```

### 8.3 Costo estimado

~$0.0005 USD por user/día. A 10.000 usuarios activos: ~$5/día = ~$150/mes.
Despreciable comparado con impacto en retention.

### 8.4 Fallback si AI falla

Si la generación falla, usar templates genéricos:

```typescript
const FALLBACK_DAILY = {
  title: 'Tu plan te espera',
  body: 'Hoy: {{minutes}} min de práctica. ¿Vamos?',
};
```

---

## 9. Rate limiting y anti-spam

### 9.1 Límites por usuario

```typescript
const RATE_LIMITS = {
  reminder:       { per_day: 1,  per_week: 7 },
  achievement:    { per_day: 3,  per_week: 15 },
  onboarding:     { per_day: 1,  per_week: 4 },
  reengagement:   { per_day: 1,  per_week: 3 },
  transactional:  { per_day: 5,  per_week: 20 },
};
```

### 9.2 Implementación con Durable Objects

```typescript
export class UserRateLimiter implements DurableObject {
  state: DurableObjectState;

  constructor(state: DurableObjectState) {
    this.state = state;
  }

  async checkAndIncrement(category: string): Promise<boolean> {
    const todayKey = `${category}_${getTodayKey()}`;
    const weekKey = `${category}_${getWeekKey()}`;

    const todayCount = (await this.state.storage.get<number>(todayKey)) ?? 0;
    const weekCount = (await this.state.storage.get<number>(weekKey)) ?? 0;

    const limits = RATE_LIMITS[category];
    if (todayCount >= limits.per_day || weekCount >= limits.per_week) {
      return false;
    }

    await this.state.storage.put(todayKey, todayCount + 1);
    await this.state.storage.put(weekKey, weekCount + 1);

    // TTL automático para cleanup
    await this.state.storage.setAlarm(Date.now() + 8 * 24 * 60 * 60 * 1000);

    return true;
  }
}
```

### 9.3 Circuit breaker

Si el usuario:
- Toca "no me interesa" en una notificación.
- Desactiva push en Settings del SO.
- No abre la app durante 30+ días.

Se aplica circuit breaker:

```typescript
const CIRCUIT_BREAKER_RULES = [
  { signal: 'user_dismissed_3_in_row',     pause_days: 7 },
  { signal: 'open_rate_under_5pct_in_30d', pause_days: 14 },
  { signal: 'inactive_30_days',            pause_days: 30 },
];
```

`user_notification_preferences.paused_until` se setea; queries lo
respetan.

---

## 10. API contracts

### 10.1 `POST /notifications/register-token`

**Request:**

```typescript
interface RegisterTokenRequest {
  token: string;
  platform: 'ios' | 'android';
  device_id?: string;
  app_version?: string;
}
```

**Response:** `200 OK` o `4xx`.

**Reglas:**
- Verificar JWT.
- Idempotente (mismo token re-registrado actualiza `last_used_at`).
- Si token ya existe en otro user: invalidarlo en el viejo, asociar al
  nuevo.

### 10.2 `POST /notifications/preferences`

**Request:**

```typescript
interface UpdatePreferencesRequest {
  push_enabled?: boolean;
  reminders_enabled?: boolean;
  achievements_enabled?: boolean;
  reengagement_enabled?: boolean;
  preferred_reminder_hour?: number;
  timezone?: string;
}
```

**Response:** `200 OK` con preferencias actualizadas.

### 10.3 `POST /notifications/track-opened`

**Llamado por:** cliente cuando user abre app desde notificación.

**Request:**

```typescript
interface TrackOpenedRequest {
  notification_log_id: string;     // UUID del log entry
  opened_at: string;               // ISO
}
```

### 10.4 Internal: `dispatchNotification`

**Llamado por:** triggers internos (no expuesto a cliente).

```typescript
async function dispatchNotification(input: {
  user_id: string;
  notification_id: string;
  category: NotificationCategory;
  title: string;
  body: string;
  data?: Record<string, string>;
  bypass_rate_limit?: boolean;     // solo para SEV-1 transactionals
}): Promise<DispatchResult>
```

---

## 11. Edge cases (tests obligatorios)

### 11.1 Tokens

1. **Token registrado por user A llega al device de user B (compartido):**
   detectar en re-register; invalidar para user A, asociar a user B.
2. **Token retorna `UNREGISTERED` de FCM:** mark `is_active = false`,
   `last_error = 'UNREGISTERED'`.
3. **User logout y vuelve a logear con mismo device:** mismo token,
   `last_used_at` se actualiza.
4. **User cambia device:** nuevo token registrado, viejo eventualmente
   marcado inactive cuando FCM lo rechace.

### 11.2 Daily reminders

5. **User practicó hace 1 minuto y ahora viene la hora del reminder:**
   query `exercise_attempts > start_of_day_in_tz` filtra; no se envía.
6. **User cambia TZ entre 18h y 19h:** edge: si nuevo TZ pone su hora
   actual fuera de las 19h, no se envía hoy. Mañana sí.
7. **User en TZ con DST cambio:** `Intl.DateTimeFormat` resuelve TZ
   correctamente. Test con `'America/Santiago'` durante cambio.

### 11.3 Streak at risk

8. **User tiene streak 5 días, son las 22:00 en su TZ y no practicó:**
   se envía `streak_at_risk` (4h antes de medianoche).
9. **User practica entre que se schedulea y que se envía:** scheduler
   debe re-verificar antes de enviar.

### 11.4 Re-engagement

10. **User vuelve a la app en día 5 y al irse pasa otra vez 7 días:**
    `inactivity_d7` se schedulea de nuevo. Anti-loop: contar desde
    último `notification.opened`, no desde signup.

### 11.5 Rate limiting

11. **3 logros desbloqueados en 1 sesión:** primer push de logro
    immediato, los siguientes acumulan en notificación grupal o se
    suprime hasta próximo evento (decisión: suprimir; user verá
    todos los logros en la pantalla de logros).
12. **Cap de 3/día de achievement alcanzado, llega 4to:** notificación
    se descarta silenciosamente. Logro queda en `user_achievements`.

### 11.6 Anti-fraud integration

13. **User con `restriction = banned`:** notificaciones no se envían
    (excepto `restriction_applied` y `payment_failed` críticos).
14. **User con `restriction = suspended` temporal:** no se envía nada
    hasta que termine.

### 11.7 FCM failures

15. **FCM 503 durante envío masivo:** retry con backoff exponencial.
    Si falla 3 veces, marcar para retry posterior.
16. **Service account JWT expira mid-flight:** regenerar, retry.
17. **Project ID equivocado en config:** todos los envíos fallan;
    alerta crítica.

---

## 12. Eventos emitidos

(Detalle del shape en `cross-cutting/data-and-events.md` §5.6.)

| Evento | Cuándo |
|--------|--------|
| `notification.scheduled` | Persistido en `notifications_scheduled` |
| `notification.sent` | FCM aceptó la solicitud |
| `notification.delivered` | FCM confirmó delivery |
| `notification.opened` | User abrió la app desde la notif |
| `notification.dismissed` | User dismisseó sin abrir |
| `notification.preferences_changed` | User editó settings |

---

## 13. Observabilidad

### 13.1 Métricas críticas

| Métrica | Definición | Target |
|---------|-----------|--------|
| Delivery rate | % de FCM accepted vs intentos | > 95% |
| Open rate `daily_reminder` | % opens vs delivered | 8–15% |
| Open rate `streak_at_risk` | % opens vs delivered | 15–25% |
| Open rate `achievement_unlocked` | % opens vs delivered | 20–30% |
| Open rate `inactivity_d3` | % opens vs delivered | 3–8% |
| Click-through rate | % que generan apertura de la app | 5–10% global |
| Conversion rate | % que generan acción objetivo (práctica, compra) | Variable |
| Opt-out rate por categoría | % que desactivan después de recibir | < 2%/mes |

### 13.2 Stack

- **Firebase Analytics:** integrado nativamente con FCM.
- **PostHog:** eventos custom de conversion.
- **Cloudflare Logs:** logs del Worker.
- **`notifications_log`:** auditoría completa con joins a métricas.

### 13.3 Alertas

- Delivery rate < 90%: SEV-2.
- Open rate `daily_reminder` cae > 30% vs baseline: SEV-2 (algo se
  rompió o user fatigue).
- Spike en opt-outs: SEV-2.
- Errors de envío > 5%: SEV-2.

---

## 14. Roadmap post-MVP

### 14.1 Cuándo agregar WhatsApp

Cuando se cumplan **todos**:

- [ ] Producto validado: D30 retention > 30%, pago recurrente.
- [ ] Volumen: > 5.000 usuarios activos.
- [ ] Push open rate `daily_reminder` < 10% (señal que push no es
  suficiente).
- [ ] Margen del producto soporta $1–3 USD/usuario/mes en WhatsApp.

### 14.2 Casos de uso prioritarios para WhatsApp

En orden:

1. **Re-engagement de usuarios inactivos D7+** (donde push falló).
2. **Concierge para Pro/Premium** ("tu tutor por WhatsApp").
3. **Recuperación de churn** (usuarios que cancelaron).
4. **Avisos críticos con CTA inmediato** (payment failed con link).

### 14.3 Otros canales post-MVP

- Email transaccional masivo (Resend) para reportes semanales detallados.
- Firebase In-App Messaging para anuncios contextuales.
- Web push cuando se lance web app (mes 9+).

### 14.4 Migraciones técnicas potenciales

- FCM HTTP v1 → Firebase Admin SDK directo si migramos a Node.js
  servers (escala muy grande).
- Cloudflare Cron Triggers → Inngest si la lógica de scheduling se
  vuelve compleja.

---

## 15. Decisiones cerradas

### 15.1 FCM directo (no Expo Push) ✓

(Ver ADR-004.)

### 15.2 Personalización con IA en batch nocturno ✓

**Razón:** open rate sube significativamente con personalización vs
templates genéricos. Costo despreciable ($150/mes a 10k users) vs
beneficio en retention.

### 15.3 NO WhatsApp en MVP ✓

**Razón:** costo + complejidad. Reconsiderar con criterios de §14.1.

### 15.4 NO email masivo en MVP ✓

**Razón:** open rate bajo para apps consumer; recibos los manejan
Stripe/RevenueCat.

### 15.5 NO snooze de notificaciones por 1 hora ✓

**Razón:** complejidad UI sin beneficio claro. User puede silenciar a
nivel SO.

### 15.6 Smart timing (aprender hora real de uso): **post-MVP** ✓

**Razón:** feature interesante pero requiere data acumulada. MVP usa
hora preferida declarada; smart override en Fase 2.

### 15.7 NO notificaciones grupales en MVP ✓

**Razón:** complejidad implementación. Suprimimos extras y user ve
todo en su perfil.

### 15.8 NO rich notifications (imágenes/audio embedded) en MVP ✓

**Razón:** complejidad payload + rendering. Texto simple alcanza.

### 15.9 A/B test de variantes con Firebase Remote Config: **post-MVP** ✓

**Razón:** PostHog feature flags suplen para empezar.

---

## 16. Plan de implementación

### 16.1 Sprint 1 (semana 1)

- Proyecto Firebase configurado iOS+Android.
- SDK en Expo dev build.
- Endpoint `/notifications/register-token`.
- Tablas `user_fcm_tokens` y `user_notification_preferences`.
- UI básica de preferencias.

### 16.2 Sprint 2 (semana 2)

- Cloudflare Worker setup.
- Generación de access tokens FCM con cache.
- Función de envío FCM con manejo de errors.
- Endpoint HTTP para test manual.

### 16.3 Sprint 3 (semana 3)

- Cron de daily reminders timezone-aware.
- Cron de detección de inactividad.
- Cron de cleanup de tokens viejos.
- Tabla `notifications_log` y logging.

### 16.4 Sprint 4 (semana 4)

- Tabla `notifications_scheduled`.
- Integración con AI Gateway para batch nocturno.
- Cron dispatcher de scheduled.

### 16.5 Sprint 5 (semana 5)

- Domain event handlers (level_complete, achievement_unlocked,
  payment_failed, etc.).
- Durable Object UserRateLimiter.
- Circuit breaker logic.

### 16.6 Sprint 6 (semana 6)

- Firebase Analytics integration.
- Eventos a PostHog.
- Dashboard interno.
- Tests de integración.

---

## 17. Costos

### 17.1 MVP (10.000 usuarios activos)

| Componente | Costo mensual |
|-----------|---------------|
| FCM | $0 |
| Cloudflare Workers | $5 |
| Cloudflare KV + DO | $5 |
| AI Gateway (personalización) | ~$150 |
| **Total** | **~$160/mes** |

### 17.2 Escala (100.000 usuarios activos)

| Componente | Costo mensual |
|-----------|---------------|
| FCM | $0 |
| Cloudflare Workers | $20 |
| Cloudflare KV + DO | $50 |
| AI Gateway | ~$1.500 |
| **Total** | **~$1.570/mes** |

---

## 18. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`../decisions/ADR-004-notifications-fcm.md`](../decisions/ADR-004-notifications-fcm.md) | Decisión arquitectónica. |
| [`authentication-system.md`](authentication-system.md) | Firebase también para Auth. |
| [`ai-gateway-strategy.md`](ai-gateway-strategy.md) §4.2.7 | Task `generate_notification_content`. |
| [`anti-fraud-system.md`](anti-fraud-system.md) | Lee `user_restrictions` antes de enviar. |
| [`../product/motivation-and-achievements.md`](../product/motivation-and-achievements.md) | Emite `achievement.unlocked` que dispara push. |
| [`../product/push-notifications-copy-bank.md`](../product/push-notifications-copy-bank.md) | Banco autoritativo de copys literales para los 15 notification_ids + 80 logros. |
| [`../product/pedagogical-system.md`](../product/pedagogical-system.md) | `block.struggling_detected` → notificación empática. |
| [`sparks-system.md`](sparks-system.md) | `sparks.balance_low` y `sparks.depleted` → push. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.6 | Eventos `notification.*`. |
| [`../cross-cutting/security-threat-model.md`](../cross-cutting/security-threat-model.md) §3.8 | Amenazas de notifications. |

---

*Documento vivo. Actualizar cuando se agreguen tipos de notificaciones,
cambien proveedores, o se agreguen canales nuevos post-MVP.*

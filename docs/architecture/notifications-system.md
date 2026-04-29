# Notifications System (MVP)

> Sistema de notificaciones para el MVP basado en Firebase Cloud Messaging.
> Cubre push notifications a iOS y Android con orquestación desde
> Cloudflare Workers. WhatsApp queda como opción futura post-validación.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Alcance:** MVP (meses 0-6)

---

## 1. Objetivo

Sostener la retención del producto durante el MVP mediante notificaciones
push efectivas que mantengan al usuario practicando con consistencia.

El aprendizaje de idiomas vive de la consistencia diaria. Sin notificaciones,
los usuarios olvidan practicar y abandonan en pocos días. Con notificaciones
bien diseñadas, se construye el hábito que el producto necesita para
demostrar valor.

---

## 2. Alcance del MVP

### 2.1 Lo que incluye

- Push notifications nativas a iOS y Android via Firebase Cloud Messaging.
- Recordatorios diarios personalizados por hora preferida del usuario.
- Notificaciones de eventos clave (logros, streaks, sistema).
- Re-engagement de usuarios inactivos.
- Preferencias granulares por tipo de notificación.
- Personalización del contenido con IA en batch nocturno.

### 2.2 Lo que NO incluye en MVP

- WhatsApp Business API (evaluado para post-MVP, sección 13).
- Email transaccional masivo (uso mínimo, solo recibos via Stripe/RevenueCat).
- SMS.
- Notificaciones in-app sofisticadas (uso simple, mensaje básico al abrir).

### 2.3 Por qué solo Firebase para MVP

- **Simplicidad operativa:** un solo servicio para gestionar.
- **Costo cero:** FCM es gratis e ilimitado.
- **Cobertura completa:** push cubre el caso de uso primario (recordatorios
  diarios) que es el más impactante para retención.
- **Foco:** sin distracciones de plantillas WhatsApp, costos por mensaje,
  o aprobaciones de Meta durante validación del producto.

---

## 3. Stack tecnológico

### 3.1 Componentes

| Componente | Servicio | Rol |
|-----------|----------|-----|
| Push delivery | Firebase Cloud Messaging | Envío real a dispositivos iOS y Android |
| SDK cliente | `@react-native-firebase/messaging` | Recepción y manejo de notificaciones en la app |
| Orquestación | Cloudflare Workers | Lógica de cuándo y a quién enviar |
| Scheduling | Cloudflare Cron Triggers | Disparos programados (diarios, semanales) |
| Estado de rate limits | Cloudflare KV o Durable Objects | Anti-spam por usuario |
| Persistencia | Postgres (Supabase) | Tokens, preferencias, historial |
| Personalización de contenido | AI Gateway (existente) | Generación batch nocturna |

### 3.2 Por qué FCM directo y no Expo Push Service

Aunque la app se construye con Expo + React Native, se opta por FCM directo
en lugar del wrapper de Expo Push Service por:

- **Control total:** acceso a todas las features de FCM (topics, conditions,
  analytics, A/B testing).
- **Sin intermediarios:** envío directo desde Cloudflare Workers a Google.
- **Mejor analytics:** Firebase Analytics integra automáticamente métricas
  de notificaciones.
- **Topics y segmentación nativa:** FCM permite enviar a grupos sin
  necesidad de iterar usuario por usuario.
- **Compatible con futuro:** si más adelante se migra fuera de Expo, FCM
  ya está configurado.

### 3.3 Configuración inicial necesaria

- Crear proyecto en Firebase Console.
- Configurar app iOS: subir certificado APNs (.p8) a Firebase.
- Configurar app Android: descargar `google-services.json`.
- Instalar SDK en la app: `@react-native-firebase/app` y
  `@react-native-firebase/messaging`.
- En Expo: usar `expo-modules` con `react-native-firebase` (requiere
  development build, no funciona con Expo Go).
- Generar service account en Google Cloud Console para el Worker.

---

## 4. Tipos de notificaciones del MVP

### 4.1 Catálogo

| ID | Tipo | Frecuencia | Trigger | Personalizada |
|----|------|------------|---------|---------------|
| `daily_reminder` | Recordatorio | 1/día a hora preferida | Cron | Sí |
| `streak_at_risk` | Alerta | 1/día si aplica | Cron | Sí |
| `level_completed` | Logro | Por evento | Domain event | No |
| `welcome_d1` | Onboarding | 1 vez | 24h post-registro | No |
| `welcome_d3` | Onboarding | 1 vez | 72h post-registro | Sí |
| `inactivity_d3` | Re-engagement | 1 vez | 72h sin actividad | Sí |
| `inactivity_d7` | Re-engagement | 1 vez | 7 días sin actividad | Sí |
| `inactivity_d14` | Re-engagement | 1 vez | 14 días sin actividad | Sí |
| `low_sparks` | Transactional | Por evento | Balance bajo | No |
| `pack_expiring` | Transactional | Por evento | Pack expira en 7 días | No |
| `payment_failed` | Transactional | Por evento | Pago rechazado | No |

### 4.2 Daily reminder: el corazón del sistema

El daily reminder es la notificación más importante. Su efectividad
determina la retención del producto.

**Características:**
- Se envía a la hora que el usuario suele practicar (configurable, default 7 PM).
- Se envía en su zona horaria.
- NO se envía si el usuario ya practicó hoy.
- Contenido personalizado al foco pedagógico de hoy.
- Menciona la racha actual si supera 3 días.

**Ejemplos de contenido (generados por IA en batch nocturno):**

```
Título: "Tu plan te espera, Juan"
Cuerpo: "Hoy practicamos pronunciación de /θ/. Solo 12 minutos para mantener tu racha de 8 días."
```

```
Título: "12 minutos de inglés, María"
Cuerpo: "Hoy: roleplay de entrevista técnica. Tu próxima entrevista se acerca."
```

### 4.3 Streak at risk

Si el usuario tiene una racha activa y faltan pocas horas para perderla
(no practicó aún ese día), se envía una alerta.

Reglas:
- Solo si la racha actual es ≥ 5 días (sin urgencia para rachas cortas).
- Se envía 4 horas antes del corte (medianoche en su tz).
- Solo si el usuario tiene `reminders_enabled = true`.
- Se envía como máximo 1 vez por día (no spam).

### 4.4 Re-engagement (d3, d7, d14)

Tres niveles de re-engagement con tono y contenido distinto:

- **D3:** suave, motivador. "Tu plan te está esperando."
- **D7:** más personal, mostrar lo que se está perdiendo. "Mejorabas X. Volvé."
- **D14:** último intento, posible incentivo (Sparks bonus por volver).

Después del D14, no se envían más. Si el usuario no volvió, se considera
churned y no tiene sentido seguir gastando notificaciones.

### 4.5 Transaccionales

Notificaciones críticas que se envían independiente de preferencias de
"engagement" (siempre que el usuario tenga push habilitado):

- `low_sparks`: cuando balance < 20% del plan mensual.
- `pack_expiring`: 7 días antes de expiración.
- `payment_failed`: inmediato, con CTA para corregir método de pago.

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
  created_at      TIMESTAMPTZ DEFAULT now(),
  last_used_at    TIMESTAMPTZ DEFAULT now(),
  is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE INDEX idx_fcm_tokens_user ON user_fcm_tokens(user_id) WHERE is_active = true;
```

Notas:
- Un usuario puede tener varios tokens (varios devices).
- Los tokens se marcan inactivos cuando FCM devuelve error de invalid token.
- Se limpian periódicamente (90 días sin uso).

### 5.2 Preferencias del usuario

```sql
CREATE TABLE user_notification_preferences (
  user_id           UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,

  -- Permiso global
  push_enabled      BOOLEAN NOT NULL DEFAULT true,

  -- Por categoría
  reminders_enabled       BOOLEAN NOT NULL DEFAULT true,
  achievements_enabled    BOOLEAN NOT NULL DEFAULT true,
  reengagement_enabled    BOOLEAN NOT NULL DEFAULT true,
  transactional_enabled   BOOLEAN NOT NULL DEFAULT true,

  -- Hora preferida y timezone
  preferred_reminder_hour INT NOT NULL DEFAULT 19,
  timezone                TEXT NOT NULL DEFAULT 'America/Mexico_City',

  updated_at              TIMESTAMPTZ DEFAULT now()
);
```

Las transactional siempre son `true` por default y solo se deshabilitan
si el usuario explícitamente lo pide. Avisos de pago fallido son críticos.

### 5.3 Historial de notificaciones enviadas

```sql
CREATE TABLE notifications_log (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  notification_id TEXT NOT NULL,        -- ej: 'daily_reminder'
  category        TEXT NOT NULL,
  title           TEXT NOT NULL,
  body            TEXT NOT NULL,
  data            JSONB DEFAULT '{}',
  fcm_message_id  TEXT,                  -- response de FCM
  sent_at         TIMESTAMPTZ DEFAULT now(),
  delivered_at    TIMESTAMPTZ,
  opened_at       TIMESTAMPTZ,
  error           TEXT,                  -- si falló el envío
  CONSTRAINT valid_status CHECK (
    error IS NULL OR (delivered_at IS NULL AND opened_at IS NULL)
  )
);

CREATE INDEX idx_notif_log_user_date ON notifications_log(user_id, sent_at DESC);
CREATE INDEX idx_notif_log_type ON notifications_log(notification_id, sent_at DESC);
```

### 5.4 Contenido personalizado pre-generado

```sql
CREATE TABLE notifications_scheduled (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  notification_id TEXT NOT NULL,
  scheduled_for   TIMESTAMPTZ NOT NULL,
  title           TEXT NOT NULL,
  body            TEXT NOT NULL,
  data            JSONB DEFAULT '{}',
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending', 'sent', 'cancelled', 'failed')),
  generated_by    TEXT NOT NULL DEFAULT 'system'  -- 'ai' | 'system'
);

CREATE INDEX idx_scheduled_pending ON notifications_scheduled(scheduled_for)
  WHERE status = 'pending';
```

El job nocturno popula esta tabla con el contenido del día siguiente.
Los Cron Triggers leen de aquí cuando es hora de enviar.

---

## 6. Flujo del cliente (React Native + Expo)

### 6.1 Setup inicial

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

  // 3. Registrar token en backend
  await api.post('/notifications/register-token', {
    token,
    platform: Platform.OS,
    device_id: await getDeviceId(),
    app_version: Application.nativeApplicationVersion
  });

  // 4. Listener para refresh de token
  messaging().onTokenRefresh(async (newToken) => {
    await api.post('/notifications/register-token', { token: newToken });
  });
}
```

### 6.2 Manejo de notificaciones recibidas

```typescript
// Notificación con app en background
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  // FCM ya muestra la notificación automáticamente
  // Aquí solo logging o pre-fetch de data
  await trackNotificationReceived(remoteMessage);
});

// Notificación con app abierta
messaging().onMessage(async (remoteMessage) => {
  // Mostrar notificación in-app custom
  showInAppNotification(remoteMessage);
});

// Usuario abre la app desde una notificación
messaging().onNotificationOpenedApp((remoteMessage) => {
  // Navegar a la pantalla relevante según data
  if (remoteMessage.data?.deeplink) {
    navigate(remoteMessage.data.deeplink);
  }
  trackNotificationOpened(remoteMessage);
});
```

### 6.3 UI de preferencias

Pantalla de settings que permite:

- Toggle global: push notifications on/off.
- Toggles por categoría: reminders, achievements, reengagement.
- Selector de hora preferida (slider o picker).
- Indicación clara: las transaccionales siempre llegan.
- Link a settings del sistema operativo si el permiso global está negado.

---

## 7. Orquestador en Cloudflare Workers

### 7.1 Estructura de archivos

```
workers/notifications/
├── src/
│   ├── index.ts                    # Worker principal con routes
│   ├── schedulers/
│   │   ├── daily-reminders.ts      # Cron handler
│   │   ├── streak-checker.ts       # Cron handler
│   │   ├── inactivity-detector.ts  # Cron handler
│   │   └── scheduled-dispatcher.ts # Lee notifications_scheduled
│   ├── senders/
│   │   └── fcm-sender.ts           # Llama FCM API
│   ├── triggers/
│   │   ├── on-level-complete.ts    # Domain event handler
│   │   ├── on-payment-failed.ts
│   │   └── on-low-sparks.ts
│   ├── utils/
│   │   ├── rate-limiter.ts         # Durable Object
│   │   ├── timezone.ts
│   │   └── fcm-auth.ts             # Generación de access tokens
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
database_id = "..."

[[kv_namespaces]]
binding = "RATE_LIMITS"
id = "..."

[[durable_objects.bindings]]
name = "USER_RATE_LIMITER"
class_name = "UserRateLimiter"

[triggers]
crons = [
  "*/15 * * * *",       # Cada 15 min: dispatch de notifications_scheduled
  "0 * * * *",          # Cada hora: chequeo de daily reminders por timezone
  "0 20 * * *",         # 8 PM UTC: detección de streaks en peligro
  "0 1 * * *",          # 1 AM UTC: detección de inactividad
  "0 3 * * 0"           # Domingos 3 AM UTC: cleanup de tokens viejos
]

[vars]
FIREBASE_PROJECT_ID = "your-project-id"

# Secrets se setean con: wrangler secret put FIREBASE_SERVICE_ACCOUNT
```

### 7.3 Handler principal de cron

```typescript
// src/index.ts
export default {
  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext) {
    const cronPattern = event.cron;

    switch (cronPattern) {
      case '*/15 * * * *':
        await dispatchScheduledNotifications(env);
        break;
      case '0 * * * *':
        await checkDailyReminders(env);
        break;
      case '0 20 * * *':
        await detectStreaksAtRisk(env);
        break;
      case '0 1 * * *':
        await detectInactivity(env);
        break;
      case '0 3 * * 0':
        await cleanupOldTokens(env);
        break;
    }
  },

  async fetch(request: Request, env: Env): Promise<Response> {
    // Endpoints HTTP para domain events (level_complete, etc.)
    return await handleDomainEvent(request, env);
  }
};
```

### 7.4 Detección de daily reminders por timezone

```typescript
async function checkDailyReminders(env: Env) {
  const currentUtcHour = new Date().getUTCHours();

  // Query: usuarios cuya hora preferida coincide con UTC actual
  // según su timezone, que no han practicado hoy
  const users = await env.DB.prepare(`
    SELECT
      u.id,
      u.name,
      np.preferred_reminder_hour,
      np.timezone
    FROM users u
    JOIN user_notification_preferences np ON np.user_id = u.id
    WHERE np.push_enabled = true
      AND np.reminders_enabled = true
      AND user_local_hour(?, np.timezone) = np.preferred_reminder_hour
      AND NOT EXISTS (
        SELECT 1 FROM user_sessions s
        WHERE s.user_id = u.id
          AND s.started_at > date_trunc('day', now() AT TIME ZONE np.timezone)
      )
  `).bind(currentUtcHour).all();

  for (const user of users.results) {
    await dispatchDailyReminder(user, env);
  }
}
```

### 7.5 Envío via FCM HTTP v1 API

```typescript
// src/senders/fcm-sender.ts
import { generateGoogleAccessToken } from '../utils/fcm-auth';

export async function sendFcmNotification(
  job: NotificationJob,
  env: Env
): Promise<FcmResult> {
  const accessToken = await generateGoogleAccessToken(env);

  const fcmEndpoint =
    `https://fcm.googleapis.com/v1/projects/${env.FIREBASE_PROJECT_ID}/messages:send`;

  // Obtener tokens activos del usuario
  const tokens = await getActiveTokens(job.userId, env);

  const results = await Promise.allSettled(tokens.map(token =>
    fetch(fcmEndpoint, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        message: {
          token: token.token,
          notification: {
            title: job.title,
            body: job.body
          },
          data: job.data,
          android: {
            priority: 'high',
            notification: {
              sound: 'default',
              channel_id: job.category
            }
          },
          apns: {
            payload: {
              aps: {
                sound: 'default',
                badge: 1
              }
            }
          }
        }
      })
    }).then(async (res) => ({
      token: token.token,
      success: res.ok,
      response: await res.json()
    }))
  ));

  // Manejar tokens inválidos
  for (const result of results) {
    if (result.status === 'fulfilled' && !result.value.success) {
      const errorCode = result.value.response?.error?.details?.[0]?.errorCode;
      if (errorCode === 'UNREGISTERED' || errorCode === 'INVALID_ARGUMENT') {
        await markTokenInactive(result.value.token, env);
      }
    }
  }

  // Logging
  await logNotification(job, results, env);

  return aggregateResults(results);
}
```

### 7.6 Generación del access token de Google

FCM requiere OAuth 2.0 access tokens, no API keys. Se generan firmando
un JWT con la service account.

```typescript
// src/utils/fcm-auth.ts
export async function generateGoogleAccessToken(env: Env): Promise<string> {
  // Cache del token (válido 1h)
  const cached = await env.RATE_LIMITS.get('fcm_access_token');
  if (cached) return cached;

  const serviceAccount = JSON.parse(env.FIREBASE_SERVICE_ACCOUNT);

  // Crear JWT firmado
  const jwt = await createSignedJwt({
    iss: serviceAccount.client_email,
    scope: 'https://www.googleapis.com/auth/firebase.messaging',
    aud: 'https://oauth2.googleapis.com/token',
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + 3600
  }, serviceAccount.private_key);

  // Intercambiar JWT por access token
  const response = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion=${jwt}`
  });

  const { access_token } = await response.json();

  // Cachear por 50 min (margen de 10 min antes de expirar)
  await env.RATE_LIMITS.put('fcm_access_token', access_token, {
    expirationTtl: 3000
  });

  return access_token;
}
```

---

## 8. Personalización con IA

### 8.1 Generación batch nocturna

El job nocturno (existente para análisis del usuario) genera el contenido
personalizado de notificaciones del día siguiente.

Flujo:

```
1. Job nocturno corre a las 2 AM en cada timezone relevante.
2. Para cada usuario activo:
   a. Recopila datos: nombre, racha, foco pedagógico de mañana, logros recientes.
   b. Llama al AI Gateway con tarea 'generate_notifications'.
   c. Recibe contenido para daily_reminder y streak_at_risk si aplica.
   d. Persiste en notifications_scheduled con scheduled_for = mañana a hora preferida.
3. El cron de cada hora dispatcha las notificaciones cuyo scheduled_for matchee.
```

### 8.2 Prompt template

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

1. daily_reminder (push, máximo 50 caracteres en título, 90 en cuerpo):
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

Aproximadamente $0.0005 USD por usuario por día. A 10.000 usuarios activos,
$5 USD/día = $150 USD/mes. Despreciable comparado con el impacto en
retención.

---

## 9. Rate limiting y anti-spam

### 9.1 Límites por usuario

| Categoría | Max por día | Max por semana |
|-----------|------------|----------------|
| Reminders | 1 | 7 |
| Achievements | 3 | 15 |
| Reengagement | 1 | 3 |
| Transactional | 5 | 20 |

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

    const todayCount = (await this.state.storage.get<number>(todayKey)) || 0;
    const weekCount = (await this.state.storage.get<number>(weekKey)) || 0;

    const limits = RATE_LIMITS[category];
    if (todayCount >= limits.maxPerDay || weekCount >= limits.maxPerWeek) {
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
- Desactiva push en settings del sistema operativo.
- No abre la app durante 30+ días.

Se aplica circuit breaker: pause de notificaciones por 7-30 días según
caso. Esto previene seguir gastando recursos en usuarios que ya no quieren
recibir o no van a volver.

---

## 10. Observabilidad

### 10.1 Métricas críticas

- **Delivery rate:** % de notificaciones que FCM entrega vs intenta enviar.
- **Open rate por tipo:** % de notificaciones que el usuario abre.
- **Click-through rate:** % que generan apertura de la app.
- **Conversion rate:** % que generan acción objetivo (práctica, compra).
- **Opt-out rate por tipo:** % que desactivan después de recibir.

### 10.2 Targets esperados (benchmarks de la industria)

| Tipo | Open rate target | CTR target |
|------|------------------|------------|
| Daily reminder | 8-15% | 5-10% |
| Streak at risk | 15-25% | 12-20% |
| Achievement | 20-30% | 15-25% |
| Inactivity | 3-8% | 2-5% |

### 10.3 Stack de observabilidad

- **Firebase Analytics:** integrado nativamente con FCM, métricas de delivery
  y open automáticas.
- **PostHog:** eventos custom de conversion (qué pasa después de abrir).
- **Cloudflare Logs:** logs del Worker con costos y errores.
- **Tabla notifications_log:** auditoría completa con joins a métricas de
  producto.

### 10.4 Alertas críticas

- Delivery rate < 90% (problema con FCM o tokens).
- Open rate de daily_reminder cae > 30% vs baseline.
- Spike en opt-outs.
- Errores de envío > 5%.

---

## 11. Plan de implementación

### 11.1 Sprint 1 (semana 1): Setup foundational

- Crear proyecto Firebase y configurar iOS/Android.
- Integrar SDK en la app (development build con react-native-firebase).
- Endpoint en backend para registrar tokens.
- Tabla `user_fcm_tokens` y `user_notification_preferences`.
- UI básica de preferencias.

### 11.2 Sprint 2 (semana 2): Worker y envío básico

- Setup de Cloudflare Worker con secrets de Firebase.
- Generación de access tokens y cache.
- Función de envío FCM con manejo de errores.
- Endpoint HTTP para test manual.

### 11.3 Sprint 3 (semana 3): Cron triggers

- Cron de daily reminders timezone-aware.
- Cron de detección de inactividad.
- Cron de cleanup de tokens viejos.
- Tabla `notifications_log` y logging.

### 11.4 Sprint 4 (semana 4): Personalización con IA

- Tabla `notifications_scheduled`.
- Integración con AI Gateway para batch nocturno.
- Cron dispatcher de notifications_scheduled.

### 11.5 Sprint 5 (semana 5): Eventos y rate limiting

- Domain events handlers (level_complete, payment_failed, etc.).
- Durable Object UserRateLimiter.
- Circuit breaker logic.

### 11.6 Sprint 6 (semana 6): Polish y observabilidad

- Firebase Analytics integration.
- Eventos a PostHog.
- Dashboard interno.
- Tests de integración.

---

## 12. Costos estimados

### 12.1 MVP (10.000 usuarios activos)

| Componente | Costo mensual |
|-----------|---------------|
| FCM | $0 (gratis ilimitado) |
| Cloudflare Workers | $5 (plan paid para Cron) |
| Cloudflare KV | $0 (tier gratuito) |
| Cloudflare Durable Objects | $5 |
| AI Gateway (personalización) | ~$150 |
| **Total** | **~$160/mes** |

### 12.2 A escala (100.000 usuarios activos)

| Componente | Costo mensual |
|-----------|---------------|
| FCM | $0 |
| Cloudflare Workers | $20 |
| Cloudflare KV + DO | $50 |
| AI Gateway | ~$1.500 |
| **Total** | **~$1.570/mes** |

El costo escala muy bien gracias a que FCM es gratis. El driver principal
es la personalización IA, que aporta valor directo a retención.

---

## 13. Roadmap post-MVP

### 13.1 Cuándo evaluar agregar WhatsApp

Considerar agregar WhatsApp Business API cuando se cumplan **todos**:

- [ ] Producto validado: pago recurrente, retention D30 > 30%.
- [ ] Volumen suficiente: > 5.000 usuarios activos.
- [ ] Push notifications con open rate < 10% (signal que push no es suficiente).
- [ ] Margen del producto soporta el costo adicional ($1-3 USD/usuario/mes
  en WhatsApp).

### 13.2 Casos de uso prioritarios para WhatsApp (cuando se agregue)

En orden de prioridad:

1. **Re-engagement de usuarios inactivos d7+**: donde push ya falló,
   WhatsApp tiene 80%+ open rate.
2. **Concierge para Pro/Premium**: feature diferenciadora "tu tutor por
   WhatsApp".
3. **Recuperación de churn**: usuarios que cancelaron suscripción.
4. **Avisos críticos con CTA inmediato**: payment failed con link de pago.

### 13.3 Otros canales a considerar post-MVP

- **Email transaccional masivo:** Resend para reportes semanales detallados,
  newsletters educativos.
- **In-app messaging avanzado:** Firebase In-App Messaging para anuncios
  contextuales.
- **Web push:** cuando se lance la web app (mes 9+).

### 13.4 Migraciones técnicas potenciales

- **FCM HTTP v1 → FCM via Firebase Admin SDK directo:** si se migra el
  backend a Node.js servers en lugar de Workers (escala muy grande).
- **Cloudflare Cron Triggers → orquestador dedicado** (Inngest, Trigger.dev):
  si la lógica de scheduling se vuelve compleja con muchos workflows.

---

## 14. Decisiones abiertas

- [ ] ¿Permitir snooze de notificaciones por 1 hora desde la propia notif?
- [ ] ¿Smart timing: aprender cuándo cada usuario suele abrir vs hora preferida declarada?
- [ ] ¿Notificaciones grupales (múltiples eventos en un solo mensaje)?
- [ ] ¿Rich notifications con imágenes/audio embebido?
- [ ] ¿A/B test de variantes de mensajes con Firebase Remote Config?

---

## 15. Referencias internas

- `docs/business/plan_de_negocio.docx` — Plan de negocio.
- `docs/product/ai-roadmap-system.md` — Sistema de roadmap.
- `docs/architecture/ai-gateway-strategy.md` — AI Gateway (provee personalización).
- `docs/architecture/platform-strategy.md` — Estrategia de plataformas.
- `docs/architecture/sparks-system.md` — Sistema de Sparks.
- `docs/decisions/ADR-004-notifications-fcm.md` — ADR de elección FCM (a crear).

---

*Documento vivo. Actualizar cuando cambien tipos de notificaciones,
proveedores o se agreguen canales nuevos post-MVP.*

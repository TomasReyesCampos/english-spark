# Data and Events

> Modelo unificado de eventos cross-system, taxonomía de tracking,
> políticas de retención, contratos de eventos. Documento autoritativo
> para cualquier emisión o consumo de eventos en el sistema.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador. Antes de emitir o
consumir un evento, leé §3 (envelope) y §5 (catálogo). No inventar
nombres ni shapes.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 establece principios (qué es un evento, qué NO).
- §2 distingue **3 tipos de streams** del sistema: domain events, product
  events, internal logs.
- §3 define el **envelope** común para domain events.
- §4 define **convenciones** (naming, idempotency, ordering).
- §5 contiene el **catálogo completo** de domain events del MVP.
- §6 lista los **product events** (PostHog) — distintos de los domain
  events.
- §7 cubre **retención** y privacidad de datos.
- §8 define **versionado** de eventos.

---

## 1. Principios

### 1.1 Eventos son la vía de notificación cross-system

Cuando el sistema A necesita avisar al sistema B que algo pasó (sin
esperar respuesta síncrona), emite un domain event. B se suscribe.

APIs son la vía para queries y comandos síncronos (necesito una
respuesta ahora). Eventos son para "ya pasó esto, haz lo que tengas
que hacer".

### 1.2 Events vs commands

- **Domain event:** "ya pasó X". Past tense. Muchos consumidores
  posibles. Idempotent. (`block.completed`)
- **Command:** "haz Y". Imperative. Un solo destinatario. Puede fallar.
  (no usamos en este sistema; las commands son llamadas RPC).

### 1.3 Loose coupling

El emisor **no sabe ni le importa** quién consume sus eventos. Un
sistema puede empezar a consumir un evento existente sin que el emisor
cambie.

Implicación: **NO** poner lógica condicional del estilo "si X consume
este evento, también incluyo este campo". El payload incluye lo que
es información esencial sobre el hecho, no lo que conviene para un
consumidor específico.

### 1.4 Idempotencia obligatoria

Consumidores deben tolerar recibir el mismo evento (mismo `event_id`)
múltiples veces sin causar efectos duplicados. Inngest puede reenviar
eventos en caso de fallo del consumidor.

### 1.5 Una sola fuente de verdad

Un evento tiene **un solo emisor**. Si dos sistemas pueden emitir el
"mismo" evento conceptualmente, hay un bug de boundaries: uno es el
dueño del fact, el otro debería consumir.

---

## 2. Streams del sistema

Tres flujos paralelos de eventos, con propósitos distintos.

### 2.1 Domain events (Inngest)

**Para qué:** comunicación entre sistemas. Reaccionar a cambios de
estado.

**Bus:** Inngest events (`inngest.send()`).

**Shape:** envelope de §3.

**Retention:** 90 días en Inngest dashboard, 12 meses en `event_log`
(tabla Postgres).

**Ejemplo:** `block.mastered` → AI Roadmap regenera plan + Motivation
verifica logros.

### 2.2 Product events (PostHog)

**Para qué:** analytics, funnels, retention, A/B testing.

**Bus:** PostHog SDK (`posthog.capture()`).

**Shape:** evento PostHog estándar (`event_name` + `properties`).

**Retention:** 12 meses en PostHog (tier que se contrate).

**Ejemplo:** `paywall_viewed` con properties `{ context, plan_shown,
country }`.

### 2.3 Internal logs (Sentry + Cloudflare Logs)

**Para qué:** debugging, errors, performance.

**Bus:** Sentry SDK (`captureException`, `captureMessage`),
`console.log` en Workers.

**Retention:** 30 días Sentry, 7 días Cloudflare Logs.

**Ejemplo:** `error: AI gateway timeout for task=score_pronunciation
user_hash=...`.

### 2.4 Cuándo usar cuál

| Caso | Stream |
|------|--------|
| "Otro sistema necesita reaccionar" | Domain event |
| "Quiero analizar cohort retention" | Product event |
| "Quiero saber por qué falló esto" | Internal log |
| "Necesito ambos" | Emitir domain event Y product event (intencional) |

**Regla práctica:** un mismo "hecho" puede generar 1 domain event + 1
product event + 1 log entry. Por ejemplo, completar un bloque genera:

- Domain: `block.completed` (Inngest, para AI Roadmap + Motivation).
- Product: `block_completed` (PostHog, para funnel).
- Log: ninguno necesario salvo error.

NO duplicar el mismo evento en dos buses con propósitos solapados.

---

## 3. Envelope de domain events

```typescript
interface DomainEvent<TPayload = unknown> {
  // Identidad del evento
  event_name: string;             // "<entity>.<action_past_tense>"
  event_id: string;               // UUID único por instancia
  event_version: string;          // semver del shape, ej: "1.0.0"

  // Origen
  emitter_system: SystemId;       // 'auth' | 'sparks' | 'pedagogical' | ...
  emitted_at: string;             // ISO-8601 timestamp con milisegundos

  // Contexto
  user_id_hash?: string;          // SHA-256 del UUID del usuario
  trace_id?: string;              // para correlacionar cross-system
  parent_event_id?: string;       // si este evento fue causado por otro

  // Payload específico
  payload: TPayload;
}

type SystemId =
  | 'auth' | 'sparks' | 'pedagogical' | 'ai-roadmap'
  | 'student-profile' | 'motivation' | 'content-creation'
  | 'notifications' | 'anti-fraud' | 'customer-support'
  | 'ai-gateway' | 'metrics' | 'webhook';
```

### 3.1 Reglas del envelope

- `event_name`: minúsculas, snake_case, formato `<entity>.<verb_past>`.
- `event_id`: generado al emitir, único en todo el sistema. Consumidores
  lo usan como idempotency key.
- `event_version`: cambia cuando el `payload` rompe compatibilidad.
- `emitter_system`: literal de `SystemId`. No "string libre".
- `emitted_at`: ISO con millisegundos (`.SSSZ`). El consumidor no debe
  asumir orden por `emitted_at` entre sistemas distintos (clock skew).
- `user_id_hash`: SHA-256 hex del `users.id` (UUID). Nunca el UUID
  crudo en eventos que vayan a sinks externos. Si un consumidor interno
  necesita el UUID, lo deriva del hash usando lookup en `users`
  (porque emisor y consumidor están en el mismo cluster).
- `trace_id`: heredado del request inicial si existe; sino generado por
  el emisor. Permite seguir un journey cross-system.
- `parent_event_id`: si este evento fue triggereado por otro, anotarlo
  para debugging de cadenas de eventos.

### 3.2 Wrapper helper

```typescript
function emitDomainEvent<TPayload>(
  eventName: string,
  payload: TPayload,
  context: {
    userId?: string;
    traceId?: string;
    parentEventId?: string;
  }
): Promise<void> {
  const event: DomainEvent<TPayload> = {
    event_name: eventName,
    event_id: crypto.randomUUID(),
    event_version: getEventVersion(eventName),
    emitter_system: getCurrentSystemId(),
    emitted_at: new Date().toISOString(),
    user_id_hash: context.userId ? sha256(context.userId) : undefined,
    trace_id: context.traceId,
    parent_event_id: context.parentEventId,
    payload,
  };

  // Persistir en event_log (Postgres) para auditoría
  await persistEventLog(event);

  // Enviar a Inngest para distribución
  await inngest.send({
    name: eventName,
    data: event,
  });
}
```

### 3.3 Tabla `event_log`

```sql
CREATE TABLE event_log (
  event_id          UUID PRIMARY KEY,
  event_name        TEXT NOT NULL,
  event_version     TEXT NOT NULL,
  emitter_system    TEXT NOT NULL,
  emitted_at        TIMESTAMPTZ NOT NULL,
  user_id_hash      TEXT,
  trace_id          TEXT,
  parent_event_id   UUID,
  payload           JSONB NOT NULL
);

CREATE INDEX idx_event_log_name_date
  ON event_log(event_name, emitted_at DESC);
CREATE INDEX idx_event_log_user_hash
  ON event_log(user_id_hash, emitted_at DESC)
  WHERE user_id_hash IS NOT NULL;
CREATE INDEX idx_event_log_trace
  ON event_log(trace_id) WHERE trace_id IS NOT NULL;

-- Retention 12 meses
CREATE INDEX idx_event_log_retention ON event_log(emitted_at);
```

### 3.4 Tabla `event_log_dlq` (dead letter queue)

```sql
CREATE TABLE event_log_dlq (
  event_id          UUID PRIMARY KEY,
  consumer_system   TEXT NOT NULL,
  failed_at         TIMESTAMPTZ NOT NULL,
  failure_count     INT NOT NULL,
  last_error        TEXT NOT NULL,
  payload           JSONB NOT NULL,
  resolved          BOOLEAN NOT NULL DEFAULT false,
  resolved_at       TIMESTAMPTZ,
  resolution_notes  TEXT
);

CREATE INDEX idx_dlq_unresolved
  ON event_log_dlq(failed_at DESC) WHERE resolved = false;
```

Eventos que fallan 3 veces consecutivas en un consumidor van al DLQ
para análisis manual.

---

## 4. Convenciones

### 4.1 Naming

- `<entity>.<action_past_tense>`. Ejemplos:
  - ✅ `user.signed_up`, `block.completed`, `payment.succeeded`
  - ❌ `signing_up_user`, `block_complete`, `payments`
- Entity es el sustantivo principal del dominio. Action es el verbo en
  pasado.
- Si la entity tiene sub-entity, usar punto: `block.attempt.started`
  solo si el evento es realmente sobre la attempt, no sobre el block.
  Generalmente preferir `attempt.started`.

### 4.2 Versionado

Seguir semver:
- **Major (1.x.x → 2.0.0):** breaking change en payload. Crear evento
  nuevo `<name>_v2` o coexistir versiones (ver §8).
- **Minor (1.0.x → 1.1.0):** campo nuevo opcional. Compatibles atrás.
- **Patch (1.0.0 → 1.0.1):** clarificación de docs sin cambio de shape.

Versión actual de cada evento se persiste en `events_registry` (§8).

### 4.3 Idempotency

```typescript
// Patrón que TODO consumer debe usar
async function consumeBlockCompleted(event: DomainEvent<BlockCompletedPayload>) {
  // 1. Verificar si ya procesamos este event_id
  const alreadyProcessed = await db.query(`
    SELECT 1 FROM consumed_events
    WHERE consumer_id = $1 AND event_id = $2
  `, [CONSUMER_ID, event.event_id]);

  if (alreadyProcessed) return; // skip silently

  // 2. Procesar dentro de una transacción
  await db.transaction(async (tx) => {
    await doBusinessLogic(event.payload, tx);
    await tx.query(`
      INSERT INTO consumed_events (consumer_id, event_id, consumed_at)
      VALUES ($1, $2, now())
    `, [CONSUMER_ID, event.event_id]);
  });
}
```

```sql
CREATE TABLE consumed_events (
  consumer_id   TEXT NOT NULL,         -- ej: 'motivation.achievement_engine'
  event_id      UUID NOT NULL,
  consumed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer_id, event_id)
);

-- Retention 90 días (suficiente para detectar duplicados)
CREATE INDEX idx_consumed_events_retention ON consumed_events(consumed_at);
```

### 4.4 Ordering

**No asumir orden** entre eventos de sistemas distintos. Inngest no
garantiza FIFO global.

Si un consumidor necesita orden, usar `emitted_at` para reordenar
internamente o forzar una API síncrona en lugar de evento.

### 4.5 Retries y DLQ

- Inngest reintenta automáticamente con backoff exponencial: 1s, 5s,
  30s, 5min.
- Tras 3 fallos consecutivos, evento va a `event_log_dlq` y se emite
  alerta a Sentry.
- Manual replay desde DLQ es soportado (admin script).

### 4.6 Sin PII en payload

Nunca incluir en `payload`:
- Email, nombre, teléfono.
- Audio crudo o transcripciones completas.
- Información de pago (números de tarjeta, billing addresses).

Usar `user_id_hash` y referencias por ID. Si un consumidor necesita el
email del usuario, lo busca vía API `getProfile`.

---

## 5. Catálogo completo de domain events

### 5.1 Auth

```typescript
// user.signed_up
interface UserSignedUpPayload {
  user_id: string;       // UUID del nuevo user (NO hasheado, evento interno)
  provider: 'google' | 'apple' | 'email' | 'phone' | 'anonymous';
  is_anonymous: boolean;
  country_code?: string;
  device_id_hash: string;
  ip_hash: string;       // SHA-256 del IP completo
}

// user.deleted (hard delete tras 30d de gracia)
interface UserDeletedPayload {
  user_id: string;
  deletion_requested_at: string;
}

// user.email_verified
interface UserEmailVerifiedPayload {
  user_id: string;
  email_hash: string;
}

// user.linked_provider
interface UserLinkedProviderPayload {
  user_id: string;
  new_provider: string;
}
```

### 5.2 Student Profile

```typescript
// onboarding.completed
interface OnboardingCompletedPayload {
  user_id: string;
  primary_goals: string[];
  professional_field: string;
  daily_minutes_available: number;
  initial_test_results: {
    cefr_estimate: string;
    fluency_score: number;
    pronunciation_score: number;
    detected_error_patterns: string[];
  };
}

// assessment.completed
interface AssessmentCompletedPayload {
  user_id: string;
  measured_cefr_level: string;
  scores: MasteryDimensions;
  strongest_areas: string[];
  weakest_areas: string[];
  duration_seconds: number;
}

// profile.updated
interface ProfileUpdatedPayload {
  user_id: string;
  changed_fields: string[];   // ej: ['daily_minutes_available', 'primary_goals']
  significant_change: boolean; // si true, AI Roadmap considera regenerar
}
```

### 5.3 Pedagogical

(Ver `product/pedagogical-system.md` §12 para catálogo completo. Resumen
de los principales:)

```typescript
// exercise.attempt_started
interface ExerciseAttemptStartedPayload {
  attempt_id: string;
  user_id: string;
  block_id: string;
  exercise_id: string;
  exercise_type: ExerciseType;
}

// exercise.attempt_completed
interface ExerciseAttemptCompletedPayload {
  attempt_id: string;
  user_id: string;
  block_id: string;
  exercise_type: ExerciseType;
  scores: Partial<MasteryDimensions>;
  duration_seconds: number;
  device: 'ios' | 'android' | 'web';
}

// block.started, block.completed, block.mastered, block.tested_out
interface BlockEventPayload {
  user_id: string;
  block_id: string;
  block_version: string;
  mastery_score?: number;
  attempts_count?: number;
}

// block.struggling_detected
interface BlockStrugglingPayload {
  user_id: string;
  block_id: string;
  signals: string[];  // ej: ['low_score_consecutive', 'time_increasing']
  attempts_count: number;
}

// subskill.level_up, subskill.regressed, subskill.review_due
interface SubskillEventPayload {
  user_id: string;
  subskill_id: string;
  previous_level: MasteryLevel;
  new_level: MasteryLevel;
  score: number;
  reason?: string;
}

// user.cefr_changed
interface CefrChangedPayload {
  user_id: string;
  previous_cefr: string;
  new_cefr: string;
  trigger: 'natural_progression' | 'reassessment' | 'recalculation';
}

// user.dimension_changed
interface DimensionChangedPayload {
  user_id: string;
  dimension: 'pronunciation' | 'fluency' | 'grammar' | 'vocabulary' | 'listening';
  previous_score: number;
  new_score: number;
}
```

### 5.4 Sparks

```typescript
// sparks.balance_changed
interface SparksBalanceChangedPayload {
  user_id: string;
  previous_balance: { plan: number; pack: number; bonus: number; total: number };
  new_balance:      { plan: number; pack: number; bonus: number; total: number };
  reason: 'plan_assignment' | 'pack_purchase' | 'gamification_reward'
        | 'system_credit' | 'operation_charge' | 'refund' | 'expiration';
  amount: number;  // negativo si es descuento
  operation_id?: string;
}

// sparks.balance_low (a 20% del plan mensual)
interface SparksBalanceLowPayload {
  user_id: string;
  current_balance: number;
  plan_assigned: number;
  pct_remaining: number;
}

// sparks.depleted (a 0)
interface SparksDepletedPayload {
  user_id: string;
  attempted_operation: string;
}

// sparks.pack_expiring (7 días antes)
interface SparksPackExpiringPayload {
  user_id: string;
  pack_id: string;
  sparks_remaining: number;
  expires_at: string;
}
```

### 5.5 Payments (RevenueCat / Stripe webhooks)

```typescript
// payment.succeeded
interface PaymentSucceededPayload {
  user_id: string;
  payment_id: string;
  payment_provider: 'apple' | 'google' | 'stripe' | 'mercadopago';
  product_type: 'subscription' | 'pack';
  product_id: string;          // ej: 'plan_pro_monthly', 'pack_medium'
  amount: number;
  currency: string;
}

// payment.failed
interface PaymentFailedPayload {
  user_id: string;
  payment_provider: string;
  product_id: string;
  reason: string;
  retry_recommended: boolean;
}

// subscription.activated, subscription.cancelled, subscription.renewed
interface SubscriptionEventPayload {
  user_id: string;
  subscription_id: string;
  plan: string;
  starts_at: string;
  ends_at?: string;
  cancellation_reason?: string;
}
```

### 5.6 Notifications

```typescript
// notification.sent
interface NotificationSentPayload {
  user_id: string;
  notification_id: string;     // ej: 'daily_reminder'
  category: string;
  fcm_message_id?: string;
}

// notification.delivered
interface NotificationDeliveredPayload {
  user_id: string;
  notification_id: string;
  delivered_at: string;
}

// notification.opened
interface NotificationOpenedPayload {
  user_id: string;
  notification_id: string;
  opened_at: string;
}

// notification.preferences_changed
interface NotificationPreferencesChangedPayload {
  user_id: string;
  changed_fields: string[];
}
```

### 5.7 Motivation

```typescript
// streak.extended
interface StreakExtendedPayload {
  user_id: string;
  current_streak: number;
  is_milestone: boolean;       // 7, 14, 30, 60, 100, 365
}

// streak.broken
interface StreakBrokenPayload {
  user_id: string;
  previous_streak: number;
  freeze_used: boolean;
}

// streak.at_risk (4h antes del corte)
interface StreakAtRiskPayload {
  user_id: string;
  current_streak: number;
  hours_until_break: number;
}

// achievement.unlocked
interface AchievementUnlockedPayload {
  user_id: string;
  achievement_id: string;
  rarity: 'common' | 'rare' | 'epic' | 'legendary' | 'unique';
  sparks_awarded: number;
}

// daily_goal.met
interface DailyGoalMetPayload {
  user_id: string;
  date: string;                // YYYY-MM-DD
  minutes_practiced: number;
  goal_minutes: number;
  sparks_awarded: number;
}

// league.week_ended (post-MVP)
interface LeagueWeekEndedPayload {
  user_id: string;
  league_id: string;
  position: number;
  promoted: boolean;
  demoted: boolean;
  sparks_awarded: number;
}
```

### 5.8 AI Roadmap

```typescript
// roadmap.generated
interface RoadmapGeneratedPayload {
  user_id: string;
  roadmap_id: string;
  type: 'initial' | 'definitive' | 'regenerated';
  total_blocks: number;
  estimated_completion_weeks: number;
}

// roadmap.updated (job nocturno con cambios)
interface RoadmapUpdatedPayload {
  user_id: string;
  roadmap_id: string;
  changes: {
    blocks_added: number;
    blocks_removed: number;
    blocks_reordered: number;
  };
}

// level.unlocked
interface LevelUnlockedPayload {
  user_id: string;
  level_id: string;
  level_name: string;
  level_order: number;
}

// level.completed
interface LevelCompletedPayload {
  user_id: string;
  level_id: string;
  level_name: string;
  total_minutes_to_complete: number;
}
```

### 5.9 Anti-fraud

```typescript
// fraud.score_calculated
interface FraudScoreCalculatedPayload {
  user_id: string;
  score: number;               // 0-100
  signals: string[];
  severity: 'low' | 'medium' | 'high' | 'critical';
}

// restriction.applied
interface RestrictionAppliedPayload {
  user_id: string;
  restriction_id: string;
  restriction_type: string;    // 'no_referrals' | 'no_premium_features' | 'suspended' | 'banned'
  reason: string;
  applied_by: 'system' | 'admin';
}

// restriction.lifted
interface RestrictionLiftedPayload {
  user_id: string;
  restriction_id: string;
  lifted_by: 'system' | 'admin' | 'appeal_approved';
}

// fraud.appeal_submitted
interface FraudAppealSubmittedPayload {
  user_id: string;
  appeal_id: string;
  restriction_id: string;
}
```

### 5.10 Customer Support

```typescript
// ticket.created
interface TicketCreatedPayload {
  ticket_id: string;
  user_id?: string;
  email?: string;
  category: string;
  priority: 'low' | 'normal' | 'high' | 'critical';
  source: 'ai_escalation' | 'direct_email' | 'in_app';
}

// ticket.resolved
interface TicketResolvedPayload {
  ticket_id: string;
  user_id?: string;
  category: string;
  resolution_time_hours: number;
}
```

### 5.11 AI Gateway

```typescript
// ai.task_completed (logged, no consumido por sistemas)
interface AiTaskCompletedPayload {
  task_id: string;             // ej: 'score_pronunciation'
  model_used: string;
  cost_usd: number;
  latency_ms: number;
  input_tokens?: number;
  output_tokens?: number;
  validation_passed: boolean;
  fallback_used: boolean;
  user_id?: string;
}
```

### 5.12 Content Creation: Atomics y Composites (v1.2)

```typescript
// atomic.generated
interface AtomicGeneratedPayload {
  atomic_id: string;
  media_format: 'audio' | 'image' | 'video';
  media_subtype: string;
  generated_by: string;        // 'ai_dalle3', 'ai_elevenlabs', ...
  cost_usd: number;
  duration_ms?: number;
  needs_human_review: boolean;
}

// atomic.approved
interface AtomicApprovedPayload {
  atomic_id: string;
  reviewed_by: string;
}

// atomic.archived
interface AtomicArchivedPayload {
  atomic_id: string;
  reason: string;
  use_count_at_archive: number;
  replaced_by_atomic_id?: string;
}

// atomic.reuse_threshold_reached
// Emitido cuando un atomic alcanza use_count = 5 (high-value asset)
interface AtomicReuseThresholdPayload {
  atomic_id: string;
  use_count: number;
  total_cost_amortized_usd: number;
}

// composite.created
interface CompositeCreatedPayload {
  asset_id: string;
  primary_media_format: string;
  interaction_type: string;
  atomic_ids_referenced: string[];
}
```

### 5.13 Sporadic questions (v1.2)

Eventos del sistema de sporadic questions
(`student-profile-and-assessment.md` §7.1).

```typescript
// sporadic_question.shown
interface SporadicQuestionShownPayload {
  user_id: string;
  question_id: string;
  category: string;
  trigger_context: 'post_exercise' | 'pre_session_close' | 'achievement_unlock';
  session_id: string;
}

// sporadic_question.answered
interface SporadicQuestionAnsweredPayload {
  user_id: string;
  question_id: string;
  response_id: string;
  duration_seconds: number;
  flagged_likely_fake: boolean;       // detección heurística
}

// sporadic_question.skipped
interface SporadicQuestionSkippedPayload {
  user_id: string;
  question_id: string;
  consecutive_skips: number;          // para throttle adaptativo
}

// sporadic.mismatch_detected
// Cuando self-perception ≥ 4 pero measured ≤ 60 (o inverso)
interface SporadicMismatchPayload {
  user_id: string;
  dimension: string;
  self_reported: number;
  measured_score: number;
  mismatch_type: 'self_high_measured_low' | 'self_low_measured_high';
}

// sporadic.paused
// Después de 3 skips consecutivos
interface SporadicPausedPayload {
  user_id: string;
  paused_until: string;
  reason: 'consecutive_skips' | 'user_setting' | 'assessment_completed';
}
```

---

## 6. Product events (PostHog)

Eventos para analytics. Distintos de domain events. Naming similar
(snake_case past tense) pero el shape es **PostHog estándar** (event
name + properties planas).

### 6.1 Lista mínima de product events del MVP

```typescript
// Lifecycle
'user_signed_up'              { method, country, ... }
'user_completed_onboarding'   { duration_seconds }
'user_completed_mini_test'    { detected_cefr }
'user_started_trial'          {}
'user_completed_assessment'   { duration_minutes, scores }
'user_subscribed'             { plan, currency, amount, country }
'user_cancelled'              { reason, days_active, plan }

// Engagement
'session_started'             { source: 'notification'|'organic' }
'session_completed'           { duration_seconds, exercises_count }
'exercise_completed'          { type, score, mastery_change }
'block_completed'             { block_id, mastery_score, attempts }
'level_completed'             { level_id, time_to_complete_days }
'daily_goal_met'              { streak_days }
'streak_at_risk'              { streak_days }
'streak_broken'               { previous_streak }

// Achievement & social
'achievement_unlocked'        { achievement_id, rarity }
'sparks_earned'               { amount, source }
'sparks_spent'                { amount, on_what }
'pack_purchased'              { pack_type, amount }
'referral_sent'               { method }
'referral_signed_up'          { referrer_hash }
'logro_shared'                { achievement_id, platform }

// Monetization
'paywall_viewed'              { context, plan_shown }
'plan_selected'               { plan }
'payment_attempted'           { method, amount }
'payment_succeeded'           { method, amount }
'payment_failed'              { reason }

// Support
'help_article_viewed'         { article_id }
'ai_assistant_opened'         { context }
'ticket_created'              { category, priority }

// Notifications
'notification_received'       { notification_id, category }
'notification_opened'         { notification_id, category }
'notification_dismissed'      { notification_id, category }
```

### 6.2 Convenciones para PostHog

- `distinct_id`: hash del `users.id`. **Nunca** usar email u otro PII.
- Properties: solo datos no-PII. Country sí, IP no.
- Properties con datos numéricos para análisis (no formatear como
  string).
- Eventos disparados desde el cliente preferentemente; backend solo si
  el evento no es visible al cliente (ej: payment webhook).

### 6.3 Anti-patterns

- ❌ Trackear cada UI click sin contexto (`button_clicked`).
- ❌ Eventos con properties completamente distintas en cada disparo
  (rompe filtros).
- ❌ PII directa en properties.
- ❌ Eventos duplicados en cliente + backend (uno solo: el que tiene el
  contexto más completo).

---

## 7. Retención y privacidad

### 7.1 Retención por tipo de dato

| Tipo de dato | Retención | Donde |
|--------------|-----------|-------|
| `event_log` (domain events) | 12 meses | Postgres |
| `event_log_dlq` | Indefinido (manual cleanup) | Postgres |
| Domain events en Inngest dashboard | 90 días | Inngest |
| Product events | 12 meses | PostHog |
| Internal logs (errors) | 30 días | Sentry |
| Internal logs (Workers) | 7 días | Cloudflare |
| Audios crudos | 30 días desde grabación | R2 |
| Transcripciones de audio | Indefinido (anonimizadas a 90d) | Postgres |
| Datos del perfil | Mientras la cuenta esté activa | Postgres |
| Resultados de assessment | Indefinido (parte del progreso) | Postgres |
| Sparks transactions (audit) | Indefinido | Postgres |

### 7.2 Hashing de identificadores

- `user_id_hash` = `SHA-256(user_id || PEPPER)` donde `PEPPER` es secret
  rotable.
- `device_id_hash`, `ip_hash`, `email_hash` usan misma estrategia.
- Hashing es **irreversible** desde el sink externo. Cross-system
  internal lookup por hash usa tabla separada (no via cracking).

### 7.3 PII boundary

PII directa (email, nombre, teléfono, foto, IP cruda, dirección, audios,
transcripciones completas) **nunca** sale de los siguientes sistemas:

- `users` table (Postgres) — accesible solo vía Auth API.
- `student_profiles` (Postgres) — accesible solo vía Profile API.
- R2 (audios) — accesible solo vía signed URLs con TTL.

**No PII** en:
- Eventos (domain o product).
- Logs de Sentry o Cloudflare.
- PostHog.
- Tablas de tracking (`event_log`, etc.).
- Comments de código, ejemplos, docs.

### 7.4 Right to erasure

Cuando un usuario ejecuta account deletion (hard delete tras 30d
gracia):

1. **Postgres:** cascade delete de tablas que poseen sistemas
   (referencias `ON DELETE CASCADE`).
2. **R2:** delete de audios bajo prefijo `users/<user_id>/`.
3. **PostHog:** API call de delete person (`POST /persons/delete`) con
   `distinct_id = sha256(user_id)`.
4. **Sentry:** purge user events vía API.
5. **Inngest:** events viejos expiran solos a 90d (no purge proactivo).
6. **`event_log`:** purge por `user_id_hash` con job manual.

Implementación: función `cascadeUserDeletion(userId)` que ejecuta los
6 pasos en transacción. Si alguno falla, escalation manual.

### 7.5 Cron de cleanup

Job diario `cleanup_expired_data` en Inngest:

```typescript
// 1. Audios > 30 días
await r2.deleteObjectsOlderThan('users/', daysAgo(30));

// 2. Transcripciones > 90 días: anonimizar (remover user_id)
await db.query(`
  UPDATE exercise_attempts
  SET user_id = NULL, audio_storage_key = NULL
  WHERE completed_at < now() - interval '90 days'
    AND user_id IS NOT NULL
`);

// 3. event_log > 12 meses
await db.query(`
  DELETE FROM event_log WHERE emitted_at < now() - interval '12 months'
`);

// 4. consumed_events > 90 días
await db.query(`
  DELETE FROM consumed_events WHERE consumed_at < now() - interval '90 days'
`);

// 5. notifications_log > 6 meses
await db.query(`
  DELETE FROM notifications_log WHERE sent_at < now() - interval '6 months'
`);

// 6. exercise_attempts > 12 meses (mantener stats agregados en
//    weekly_mastery_snapshots)
await db.query(`
  DELETE FROM exercise_attempts WHERE completed_at < now() - interval '12 months'
`);
```

---

## 8. Versionado de eventos

### 8.1 Tabla `events_registry`

```sql
CREATE TABLE events_registry (
  event_name        TEXT PRIMARY KEY,
  current_version   TEXT NOT NULL,
  schema_jsonschema JSONB NOT NULL,    -- JSON Schema del payload
  emitter_system    TEXT NOT NULL,
  consumers         TEXT[] NOT NULL DEFAULT '{}',
  introduced_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  deprecated_at     TIMESTAMPTZ,
  deprecation_notes TEXT
);
```

Source of truth de qué eventos existen, su versión actual, su schema y
sus consumidores conocidos.

### 8.2 Patrón para breaking changes

Si necesitás cambiar el shape del payload de manera incompatible:

**Opción A — Coexistencia temporal:**

1. Emitir versión nueva como `<event_name>_v2` durante transición.
2. Consumidores nuevos migran a `_v2`.
3. Cuando todos migraron, dejar de emitir el original.

**Opción B — Versión en envelope:**

1. Mantener mismo `event_name`.
2. Bumpear `event_version` en envelope.
3. Consumidores viejos que reciben evento de versión nueva pueden
   ignorarlo (`if event.event_version !== EXPECTED return`).
4. Migración gradual de consumidores.

**Decisión:** usar Opción B preferentemente. A solo si el cambio es muy
grande y el evento tiene muchos consumidores.

### 8.3 Cambios compatibles atrás (no requieren versión nueva)

- Agregar campo opcional al payload.
- Agregar nuevo valor a un enum (los consumidores deben tener default).
- Cambiar tipo de `string` a `string | null` (igual default a null).

### 8.4 Cambios incompatibles (requieren versión nueva)

- Remover campo.
- Renombrar campo.
- Cambiar tipo de campo (string → number).
- Hacer campo opcional → required.
- Cambiar significado semántico de un campo.

---

## 9. Implementación

### 9.1 Sprint 1 — Foundation

- Tabla `event_log`, `consumed_events`, `event_log_dlq`,
  `events_registry`.
- Helper `emitDomainEvent` y wrapper de Inngest.
- Convención de hashing (`user_id_hash` con pepper).

### 9.2 Sprint 2 — Primeros eventos

- Auth: `user.signed_up`, `user.email_verified`.
- Student Profile: `onboarding.completed`.
- Sparks: `sparks.balance_changed`.
- AI Roadmap: `roadmap.generated`.

### 9.3 Sprint 3+ — Resto del catálogo según prioridad

- Pedagogical events (críticos para Motivation y AI Roadmap).
- Motivation events.
- Notifications events.
- Anti-fraud events.

### 9.4 Cleanup cron

Implementar al final de Sprint 3 cuando ya hay datos viejos para limpiar.

---

## 10. Métricas y alertas

### 10.1 Métricas

- **Throughput por evento:** eventos/min por `event_name`.
- **DLQ size:** cantidad de eventos no resueltos en
  `event_log_dlq`.
- **Latencia consumer-to-emit:** tiempo desde `emitted_at` hasta
  `consumed_at` (p50, p95).
- **Validation rate de eventos:** % de eventos que matchean su schema
  registrado en `events_registry`.

### 10.2 Alertas

- DLQ size > 100 eventos: investigar consumer caído.
- Validation rate < 99%: hay emisor enviando shapes incorrectos.
- Latencia p95 > 60s: Inngest o consumer lento.
- Cualquier evento crítico (`user.deleted`, `payment.succeeded`,
  `restriction.applied`) que falle en consumer: alerta inmediata.

---

## 11. Referencias internas

- [`../architecture/01-overview.md`](../architecture/01-overview.md) §5
  — contratos entre sistemas.
- [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md)
  — cómo el Gateway loggea eventos `ai.task_completed`.
- [`../business/metrics-and-experimentation.md`](../business/metrics-and-experimentation.md)
  — product events en PostHog.
- [`security-threat-model.md`](security-threat-model.md) — privacidad y
  compliance.

---

*Documento vivo. Actualizar cuando se agreguen eventos nuevos al
catálogo, cambien convenciones o se descubran patrones nuevos.*

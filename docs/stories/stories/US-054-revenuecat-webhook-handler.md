# US-054: RevenueCat webhook handler (in-app purchase events)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 2
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

RevenueCat es la abstracción cross-platform de Apple App Store y
Google Play Store. Sus webhooks notifican de eventos de in-app
purchases (subscriptions y consumables/packs).

Eventos clave:
- `INITIAL_PURCHASE`: primera compra de subscription.
- `RENEWAL`: renovación mensual.
- `CANCELLATION`: usuario canceló (no quiere renovar).
- `EXPIRATION`: subscription expiró.
- `BILLING_ISSUE`: cobro falló (Apple/Google retry interno).
- `NON_RENEWING_PURCHASE`: pack one-time comprado.

Esta story es la contraparte mobile de US-053 (Stripe).

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §13.

## Scope

### In

- Endpoint `POST /webhooks/revenuecat`:
  - **Verifica auth** vía `Authorization` header bearer token
    (`REVENUECAT_WEBHOOK_AUTH`).
  - Idempotency: `event.id` de RevenueCat único.
  - Routing por `event.type`.
- Handlers:
  - `INITIAL_PURCHASE`:
    - Parse `app_user_id` (= nuestro `user_id`) y `product_id`.
    - Map `product_id` → `plan_id` interno (config table).
    - Emit `payment.succeeded` event con `is_renewal: false`.
  - `RENEWAL`:
    - Emit `payment.succeeded` con `is_renewal: true`.
  - `CANCELLATION`:
    - Emit `subscription.cancelled` con
      `expiration_at = current_period_end` (no inmediato).
  - `EXPIRATION`:
    - Emit `subscription.expired`.
  - `BILLING_ISSUE`:
    - Emit `payment.failed` con `source: 'revenuecat'`.
  - `NON_RENEWING_PURCHASE`:
    - Sparks pack one-time.
    - Emit `pack.purchased`.
- Tabla `revenuecat_events_processed` para idempotency.
- Telemetry: `revenuecat.webhook_received`,
  `revenuecat.webhook_processed`, `revenuecat.auth_failed`.

### Out

- Validación de receipt original con Apple/Google
  (post-MVP — RevenueCat lo hace upstream).
- Subscription pause/resume (post-MVP).
- Promotional offers (post-MVP).

## Acceptance criteria

- **Given** webhook con `Authorization: Bearer <correct>`,
  **When** llega, **Then** procesa.
- **Given** webhook sin auth o token mal, **When** llega, **Then**
  401 sin procesar.
- **Given** mismo `event.id` 2x, **When** segundo, **Then** detect
  duplicate y 200 OK sin re-procesar.
- **Given** `INITIAL_PURCHASE` evento con `app_user_id, product_id`,
  **When** procesa, **Then** emit `payment.succeeded` con campos
  mapeados.
- **Given** `NON_RENEWING_PURCHASE` (pack), **When** procesa,
  **Then** emit `pack.purchased` con
  `{ user_id, pack_id, price_paid, currency }`.
- **Given** `EXPIRATION`, **When** procesa, **Then** emit
  `subscription.expired` (US-055 consumer maneja desactivación
  del plan).
- **Given** `BILLING_ISSUE`, **When** procesa, **Then** emit
  `payment.failed` con flag `source: 'revenuecat'`.
- **Given** event_type desconocido, **When** llega, **Then** log
  + 200 OK.

## Developer details

### Owning service

`apps/workers/api/handlers/webhooks-revenuecat.ts`.

### Dependencies

- US-044/045/047.
- US-055 consumer.
- RevenueCat dashboard config (webhook URL + auth token).

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §13.
- RevenueCat webhook docs (external).

### Implementación esperada

```typescript
// apps/workers/api/handlers/webhooks-revenuecat.ts
const PRODUCT_TO_PLAN_MAP: Record<string, string> = {
  'plan_basic_monthly_v1':   'basic',
  'plan_pro_monthly_v1':     'pro',
  'plan_premium_monthly_v1': 'premium',
};

const PRODUCT_TO_PACK_MAP: Record<string, { pack_id: string; sparks: number }> = {
  'pack_small_v1':  { pack_id: 'small', sparks: 50 },
  'pack_medium_v1': { pack_id: 'medium', sparks: 130 },
  'pack_large_v1':  { pack_id: 'large', sparks: 350 },
};

export async function handleRevenueCatWebhook(request: Request, env: Env) {
  // Auth
  const auth = request.headers.get('Authorization');
  if (auth !== `Bearer ${env.REVENUECAT_WEBHOOK_AUTH}`) {
    track('revenuecat.auth_failed');
    return jsonResponse({ error: 'unauthorized' }, 401);
  }

  const payload = await request.json();
  const event = payload.event;
  if (!event) {
    return jsonResponse({ error: 'invalid_payload' }, 400);
  }

  track('revenuecat.webhook_received', { type: event.type });

  // Idempotency
  const existing = await env.DB.query(`
    SELECT event_id FROM revenuecat_events_processed WHERE event_id = $1
  `, [event.id]);
  if (existing.length > 0) {
    return jsonResponse({ received: true, duplicate: true });
  }

  try {
    switch (event.type) {
      case 'INITIAL_PURCHASE':
      case 'RENEWAL':
        await handleSubscriptionPayment(event, env);
        break;
      case 'CANCELLATION':
        await emitEvent('subscription.cancelled', {
          user_id: event.app_user_id,
          expiration_at: event.expiration_at_ms ? new Date(event.expiration_at_ms).toISOString() : null,
          source: 'revenuecat',
        });
        break;
      case 'EXPIRATION':
        await emitEvent('subscription.expired', {
          user_id: event.app_user_id,
          source: 'revenuecat',
        });
        break;
      case 'BILLING_ISSUE':
        await emitEvent('payment.failed', {
          user_id: event.app_user_id,
          source: 'revenuecat',
          product_id: event.product_id,
        });
        break;
      case 'NON_RENEWING_PURCHASE':
        const pack = PRODUCT_TO_PACK_MAP[event.product_id];
        if (!pack) {
          throw new Error(`Unknown pack product: ${event.product_id}`);
        }
        await emitEvent('pack.purchased', {
          user_id: event.app_user_id,
          pack_id: pack.pack_id,
          sparks_amount: pack.sparks,
          price_paid: event.price,
          currency: event.currency,
          source: 'revenuecat',
        });
        break;
      default:
        console.log({ event: 'revenuecat.webhook_unknown_type', type: event.type });
    }

    await env.DB.execute(`
      INSERT INTO revenuecat_events_processed (event_id, event_type)
      VALUES ($1, $2) ON CONFLICT (event_id) DO NOTHING
    `, [event.id, event.type]);

    track('revenuecat.webhook_processed', { type: event.type });
    return jsonResponse({ received: true });
  } catch (error: any) {
    console.error({ event: 'revenuecat.webhook_error', error: error.message });
    return jsonResponse({ error: 'processing_failed' }, 500);
  }
}

async function handleSubscriptionPayment(event: any, env: Env) {
  const planId = PRODUCT_TO_PLAN_MAP[event.product_id];
  if (!planId) {
    throw new Error(`Unknown plan product: ${event.product_id}`);
  }
  await emitEvent('payment.succeeded', {
    user_id: event.app_user_id,
    plan_id: planId,
    revenuecat_subscription_id: event.original_transaction_id,
    amount_total: event.price * 100, // Stripe-style cents
    currency: event.currency,
    is_renewal: event.type === 'RENEWAL',
    source: 'revenuecat',
  });
}
```

### Schema

```sql
CREATE TABLE revenuecat_events_processed (
  event_id    TEXT PRIMARY KEY,
  event_type  TEXT NOT NULL,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Integration points

- RevenueCat (incoming).
- US-055 (consumer del `payment.succeeded`).
- Inngest event bus.

### Notas técnicas

- `app_user_id` debe ser nuestro `user_id` (UUID) — configurar en
  mobile app cuando init RevenueCat SDK.
- `original_transaction_id` permite trace cross-renewal.
- RevenueCat retries con backoff si recibimos 5xx.

## Definition of Done

- [ ] Endpoint con auth bearer.
- [ ] 6 event types handlers.
- [ ] Idempotency table + check.
- [ ] Product → plan/pack mappings configurados.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration con webhook fixtures.
- [ ] Validation contra spec `sparks-system.md` §13.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-055 (mobile path).*

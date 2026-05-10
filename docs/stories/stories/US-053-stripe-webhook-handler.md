# US-053: Stripe webhook handler (subscription events)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 2
**Story points:** 5
**Persona:** Admin (procesa payments)
**Owner:** —

---

## Contexto

Stripe procesa subscriptions de usuarios web. Su webhook nos
notifica de eventos críticos:
- `checkout.session.completed`: user pagó plan/pack.
- `customer.subscription.updated`: plan cambió.
- `customer.subscription.deleted`: subscription cancelada.
- `invoice.payment_succeeded`: ciclo mensual cobrado OK.
- `invoice.payment_failed`: cobro falló (3 retries).

Sin esta story, no hay forma de saber que el user pagó → no se
acreditan Sparks → user paga y queda sin acceso.

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §13
(payment integration).

## Scope

### In

- Endpoint `POST /webhooks/stripe`:
  - **Verifica signature** del webhook con
    `stripe.webhooks.constructEvent` usando `STRIPE_WEBHOOK_SECRET`.
  - Si signature inválida: 400 sin procesar (defensa contra spoof).
  - Idempotency: `event_id` de Stripe único; si ya procesado:
    200 OK sin re-procesar (Stripe retry-friendly).
  - Routing por `event.type` a handler específico.
- Handlers implementados:
  - `checkout.session.completed`:
    - Parse subscription metadata (`user_id`, `plan_id`).
    - Trigger `payment.succeeded` event (para US-055).
  - `customer.subscription.updated`:
    - Si `plan_id` cambió: trigger `subscription.plan_changed`.
    - Si `status` cambió a `active`: trigger
      `subscription.activated`.
  - `customer.subscription.deleted`:
    - Trigger `subscription.cancelled`.
    - Plan stays activo hasta fin del cycle pagado (Stripe
      semantics).
  - `invoice.payment_succeeded`:
    - Trigger `payment.succeeded` con `is_renewal = true`.
  - `invoice.payment_failed`:
    - Trigger `payment.failed` con
      `attempt_count` (1, 2, 3 — Stripe Smart Retries).
- Tabla `stripe_events_processed` para idempotency:
  ```sql
  CREATE TABLE stripe_events_processed (
    event_id    TEXT PRIMARY KEY,
    event_type  TEXT NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    payload_summary JSONB
  );
  ```
- Telemetry: `stripe.webhook_received`,
  `stripe.webhook_processed`, `stripe.webhook_duplicate`,
  `stripe.webhook_signature_failed`.

### Out

- UI checkout flow (post-MVP — está en spec de payment, story
  separada).
- Refund handling vía Stripe Dashboard (post-MVP, manual support).
- Tax handling, invoicing customizado (post-MVP).

## Acceptance criteria

- **Given** webhook llega con signature válido del Stripe SDK,
  **When** handler procesa, **Then** retorna 200 OK rápido
  (<2s) para no triggerar retry.
- **Given** webhook con signature inválido, **When** llega,
  **Then** retorna 400 sin procesar y sin emit events.
- **Given** mismo `event_id` se reciba 2x (Stripe retry),
  **When** segundo, **Then** detecta duplicate via tabla y retorna
  200 sin re-procesar.
- **Given** `checkout.session.completed` con metadata
  `user_id, plan_id`, **When** procesa, **Then** emit
  `payment.succeeded` event con payload completo.
- **Given** `customer.subscription.deleted`, **When** handler,
  **Then** emit `subscription.cancelled` con
  `cancel_at_period_end` data.
- **Given** `invoice.payment_failed` con `attempt_count: 1`,
  **When** procesa, **Then** emit `payment.failed` con
  `retry_will_happen: true`.
- **Given** `invoice.payment_failed` con `attempt_count: 3`
  (último), **When** procesa, **Then** emit `payment.failed` con
  `retry_will_happen: false, subscription_at_risk: true`.
- **Given** webhook event_type desconocido (Stripe agregó nuevo),
  **When** llega, **Then** se loguea + retorna 200 (no fallar
  por evento que no manejamos).

## Developer details

### Owning service

`apps/workers/api/handlers/webhooks-stripe.ts`.

### Dependencies

- US-044/045/047: foundation Sparks.
- US-055: consumer del `payment.succeeded` event.
- Stripe webhook secret configurado en `STRIPE_WEBHOOK_SECRET`.

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §13.
- Stripe webhook docs (external).

### Implementación esperada

```typescript
// apps/workers/api/handlers/webhooks-stripe.ts
import Stripe from 'stripe';

export async function handleStripeWebhook(request: Request, env: Env) {
  const signature = request.headers.get('stripe-signature');
  if (!signature) {
    return jsonResponse({ error: 'missing_signature' }, 400);
  }

  const body = await request.text();
  let event: Stripe.Event;
  try {
    const stripe = new Stripe(env.STRIPE_SECRET_KEY);
    event = await stripe.webhooks.constructEventAsync(
      body, signature, env.STRIPE_WEBHOOK_SECRET
    );
  } catch (error) {
    track('stripe.webhook_signature_failed');
    return jsonResponse({ error: 'invalid_signature' }, 400);
  }

  track('stripe.webhook_received', { type: event.type });

  // Idempotency check
  const existing = await env.DB.query(`
    SELECT event_id FROM stripe_events_processed WHERE event_id = $1
  `, [event.id]);
  if (existing.length > 0) {
    track('stripe.webhook_duplicate', { type: event.type });
    return jsonResponse({ received: true, duplicate: true });
  }

  try {
    switch (event.type) {
      case 'checkout.session.completed':
        await handleCheckoutCompleted(event.data.object as any, env);
        break;
      case 'customer.subscription.updated':
        await handleSubscriptionUpdated(event.data.object as any, env);
        break;
      case 'customer.subscription.deleted':
        await handleSubscriptionDeleted(event.data.object as any, env);
        break;
      case 'invoice.payment_succeeded':
        await handleInvoicePaymentSucceeded(event.data.object as any, env);
        break;
      case 'invoice.payment_failed':
        await handleInvoicePaymentFailed(event.data.object as any, env);
        break;
      default:
        console.log({ event: 'stripe.webhook_unknown_type', type: event.type });
    }

    // Mark as processed
    await env.DB.execute(`
      INSERT INTO stripe_events_processed (event_id, event_type, payload_summary)
      VALUES ($1, $2, $3)
      ON CONFLICT (event_id) DO NOTHING
    `, [event.id, event.type, JSON.stringify({
      object_id: (event.data.object as any).id,
    })]);

    track('stripe.webhook_processed', { type: event.type });
    return jsonResponse({ received: true });
  } catch (error: any) {
    console.error({ event: 'stripe.webhook_error', type: event.type, error: error.message });
    // Return 500 → Stripe will retry
    return jsonResponse({ error: 'processing_failed' }, 500);
  }
}

async function handleCheckoutCompleted(session: any, env: Env) {
  const { user_id, plan_id } = session.metadata ?? {};
  if (!user_id || !plan_id) {
    throw new Error(`Missing metadata in checkout: ${session.id}`);
  }

  await emitEvent('payment.succeeded', {
    user_id,
    plan_id,
    stripe_session_id: session.id,
    stripe_subscription_id: session.subscription,
    amount_total: session.amount_total,
    currency: session.currency,
    is_renewal: false,
  });
}

async function handleInvoicePaymentFailed(invoice: any, env: Env) {
  const attemptCount = invoice.attempt_count;
  await emitEvent('payment.failed', {
    user_id: invoice.metadata?.user_id,
    stripe_subscription_id: invoice.subscription,
    attempt_count: attemptCount,
    retry_will_happen: attemptCount < 3,
    subscription_at_risk: attemptCount >= 3,
    next_payment_attempt: invoice.next_payment_attempt,
  });
}

// ... otros handlers similares
```

### Integration points

- Stripe (incoming webhooks).
- Inngest event bus (downstream consumers).
- US-055 (consumer principal del `payment.succeeded`).

### Notas técnicas

- **Critical:** signature verification ANTES de cualquier
  parsing. Spoofed webhooks podrían dar acceso premium gratis.
- 200 OK rápido es importante: Stripe retries si timeout >5s.
- Idempotency table: keep 90 días, purge older (cron).
- Metadata en checkout session: setear `user_id` y `plan_id` al
  crear sessions (story separada de UI checkout).

## Definition of Done

- [ ] Endpoint con signature verification.
- [ ] 5 event types handlers implementados.
- [ ] Idempotency table + check.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit con webhooks fixtures.
- [ ] Test integration con Stripe CLI (`stripe trigger`).
- [ ] Validation contra spec `sparks-system.md` §13.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-055 (plan activation). Web-only.*

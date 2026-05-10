# US-048: Sparks refund (en caso de fallo de operación)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 1
**Story points:** 3
**Persona:** Admin (consumido por pedagogical engine + AI Gateway)
**Owner:** —

---

## Contexto

Complemento del US-047: si la operación que cobramos falla
(LLM error, audio inválido, network timeout), debemos hacer
**refund automático** al user. Sin esto, user pagaría por
operaciones que nunca produjeron output útil.

Esta story es el **escudo de equidad económica** del producto.

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §5.

## Scope

### In

- Endpoint `POST /sparks/refund`:
  - Auth: dual JWT user + X-Internal-Auth.
  - Body: `{ transaction_id, reason }`.
  - Validar que `transaction_id` existe, es del user, y es de
    `type='charge'`.
  - Validar que NO tiene refund previo (1 refund per charge max).
  - Insertar transaction `type='refund'`,
    `amount = +X` (mismo monto que el charge original),
    `reason`,
    `related_id = transaction_id` del original,
    `idempotency_key = 'refund_${transaction_id}'`.
  - Invalidate cache.
  - Emit `sparks.refunded` event.
- Auto-refund flow: clients que llamaron charge deberían wrap
  call en try/catch:
  ```typescript
  const charge = await sparks.charge(...);
  try {
    const result = await operationThatMightFail();
    return result;
  } catch (error) {
    await sparks.refund(charge.transaction_id, 'operation_failed');
    throw error;
  }
  ```
- Telemetry: `sparks.refund_requested`, `.refund_succeeded`,
  `.refund_failed_already_refunded`,
  `.refund_failed_invalid_tx`.

### Out

- Partial refunds (post-MVP — MVP es full refund only).
- Refund de transactions que NO fueron charges (ej: refund de
  award): NO permitido.
- Manual refund por support agent (post-MVP — endpoint admin
  separado).

## Acceptance criteria

- **Given** charge transaction `tx_001` con amount 10 para user X,
  **When** POST refund con `transaction_id: 'tx_001'`, **Then**
  balance sube +10, nueva transaction insertada con
  `type='refund', amount=10, related_id='tx_001'`.
- **Given** mismo `tx_001` ya tiene refund, **When** POST refund
  segunda vez, **Then** retorna 409 Conflict
  `{ error: 'already_refunded', original_refund_id }`.
- **Given** `transaction_id` inexistente, **When** valida, **Then**
  retorna 404 `{ error: 'transaction_not_found' }`.
- **Given** transaction de otro user, **When** user X intenta
  refund, **Then** retorna 403
  `{ error: 'transaction_not_yours' }`.
- **Given** transaction de tipo `award_trial`, **When** se intenta
  refund, **Then** retorna 400
  `{ error: 'only_charges_can_be_refunded' }`.
- **Given** request sin auth interna, **When** llega, **Then**
  retorna 401.
- **Given** refund exitoso, **When** completa, **Then**
  `sparks.refunded` event emitido + cache invalidated.
- **Given** Postgres falla mid-transaction, **When** función
  procesa, **Then** rollback automático (atomic) y retorna 500.

## Developer details

### Owning service

`apps/workers/api/handlers/sparks-refund.ts`.

### Dependencies

- US-044: schema + `sparks_log`.
- US-045: cache invalidation.
- US-047: charge (provee transaction_ids para refund).

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §5.

### Implementación esperada

```typescript
// apps/workers/api/handlers/sparks-refund.ts
const RefundSchema = z.object({
  transaction_id: z.string().uuid(),
  reason: z.string().min(1).max(200),
});

export async function handleSparksRefund(request: AuthedRequest, env: Env) {
  if (request.headers.get('X-Internal-Auth') !== env.INTERNAL_AUTH_TOKEN) {
    return jsonResponse({ error: 'unauthorized' }, 401);
  }

  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const body = await request.json();
  const parsed = RefundSchema.safeParse(body);
  if (!parsed.success) {
    return jsonResponse({ error: parsed.error.issues[0].message }, 400);
  }

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  track('sparks.refund_requested', { transaction_id: parsed.data.transaction_id });

  // Fetch original transaction
  const original = await env.DB.query(`
    SELECT id, user_id, type, amount, balance_after
    FROM sparks_transactions
    WHERE id = $1
  `, [parsed.data.transaction_id]);

  if (original.length === 0) {
    track('sparks.refund_failed_invalid_tx', { reason: 'not_found' });
    return jsonResponse({ error: 'transaction_not_found' }, 404);
  }

  const tx = original[0];
  if (tx.user_id !== userId) {
    track('sparks.refund_failed_invalid_tx', { reason: 'wrong_user' });
    return jsonResponse({ error: 'transaction_not_yours' }, 403);
  }

  if (tx.type !== 'charge') {
    track('sparks.refund_failed_invalid_tx', { reason: 'not_a_charge' });
    return jsonResponse({ error: 'only_charges_can_be_refunded' }, 400);
  }

  // Check if already refunded
  const existingRefund = await env.DB.query(`
    SELECT id FROM sparks_transactions
    WHERE related_id = $1 AND type = 'refund'
  `, [parsed.data.transaction_id]);

  if (existingRefund.length > 0) {
    track('sparks.refund_failed_already_refunded', {});
    return jsonResponse({
      error: 'already_refunded',
      original_refund_id: existingRefund[0].id,
    }, 409);
  }

  // Atomic refund via sparks_log
  const result = await env.DB.query(`
    SELECT * FROM sparks_log($1, 'refund', $2, $3, $4, $5)
  `, [
    userId,
    -tx.amount, // refund is positive of (negated negative original)
    parsed.data.reason,
    parsed.data.transaction_id, // related_id
    `refund_${parsed.data.transaction_id}`,
  ]);

  const { transaction_id: refundTxId, new_balance } = result[0];

  await invalidateBalanceCache(userId, env);

  await emitEvent('sparks.refunded', {
    user_id: userId,
    original_transaction_id: parsed.data.transaction_id,
    refund_transaction_id: refundTxId,
    amount: -tx.amount,
    reason: parsed.data.reason,
  });

  track('sparks.refund_succeeded', { amount: -tx.amount });

  return jsonResponse({
    success: true,
    refund_transaction_id: refundTxId,
    new_balance,
    refunded_amount: -tx.amount,
  });
}
```

### Client wrapper helper

Documentado en `docs/architecture/sparks-system.md` para que
implementers de pedagogical engine + AI Gateway adopten el
pattern:

```typescript
// apps/workers/shared/with-sparks-charge.ts
export async function withSparksCharge<T>(
  user: { firebase_uid: string },
  amount: number,
  reason: string,
  related_id: string,
  operation: () => Promise<T>
): Promise<T> {
  const idempotencyKey = `${reason}_${related_id}`;
  const charge = await sparksClient.charge(user, {
    amount, reason, related_id, idempotency_key: idempotencyKey,
  });

  try {
    return await operation();
  } catch (error) {
    await sparksClient.refund(user, {
      transaction_id: charge.transaction_id,
      reason: 'operation_failed',
    });
    throw error;
  }
}
```

### Integration points

- Pedagogical engine + AI Gateway (callers).
- US-047 (charge) provee transaction_id.
- Notifications (event consumer puede mostrar "Te devolvimos X
  Sparks" — opcional, post-MVP).

### Notas técnicas

- Refund usa idempotency_key pattern `refund_${original_tx_id}` —
  mismo refund llamado 2x detecta el original.
- Match strictly type='charge': refund de awards o subscriptions
  no permitido (esos tienen flow propio).
- Audit trail: refund queda visible en transactions con
  `related_id` apuntando al charge.

## Definition of Done

- [ ] Endpoint implementado.
- [ ] Validation completa (transaction_id existe, es del user, es
  charge, no refunded).
- [ ] Idempotency funcional.
- [ ] Helper `withSparksCharge` documentado.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Test de double-refund (segundo retorna 409).
- [ ] Validation contra spec `sparks-system.md` §5.
- [ ] PR aprobada y mergeada.

---

*Depende de US-047. Bloqueante para EPIC-03 (pedagogical scoring
necesita pattern charge+refund).*

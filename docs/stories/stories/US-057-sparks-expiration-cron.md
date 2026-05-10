# US-057: Sparks expiration cron (packs 6 meses)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 2
**Story points:** 2
**Persona:** Admin (mantenimiento financiero)
**Owner:** —

---

## Contexto

Algunos Sparks expiran:
- Pack purchases: 6 meses post-compra.
- Trial Sparks: al fin del trial (7 días).

Esta story implementa el cron diario que:
1. Detecta transactions con `expires_at < now()` y
   `expired = false`.
2. Mark `expired = true`.
3. Insert offsetting transaction `type='expiration'` con
   `amount = -original_amount` (negativo para reducir balance).
4. Update balance.
5. Notifica al user 7 días antes de expiration (event
   `sparks.expiring_soon`).

Sin esto, packs caducados quedan eternamente en balance
(financialmente incorrecto).

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §3
(packs expiration).

## Scope

### In

- Cron Worker `apps/workers/crons/sparks-expiration.ts`:
  - Trigger: diario 02:00 UTC.
  - Para cada transaction con `expires_at < now() AND expired =
    false`:
    - Mark expired=true.
    - Insert offsetting transaction
      `type='expiration', amount=-original.amount,
       reason='expired_${original.type}', related_id=original.id`.
    - Update balance row.
    - Emit `sparks.expired` event (post-MVP notify user).
- Pre-expiration warning cron (separate, también daily):
  - Detect transactions con
    `expires_at BETWEEN now() AND now() + 7 days
     AND expired = false`.
  - Emit `sparks.expiring_soon` event con
    `{ user_id, amount, expires_at }`.
  - Cooldown: max 1 evento per user per pack per 7 days (KV).
- Telemetry: `sparks.expirations_processed`,
  `sparks.expiring_warnings_sent`.

### Out

- Notificación push real (esa es responsabilidad de
  notifications-system, post-MVP).
- Auto-renewal de packs (no, packs son one-time).

## Acceptance criteria

- **Given** transaction con `expires_at: yesterday, expired: false,
  amount: 50`, **When** cron ejecuta, **Then** mark expired=true +
  insert offsetting transaction de -50, balance baja por -50.
- **Given** transaction `expires_at: tomorrow`, **When** cron,
  **Then** NO procesa expiration (pero SÍ envía warning si dentro
  de 7 días).
- **Given** transaction ya expired (expired=true), **When** cron,
  **Then** skip (idempotent).
- **Given** transaction expira con amount 100 pero current balance
  es 30 (user gastó parte), **When** cron procesa, **Then** balance
  baja a -70? **NO** — clamp a 0 (no negative). Diff queda como
  "loss" log para audit.

  > Decisión: si user gastó más Sparks que el pack en sí, eso
  > significa que ya consumió el pack via FIFO (otros bonus
  > anteriores). Expiration removes lo que queda.

- **Given** warning cron detecta pack expira en 5 días, **When**
  emite, **Then** event `sparks.expiring_soon` con detalles.
- **Given** warning ya emitido para mismo pack en últimos 7 días,
  **When** próximo cron, **Then** skip cooldown.
- **Given** 1000 transactions a expirar, **When** cron procesa,
  **Then** completa en <2 min y emite count.
- **Given** Postgres falla mid-cron, **When** crash, **Then** parcial
  processed; próxima corrida resume (transactions con expired=false
  detectadas otra vez).

## Developer details

### Owning service

`apps/workers/crons/sparks-expiration.ts`.

### Dependencies

- US-044 (schema).
- US-045 (cache invalidation).
- US-046 trial (caduca al fin trial).
- US-056 packs (caducan +6 meses).

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §3.

### Implementación esperada

```typescript
// apps/workers/crons/sparks-expiration.ts
export async function runSparksExpiration(env: Env) {
  const startTime = Date.now();
  let processedCount = 0;

  while (true) {
    // Process in batches of 100 (transactional)
    const batch = await env.DB.query(`
      SELECT id, user_id, amount, type
      FROM sparks_transactions
      WHERE expires_at < now() AND expired = false
      ORDER BY expires_at ASC
      LIMIT 100
    `);

    if (batch.length === 0) break;

    for (const tx of batch) {
      await env.DB.transaction(async (txn) => {
        // 1. Mark original as expired
        await txn.execute(`
          UPDATE sparks_transactions SET expired = true WHERE id = $1
        `, [tx.id]);

        // 2. Get current balance to compute offset (clamp to 0)
        const balance = await txn.query(`
          SELECT current FROM sparks_balances WHERE user_id = $1
        `, [tx.user_id]);
        const currentBalance = balance[0]?.current ?? 0;
        const offsetAmount = Math.min(tx.amount, currentBalance);

        if (offsetAmount > 0) {
          // 3. Insert expiration transaction
          await txn.execute(`
            INSERT INTO sparks_transactions
              (user_id, type, amount, balance_after, reason, related_id, idempotency_key)
            VALUES ($1, 'expiration', $2, $3, $4, $5, $6)
          `, [
            tx.user_id,
            -offsetAmount,
            currentBalance - offsetAmount,
            `expired_${tx.type}`,
            tx.id,
            `expiration_${tx.id}`,
          ]);

          // 4. Update balance
          await txn.execute(`
            UPDATE sparks_balances SET current = current - $1 WHERE user_id = $2
          `, [offsetAmount, tx.user_id]);
        }
      });

      await invalidateBalanceCache(tx.user_id, env);
      processedCount++;
    }
  }

  const elapsed = Date.now() - startTime;
  track('sparks.expirations_processed', { count: processedCount, elapsed_ms: elapsed });

  return { processed: processedCount };
}

export async function runExpiringSoonWarnings(env: Env) {
  let warningsSent = 0;

  const expiring = await env.DB.query(`
    SELECT id, user_id, amount, expires_at, type
    FROM sparks_transactions
    WHERE expires_at BETWEEN now() AND now() + interval '7 days'
      AND expired = false
  `);

  for (const tx of expiring) {
    const cooldownKey = `expiring_warning:${tx.id}`;
    const inCooldown = await env.SPARKS_KV.get(cooldownKey);
    if (inCooldown) continue;

    await emitEvent('sparks.expiring_soon', {
      user_id: tx.user_id,
      amount: tx.amount,
      expires_at: tx.expires_at,
      transaction_type: tx.type,
    });

    await env.SPARKS_KV.put(cooldownKey, '1', { expirationTtl: 7 * 24 * 3600 });
    warningsSent++;
  }

  track('sparks.expiring_warnings_sent', { count: warningsSent });
  return { warnings_sent: warningsSent };
}
```

### Cron triggers

```toml
[triggers]
crons = [
  "0 2 * * *",   # Daily 02:00 UTC: process expirations
  "0 14 * * *",  # Daily 14:00 UTC: send warnings (timing-friendly)
]
```

### Integration points

- US-044 (schema).
- US-045 (cache).
- Notifications system (consume `sparks.expiring_soon`).

### Notas técnicas

- Clamp to 0 (no negative balance) es decisión defensiva. Caso
  raro pero posible si user gastó más que el pack original (otras
  awards lo subsidiaron).
- Audit anomaly cron (US-049) detectará si clamp aplicó (drift
  expected vs actual).
- 02:00 UTC es low-traffic.
- Warning a 14:00 UTC para que llegue durante hábito típico de
  uso (no de madrugada).

## Definition of Done

- [ ] 2 cron functions implementadas.
- [ ] Cron triggers configurados.
- [ ] Clamp to 0 logic correcta.
- [ ] Cooldown KV de warnings funcional.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration con fixtures.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-044, US-045, US-056. Cierra el lifecycle de packs.*

# US-058: Edge cases del Sparks system (race conditions, concurrent ops)

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 2
**Story points:** 3
**Persona:** Admin (financial integrity)
**Owner:** —

---

## Contexto

Las stories US-044 a US-057 cubren happy paths y errors típicos.
Esta story consolida **edge cases críticos** identificados durante
diseño, asegurando que el sistema económico es robusto bajo:
- Concurrent operations.
- Network partition durante payment.
- User actions fast/repetitive.
- State transitions complejas (downgrade, cancellation, refund
  legal).

Sin esta story consolidada, edge cases quedan diluidos y algunos
no se testean → risk de bugs financieros en producción.

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §11
(edge cases).

## Scope

### In

- **Test suite cubriendo 9 edge cases:**

  1. **Concurrent charges race:**
     - 5 charges simultáneos para user con balance 50, cada uno
       de 12 Sparks.
     - Esperado: 4 succeed (balance → 2), 5to fail con
       insufficient. Suma exacta sin negativo.

  2. **Charge + concurrent refund:**
     - User hace 2 charges sequential, 1 refund concurrent al
       primero.
     - Esperado: refund procesa el primer charge correcto sin
       afectar el segundo.

  3. **Webhook duplicate (network retry):**
     - Stripe envía webhook 3x (retry por timeout).
     - Esperado: payment procesado solo 1 vez.

  4. **User cancels subscription during cycle:**
     - User cancela plan Pro mid-cycle (10 días into 30).
     - Esperado: plan stays activo hasta cycle end, Sparks no se
       remove, status = 'cancelled' pero functional hasta
       expiration.

  5. **Plan downgrade Pro → Basic:**
     - User downgrade Pro (200) a Basic (30) mid-cycle.
     - Esperado: cycle reset al new plan, Sparks ya gastados no
       refund, NEW allotment 30 (reduce si existía cycle_allotment
       anterior).

  6. **Sparks expiration coincide con charge:**
     - User tiene 100 Sparks (50 trial expirando hoy + 50 pack).
     - Cron expiration corre 02:00.
     - User intenta charge 60 Sparks a 02:01.
     - Esperado: si expiration procesó primero (current=50),
       charge succeeds (50-60 fail). Si charge primero, expiration
       sees current=40 y clamp.
     - **Test:** ambos órdenes funcionan sin negative balance.

  7. **Refund de transaction old (>90 días):**
     - Sparks transactions retention pol es indefinite, but
       practicality may limit refunds.
     - Decisión: NO time limit en refunds (refund permitido siempre
       que transaction exista + no refunded prev).

  8. **User borrado mid-flow:**
     - User elimina cuenta mientras tiene charges en flight.
     - Esperado: CASCADE delete elimina balance + transactions.
       Charges en flight detectan user_not_found y fallar
       gracefully.

  9. **Idempotency key reuse cross-user:**
     - User A usa key `ex_attempt_001` para charge.
     - User B usa MISMO key `ex_attempt_001` para charge.
     - Esperado: dos charges separados (idempotency es
       per-user, no global).

- Tests automatizados para los 9 cases (integration tests).
- Documentation `docs/qa/sparks-edge-cases.md` (a crear) con
  scenarios manuales.
- Fix de bugs detectados durante test development.

### Out

- Edge cases de Stripe/RevenueCat fraude (post-MVP, anti-fraud
  system propio).
- Migration de Sparks históricos si cambia precio (post-MVP).

## Acceptance criteria

- **Given** 5 concurrent charges via Promise.all para user balance
  50 amount 12 each, **When** se ejecutan, **Then** exactamente 4
  succeed, 1 fail con insufficient. Final balance = 2.
- **Given** webhook Stripe llega 3x (network retry simulado),
  **When** procesados, **Then** sparks_transactions tiene 1 sola
  entry para esa subscription.
- **Given** user cancela subscription en día 10 de 30, **When**
  procesado, **Then** subscription.status='cancelled',
  cycle_ends_at preserved, balance no cambia. Día 30: cron
  detecta cycle ended → status='expired', plan se desactiva.
- **Given** downgrade Pro (200 → Basic 30) mid-cycle, **When**
  webhook llega, **Then** new cycle starts immediately con
  allotment 30, Sparks current se preserva (no refund).
- **Given** scenario 6 (concurrent expiration + charge), **When**
  ambos órdenes test, **Then** balance final correcto en ambos
  casos sin negative.
- **Given** refund de transaction de hace 100 días, **When**
  user/admin solicita, **Then** procesa OK (no time limit).
- **Given** user delete mid-charge, **When** charge intenta
  procesar, **Then** retorna error `user_not_found` graceful (no
  crash).
- **Given** mismo idempotency_key usado por user A y user B,
  **When** ambos charge, **Then** 2 transactions separadas (key
  unique per (user_id, idempotency_key)).
- **Given** test suite ejecuta, **When** completa, **Then** los 9
  edge cases pasan + audit cron (US-049) NO detecta anomalies
  post-test.

## Developer details

### Owning service

`apps/workers/api/__tests__/sparks-edge-cases.integration.test.ts`
+ fixes en handlers según hallazgos.

### Dependencies

- US-044 a US-057 implementadas.

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §11.

### Implementación esperada

Cada edge case = 1 test integration. Ejemplo del case 1
(concurrent race):

```typescript
// __tests__/sparks-edge-cases.integration.test.ts
describe('Sparks edge case: concurrent charges race', () => {
  it('5 concurrent charges sum correctly without negative balance', async () => {
    const userId = await setupUserWithBalance(50);
    const promises = Array.from({ length: 5 }, (_, i) =>
      sparksClient.charge({
        amount: 12,
        reason: 'exercise_scoring',
        idempotency_key: `race_test_${i}`,
      }).catch(err => ({ error: err.code }))
    );

    const results = await Promise.all(promises);
    const successes = results.filter(r => 'success' in r && r.success);
    const failures = results.filter(r => 'error' in r);

    expect(successes).toHaveLength(4);
    expect(failures).toHaveLength(1);
    expect(failures[0].error).toBe('insufficient_sparks');

    const finalBalance = await sparksClient.getBalance(userId);
    expect(finalBalance.current).toBe(2); // 50 - (4 * 12) = 2
  });
});

// ... otros 8 tests similares por edge case
```

### Implementación de fix scenario 6

Agregar advisory lock en cron expiration para coordinar con
charges:

```sql
-- En sparks_log helper, antes de UPDATE balance:
PERFORM pg_advisory_xact_lock(hashtext('sparks_balance_' || p_user_id::text));

-- En cron expiration, mismo lock antes de procesar user:
PERFORM pg_advisory_xact_lock(hashtext('sparks_balance_' || tx.user_id::text));
```

Esto serializa operations sobre mismo balance row → no race.

### Documentation

`docs/qa/sparks-edge-cases.md`:
- Scenarios manualmente testables.
- Steps reproducción.
- Expected behavior por scenario.
- Si algún case falla en producción: link al test que lo cubre.

### Integration points

- Todos los handlers del epic.
- US-049 audit cron (verifica no anomalies post-test).

### Notas técnicas

- Postgres advisory locks son per-session, automaticamente released
  al commit. Útil para coordinar Workers concurrent.
- Tests integration corren contra DB real (no mock) para validar
  constraints reales.
- Failures detectados durante test → bugfixes en stories US-044 a
  US-057.

## Definition of Done

- [ ] 9 edge case tests integration implementados.
- [ ] Bugs detectados durante test development arreglados (en
  PRs propias linked).
- [ ] Doc `docs/qa/sparks-edge-cases.md` creado.
- [ ] Audit cron (US-049) corre post-suite y NO detecta
  anomalies.
- [ ] 9 acceptance criteria pasan.
- [ ] Validation contra spec `sparks-system.md` §11.
- [ ] PR aprobada y mergeada.

---

*Cross-cutting. Cierra EPIC-04 con confidence en robustez.*

# EPIC-04: Sparks system

**Status:** Draft
**Sprint target:** Sprint 1-2 (paralelo a EPIC-01 + EPIC-05)
**Owner:** —
**Estimación total:** ~34 puntos (15 stories aprox.)

---

## Resumen

Implementar el sistema económico del producto: **Sparks** como
unidad atómica de consumo para operaciones que cuestan IA (LLM
calls, scoring, conversaciones).

Componentes:
1. Balance + transactions schema con audit trail inmutable.
2. **Cobro previo + refund pattern** (regla de oro):
   - Antes de invocar operación cara: cobrar Sparks.
   - Si operación falla: refund automático.
3. Award flows: trial inicial (50), bonus por logros, packs
   comprados.
4. Integración con Stripe (web) + RevenueCat (mobile in-app).
5. Edge cases: race conditions, packs expiration, low balance
   notifications.

---

## Por qué este epic

Sparks son la **unidad económica** del producto:
- Sin Sparks tracking, no hay control de costos LLM (user
  podría agotarnos por error).
- Sin cobro previo + refund, una operación que falla mid-call
  cobra al user igual.
- Sin audit log inmutable, disputes financieros / fraude
  imposible de resolver.
- Sin pack purchases, no hay revenue alternativo a subscriptions.

EPIC-04 es **bloqueante para EPIC-03** (pedagogical engine cobra
Sparks por exercise scoring) y **bloqueante para go-live**.

---

## Specs autoritativos referenciados

| Spec | Cubre |
|------|-------|
| [`architecture/sparks-system.md`](../../architecture/sparks-system.md) | Sistema económico completo |
| [`decisions/ADR-003-sparks-system.md`](../../decisions/ADR-003-sparks-system.md) | Decisión arquitectónica |
| [`reglas.md`](../../reglas.md) | Cobro previo + refund, audit inmutable, preassets gratis siempre |
| [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md) §5 | Trial 50 Sparks |

---

## Stories del epic

### Slice 1: Foundation (~7 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-044 | Schema `sparks_balances` + `sparks_transactions` | 3 |
| US-045 | Sparks balance read endpoint + cache | 2 |
| US-046 | Sparks initial trial award (50 al signup) | 2 |

### Slice 2: Cobro previo + refund pattern (~8 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-047 | Sparks charge endpoint (cobro previo idempotente) | 5 |
| US-048 | Sparks refund (en caso de fallo de operación) | 3 |
| US-049 | Audit log inmutable validation cron | 2 |

### Slice 3: Bonus + low balance (~5 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-050 | Sparks awardBonus (logros, referrals, comeback) | 2 |
| US-051 | Low Sparks event trigger (delega a notifications) | 2 |
| US-052 | UI de balance display en home + paywall | 1 |

### Slice 4: Stripe + RevenueCat (~9 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-053 | Stripe webhook handler (subscription events) | 5 |
| US-054 | RevenueCat webhook handler (in-app events) | 3 |
| US-055 | Plan activation + Sparks credit on payment success | 2 |

### Slice 5: Packs + edge cases (~5 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-056 | Pack purchase flow (one-time Sparks pack) | 2 |
| US-057 | Sparks expiration cron (packs 6 meses) | 2 |
| US-058 | Edge cases (race conditions, concurrent ops) | 3 |

---

## Total estimado

~34 puntos en 15 stories. Asumiendo velocity 20-25 pts/sprint:
**~2 sprints** (semanas 3-5), paralelo a EPIC-01 + EPIC-05.

---

## Dependencies externas

- Cuenta Stripe + webhook secret configurado.
- Cuenta RevenueCat + webhook secret.
- Postgres tablas (US-002 ya creó base; esta extiende).
- Inngest para event handling (compartido con otros sistemas).

---

## Dependencies inter-EPIC

- **EPIC-01:**
  - US-008 emit `user.signed_up` → US-046 acredita 50 trial.
- **EPIC-05:**
  - US-031 cost tracking (orthogonal a Sparks: cost real vs Sparks
    son distintos units).
- **EPIC-03 (futuro):**
  - Pedagogical engine llama US-047 (charge) antes de scoring +
    US-048 (refund) si falla.

---

## Definition of Done del epic

- [ ] Schema completo + audit trail inmutable.
- [ ] Cobro previo + refund pattern funcionando.
- [ ] Trial 50 Sparks acreditado al signup.
- [ ] Bonus flows operacionales (logros, comeback).
- [ ] Stripe + RevenueCat webhooks procesando payments.
- [ ] Plan activation atómico (payment + Sparks credit en una
  transacción).
- [ ] Pack purchases con expiration cron operativo.
- [ ] Low balance notifications triggered.
- [ ] Edge cases (race conditions) cubiertos en tests.
- [ ] Audit log inmutable verificado por validation cron.

---

## Riesgos identificados

| Riesgo | Mitigación |
|--------|------------|
| Race condition en cobro concurrente | UPDATE ... RETURNING + advisory locks de Postgres |
| Webhook duplicado de Stripe | Idempotency key del event_id de Stripe |
| Refund incorrecto (refund de algo nunca cobrado) | Match transaction_id en refund + validation |
| Audit log mutado accidentalmente | INSERT-only, no UPDATE permitido por permissions |
| Pack expirado pero balance positivo | Cron mensual que reconcilia + notifica al user antes |

---

## A/B tests post-MVP

(Coordinar con `sparks-system.md`.)

1. Trial Sparks: 30 vs 50 vs 100.
2. Pricing per pack (3 sizes).
3. Bonus de comeback D14: 20 vs 10 vs 30 Sparks.

---

*Documento vivo. Actualizar cuando se ajusten precios, se agreguen
nuevos flows de bonus o packs.*

# US-044: Schema `sparks_balances` + `sparks_transactions`

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 1
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Schema base del sistema económico:
- `sparks_balances`: 1 row por user con balance actual + métricas
  agregadas.
- `sparks_transactions`: log inmutable append-only de cada cambio
  al balance (charge, refund, award, expiration).

Es **append-only por diseño**: no se permite UPDATE ni DELETE en
transactions (audit trail sagrado).

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §6
(schema autoritativo).

## Scope

### In

- Migration SQL `0XXX_sparks_schema.sql`:
  ```sql
  CREATE TABLE sparks_balances (
    user_id              UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    current              INT NOT NULL DEFAULT 0,
    cycle_allotment      INT NOT NULL DEFAULT 0,        -- de plan suscripto
    cycle_started_at     TIMESTAMPTZ,
    cycle_ends_at        TIMESTAMPTZ,
    lifetime_earned      INT NOT NULL DEFAULT 0,
    lifetime_spent       INT NOT NULL DEFAULT 0,
    last_transaction_id  UUID,                          -- denormalized for race detection
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (current >= 0)                               -- no negative balance
  );

  CREATE TABLE sparks_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type            TEXT NOT NULL CHECK (type IN (
                      'charge', 'refund', 'award_trial', 'award_bonus',
                      'award_pack', 'award_subscription_cycle',
                      'expiration'
                    )),
    amount          INT NOT NULL,                     -- positive: credit; negative: debit
    balance_after   INT NOT NULL,                     -- snapshot post-transaction
    reason          TEXT NOT NULL,                    -- 'exercise_scoring', 'achievement_streak_7', etc.
    related_id      TEXT,                             -- exercise_attempt_id, achievement_id, etc.
    idempotency_key TEXT UNIQUE,                      -- previene duplicados
    metadata        JSONB DEFAULT '{}',
    expires_at      TIMESTAMPTZ,                      -- para Sparks que caducan (packs)
    expired         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
  );

  CREATE INDEX idx_sparks_tx_user_date ON sparks_transactions(user_id, created_at DESC);
  CREATE INDEX idx_sparks_tx_idempotency ON sparks_transactions(idempotency_key)
    WHERE idempotency_key IS NOT NULL;
  CREATE INDEX idx_sparks_tx_expiring ON sparks_transactions(expires_at)
    WHERE expires_at IS NOT NULL AND expired = false;

  -- Append-only enforcement via row-level rule
  CREATE RULE sparks_tx_no_update AS ON UPDATE TO sparks_transactions
    DO INSTEAD NOTHING;
  CREATE RULE sparks_tx_no_delete AS ON DELETE TO sparks_transactions
    DO INSTEAD NOTHING;
  ```
- Trigger `updated_at` para sparks_balances.
- Function helper `sparks_log(user_id, type, amount, reason, ...)`
  que en una transacción:
  - Inserta row en transactions.
  - Updates balance.
  - Retorna nuevo balance.
- Migration down (`_down.sql`) para rollback.
- Tests de migration: idempotency, append-only enforcement, FK
  cascade.

### Out

- Endpoints CRUD (US-045+).
- UI (US-052).
- Stripe webhook persist (US-053).

## Acceptance criteria

- **Given** la migration aplicada, **When** se ejecuta, **Then** las
  2 tablas existen con todos los columnas, constraints, índices.
- **Given** un user nuevo, **When** se intenta INSERT con
  `current = -10`, **Then** rechaza por CHECK constraint.
- **Given** una transaction persistida, **When** se intenta UPDATE,
  **Then** el rule la convierte en NOTHING (no error pero no
  cambio).
- **Given** una transaction persistida, **When** DELETE, **Then**
  mismo behavior (NOTHING).
- **Given** mismo `idempotency_key` se inserta 2x, **When** segunda,
  **Then** unique constraint violation.
- **Given** user borrado (CASCADE), **When** se elimina, **Then**
  todas sus transactions y balance se borran.
- **Given** la migration es idempotent, **When** se ejecuta 2x,
  **Then** segunda corrida no error (uses IF NOT EXISTS).
- **Given** rollback ejecuta, **When** tablas se borran, **Then**
  sin errors.

## Developer details

### Owning service

Database / migrations.

### Dependencies

- US-002: schema users (FK target).

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §6 —
  schema autoritativo.
- [`reglas.md`](../../reglas.md) — audit log inmutable.

### Implementación helper

```sql
CREATE OR REPLACE FUNCTION sparks_log(
  p_user_id UUID,
  p_type TEXT,
  p_amount INT,
  p_reason TEXT,
  p_related_id TEXT DEFAULT NULL,
  p_idempotency_key TEXT DEFAULT NULL,
  p_expires_at TIMESTAMPTZ DEFAULT NULL,
  p_metadata JSONB DEFAULT '{}'
) RETURNS TABLE(transaction_id UUID, new_balance INT) AS $$
DECLARE
  v_new_balance INT;
  v_tx_id UUID;
BEGIN
  -- Lock balance row + update atomically
  UPDATE sparks_balances
  SET current = current + p_amount,
      lifetime_earned = lifetime_earned + GREATEST(p_amount, 0),
      lifetime_spent = lifetime_spent + GREATEST(-p_amount, 0)
  WHERE user_id = p_user_id
  RETURNING current INTO v_new_balance;

  IF v_new_balance IS NULL THEN
    RAISE EXCEPTION 'sparks_balance_not_found';
  END IF;

  IF v_new_balance < 0 THEN
    RAISE EXCEPTION 'insufficient_sparks';
  END IF;

  -- Insert transaction
  INSERT INTO sparks_transactions
    (user_id, type, amount, balance_after, reason, related_id,
     idempotency_key, expires_at, metadata)
  VALUES
    (p_user_id, p_type, p_amount, v_new_balance, p_reason, p_related_id,
     p_idempotency_key, p_expires_at, p_metadata)
  RETURNING id INTO v_tx_id;

  -- Update last_transaction_id for race detection
  UPDATE sparks_balances
  SET last_transaction_id = v_tx_id
  WHERE user_id = p_user_id;

  RETURN QUERY SELECT v_tx_id, v_new_balance;
END;
$$ LANGUAGE plpgsql;
```

### Integration points

- US-046, US-047, US-048, US-049, US-050, US-053, US-054, US-056,
  US-057 (todas las stories del epic).

### Notas técnicas

- Append-only via RULE en lugar de revoke UPDATE/DELETE permissions:
  funciona aún si admin se equivoca con role.
- Trigger `expired` flag se setea con cron de US-057.
- Helper function `sparks_log` es la **única vía** de modificar
  balance — todos los handlers deberán llamarla.

## Definition of Done

- [ ] Migration `0XXX_sparks_schema.sql` aplicada en dev.
- [ ] Down migration funcional.
- [ ] Append-only rules verificados.
- [ ] Helper function `sparks_log` testeada.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests SQL: idempotency, constraints, CASCADE.
- [ ] Validation contra spec `sparks-system.md` §6.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para todas las stories de EPIC-04.*

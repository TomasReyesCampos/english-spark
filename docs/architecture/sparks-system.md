# Sparks System

> Sistema de tokens (Sparks) que controla el consumo de funciones de IA,
> ajusta dinámicamente costos sin alterar precios al usuario, y habilita
> gamificación. Es el corazón económico del producto.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador. Crítico para correctness
financiera. Antes de modificar cualquier cosa que toque balance,
transactions o costos: leer el doc completo y los tests obligatorios
(§14).
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

Este sistema tiene **alta criticidad financiera**. Los siguientes
puntos son **non-negotiables**:

- Schemas (§7) y operaciones (§8) son autoritativos. Tests con coverage
  ≥ 95% (ver `cross-cutting/testing-strategy.md` §4.2).
- Toda modificación de balance pasa por las funciones de §8. **No SQL
  ad-hoc en producción** sobre `user_sparks_balance` o
  `sparks_transactions`.
- `sparks_transactions` es **append-only**. NUNCA `UPDATE` o `DELETE`.
- Concurrencia siempre con `SELECT ... FOR UPDATE` sobre
  `user_sparks_balance` antes de modificar.
- Idempotency en cualquier operación con webhook o reintento (§8).

Decisiones cerradas en §17. Si encontrás una abierta, es bug del
diseño.

---

## 1. Visión general

Los Sparks son la unidad de consumo del producto. Cada función que
requiere cómputo significativo de IA tiene un costo en Sparks. Los
planes de suscripción incluyen una cantidad mensual de Sparks; los
usuarios pueden comprar packs adicionales.

Cumple cuatro funciones críticas:

1. **Control de costos:** límites duros sobre consumo de IA por
   usuario.
2. **Flexibilidad de pricing interno:** ajustar costo por operación sin
   tocar precio del plan al usuario.
3. **Educación del valor:** hace tangible para el usuario el costo de
   cada interacción intensiva en IA.
4. **Gamificación:** habilita recompensas naturales sin economía
   paralela.

---

## 2. Boundaries (responsabilidades)

### 2.1 Es responsable de

- Mantener balance de Sparks por usuario (plan + pack + bonus).
- Cobrar (`chargeOperation`) y reembolsar (`refundOperation`)
  operaciones.
- Asignación mensual del plan, expiración, rollover.
- Procesar compra de packs (vía webhooks de RevenueCat / Stripe).
- Otorgar bonos por gamificación (consumiendo eventos de
  `motivation-and-achievements`).
- Audit log inmutable de todas las transacciones.
- Reconciliación nocturna (verifica `SUM(transactions) == balance`).
- Emitir eventos de cambio de balance.

### 2.2 NO es responsable de

- **Pricing del plan al usuario** ($30 MXN, $100 MXN, etc.) — eso es
  configuración del producto, vive en `plans` table o config.
- **Costos en Sparks de cada operación** — son configuración (tabla
  `sparks_operation_costs`), no constantes hardcoded. Quien los ajusta
  es admin, no este sistema.
- **Detección de fraude / farming** — eso es `anti-fraud-system`. Este
  sistema **lee** `user_restrictions` y deniega operaciones si el
  usuario tiene restricción aplicable.
- **Ofrecer paywall al usuario** — eso es la UI consumiendo
  `getBalance` + producto.
- **Decidir qué task de IA ejecutar** — eso es el AI Gateway. Este
  sistema solo cobra el `taskId` que le piden.

### 2.3 Tensiones con otros sistemas

| Tensión | Resolución |
|---------|-----------|
| Costo de IA cambia → ajustar costo en Sparks | Admin actualiza `sparks_operation_costs` con notice de 30d. Este sistema lee la nueva tabla en runtime. |
| Usuario con `restriction = no_premium_features` quiere ejecutar operación | `chargeOperation` lee `user_restrictions` y deniega con `RESTRICTED_FEATURE` antes de cobrar. |
| Operación falla → necesito refund | `chargeOperation` retorna `operationId`; ante fallo, llamador invoca `refundOperation(operationId)`. |
| Webhook llega dos veces (Stripe retry) | Idempotency key = `payment_id`. Tabla `sparks_idempotency_keys` previene doble crédito. |

---

## 3. Modelo conceptual

### 3.1 ¿Qué es un Spark?

Un Spark es la unidad mínima de consumo. Equivale aproximadamente a:

- 1 minuto de conversación 1 a 1 con IA.
- 5 ejercicios cortos con análisis IA.

La equivalencia exacta es ajustable internamente sin afectar la
experiencia del usuario.

### 3.2 Origen de los Sparks

| Fuente | Identificador `source` | Notas |
|--------|------------------------|-------|
| Asignación mensual del plan | `plan_assignment` | Automática al inicio del ciclo. |
| Compra de pack | `pack_purchase` | Vía webhook RevenueCat/Stripe. |
| Recompensa de gamificación | `gamification_reward` | Streaks, logros, referidos. |
| Compensación del sistema | `system_credit` | Bugs, incidentes, manual override por admin. |
| Reembolso | `refund` | Cuando una operación falla. |

### 3.3 Tipos de Sparks (3 buckets)

```typescript
interface UserSparksBalance {
  user_id: string;
  plan_sparks: number;    // del plan mensual actual
  pack_sparks: number;    // de packs comprados
  bonus_sparks: number;   // de gamificación
  total_sparks: number;   // computed: plan + pack + bonus
}
```

**Reglas:**
- Cada bucket tiene reglas distintas de expiración (§3.4).
- Cobro consume buckets en orden: **plan → bonus → pack** (favorece al
  usuario: no quema sparks comprados primero).
- Bonus expira con el ciclo del plan (mismo trato).

### 3.4 Expiración

| Bucket | Regla de expiración |
|--------|--------------------|
| `plan_sparks` | Al cierre del ciclo: rollover hasta 2x el plan; el resto expira. |
| `pack_sparks` | 6 meses desde la compra del pack (FIFO por pack). |
| `bonus_sparks` | Al cierre del ciclo, mismas reglas que plan_sparks. |

---

## 4. Estructura de planes

### 4.1 Tabla de planes

| Plan ID | Precio MXN/mes | Sparks/mes | Rollover máx | Costo objetivo IA | Acceso premium |
|---------|---------------:|-----------:|-------------:|-------------------:|:---------------|
| `free` | $0 | 5 (one-time trial 50) | n/a | < $0.05 | Solo preassets |
| `basico` | $30 | 30 | 60 | < $0.30 | Preassets + Sparks |
| `pro` | $100 | 200 | 400 | < $1.50 | Preassets + Sparks + insights IA |
| `premium` | $250 | 600 | 1200 | < $4.00 | Todo Pro + roleplays personalizados |

### 4.2 Packs adicionales

| Pack ID | Sparks | Precio MXN | $/Spark efectivo | Validez |
|---------|-------:|-----------:|-----------------:|--------:|
| `pack_small` | 100 | $50 | $0.50 | 6 meses |
| `pack_medium` | 300 | $130 | $0.43 | 6 meses |
| `pack_large` | 1.000 | $350 | $0.35 | 6 meses |

### 4.3 Trial gratuito

`free` tiene un comportamiento especial: al activarse el trial (en
onboarding), recibe **50 Sparks one-time** (no 5). Estos 50 son
`bonus_sparks` con expiración a los 7 días.

```typescript
const TRIAL_CONFIG = {
  initial_sparks: 50,
  duration_days: 7,
  bucket: 'bonus_sparks',
  source_subtype: 'trial_initial',
};
```

---

## 5. Costos por operación

### 5.1 Tabla de costos canónica

Persistida en `sparks_operation_costs` (§7.5). Valores iniciales:

| `operation_id` | Costo (Sparks) | Categoría | Notas |
|----------------|---------------:|-----------|-------|
| `conversation_minute` | 1 / minuto | realtime | Limitado a 15 min por sesión |
| `analyze_long_audio` | 2 | realtime | Audio > 2 min |
| `generate_personalized_roleplay` | 5 | realtime | Solo si no hay match en biblioteca |
| `pronunciation_session` | 1 / 10 ejercicios | batch | Eficiente |
| `weekly_summary_generation` | 3 | batch | Generado domingos |
| `generate_initial_roadmap` | 0 (free, incluido en onboarding) | one-time | Una vez por usuario |
| `nightly_roadmap_update` | 0 | batch | Incluido en plan |
| `preasset_access` | 0 | n/a | Ilimitado |
| `morning_message` | 0 | batch | Incluido |
| `test_out_attempt` | 2 | one-time | Cooldown 7d (ver pedagogical §4.5) |
| `ai_assistant_conversation` | 0 (incluido en plan, hasta cierto cap) | realtime | Cap en `ai-assistant-system` |

### 5.2 Categorías

```typescript
type OperationCategory =
  | 'realtime'   // user-facing immediate, cobro previo crítico
  | 'batch'      // job nocturno, sin urgencia
  | 'one_time'   // una vez por evento (trial, assessment, test_out)
  | 'free';      // sin costo en Sparks
```

### 5.3 Principio de costo

Sparks reflejan **operaciones intensivas en IA en tiempo real o
cercanas**. Operaciones batch nocturnas, generación de mensajes,
análisis SQL están **incluidas** en el plan y no consumen Sparks
individualmente.

### 5.4 Costos ajustables

Tabla mutable. Ajustes siguen el procedimiento de §13. Cualquier
cambio:

- 30 días de notice a usuarios activos.
- Aplica solo a operaciones futuras desde la fecha del cambio.
- Documentado en `docs/decisions/sparks-pricing-changes.md`.

---

## 6. Estado y ciclos

### 6.1 Ciclo de facturación

```
┌─────────────────────────────────────────────────────────┐
│ Day 0: Cycle starts                                     │
│   - Asignar plan_sparks del plan actual                 │
│   - Calcular rollover de prev cycle (cap 2x plan)       │
│   - Crear billing_cycle row                             │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Days 1-29: User uses Sparks                             │
│   - Cobros via chargeOperation                          │
│   - Compras de pack acreditan pack_sparks               │
│   - Recompensas de gamificación acreditan bonus_sparks  │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Day 30: Cycle ends                                      │
│   - billing_cycle.status = 'closed'                     │
│   - Calcular sparks_unused                              │
│   - Expirar plan_sparks excedente del rollover          │
│   - Expirar bonus_sparks (todos)                        │
│   - Day 30 = Day 0 of next cycle                        │
└─────────────────────────────────────────────────────────┘
```

### 6.2 Cambio de plan mid-cycle

| Caso | Comportamiento |
|------|----------------|
| Upgrade (Pro → Premium) | Inmediato. Se acreditan los Sparks adicionales (proporcional al tiempo restante del ciclo) en `plan_sparks`. |
| Downgrade (Premium → Pro) | Efectivo al final del ciclo actual. Sparks comprados no afectan. |
| Cancel suscripción | Efectivo al final del ciclo actual. Permite uso hasta entonces. Pack_sparks siguen vigentes hasta su expiración propia. |

### 6.3 Edge case: usuario en `free` activa trial

- Si un usuario en `free` (sin trial) inicia onboarding: recibe los 50
  Sparks de trial como `bonus_sparks` con expiración 7d.
- Si pasa el trial: balance vuelve a 0 (los bonus expiran).
- No puede reactivar trial (verificado por `student_profile.trial_status`).

### 6.4 Pack purchase mid-cycle

- Inmediato. `pack_sparks += pack.amount` con expiración a 6 meses.
- No interactúa con el ciclo del plan.

---

## 7. Modelo de datos

### 7.1 Tabla de balance

```sql
CREATE TABLE user_sparks_balance (
  user_id           UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  plan_sparks       INT NOT NULL DEFAULT 0 CHECK (plan_sparks >= 0),
  pack_sparks       INT NOT NULL DEFAULT 0 CHECK (pack_sparks >= 0),
  bonus_sparks      INT NOT NULL DEFAULT 0 CHECK (bonus_sparks >= 0),
  total_sparks      INT GENERATED ALWAYS AS
                      (plan_sparks + pack_sparks + bonus_sparks) STORED,
  current_cycle_id  UUID REFERENCES billing_cycles(id),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_balance_low ON user_sparks_balance(total_sparks)
  WHERE total_sparks < 20;
```

### 7.2 Tabla de transacciones (audit log inmutable)

```sql
CREATE TABLE sparks_transactions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  amount          INT NOT NULL,                     -- + ingreso, - consumo
  bucket_changes  JSONB NOT NULL,                   -- {"plan": -10, "pack": 0, "bonus": 0}
  source          TEXT NOT NULL CHECK (source IN (
                    'plan_assignment',
                    'pack_purchase',
                    'gamification_reward',
                    'system_credit',
                    'operation_charge',
                    'refund',
                    'expiration',
                    'correction'
                  )),
  source_subtype  TEXT,
  operation_id    UUID,                              -- ref a la operation que generó el cargo
  reference_id    UUID,                              -- ref a otra transaction (refund → original)
  balance_before  INT NOT NULL,
  balance_after   INT NOT NULL,
  metadata        JSONB NOT NULL DEFAULT '{}',
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by      TEXT NOT NULL DEFAULT 'system'    -- 'system' | 'admin:<id>' | 'webhook:<provider>'
);

CREATE INDEX idx_tx_user_date ON sparks_transactions(user_id, created_at DESC);
CREATE INDEX idx_tx_operation ON sparks_transactions(operation_id) WHERE operation_id IS NOT NULL;
CREATE INDEX idx_tx_reference ON sparks_transactions(reference_id) WHERE reference_id IS NOT NULL;
CREATE INDEX idx_tx_source_date ON sparks_transactions(source, created_at DESC);
```

**Reglas inviolables:**
- Append-only. No `UPDATE`. No `DELETE` (excepto retention si se decide
  archivar después de N años).
- `balance_before` + `amount` = `balance_after` (verificable post-hoc).
- `bucket_changes` debe sumar a `amount`.

### 7.3 Tabla de packs comprados

```sql
CREATE TABLE sparks_packs_purchased (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  pack_type       TEXT NOT NULL,                    -- 'pack_small' | 'pack_medium' | ...
  sparks_amount   INT NOT NULL,
  sparks_remaining INT NOT NULL,                    -- decrementa al consumir
  price_paid      DECIMAL(10,2) NOT NULL,
  currency        TEXT NOT NULL,
  payment_id      TEXT NOT NULL,                    -- Stripe / RevenueCat ID
  payment_provider TEXT NOT NULL CHECK (payment_provider IN (
                    'apple', 'google', 'stripe', 'mercadopago'
                  )),
  purchased_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at      TIMESTAMPTZ NOT NULL,
  fully_consumed_at TIMESTAMPTZ,
  CHECK (sparks_remaining >= 0 AND sparks_remaining <= sparks_amount)
);

CREATE INDEX idx_packs_user_active ON sparks_packs_purchased(user_id)
  WHERE sparks_remaining > 0 AND expires_at > now();
CREATE INDEX idx_packs_payment ON sparks_packs_purchased(payment_id);
```

### 7.4 Tabla de ciclos de facturación

```sql
CREATE TABLE billing_cycles (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  plan_id         TEXT NOT NULL,
  cycle_start     TIMESTAMPTZ NOT NULL,
  cycle_end       TIMESTAMPTZ NOT NULL,
  sparks_assigned INT NOT NULL,
  sparks_unused   INT,                              -- calculado al cierre
  status          TEXT NOT NULL CHECK (status IN ('active', 'closed', 'cancelled'))
);

CREATE INDEX idx_cycles_user_active ON billing_cycles(user_id)
  WHERE status = 'active';
```

### 7.5 Tabla de configuración de costos

```sql
CREATE TABLE sparks_operation_costs (
  operation_id    TEXT PRIMARY KEY,
  cost_sparks     NUMERIC(10,2) NOT NULL CHECK (cost_sparks >= 0),
  category        TEXT NOT NULL CHECK (category IN (
                    'realtime', 'batch', 'one_time', 'free'
                  )),
  unit            TEXT NOT NULL DEFAULT 'fixed',    -- 'fixed' | 'per_minute' | 'per_n'
  per_n_value     INT,                              -- si unit='per_n'
  notes           TEXT,
  effective_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
  superseded_by   TEXT REFERENCES sparks_operation_costs(operation_id)
);

-- Seed inicial
INSERT INTO sparks_operation_costs (operation_id, cost_sparks, category, unit, per_n_value)
VALUES
  ('conversation_minute',            1,    'realtime',  'per_minute',  NULL),
  ('analyze_long_audio',             2,    'realtime',  'fixed',       NULL),
  ('generate_personalized_roleplay', 5,    'realtime',  'fixed',       NULL),
  ('pronunciation_session',          1,    'batch',     'per_n',       10),
  ('weekly_summary_generation',      3,    'batch',     'fixed',       NULL),
  ('test_out_attempt',               2,    'one_time',  'fixed',       NULL),
  ('generate_initial_roadmap',       0,    'free',      'fixed',       NULL),
  ('nightly_roadmap_update',         0,    'free',      'fixed',       NULL),
  ('preasset_access',                0,    'free',      'fixed',       NULL),
  ('morning_message',                0,    'free',      'fixed',       NULL),
  ('ai_assistant_conversation',      0,    'free',      'fixed',       NULL);
```

### 7.6 Tabla de idempotency keys

```sql
CREATE TABLE sparks_idempotency_keys (
  key             TEXT PRIMARY KEY,                  -- ej: 'webhook:stripe:pi_xxx'
  user_id         UUID REFERENCES users(id),
  operation_type  TEXT NOT NULL,                     -- 'pack_purchase' | 'charge' | ...
  result_tx_id    UUID REFERENCES sparks_transactions(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at      TIMESTAMPTZ NOT NULL DEFAULT (now() + interval '90 days')
);

CREATE INDEX idx_idempotency_expires ON sparks_idempotency_keys(expires_at);
```

### 7.7 Cleanup de tablas

- `sparks_idempotency_keys`: cron diario borra rows con `expires_at <
  now()`.
- `sparks_transactions`: indefinido (compliance financiero).
- `billing_cycles`: indefinido.
- `sparks_packs_purchased`: indefinido (necesario para audit incluso
  después de expirar).

---

## 8. API contracts

### 8.1 `chargeOperation`

**Llamado por:** AI Gateway (antes de invocar LLM), `pedagogical-system`
(test-out), otros sistemas que cobran Sparks.

**Request:**

```typescript
interface ChargeOperationRequest {
  user_id: string;
  operation_id: string;          // ej: 'conversation_minute'
  amount?: number;               // si unit='per_minute', cantidad de minutos
  idempotency_key?: string;      // opcional, recomendado para realtime
  metadata?: Record<string, unknown>;
}
```

**Response:**

```typescript
interface ChargeOperationResponse {
  charge_id: string;             // UUID, usar para refund
  cost_sparks: number;
  bucket_breakdown: {
    plan: number;
    bonus: number;
    pack: number;
  };
  balance_after: number;
}
```

**Errores:**

| Código | Causa | Acción del llamador |
|--------|-------|---------------------|
| `INSUFFICIENT_SPARKS` | Balance < cost | Mostrar paywall |
| `RESTRICTED_FEATURE` | `user_restrictions` deniega | Mostrar mensaje + apelación |
| `OPERATION_NOT_FOUND` | `operation_id` no existe en `sparks_operation_costs` | Bug en llamador, escalar |
| `IDEMPOTENT_REPLAY` | `idempotency_key` ya procesada | Devolver mismo resultado, no double-charge |
| `INTERNAL_ERROR` | Error inesperado | Retry con backoff |

**Implementación (pseudo):**

```typescript
async function chargeOperation(req: ChargeOperationRequest): Promise<ChargeOperationResponse> {
  // 1. Idempotency check
  if (req.idempotency_key) {
    const cached = await getIdempotencyResult(req.idempotency_key);
    if (cached) return cached;
  }

  // 2. Resolver costo
  const config = await getOperationCost(req.operation_id);
  if (!config) throw new Error('OPERATION_NOT_FOUND');
  const cost = config.unit === 'per_minute'
    ? config.cost_sparks * (req.amount ?? 1)
    : config.unit === 'per_n'
    ? Math.ceil((req.amount ?? 1) / config.per_n_value) * config.cost_sparks
    : config.cost_sparks;

  // 3. Verificar restricciones
  const restrictions = await getActiveRestrictions(req.user_id);
  if (restrictionDeniesOperation(restrictions, config.category)) {
    throw new Error('RESTRICTED_FEATURE');
  }

  // 4. Transacción atómica
  return await db.transaction(async (tx) => {
    // Lock balance row
    const balance = await tx.query(
      `SELECT * FROM user_sparks_balance WHERE user_id = $1 FOR UPDATE`,
      [req.user_id]
    );

    if (balance.total_sparks < cost) {
      throw new Error('INSUFFICIENT_SPARKS');
    }

    // Decrementar en orden: plan → bonus → pack
    const breakdown = deductInOrder(balance, cost, ['plan_sparks', 'bonus_sparks', 'pack_sparks']);

    await tx.query(`
      UPDATE user_sparks_balance
      SET plan_sparks = plan_sparks - $1,
          bonus_sparks = bonus_sparks - $2,
          pack_sparks = pack_sparks - $3,
          updated_at = now()
      WHERE user_id = $4
    `, [breakdown.plan, breakdown.bonus, breakdown.pack, req.user_id]);

    // Si tocó pack_sparks, decrementar packs específicos (FIFO)
    if (breakdown.pack > 0) {
      await consumePacksFIFO(tx, req.user_id, breakdown.pack);
    }

    // Crear transaction
    const chargeId = crypto.randomUUID();
    await tx.query(`
      INSERT INTO sparks_transactions
        (id, user_id, amount, bucket_changes, source, source_subtype,
         operation_id, balance_before, balance_after, metadata)
      VALUES ($1, $2, $3, $4, 'operation_charge', $5, $1, $6, $7, $8)
    `, [chargeId, req.user_id, -cost, JSON.stringify({
      plan: -breakdown.plan, bonus: -breakdown.bonus, pack: -breakdown.pack,
    }), req.operation_id, balance.total_sparks, balance.total_sparks - cost, req.metadata ?? {}]);

    // Persistir idempotency key si aplica
    if (req.idempotency_key) {
      await persistIdempotencyResult(tx, req.idempotency_key, chargeId);
    }

    // Emitir evento
    await emitDomainEvent('sparks.balance_changed', {
      user_id: req.user_id,
      previous_balance: balance,
      new_balance: { ...balance, total: balance.total_sparks - cost },
      reason: 'operation_charge',
      amount: -cost,
      operation_id: chargeId,
    });

    // Si balance < 20% del plan, emitir balance_low
    if (shouldEmitLowBalance(balance, cost)) {
      await emitDomainEvent('sparks.balance_low', { /* ... */ });
    }

    return {
      charge_id: chargeId,
      cost_sparks: cost,
      bucket_breakdown: breakdown,
      balance_after: balance.total_sparks - cost,
    };
  });
}
```

### 8.2 `refundOperation`

**Llamado por:** AI Gateway cuando una operación falla post-charge.

**Request:**

```typescript
interface RefundOperationRequest {
  charge_id: string;             // ID retornado por chargeOperation
  reason: string;                // ej: 'llm_provider_unavailable'
  metadata?: Record<string, unknown>;
}
```

**Response:**

```typescript
interface RefundOperationResponse {
  refund_id: string;
  amount_refunded: number;
  balance_after: number;
}
```

**Errores:**

| Código | Causa |
|--------|-------|
| `CHARGE_NOT_FOUND` | `charge_id` no corresponde a transaction |
| `ALREADY_REFUNDED` | charge ya tiene refund asociado |
| `NOT_REFUNDABLE` | charge es de tipo no-refundable (ej: trial assignment) |

**Reglas:**
- Refund retorna Sparks **al mismo bucket** del que se cobraron.
- Si el bucket cambió de estado (ej: bonus_sparks expiraron entre
  charge y refund): el refund se acredita a `plan_sparks` con nota.
- No se permite refund parcial. Es todo o nada.

### 8.3 `awardBonus`

**Llamado por:** `motivation-and-achievements` al desbloquear logros,
streaks, daily goals; admin para créditos manuales.

**Request:**

```typescript
interface AwardBonusRequest {
  user_id: string;
  amount: number;                // > 0
  source_subtype: string;        // ej: 'streak_7_days', 'achievement_first_steps'
  reference_id?: string;         // ej: achievement_id
  expiry_policy: 'cycle_end' | 'never' | 'days_30';
  idempotency_key?: string;
}
```

**Response:**

```typescript
interface AwardBonusResponse {
  award_id: string;
  amount: number;
  balance_after: number;
}
```

**Reglas:**
- `expiry_policy = 'cycle_end'` (default): expira con el ciclo del plan.
- `'never'`: no expira (raro, solo para créditos especiales).
- `'days_30'`: expira a 30 días (eventos especiales).

### 8.4 `processPackPurchase`

**Llamado por:** webhook handlers (Stripe, RevenueCat, MercadoPago).

**Request:**

```typescript
interface ProcessPackPurchaseRequest {
  user_id: string;
  pack_type: string;             // 'pack_small' | 'pack_medium' | 'pack_large'
  payment_id: string;            // ID del proveedor (Stripe pi_xxx, etc.)
  payment_provider: 'apple' | 'google' | 'stripe' | 'mercadopago';
  amount_paid: number;
  currency: string;
}
```

**Response:**

```typescript
interface ProcessPackPurchaseResponse {
  pack_id: string;
  sparks_credited: number;
  expires_at: string;
}
```

**Idempotency obligatoria:** `idempotency_key = 'pack_purchase:' +
payment_id`. Webhook duplicado retorna mismo resultado sin
double-credit.

### 8.5 `assignMonthlyPlanSparks`

**Llamado por:** job al cierre de cada `billing_cycle`.

**Request:**

```typescript
interface AssignMonthlyPlanSparksRequest {
  user_id: string;
  plan_id: string;
  new_cycle_id: string;
}
```

**Response:**

```typescript
interface AssignMonthlyPlanSparksResponse {
  sparks_assigned: number;
  rollover_amount: number;
  expired_amount: number;
}
```

### 8.6 `getBalance`

**Llamado por:** UI, otros sistemas que necesitan info no-modificable.

**Request:**

```typescript
interface GetBalanceRequest {
  user_id: string;
}
```

**Response:**

```typescript
interface GetBalanceResponse {
  plan_sparks: number;
  pack_sparks: number;
  bonus_sparks: number;
  total_sparks: number;
  packs: Array<{
    pack_id: string;
    pack_type: string;
    sparks_remaining: number;
    expires_at: string;
  }>;
  cycle: {
    cycle_id: string;
    starts_at: string;
    ends_at: string;
    plan_id: string;
  };
}
```

### 8.7 `getRecentTransactions`

```typescript
interface GetRecentTransactionsRequest {
  user_id: string;
  limit?: number;                // default 30
  offset?: number;
  filter_source?: string;
}

interface GetRecentTransactionsResponse {
  transactions: Array<{
    id: string;
    amount: number;
    source: string;
    source_subtype: string | null;
    created_at: string;
    description: string;          // human-readable
  }>;
  has_more: boolean;
}
```

---

## 9. Eventos emitidos

Detalle del shape en `cross-cutting/data-and-events.md` §5.4.

| Evento | Cuándo se emite |
|--------|----------------|
| `sparks.balance_changed` | Cualquier cambio de balance (charge, refund, bonus, expire, plan assignment) |
| `sparks.balance_low` | Cuando balance cae a 20% del plan asignado |
| `sparks.depleted` | Balance llega a 0 e intentan operación |
| `sparks.pack_expiring` | 7 días antes de expirar un pack con `sparks_remaining > 0` |
| `sparks.pack_expired` | Al expirar un pack con sparks remaining (loss reporting) |

---

## 10. Edge cases (tests obligatorios)

### 10.1 Concurrencia

1. **Doble charge concurrente del mismo user:** uno gana
   `FOR UPDATE`, el otro espera. Si el segundo deja insufficient
   balance, retorna `INSUFFICIENT_SPARKS`. Ambas transactions
   correctamente persistidas o ninguna.
2. **Charge + refund concurrentes:** refund espera FOR UPDATE; ejecuta
   después. Balance correcto.
3. **Webhook duplicado en <1s:** idempotency key bloquea segundo. Mismo
   resultado retornado.

### 10.2 Estados de borde

4. **Balance == cost exacto:** charge succeeds, balance queda 0.
5. **Insufficient: cost = 10, balance = 9:** `INSUFFICIENT_SPARKS`
   sin tocar balance.
6. **Pack expira durante operación in-flight:** charge ya tomó FOR
   UPDATE; consume del pack válido al inicio. Si pack expira segundos
   después, no afecta el charge en curso.
7. **Pack con `sparks_remaining = 1` y user pide charge de 2 que toca
   pack:** charge consume 1 del pack + lo restante de otro bucket. Si
   no hay otro bucket: insufficient.

### 10.3 Plan changes

8. **Upgrade Pro → Premium mid-cycle:** prorate. Si quedan 15 días
   del ciclo actual de Pro (200 Sparks/mes), el upgrade añade `(600 -
   200) * (15/30) = 200 Sparks` al `plan_sparks` actual.
9. **Downgrade Premium → Pro:** efectivo al fin del ciclo. Mid-cycle
   no cambia.
10. **Cancel suscripción:** cycle se marca `cancelled`. Permite uso
    hasta `cycle_end`. No se asignan plan_sparks del próximo ciclo.

### 10.4 Refunds

11. **Refund de un charge ya refunded:** `ALREADY_REFUNDED`.
12. **Refund de un charge cuando el bucket original (bonus) ya
    expiró:** acredita a `plan_sparks` con metadata indicando
    bucket_swap.
13. **Refund de pack purchase (raro, solo via admin):** decrementa
    `pack_sparks`, marca pack como refunded. Sólo permitido por admin
    + comando explícito.

### 10.5 Trial

14. **Trial de 50 Sparks: usuario los gasta y pasa al día 8:** balance
    queda 0. Usuario puede practicar preassets gratis. No reactiva
    trial.
15. **Trial activo + usuario compra plan Pro mid-trial:** plan
    activado inmediatamente, plan_sparks = 200 + bonus restante (50 -
    consumido). Bonus expira con el ciclo del Pro.

### 10.6 Reconciliation

16. **Sum de transactions diverge de balance:** alerta crítica.
    Reconciliation job recalcula desde transactions y corrige balance
    con transaction `correction`.

---

## 11. Reconciliation y audit

### 11.1 Daily reconciliation job

Cron diario 03:00 UTC. Verifica integridad por user:

```typescript
async function dailyReconciliation() {
  // Para cada user activo en últimos 30 días
  const users = await getActiveUsers(30);

  for (const userId of users) {
    const balance = await getBalance(userId);

    // Sumar transactions del usuario
    const txSum = await db.query(`
      SELECT COALESCE(SUM(amount), 0) as sum_amount,
             COALESCE(SUM((bucket_changes->>'plan')::int), 0) as sum_plan,
             COALESCE(SUM((bucket_changes->>'bonus')::int), 0) as sum_bonus,
             COALESCE(SUM((bucket_changes->>'pack')::int), 0) as sum_pack
      FROM sparks_transactions
      WHERE user_id = $1
    `, [userId]);

    const expected = {
      plan_sparks: txSum.sum_plan,
      bonus_sparks: txSum.sum_bonus,
      pack_sparks: txSum.sum_pack,
      total_sparks: txSum.sum_amount,
    };

    // Comparar con balance actual
    if (!matchesExactly(balance, expected)) {
      await alertReconciliationMismatch(userId, balance, expected);
      // Auto-corrección si diff es < 5 Sparks (probablemente bug menor)
      // Sino: alerta humano
    }
  }
}
```

### 11.2 Audit reportes mensuales

Primer día del mes, reporte automático:

```
═══════════════════════════════════════════
   SPARKS AUDIT REPORT - 2026-04
═══════════════════════════════════════════

USERS:                            10,432
ACTIVE BALANCES TOTAL:            1,234,567 Sparks

TRANSACTIONS:
  plan_assignment:    2,345 (acreditando 234,500 Sparks)
  pack_purchase:        678 (acreditando 89,400 Sparks)
  gamification_reward: 12,345 (acreditando 56,789 Sparks)
  operation_charge:   45,678 (debitando 234,567 Sparks)
  refund:               123 (acreditando 1,234 Sparks)
  expiration:         5,432 (debitando 23,456 Sparks)

REVENUE FROM PACKS:               $5,432 USD

COST OF AI:                       $1,234 USD
COST PER SPARK CONSUMED:          $0.0053
SPARKS SOLD VS SPENT RATIO:       1.4

ANOMALIES:
  Users with reconciliation mismatch:  0  ✓
  Users with consumption > $5/day:     2  (review)
  Negative balances detected:          0  ✓
```

### 11.3 Audit log inmutable

`sparks_transactions` es **append-only**. Reglas en DB:

```sql
-- Trigger que previene UPDATE/DELETE
CREATE OR REPLACE FUNCTION prevent_tx_modification() RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'sparks_transactions is append-only. Use a correction transaction instead.';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_no_update_tx BEFORE UPDATE ON sparks_transactions
  FOR EACH ROW EXECUTE FUNCTION prevent_tx_modification();

CREATE TRIGGER trg_no_delete_tx BEFORE DELETE ON sparks_transactions
  FOR EACH ROW EXECUTE FUNCTION prevent_tx_modification();
```

Excepción: cleanup script con flag `ALLOW_TX_DELETE` para retention
después de N años. Default: nunca.

---

## 12. Gamificación con Sparks

### 12.1 Recompensas estándar

(Detalle en `motivation-and-achievements.md` §5.2.) Cada logro tiene
`sparks_reward` en su definición. Engine de logros llama `awardBonus`.

### 12.2 Anti-abuso

- Cap de bonus: 100 Sparks por usuario por mes en bonos de
  gamificación, EXCEPTO referral pagos (50 cada uno, sin cap).
- Detección de cuentas duplicadas: ver `anti-fraud-system`.
- Auditoría manual de usuarios con > 200 Sparks bonus en mes.

### 12.3 Por qué Sparks como recompensa

Alto valor percibido, bajo costo real:
- Para el usuario: 30 Sparks bonus = $15 MXN de valor percibido.
- Para el negocio: el costo real de generar esos 30 Sparks de uso es
  $1-2 USD ($20-40 MXN).
- Multiplicador de retention >> costo.

---

## 13. Manejo de cambios de costo

### 13.1 Triggers de ajuste

- Costo real de la operación cambia más de 20% respecto a baseline.
- Nuevo modelo más barato/caro reemplaza al actual.
- Comportamiento de uso revela mal calibración.

### 13.2 Procedimiento

1. **Análisis:** propuesta documentada con datos.
2. **Notificación:** email + in-app banner con 30 días de notice.
3. **Aplicación:** insert nueva fila en `sparks_operation_costs` con
   `effective_from = now() + 30d`. Old row se mantiene con
   `superseded_by` apuntando a la nueva.
4. **Operaciones en curso:** mantienen costo previo si el charge ya
   sucedió antes de `effective_from`.

### 13.3 Comunicación al usuario

```
"A partir del 1 de junio, las conversaciones 1 a 1 con IA costarán
1.2 Sparks por minuto en lugar de 1 Spark por minuto, debido a
mejoras significativas en la calidad del modelo. Tu plan mensual
sigue siendo el mismo. Esto significa aproximadamente 4 minutos
menos de conversación al mes en el plan Pro."
```

### 13.4 Compensaciones por cambios significativos

Si un cambio reduce significativamente valor para usuarios existentes
(>50% más caro):

- Sparks bonus de compensación a usuarios afectados.
- Grandfathering temporal (precio antiguo) para usuarios anteriores.
- Mejora simultánea de feature para balancear.

### 13.5 Provisioned throughput (futuro)

Para tareas críticas y alto volumen, evaluar contratos de capacidad
reservada con proveedores LLM. Da:
- Precio fijo durante contrato.
- Capacidad garantizada.
- Aislamiento de cambios de precio.

Solo justificable a escala (post-soft-launch).

---

## 14. Plan de implementación

### 14.1 Sprint 1 (semana 1-2): Foundation

- Schemas §7 completos en migración inicial.
- Seed `sparks_operation_costs` con tabla §5.1.
- `getBalance`, `chargeOperation`, `refundOperation` implementados.
- Append-only trigger en `sparks_transactions`.
- Tests unitarios ≥ 95% coverage en estas funciones.
- Tests de concurrencia (§10.1).

### 14.2 Sprint 2 (semana 3-4): Integration

- Integración con AI Gateway (cobro previo a llamadas LLM).
- `awardBonus` integrado con `motivation-and-achievements`.
- UI de balance visible en cliente.
- Indicación previa de costo en operaciones realtime.

### 14.3 Sprint 3 (semana 5-6): Pagos

- `processPackPurchase` con webhooks de RevenueCat (in-app) y Stripe
  (web).
- Webhook signature verification.
- Idempotency en webhooks.
- UI de tienda de packs.

### 14.4 Sprint 4 (semana 7-8): Ciclos

- `assignMonthlyPlanSparks` job.
- Cron de cierre de ciclo (expiración + rollover).
- Cron de cleanup de idempotency_keys.
- Tests de edge cases §10.

### 14.5 Sprint 5 (semana 9-10): Audit

- Daily reconciliation job.
- Audit log inmutable verificado.
- Audit reportes mensuales.
- Dashboard interno con métricas (§15).
- Alertas críticas configuradas.

---

## 15. Métricas

### 15.1 Métricas críticas

| Métrica | Fuente | Target |
|---------|--------|--------|
| Cost per Spark consumed | reconciliation report | < $0.005 |
| Sparks consumption rate por plan | aggregations | Pro > 50%, Premium > 60% |
| Pack conversion rate | aggregations | > 5% de usuarios pago |
| Sparks por sesión activa | events | > 1 por sesión engagement |
| Refund rate | transactions | < 1% |
| Reconciliation mismatch rate | daily job | 0% (cualquier > 0 = alerta) |
| Time to credit pack post-payment | webhook latency | < 60s para 99% de casos |

### 15.2 Alertas críticas

- Reconciliation mismatch en cualquier usuario: SEV-1 inmediato.
- Usuario individual con consumo > $5 USD/día: review (potencial bug
  o abuso).
- Pack purchase webhook fallando consistentemente: SEV-1.
- Caída > 20% en compras de packs vs semana anterior: review (bug en
  flujo de compra?).
- Spike en `INSUFFICIENT_SPARKS` errors > 5x baseline: posible
  mal calibrado un costo.
- Discrepancia en `total_sparks` calculated column vs sum: bug grave.

---

## 16. Aspectos legales y financieros

### 16.1 Naturaleza contable

Sparks comprados (packs) son **pasivo contable** hasta consumir.
Sparks asignados con plan no se contabilizan como pasivo (vienen con
suscripción).

**Revenue recognition:**
- Pack: se reconoce al **consumir** los Sparks, no al venderlos
  (US GAAP / IFRS).
- Sparks no consumidos al expirar (6 meses): se reconocen como
  revenue en ese momento ("breakage").

Para MVP esto puede simplificarse, pero debe ajustarse con contabilidad
formal post-validación.

### 16.2 Reembolsos

- Sparks de plan: no reembolsables individualmente.
- Sparks de packs: reembolsables proporcionalmente si:
  - < 14 días de la compra.
  - < 30% consumido.
  - Política de la store permite (Apple/Google).

### 16.3 T&C (cláusulas obligatorias)

(Documentadas en `business/legal-compliance.md` §2.2.)

- Sparks no son moneda de curso legal.
- Sparks no son transferibles entre usuarios.
- Sparks no se pueden convertir a dinero.
- Empresa puede ajustar costo en Sparks con notice de 30 días.

---

## 17. Decisiones cerradas

### 17.1 Transferencia entre usuarios (gifting): **NO** ✓

**Razón:** complejidad anti-abuse alta (vector de farming),
beneficio marginal en viralidad. Reconsiderar post-validación.

### 17.2 Sparks negativos (deuda): **NO** ✓

**Razón:** complejidad alta, riesgo de churn cuando usuarios
recuerden que "deben" Sparks. No alinea con el modelo conceptual.

### 17.3 Programa de fidelidad multiplicador: **NO en MVP** ✓

**Razón:** no agrega valor pre-validación. Reconsiderar post-soft-launch
si retention de cohorts antiguos es alta.

### 17.4 Sparks comunitarios (sorteos): **NO en MVP** ✓

**Razón:** marketing tool prematuro. Después de soft launch.

### 17.5 Gift cards: **NO en MVP** ✓

**Razón:** vendor de gift cards (Stripe Gift, etc.) agrega complejidad.
Considerar para campaña navideña año 2.

---

## 18. Casos de uso ilustrativos

### 18.1 Usuario Plan Pro típico

- Mes 1: recibe 200 Sparks. Usa 180 en conversaciones (15 sesiones de
  12 min). Sobran 20.
- Inicio mes 2: rollover de 20 + 200 nuevos = 220.
- Mes 2: usa 200 conversaciones, 5 pronunciation. Sobran 15.
- Inicio mes 3: rollover de 15 + 200 = 215.

Todo dentro del plan, sin compras.

### 18.2 Power user

- Plan Pro 200 Sparks/mes. Usa todo en primera semana.
- Compra Pack Mediano: +300 Sparks por $130.
- Costo total mensual: $230 MXN vs $250 del Plan Premium.
- **Análisis interno:** mejor sugerir upgrade a Premium directamente.

### 18.3 Plan Básico explorando conversación

- Plan Básico no incluye conversación normalmente.
- Tiene 30 Sparks plan + 50 bonus (referidos) = 80.
- Compra sesión de 5 min: -5 Sparks. Quedan 75.
- Permite upselling: "¿el Plan Pro te daría 200 Sparks/mes?"

---

## 19. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`ai-gateway-strategy.md`](ai-gateway-strategy.md) | Llama `chargeOperation` antes de invocar LLM. |
| [`anti-fraud-system.md`](anti-fraud-system.md) | Provee `user_restrictions` que `chargeOperation` consulta. |
| [`../product/pedagogical-system.md`](../product/pedagogical-system.md) | Cobra Sparks por test-out (§4.5 de pedagogical). |
| [`../product/motivation-and-achievements.md`](../product/motivation-and-achievements.md) | Llama `awardBonus` para recompensas. |
| [`../product/student-profile-and-assessment.md`](../product/student-profile-and-assessment.md) | Trial de 50 Sparks. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.4 | Catálogo de eventos `sparks.*`. |
| [`../cross-cutting/security-threat-model.md`](../cross-cutting/security-threat-model.md) §3.7 | Amenazas y mitigaciones de Sparks. |
| [`../cross-cutting/testing-strategy.md`](../cross-cutting/testing-strategy.md) §4.2 | Tests obligatorios (≥ 95% coverage). |
| [`../decisions/ADR-003-sparks-system.md`](../decisions/ADR-003-sparks-system.md) | Decisión arquitectónica. |
| [`../business/plan_de_negocio.docx`](../business/plan_de_negocio.docx) | Pricing. |

---

*Documento vivo. Cualquier ajuste a costos, schemas o estructura de
planes debe reflejarse aquí inmediatamente y comunicarse a usuarios.*

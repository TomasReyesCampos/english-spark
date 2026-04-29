# Sparks System

> Sistema de tokens (Sparks) que controla el consumo de funciones de IA,
> ajusta dinámicamente costos sin alterar precios al usuario, y habilita
> gamificación. Es el corazón económico del producto.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —

---

## 1. Visión general

Los Sparks son la unidad de consumo del producto. Cada función que requiere
cómputo significativo de IA tiene un costo en Sparks. Los planes de
suscripción incluyen una cantidad mensual de Sparks; los usuarios pueden
comprar packs adicionales si los requieren.

Este sistema cumple cuatro funciones críticas:

1. **Control de costos:** establece límites duros sobre cuánto cómputo de
   IA puede consumir un usuario, evitando que costos variables superen el
   revenue de su plan.
2. **Flexibilidad de precios:** permite ajustar el costo en Sparks de cada
   operación según fluctúen los costos de proveedores, sin tocar el precio
   mensual al usuario.
3. **Educación del valor:** hace tangible para el usuario el valor de cada
   interacción intensiva en IA, generando uso más consciente.
4. **Gamificación:** habilita recompensas y mecánicas de retención que se
   sienten naturales sin agregar economía paralela.

---

## 2. Modelo conceptual

### 2.1 ¿Qué es un Spark?

Un Spark es la unidad mínima de consumo. Representa aproximadamente
**1 minuto de conversación 1 a 1 con IA** o **5 ejercicios cortos con
análisis IA**.

La equivalencia exacta es ajustable internamente sin afectar la experiencia
del usuario; el usuario solo ve "esto cuesta X Sparks".

### 2.2 Origen de los Sparks

Los Sparks pueden ingresar a la cuenta del usuario por cuatro vías:

1. **Asignación mensual del plan:** automática al inicio de cada ciclo.
2. **Compra de packs adicionales:** transacción puntual.
3. **Recompensas (gamificación):** streaks, referidos, completar niveles.
4. **Compensaciones del sistema:** créditos por bugs, fallos, etc.

### 2.3 Consumo de Sparks

Los Sparks se consumen al ejecutar operaciones costosas. El sistema
descuenta el costo de la operación del balance del usuario antes de
ejecutarla, devolviendo el balance si la operación falla.

### 2.4 Expiración

- **Sparks del plan mensual:** expiran al cierre del ciclo, con un máximo
  de rollover de 2x el plan (evita acumulación excesiva).
- **Sparks comprados (packs):** no expiran durante 6 meses desde la compra.
- **Sparks de gamificación:** mismas reglas que los del plan mensual.

---

## 3. Estructura de planes

### 3.1 Tabla de planes

| Plan | Precio (MXN/mes) | Sparks/mes | Rollover máx | Costo objetivo IA |
|------|------------------|------------|--------------|---------------------|
| Free | $0 | 5 (one-time trial) | No aplica | < $0.05 |
| Básico | $30 | 30 | 60 | < $0.30 |
| Pro | $100 | 200 | 400 | < $1.50 |
| Premium | $250 | 600 | 1.200 | < $4.00 |

### 3.2 Packs adicionales

| Pack | Sparks | Precio (MXN) | Precio efectivo/Spark | Validez |
|------|--------|--------------|----------------------|---------|
| Pequeño | 100 | $50 | $0.50 | 6 meses |
| Mediano | 300 | $130 | $0.43 | 6 meses |
| Grande | 1.000 | $350 | $0.35 | 6 meses |

### 3.3 Estructura de descuentos

Los packs más grandes ofrecen mejor valor unitario, incentivando compras
mayores. La diferencia entre el plan Básico ($1.00 por Spark efectivo) y
los packs ($0.35-$0.50) refleja que el plan incluye también acceso a
preassets ilimitados, mientras que los packs son sólo Sparks puros.

---

## 4. Costo en Sparks por operación

### 4.1 Tabla de costos

| Operación | Costo en Sparks | Notas |
|-----------|-----------------|-------|
| Conversación 1 a 1 con IA | 1 por minuto | Limitado a 15 min por sesión |
| Análisis profundo de audio largo | 2 | Audio > 2 minutos |
| Generación de roleplay personalizado | 5 | Solo si no hay match en biblioteca |
| Sesión de pronunciación con feedback | 1 / 10 ejercicios | Batch eficiente |
| Resumen semanal personalizado | 3 | Generado domingos |
| Generación inicial de roadmap | Gratis (incluido en onboarding) | Una vez por usuario |
| Actualización de roadmap | Gratis (en plan) | Automática nocturna |
| Acceso a preassets | Gratis (ilimitado en plan) | No consume Sparks |
| Listening con audio pre-generado | Gratis | No consume Sparks |
| Mensaje matutino humanizado | Gratis (incluido en plan) | Generado batch |

### 4.2 Principio de costo

Los Sparks reflejan **operaciones intensivas en IA en tiempo real o cercanas**.
Las operaciones batch (job nocturno, generación de mensajes, análisis SQL)
están incluidas en el plan y no consumen Sparks individualmente.

### 4.3 Tabla ajustable

Los costos en Sparks son configuración del sistema, no constantes hardcodeadas.
Se ajustan según fluctúen los costos reales de proveedores. Cualquier ajuste:

- Se comunica con 30 días de anticipación a usuarios activos.
- Aplica solo a operaciones futuras, no retroactivamente.
- Se documenta en `docs/decisions/sparks-pricing-changes.md`.

---

## 5. Modelo de datos

### 5.1 Tabla de balance

```sql
CREATE TABLE user_sparks_balance (
  user_id           UUID PRIMARY KEY REFERENCES users(id),
  plan_sparks       INT NOT NULL DEFAULT 0,    -- del plan mensual actual
  pack_sparks       INT NOT NULL DEFAULT 0,    -- de packs comprados
  bonus_sparks      INT NOT NULL DEFAULT 0,    -- de gamificación
  total_sparks      INT GENERATED ALWAYS AS (plan_sparks + pack_sparks + bonus_sparks) STORED,
  current_cycle_id  UUID NOT NULL REFERENCES billing_cycles(id),
  updated_at        TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_balance_user ON user_sparks_balance(user_id);
```

### 5.2 Tabla de transacciones (audit log inmutable)

```sql
CREATE TABLE sparks_transactions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  amount          INT NOT NULL,  -- positivo = ingreso, negativo = consumo
  source          TEXT NOT NULL CHECK (source IN (
                    'plan_assignment',
                    'pack_purchase',
                    'gamification_reward',
                    'system_credit',
                    'operation_charge',
                    'refund',
                    'expiration'
                  )),
  source_subtype  TEXT,           -- ej: 'streak_7_days', 'referral_signup'
  operation_id    UUID,           -- referencia a la operación que generó el cargo
  balance_before  INT NOT NULL,
  balance_after   INT NOT NULL,
  metadata        JSONB DEFAULT '{}',
  created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_transactions_user_date ON sparks_transactions(user_id, created_at DESC);
CREATE INDEX idx_transactions_operation ON sparks_transactions(operation_id);
```

### 5.3 Tabla de packs comprados

```sql
CREATE TABLE sparks_packs_purchased (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  pack_type       TEXT NOT NULL,
  sparks_amount   INT NOT NULL,
  sparks_remaining INT NOT NULL,
  price_paid      DECIMAL(10,2) NOT NULL,
  currency        TEXT NOT NULL,
  payment_id      TEXT NOT NULL,
  purchased_at    TIMESTAMPTZ DEFAULT now(),
  expires_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_packs_user_active ON sparks_packs_purchased(user_id)
  WHERE sparks_remaining > 0 AND expires_at > now();
```

### 5.4 Tabla de ciclos de facturación

```sql
CREATE TABLE billing_cycles (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  plan_id         TEXT NOT NULL,
  cycle_start     TIMESTAMPTZ NOT NULL,
  cycle_end       TIMESTAMPTZ NOT NULL,
  sparks_assigned INT NOT NULL,
  sparks_unused   INT,                -- calculado al cierre
  status          TEXT NOT NULL CHECK (status IN ('active', 'closed', 'cancelled'))
);
```

---

## 6. Operaciones del sistema

### 6.1 Asignación mensual

Al inicio de cada ciclo de facturación:

```typescript
async function assignMonthlyPlanSparks(userId: string, planId: string) {
  const planConfig = getPlanConfig(planId);
  const previousBalance = await getBalance(userId);

  // Calcular rollover (sparks no usados, hasta el límite)
  const carryOver = Math.min(
    previousBalance.plan_sparks,
    planConfig.maxRollover - planConfig.monthlySparks
  );

  // Sparks expirados (los que no caben en rollover)
  const expired = previousBalance.plan_sparks - carryOver;
  if (expired > 0) {
    await recordTransaction({
      userId,
      amount: -expired,
      source: 'expiration',
      metadata: { reason: 'cycle_end_overflow' }
    });
  }

  // Asignar nuevos sparks del plan
  const newPlanBalance = carryOver + planConfig.monthlySparks;
  await updateBalance(userId, { plan_sparks: newPlanBalance });

  await recordTransaction({
    userId,
    amount: planConfig.monthlySparks,
    source: 'plan_assignment',
    metadata: { plan: planId, cycle: newCycleId }
  });
}
```

### 6.2 Cobro de operación

```typescript
async function chargeOperation(userId: string, operationType: string, params: any) {
  const cost = getOperationCost(operationType, params);

  // Validación previa
  const balance = await getBalance(userId);
  if (balance.total_sparks < cost) {
    throw new InsufficientSparksError({
      required: cost,
      available: balance.total_sparks,
      suggested_action: 'buy_pack_or_upgrade'
    });
  }

  // Cobro en orden: primero plan, luego pack, finalmente bonus
  // (favorece al usuario: no quemar sus sparks comprados primero)
  const charged = await deductInOrder(userId, cost, [
    'plan_sparks',
    'bonus_sparks',
    'pack_sparks'
  ]);

  const operationId = uuidv4();
  await recordTransaction({
    userId,
    amount: -cost,
    source: 'operation_charge',
    source_subtype: operationType,
    operation_id: operationId,
    metadata: { params, charged_breakdown: charged }
  });

  return { operationId, costInSparks: cost };
}
```

### 6.3 Refund por operación fallida

```typescript
async function refundOperation(operationId: string, reason: string) {
  const originalTx = await getTransactionByOperation(operationId);
  if (!originalTx) throw new Error('Original transaction not found');
  if (await alreadyRefunded(operationId)) return;

  await updateBalance(originalTx.user_id, {
    plan_sparks: { increment: Math.abs(originalTx.amount) }
  });

  await recordTransaction({
    userId: originalTx.user_id,
    amount: Math.abs(originalTx.amount),
    source: 'refund',
    operation_id: operationId,
    metadata: { reason, original_tx: originalTx.id }
  });
}
```

### 6.4 Compra de pack

```typescript
async function processPackPurchase(userId: string, packType: string, paymentId: string) {
  const packConfig = getPackConfig(packType);

  // Verificar pago confirmado (RevenueCat webhook, Stripe, etc.)
  const payment = await verifyPayment(paymentId);
  if (!payment.confirmed) throw new Error('Payment not confirmed');

  // Crear pack
  const expiresAt = addMonths(new Date(), 6);
  await createPack({
    userId,
    packType,
    sparksAmount: packConfig.sparks,
    sparksRemaining: packConfig.sparks,
    pricePaid: payment.amount,
    currency: payment.currency,
    paymentId,
    expiresAt
  });

  // Acreditar balance
  await updateBalance(userId, {
    pack_sparks: { increment: packConfig.sparks }
  });

  await recordTransaction({
    userId,
    amount: packConfig.sparks,
    source: 'pack_purchase',
    source_subtype: packType,
    metadata: { paymentId, expires_at: expiresAt }
  });
}
```

---

## 7. Gamificación con Sparks

### 7.1 Recompensas estándar

| Acción | Recompensa | Notas |
|--------|-----------|-------|
| Streak de 7 días | 5 Sparks | Reset si se rompe |
| Streak de 30 días | 30 Sparks | Reset si se rompe |
| Completar primer nivel | 20 Sparks | Una vez por track |
| Referido que se registra | 10 Sparks | Recompensa al referente |
| Referido que paga primer mes | 50 Sparks | Recompensa al referente |
| Completar onboarding | 10 Sparks | Una vez por usuario |
| Reseña en App Store / Play Store | 20 Sparks | Verificable manualmente |

### 7.2 Reglas anti-abuso

- **Detección de cuentas duplicadas:** análisis de IP, device fingerprint,
  patrones de uso para detectar referidos auto-generados.
- **Límite global de bonos:** máximo 100 Sparks de bonus por mes (excepto
  referidos legítimos).
- **Auditoría de patrones sospechosos:** revisión manual de cuentas con
  >200 Sparks bonus en un mes.

### 7.3 Por qué la gamificación funciona

Los Sparks como recompensa tienen alto valor percibido y bajo costo real
para el negocio:

- Para el usuario: 30 Sparks bonus se sienten como un regalo significativo
  ($15 MXN de valor percibido).
- Para el negocio: el costo real de generar esos 30 Sparks de uso es
  $1-2 USD ($20-40 MXN al tipo de cambio), pero el efecto en retención y
  engagement multiplica el LTV mucho más que ese costo.

---

## 8. Manejo de cambios de costo

### 8.1 Cuándo ajustar costos en Sparks

El costo en Sparks de una operación se ajusta cuando:

- Costo real de la operación cambia más de 20% respecto a baseline.
- Nuevo modelo más barato/caro reemplaza al actual para esa operación.
- Comportamiento de uso revela que el costo está mal calibrado.

### 8.2 Proceso de ajuste

1. Análisis interno: nueva propuesta de costo en Sparks documentada con
   datos.
2. Comunicación a usuarios activos con 30 días de anticipación.
3. Aplicación solo a operaciones futuras desde la fecha del cambio.
4. Operaciones en curso al momento del cambio mantienen el costo previo.

### 8.3 Comunicación al usuario

```
"A partir del 1 de junio, las conversaciones 1 a 1 con IA costarán
1.2 Sparks por minuto en lugar de 1 Spark por minuto, debido a mejoras
significativas en la calidad del modelo. Tu plan mensual sigue siendo
el mismo. Esto significa aproximadamente 4 minutos menos de
conversación al mes en el plan Pro."
```

Transparencia genera confianza. Esconder o disfrazar cambios genera
desconfianza permanente.

### 8.4 Compensaciones por cambios significativos

Si un cambio de costo en Sparks reduce significativamente el valor para
usuarios existentes (ej: 50% más caro), considerar:

- Sparks bonus de compensación a usuarios afectados.
- Grandfathering temporal (precio antiguo para usuarios anteriores).
- Mejora simultánea de algún feature para balancear.

---

## 9. Transparencia y UX

### 9.1 Visibilidad del balance

El usuario debe ver siempre:

- Balance total de Sparks.
- Desglose por origen (plan, packs, bonus).
- Fecha de expiración de packs.
- Historial de transacciones (últimos 30 días mínimo).

### 9.2 Indicación previa de costo

Antes de iniciar cualquier operación que cobre Sparks:

```
"Iniciar conversación de 10 minutos
Costo: 10 Sparks
Tu balance: 187 Sparks (después: 177 Sparks)
[Iniciar] [Cancelar]"
```

Esto evita sorpresas y educa al usuario sobre el valor.

### 9.3 Avisos de balance bajo

- A 20% del plan mensual restante: notificación suave.
- A 5% del plan mensual restante: notificación con sugerencia de upgrade
  o pack.
- A 0 Sparks: bloqueo de operaciones de pago, oferta de pack.

### 9.4 Acceso ininterrumpido a preassets

Crítico: aún cuando el usuario tenga 0 Sparks, debe seguir teniendo
acceso completo a la biblioteca de preassets. Los Sparks son para
**operaciones premium**, no para acceso básico.

Esto:

- Mantiene retención incluso si el usuario no compra packs.
- Construye confianza (el plan mensual entrega valor real, no fricción).
- Genera oportunidades futuras de upgrade cuando el usuario sienta el
  límite.

---

## 10. Reportes internos

### 10.1 Métricas críticas a monitorear

- **Cost per Spark consumed** (interno): debe mantenerse por debajo del
  precio efectivo del Spark vendido.
- **Sparks consumption rate** por plan: ¿usan los del Pro todos sus 200
  Sparks? Si <50%, el plan está sobrevaluado y se puede ajustar.
- **Pack conversion rate**: % de usuarios que compran packs. Indicador
  de saturación del plan principal.
- **Sparks por sesión activa**: salud del engagement.
- **Refund rate**: % de operaciones que requirieron refund. Alerta de
  bugs si sube.

### 10.2 Dashboard interno

Página en admin panel que muestre:

- Distribución de balances de usuarios.
- Cohorte de consumo (cuándo en el ciclo se gastan los Sparks).
- Top operaciones por consumo total de Sparks.
- Alertas: cuentas con consumo anómalo, posibles abusos, errores de
  cobro.

### 10.3 Alertas automáticas

- Usuario individual con consumo > $5 USD equivalente en un día (potencial
  bug o abuso).
- Caída de >20% en compras de packs vs semana anterior.
- Spike en errores de "InsufficientSparks" (puede indicar mal calibrado
  algún costo).
- Discrepancia entre balance calculado y suma de transacciones (integrity
  check).

---

## 11. Integración con otros sistemas

### 11.1 Con AI Gateway

Antes de cada llamada a un LLM, el AI Gateway:

1. Calcula el costo en Sparks de la operación.
2. Llama a `chargeOperation()` del Sparks system.
3. Si el cobro tiene éxito, ejecuta la llamada al LLM.
4. Si la llamada falla, llama a `refundOperation()`.

### 11.2 Con sistema de pagos

- RevenueCat webhook → confirmación de compra → `processPackPurchase()`.
- Stripe webhook → mismo flujo.
- Cancelación de suscripción → marcar `billing_cycles.status = 'cancelled'`,
  permitir uso hasta fin de ciclo.

### 11.3 Con gamificación

- Engine de gamificación detecta logros y llama a
  `awardBonusSparks(userId, amount, reason)`.
- Cada bono se registra como transacción con `source = 'gamification_reward'`.

---

## 12. Aspectos legales y financieros

### 12.1 Naturaleza contable de los Sparks

Los Sparks comprados en packs representan un **pasivo contable** hasta
ser consumidos. Los Sparks asignados con el plan no se contabilizan como
pasivo (vienen con la suscripción).

Implicaciones:

- Reconocimiento de revenue: el revenue del pack se reconoce al consumir
  los Sparks, no al venderlos (US GAAP / IFRS).
- Sparks no consumidos al expirar (6 meses): se reconocen como revenue
  en ese momento ("breakage").

Para etapa MVP esto puede simplificarse, pero debe ajustarse cuando el
producto tenga contabilidad formal.

### 12.2 Reembolsos

Política sugerida:

- Sparks de plan mensual: no reembolsables individualmente; reembolso de
  suscripción según política de Stripe/Apple/Google.
- Sparks de packs: reembolsables proporcionalmente si:
  - El usuario solicita en menos de 14 días.
  - No ha consumido más del 30% del pack.
  - Política de las stores de Apple/Google permite el reembolso.

### 12.3 Términos y condiciones

Documentar claramente en T&C:

- Sparks no son moneda de curso legal.
- Sparks no son transferibles entre usuarios.
- Sparks no se pueden convertir a dinero.
- La empresa puede ajustar el costo en Sparks de las operaciones con
  notice previo de 30 días.

---

## 13. Plan de implementación incremental

### 13.1 Sprint 1: tablas y operaciones básicas

- Schema de tablas (balance, transactions, packs, cycles).
- Funciones core: `getBalance`, `chargeOperation`, `refundOperation`,
  `assignMonthlyPlanSparks`.
- Tests unitarios completos.

### 13.2 Sprint 2: integración con producto

- Integración con AI Gateway: cargo previo a llamadas.
- UI de balance visible en la app.
- Indicación previa de costo en operaciones.
- Notificación de balance bajo.

### 13.3 Sprint 3: pagos y packs

- Integración con RevenueCat para packs in-app.
- Integración con Stripe para packs web.
- UI de tienda de packs.
- Webhooks y verificación de pagos.

### 13.4 Sprint 4: ciclos y rollover

- Job nocturno de rollover y expiración.
- Asignación automática de plan mensual.
- Auditoría de integridad (suma de transacciones = balance).

### 13.5 Sprint 5: gamificación

- Engine de detección de logros.
- Recompensas en Sparks.
- Anti-abuso básico.

### 13.6 Sprint 6: dashboard y observabilidad

- Dashboard interno con métricas.
- Alertas críticas configuradas.
- Reportes internos.

---

## 14. Casos de uso ilustrativos

### 14.1 Usuario Plan Pro típico

- Mes 1: recibe 200 Sparks. Usa 180 en conversaciones (15 sesiones de
  12 min). Sobran 20.
- Inicio mes 2: rollover de 20 + 200 nuevos = 220 Sparks disponibles.
- Mes 2: usa 200 en conversaciones, 5 en pronunciación. Sobran 15.
- Inicio mes 3: rollover de 15 + 200 = 215 Sparks.

Todo dentro del plan, sin necesidad de comprar packs.

### 14.2 Usuario power que necesita más

- Plan Pro con 200 Sparks/mes.
- Usa todo en la primera semana (45 min de conversación diaria).
- Compra Pack Mediano: +300 Sparks por $130.
- Tiene 300 Sparks adicionales para 4 semanas más.
- Costo total: $230 MXN al mes vs $250 MXN del plan Premium.
- Análisis: para este usuario, mejor upgrade a Premium directamente.

### 14.3 Usuario Plan Básico que prueba conversación

- Plan Básico no incluye conversación 1 a 1 normalmente.
- Pero el usuario tiene 30 Sparks del plan + 50 de gamificación
  (referidos) = 80 Sparks.
- Compra una sesión de 5 min: -5 Sparks. Le quedan 75.
- Permite upselling: "te encantó la conversación; con el Plan Pro tendrías
  200 Sparks/mes para hacerlo todos los días".

---

## 15. Decisiones abiertas

- [ ] ¿Permitir transferencia de Sparks entre usuarios (gifting)? Pros:
  viralidad. Contras: complejidad anti-abuso.
- [ ] ¿Sparks negativos (deuda) para usuarios premium con buen historial?
  Probablemente no, agrega complejidad.
- [ ] ¿Programa de fidelidad con multiplicador de Sparks (bonus % por
  antigüedad)? Evaluar después de validar churn.
- [ ] ¿Sparks comunitarios (eventos, sorteos)? Marketing tool, evaluar
  cuando haya tracción.
- [ ] ¿Sparks comprables como gift card para terceros? Útil en navidad,
  posible feature secundaria.

---

## 16. Referencias internas

- `docs/business/plan_de_negocio.docx` — Plan de negocio.
- `docs/product/ai-roadmap-system.md` — Sistema de roadmap.
- `docs/architecture/ai-gateway-strategy.md` — Estrategia de proveedores LLM.
- `docs/architecture/platform-strategy.md` — Estrategia de plataformas.
- `docs/decisions/ADR-003-sparks-system.md` — ADR formal (a crear).

---

*Documento vivo. Cualquier ajuste a costos en Sparks o estructura de
planes debe reflejarse aquí inmediatamente y comunicarse a usuarios.*

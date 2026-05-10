# US-049: Audit log inmutable validation cron

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 1
**Story points:** 2
**Persona:** Admin (compliance + financial integrity)
**Owner:** —

---

## Contexto

El audit log de transactions es **legalmente sagrado** —
disputes, fraud investigations, accounting reconciliation
dependen de que sea íntegro.

US-044 ya implementa append-only via RULE. Esta story agrega:
- Validation cron diario que verifica integridad.
- Detección de anomalías (balance mismatch vs transactions sum).
- Alerta inmediata si anomaly detectada.

Si alguna vez detectamos drift entre balance y suma de
transactions: SEV-1 alarm + investigation manual.

Backend en
[`sparks-system.md`](../../architecture/sparks-system.md) §6 +
[`reglas.md`](../../reglas.md) (audit inmutable).

## Scope

### In

- Cron Worker `apps/workers/crons/sparks-audit-validation.ts`:
  - Trigger: diario 03:00 UTC.
  - Para cada user con balance != 0:
    - Computar suma de transactions desde el primer transaction.
    - Verificar `current = SUM(amount) FROM transactions WHERE
      expired = false`.
    - Verificar `lifetime_earned = SUM positive amounts`.
    - Verificar `lifetime_spent = SUM negative amounts (abs)`.
  - Si mismatch detectado:
    - Insert row en `audit_anomalies` con detalles.
    - Emit event `sparks.audit_anomaly_detected` (SEV-1).
    - Alerta Slack #alerts-financial.
- Schema `audit_anomalies`:
  ```sql
  CREATE TABLE audit_anomalies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    anomaly_type    TEXT NOT NULL,
    expected_balance INT NOT NULL,
    actual_balance  INT NOT NULL,
    drift_amount    INT NOT NULL,
    investigated    BOOLEAN NOT NULL DEFAULT false,
    investigation_notes TEXT,
    investigated_at TIMESTAMPTZ,
    investigated_by TEXT
  );
  ```
- Endpoint `GET /admin/audit-anomalies` para listar pendientes
  (auth interna).
- Telemetry: `sparks.audit_validated_users`,
  `sparks.audit_anomaly_detected`.

### Out

- Auto-fix de anomalies (post-MVP — MVP es solo detection +
  manual investigation).
- Validation real-time (post-MVP — MVP es batch nocturno).

## Acceptance criteria

- **Given** cron triggered, **When** ejecuta, **Then** itera todos
  los users con balance > 0 (o lifetime_earned > 0).
- **Given** user con balance consistente (50 = SUM(awards) -
  SUM(charges)), **When** validate, **Then** NO insert en
  audit_anomalies.
- **Given** balance.current = 30 pero
  SUM(transactions.amount where !expired) = 50 (drift de -20),
  **When** validate detecta, **Then** insert audit_anomaly con
  drift_amount = -20.
- **Given** anomaly detected, **When** insert completa, **Then**
  event SEV-1 emitido + Slack alert disparada.
- **Given** GET `/admin/audit-anomalies?investigated=false`,
  **When** se llama, **Then** retorna lista de anomalies pendientes
  con detalle.
- **Given** cron procesa 10000 users en lote, **When** completa,
  **Then** termina en < 5 min y emite total
  `audit_validated_users: 10000`.
- **Given** Postgres timeout mid-cron, **When** falla, **Then**
  Inngest retry; si falla 3x: SEV-2 (cron itself broken).
- **Given** anomaly investigada y resuelta, **When** admin updates
  `investigated = true`, **Then** próximas runs del cron ignoran
  esa row (no re-alert).

## Developer details

### Owning service

`apps/workers/crons/sparks-audit-validation.ts` +
`/admin/audit-anomalies` handler.

### Dependencies

- US-044: schema (transactions + balances).

### Specs referenciados

- [`sparks-system.md`](../../architecture/sparks-system.md) §6.
- [`reglas.md`](../../reglas.md) — audit inmutable.

### Implementación esperada

```typescript
// apps/workers/crons/sparks-audit-validation.ts
export async function runSparksAuditValidation(env: Env) {
  const startTime = Date.now();
  let validatedCount = 0;
  let anomalyCount = 0;

  // Process in batches of 1000
  let offset = 0;
  while (true) {
    const batch = await env.DB.query(`
      SELECT user_id, current, lifetime_earned, lifetime_spent
      FROM sparks_balances
      WHERE current > 0 OR lifetime_earned > 0
      ORDER BY user_id
      LIMIT 1000 OFFSET $1
    `, [offset]);

    if (batch.length === 0) break;

    for (const row of batch) {
      validatedCount++;
      const sums = await env.DB.query(`
        SELECT
          COALESCE(SUM(amount) FILTER (WHERE expired = false), 0) AS expected_balance,
          COALESCE(SUM(amount) FILTER (WHERE amount > 0), 0) AS expected_lifetime_earned,
          COALESCE(ABS(SUM(amount) FILTER (WHERE amount < 0)), 0) AS expected_lifetime_spent
        FROM sparks_transactions
        WHERE user_id = $1
      `, [row.user_id]);

      const expected = sums[0];

      if (expected.expected_balance !== row.current
       || expected.expected_lifetime_earned !== row.lifetime_earned
       || expected.expected_lifetime_spent !== row.lifetime_spent) {
        anomalyCount++;
        const drift = row.current - expected.expected_balance;

        await env.DB.execute(`
          INSERT INTO audit_anomalies
            (user_id, anomaly_type, expected_balance, actual_balance, drift_amount)
          VALUES ($1, 'balance_mismatch', $2, $3, $4)
        `, [row.user_id, expected.expected_balance, row.current, drift]);

        await emitEvent('sparks.audit_anomaly_detected', {
          severity: 'SEV-1',
          user_id: row.user_id,
          drift_amount: drift,
          expected: expected.expected_balance,
          actual: row.current,
        });

        // Slack alert (best-effort)
        await sendSlackAlert({
          channel: '#alerts-financial',
          severity: 'SEV-1',
          message: `Sparks audit anomaly: user=${row.user_id}, drift=${drift}`,
        });
      }
    }

    offset += batch.length;
  }

  const elapsed = Date.now() - startTime;
  track('sparks.audit_validated_users', { count: validatedCount, elapsed_ms: elapsed });
  if (anomalyCount > 0) {
    track('sparks.audit_anomaly_summary', { anomaly_count: anomalyCount });
  }

  return { validated: validatedCount, anomalies: anomalyCount };
}
```

### Cron trigger

```toml
# apps/workers/crons/wrangler.toml
[triggers]
crons = ["0 3 * * *"]  # Diariamente 03:00 UTC
```

### Admin endpoint

```typescript
export async function handleAdminAuditAnomalies(request: Request, env: Env) {
  if (!isInternalAuth(request, env)) return unauthorized();

  const url = new URL(request.url);
  const investigatedFilter = url.searchParams.get('investigated') ?? 'false';

  const anomalies = await env.DB.query(`
    SELECT a.*, u.email
    FROM audit_anomalies a
    JOIN users u ON u.id = a.user_id
    WHERE a.investigated = $1
    ORDER BY a.detected_at DESC
    LIMIT 200
  `, [investigatedFilter === 'true']);

  return jsonResponse({ anomalies });
}
```

### Integration points

- US-044 schema (queries lectura).
- Slack webhook (alerts).
- US-041 logging dashboard (visualizar anomalies count).

### Notas técnicas

- Batch processing 1000 rows per loop evita memory issues con
  10k+ users.
- Cron debería NO bloquear flujo del producto — read-only.
- Anomaly detection se basa en SUM exacto de transactions —
  cualquier mutación (rule fail, manual SQL bypass) se detecta.

## Definition of Done

- [ ] Schema `audit_anomalies` migration aplicada.
- [ ] Cron Worker implementado con cron trigger.
- [ ] Admin endpoint funcional.
- [ ] Slack alert configurado.
- [ ] 8 acceptance criteria pasan.
- [ ] Test integration: simular drift, verificar detección.
- [ ] Performance: 10k users validated en <5 min.
- [ ] Validation contra spec `sparks-system.md` §6.
- [ ] PR aprobada y mergeada.

---

*Independent — corre en background. Genera confidence en la
integridad financial del producto.*

# Security Threat Model

> Modelo de amenazas usando STRIDE por sistema, mitigaciones existentes
> y pendientes, política de secrets, audit log, requisitos de compliance.
> Documento de referencia para cualquier cambio que toque seguridad.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador. Antes de modificar
cualquier sistema, leer §3 (amenazas por sistema) para identificar
mitigaciones que **no** debés romper.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 explica STRIDE en breve.
- §2 lista activos críticos a proteger.
- §3 enumera amenazas y mitigaciones **por sistema**.
- §4 cubre **secrets management** (cómo se manejan API keys, JWT
  secrets, etc.).
- §5 cubre **audit log** (qué se loggea, dónde, retención).
- §6 cubre **compliance** (GDPR, LFPDPPP México, Ley 25.326 Argentina).
- §7 cubre **incident response** (link a `ops/incidents-runbook.md`).
- §8 lista checklist de seguridad para code review.

Si tu cambio toca:
- **Auth** → leer §3.1.
- **Datos de usuario** → leer §3.2 (Postgres) y §3.6 (R2/audios).
- **Endpoints públicos** → leer §3.3 (API).
- **Llamadas a LLM** → leer §3.4 (LLM-specific).
- **Pagos** → leer §3.5 (Stripe/RevenueCat).

---

## 1. Metodología STRIDE

STRIDE clasifica amenazas en 6 categorías:

| Categoría | Definición |
|-----------|-----------|
| **S**poofing | Suplantar identidad |
| **T**ampering | Modificar datos no autorizados |
| **R**epudiation | Negar haber hecho una acción |
| **I**nformation Disclosure | Exponer información a quien no debería verla |
| **D**enial of Service | Hacer el sistema inaccesible |
| **E**levation of Privilege | Obtener permisos que no corresponden |

Cada amenaza se evalúa por:
- **Probabilidad:** alta / media / baja.
- **Impacto:** crítico / alto / medio / bajo.
- **Mitigación actual:** sí / parcial / no.
- **Mitigación pendiente:** acción tracked en `pendientes.md`.

---

## 2. Activos críticos

Lo que estamos protegiendo, ranqueado por sensibilidad:

| Activo | Sensibilidad | Donde vive |
|--------|--------------|-----------|
| Credenciales de usuario | Crítico | Firebase Auth |
| Datos personales (nombre, email, teléfono) | Alto | Postgres `users` |
| Audios crudos del usuario | Alto | Cloudflare R2 |
| Datos de pago | Crítico | Stripe / RevenueCat (no en nuestros servers) |
| Resultados de assessment | Medio | Postgres `student_profiles` |
| Balance de Sparks | Alto (riesgo financiero) | Postgres `user_sparks_balance` |
| Audit log de Sparks | Crítico (forense) | Postgres `sparks_transactions` |
| Llaves de proveedores LLM | Crítico (riesgo financiero) | Cloudflare Secrets |
| Firebase service account | Crítico | Cloudflare Secrets |
| Datos de fraude (fingerprints, scores) | Medio | Postgres |

---

## 3. Amenazas por sistema

### 3.1 Authentication

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| AUTH-1 | Account takeover via password leak | S | Med | Crítico | Firebase password breach detection; email verification | MFA para cuentas con >500 Sparks comprados (post-MVP) |
| AUTH-2 | Brute force a email/password | S | Med | Alto | Firebase throttling automático; rate limit en endpoint | OK |
| AUTH-3 | Token replay después de logout | S | Baja | Alto | idTokens de Firebase TTL 1h; logout invalida refresh | OK |
| AUTH-4 | JWT validation bypass | S/E | Baja | Crítico | jose lib + JWKS validation; cache JWKS 24h | Tests de integración del validator |
| AUTH-5 | OAuth callback hijacking | S | Baja | Alto | Redirect URIs whitelisted en Google/Apple console | OK |
| AUTH-6 | Anonymous auth abuse (bot signups) | E | Alta | Med | Rate limit por IP/device; cap 50 Sparks trial | Anti-fraud system §3.6 |
| AUTH-7 | Account deletion sin verificación | T | Baja | Alto | Período de gracia 30d con email confirmation | OK |
| AUTH-8 | Phone OTP fraud (cuando se agregue) | S | Med | Med | Rate limit estricto + Recaptcha | Diseñado en doc auth §10 |

### 3.2 Postgres (datos)

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| DB-1 | SQL injection | T/I | Baja | Crítico | Solo parameterized queries (kysely o SQL directo con bindings); nunca string concat | Lint rule en CI |
| DB-2 | Acceso directo a tablas de otro sistema | E | Baja | Med | Convención: cada sistema tiene un módulo de datos; PR review | Schemas Postgres con permissions a roles distintos por sistema (post-MVP) |
| DB-3 | Filtración por error en queries (mostrar X user a Y user) | I | Med | Crítico | Validación de `user_id` en cada query; tests de integración con multi-tenant | Row Level Security (RLS) si migramos a Supabase Auth (post-MVP) |
| DB-4 | Dump de DB en backup robado | I | Baja | Crítico | Supabase encryption at rest; backups en region GCP us-central1 | Encriptación de campos PII a nivel de aplicación (deferred, evaluar costo/beneficio) |
| DB-5 | Borrado masivo accidental | T/D | Baja | Crítico | Soft-delete con período de gracia; backups diarios; admin queries con confirmación | Plan de recovery documentado en `ops/incidents-runbook.md` |
| DB-6 | DB compromise → exfiltración total | I | Baja | Crítico | Network ACL Supabase, no internet-public; conexiones solo desde Cloudflare Workers | Auditoría anual de accesos a Supabase |

### 3.3 API endpoints (Cloudflare Workers)

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| API-1 | Endpoint sin auth llamado por anónimo | E | Med | Crítico | `verifyFirebaseToken` middleware obligatorio; default deny | Test de integración: cada endpoint tiene auth |
| API-2 | IDOR (insecure direct object reference) | E | Med | Crítico | Validar `userId == JWT.sub` en cada query con fila de usuario | Code review explícito de cualquier endpoint que reciba `user_id` del cliente |
| API-3 | Rate limit bypass | D | Med | Med | Cloudflare rate limiting por IP; Durable Object por usuario en endpoints sensibles | Configurar limits específicos en `wrangler.toml` |
| API-4 | DDoS volumétrico | D | Baja | Alto | Cloudflare automatic DDoS protection | OK |
| API-5 | Cross-origin abuse | I | Baja | Med | CORS whitelisting de dominios propios | Lockdown CORS en producción |
| API-6 | Request smuggling / desync | T | Muy baja | Med | Workers stack moderno, no afectado | OK |
| API-7 | Mass assignment (cliente envía campos prohibidos) | T | Med | Alto | Validación con Zod en cada endpoint; whitelist de campos editables | OK |
| API-8 | Logs leak de datos sensibles | I | Med | Med | Convención: no loggear payloads enteros; redactar PII | Lint helper para detectar `console.log(req.body)` |

### 3.4 LLM-specific (AI Gateway)

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| LLM-1 | Prompt injection vía input del usuario | T | Alta | Med | Sanitización de inputs antes de prompt; validation Zod del output; system prompts con guardrails explícitos | Tests con dataset de attack prompts |
| LLM-2 | Modelo devuelve PII de otros usuarios | I | Baja | Crítico | Nunca enviar datos de un user a otro en prompts; queries siempre scoped por user_id | Code review |
| LLM-3 | Costo descontrolado (loop / abuso) | D | Med | Alto | Cap por usuario en Sparks; alert si user individual > $1/día; circuit breaker | Cost tracking dashboard |
| LLM-4 | Output malicioso (prompts que generan instrucciones dañinas) | T | Med | Med | Output filtering + validation; limit de tokens; cliente solo muestra outputs validados | Reportar al usuario si detectamos abuse pattern |
| LLM-5 | Modelo deprecado → fallo en runtime | D | Med | Alto | Fallback chain en AI Gateway; alertas de deprecación detection | Job semanal de check de modelos |
| LLM-6 | Provider compromise → API key robada | E | Baja | Crítico | Secrets en Cloudflare Secrets, no en código; rotation cuatrimestral | Setup de rotation playbook |
| LLM-7 | Audio del usuario subido a LLM externo de bajo trust | I | Baja | Alto | Solo Azure (acuerdo enterprise) y eventualmente self-hosted; ningún LLM Tier 3 procesa audios | Documentar lista de proveedores autorizados |

### 3.5 Pagos (Stripe / RevenueCat)

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| PAY-1 | Webhook spoofing (atacante envía POST falso) | S | Med | Crítico | Verificación de signature de Stripe / RevenueCat en cada webhook | Tests de integración del webhook handler |
| PAY-2 | Replay attack en webhook | T | Baja | Alto | Idempotency en handler con `payment_id` como key | OK |
| PAY-3 | Pago confirmed pero plan no activado | T | Baja | Crítico | Polling cliente post-compra; reconciliación nocturna con RevenueCat API | Cron de reconciliación documentado |
| PAY-4 | Chargebacks fraudulentos | T | Med | Med | Stripe Radar; 3DS para tarjetas nuevas; ban de chargebacks repetidos | Documentado en anti-fraud §6.5 |
| PAY-5 | PCI scope (no procesamos tarjetas) | I | N/A | N/A | Stripe handles all card data; nunca tocamos PAN | OK (PCI scope mínimo) |

### 3.6 Audio storage (R2)

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| R2-1 | Acceso público a audios privados | I | Baja | Crítico | R2 bucket privado; acceso solo via signed URLs con TTL corto (5 min) | Audit de bucket public access trimestral |
| R2-2 | URL signed leak en logs | I | Med | Med | No loggear URLs signed completas; truncar | Lint rule |
| R2-3 | Path traversal en upload | T | Baja | Alto | Path generado por backend (no por cliente); usar UUIDs | OK |
| R2-4 | Bucket scan / enumeration | I | Baja | Med | Object keys con UUID, no predecibles | OK |
| R2-5 | Retention no aplicada (audios viejos quedan) | I | Med | Med | Cron diario de cleanup (§7.5 data-and-events.md) | Test de integración del cron |

### 3.7 Sparks system (riesgo financiero)

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| SPK-1 | Manipulación de balance vía endpoint | T | Baja | Crítico | Balance solo se modifica vía funciones internas (`chargeOperation`, `awardBonus`); endpoints de cliente nunca aceptan `amount` directo | Code review |
| SPK-2 | Race condition en cobro (double spend) | T | Baja | Alto | `SELECT ... FOR UPDATE` en transacción Postgres; balance check + decrement atómico | Tests de concurrencia |
| SPK-3 | Replay de webhook de purchase | T | Baja | Alto | Idempotency con `payment_id` único | OK (PAY-2) |
| SPK-4 | Farming via cuentas duplicadas | E | Alta | Med | Anti-fraud system; cap absoluto del trial; device fingerprinting | Detalle en anti-fraud §3 |
| SPK-5 | Discrepancia balance ↔ transactions sum | T | Baja | Crítico | Audit job nocturno: verifica `SUM(transactions) == balance` por user; alerta si diverge | Implementación pendiente |

### 3.8 Notifications

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| NOT-1 | Spam push a usuarios | D | Baja | Med | Rate limiting por categoría (§9 notifications-system); circuit breaker | OK |
| NOT-2 | FCM token leak | I | Baja | Bajo | Tokens en Postgres, no expuestos a cliente | Rotation: Firebase rota tokens automáticamente |
| NOT-3 | Push con contenido malicioso | T | Baja | Bajo | Contenido pre-generado en batch nocturno con validation Zod del LLM | Tests del pipeline de generación |

### 3.9 Anti-fraud (recursive)

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| AF-1 | Falsos positivos baneando usuarios legítimos | T | Med | Alto | Niveles 4-5 requieren revisión humana; sistema de apelaciones | Métrica false positive rate < 5% |
| AF-2 | Atacante aprende cómo evadir el scoring | E | Med | Med | Reglas de scoring en backend, no expuestas a cliente; rotación de pesos cuatrimestral | OK |
| AF-3 | Restricciones aplicadas al usuario equivocado | T | Baja | Alto | Logs de cada `applyRestriction` con admin/system origen | Audit log |

### 3.10 Customer Support (AI Assistant)

| ID | Amenaza | Categoría | Prob. | Impacto | Mitigación actual | Pendiente |
|----|---------|-----------|-------|---------|-------------------|-----------|
| CS-1 | Usuario obtiene refund no autorizado vía AI | T | Baja | Med | AI nunca ejecuta refund directamente; siempre crea ticket que humano aprueba | Tests del prompt |
| CS-2 | Filtración de datos de otro usuario via AI conversation | I | Baja | Crítico | Context del AI scoped al user_id del usuario logueado; nunca cross-user data en prompt | Code review |
| CS-3 | AI promete cosas que no podemos cumplir | T | Med | Med | System prompt explícito: "Nunca prometas descuentos o refunds sin verificar"; tests con casos | OK |

---

## 4. Secrets management

### 4.1 Donde viven los secrets

**Producción (Cloudflare Workers):**
- `wrangler secret put <NAME>` → Cloudflare Secrets (encrypted at rest).
- Accesibles via `env.<NAME>` en Workers.

**Local development:**
- `.dev.vars` (gitignored). Contiene los secrets de dev.
- **Nunca** commitear `.dev.vars`.

**CI/CD:**
- GitHub Actions secrets (encrypted, accesibles solo en jobs autorizados).

### 4.2 Lista de secrets críticos

| Secret | Donde se usa | Rotation |
|--------|-------------|----------|
| `FIREBASE_SERVICE_ACCOUNT` | Workers (FCM, Auth admin) | Cuatrimestral |
| `FIREBASE_PROJECT_ID` | Workers | No rota (no es secret estricto) |
| `SUPABASE_DB_URL` | Workers (Postgres connection) | Cuatrimestral |
| `SUPABASE_SERVICE_ROLE_KEY` | Workers (admin DB ops) | Cuatrimestral |
| `STRIPE_SECRET_KEY` | Workers (webhook + reconciliation) | Cuatrimestral |
| `STRIPE_WEBHOOK_SIGNING_SECRET` | Workers | Anual |
| `REVENUECAT_API_KEY` | Workers | Cuatrimestral |
| `REVENUECAT_WEBHOOK_SECRET` | Workers | Anual |
| `ANTHROPIC_API_KEY` | Workers (AI Gateway) | Cuatrimestral |
| `OPENAI_API_KEY` | Workers (AI Gateway fallback) | Cuatrimestral |
| `GEMINI_API_KEY` | Workers (AI Gateway fallback) | Cuatrimestral |
| `AZURE_SPEECH_KEY` | Workers (pronunciation) | Cuatrimestral |
| `AZURE_SPEECH_REGION` | Workers | No rota |
| `USER_ID_HASH_PEPPER` | Workers (hashing user_id) | Anual (con migración de hashes) |
| `SENTRY_DSN` | Workers + cliente | Anual |
| `POSTHOG_API_KEY` | Cliente + Workers | Anual |

### 4.3 Política de rotación

- **Trigger:** rotación programada (calendar reminder) + rotación
  forzada si hay sospecha de leak.
- **Procedimiento:**
  1. Generar nuevo secret en el proveedor.
  2. Setear el nuevo valor en Cloudflare Secrets.
  3. Verificar que Workers funcionan con el nuevo valor.
  4. Revocar el secret viejo en el proveedor.
  5. Documentar rotación con fecha en `ops/secrets-rotation.log` (a
     crear).

### 4.4 Reglas de no-leak

- ❌ Nunca commitear secrets a git. Pre-commit hook con `gitleaks` o
  similar.
- ❌ Nunca loggear secrets (incluso parciales).
- ❌ Nunca enviar secrets en eventos o tracking.
- ❌ Nunca compartir secrets en chat (Slack, etc.) — usar 1Password o
  similar.
- ✅ Si un secret se filtró, **rotarlo inmediatamente** y auditar uso
  durante el período de exposición.

### 4.5 Pre-commit protection

`.gitleaks.toml` configurado. Pre-commit hook escanea staging area.

```bash
# .githooks/pre-commit
gitleaks protect --staged --redact --verbose
```

---

## 5. Audit log

### 5.1 Qué se loggea

Acciones críticas que requieren forensia:

| Acción | Donde | Quién |
|--------|-------|-------|
| Sparks transaction (cualquier cambio de balance) | `sparks_transactions` | Sparks system |
| Aplicación/lift de restriction | `user_restrictions` + event log | Anti-fraud |
| Account deletion request | `users.deleted_at` + event log | Auth |
| Manual override de fraud score | `user_fraud_scores.override_*` | Admin (humano) |
| Cambio de `pedagogical_config` | `pedagogical_config.updated_*` | Admin |
| Cambio de costos en Sparks (`sparks_operation_costs`) | tabla con `updated_*` | Admin |
| Login event | Firebase Auth (loggeado por Firebase) | Auth |
| Refund manual | `support_tickets.resolution` | Soporte humano |
| Apelación resolved | `fraud_appeals.reviewed_*` | Soporte humano |

### 5.2 Inmutabilidad

Tablas de audit son **append-only**. No `UPDATE`, no `DELETE`, ni
siquiera por admins.

Si hay que corregir un dato, se inserta una fila nueva que "anula" la
anterior con referencia.

```sql
-- Ejemplo: corregir un Sparks transaction errónea
INSERT INTO sparks_transactions (
  user_id, amount, source, source_subtype,
  metadata
) VALUES (
  $user_id, $reverse_amount, 'system_credit', 'correction',
  '{"corrects_transaction_id": "<original_id>", "reason": "..."}'::jsonb
);
```

### 5.3 Retención de audit log

- `sparks_transactions`: indefinido (compliance financiero).
- `user_restrictions`: indefinido.
- `fraud_events`: 5 años.
- `fraud_appeals`: 5 años.
- Login events (en Firebase): según política de Firebase (~1 año).
- `event_log`: 12 meses (eventos generales, no audit estricto).

### 5.4 Acceso al audit log

Solo:
- Admin (humano, vía admin panel autenticado con MFA).
- Procesos automáticos (jobs nocturnos para reconciliación).

**No** accesible vía API pública. **No** accesible al usuario salvo
extracto via account export (data portability under GDPR).

---

## 6. Compliance

### 6.1 GDPR (usuarios europeos)

Aunque target es Latam, podemos tener users europeos. Cumplimiento
mínimo:

- **Consent informado:** privacy policy clara, opt-ins separados para
  data sensible.
- **Derecho al borrado:** account deletion completo en 30 días.
- **Derecho al acceso:** export en JSON desde settings.
- **Derecho a rectificación:** UI para editar perfil.
- **Portabilidad:** export en formato estándar (JSON).
- **Notificación de breach:** dentro de 72 horas a la autoridad y
  usuarios afectados.

### 6.2 LFPDPPP (México)

- **Aviso de privacidad accesible:** en website y dentro de la app.
- **Mecanismos para ejercer derechos ARCO** (Acceso, Rectificación,
  Cancelación, Oposición): vía endpoints en settings + email a
  `privacy@<dominio>`.

### 6.3 Ley 25.326 (Argentina)

- **Registro ante AAIP** si procesamos datos personales.
- **Consentimiento informado.**

### 6.4 App Store / Play Store

- **Account deletion in-app:** obligatorio. Implementado en
  `authentication-system.md` §8.3.
- **Privacy nutrition labels (Apple):** declarar tipos de datos
  recolectados.
- **Data safety form (Google):** equivalente.
- **Sign in with Apple:** obligatorio si ofrecemos otros SSO.

### 6.5 Audios y biometría

Audios de voz **pueden** considerarse data biométrica en algunas
jurisdicciones (depende de uso). Mitigación:

- Privacy policy declara captura de audios y propósito.
- No vendemos audios a terceros.
- Audios crudos retenidos solo 30 días.
- Modelos propios entrenados con audios consentidos.

### 6.6 Documento detallado

Compliance específico, T&C, privacy policy se desarrollan en
`docs/business/legal-compliance.md` (a crear).

---

## 7. Incident response

Detalle en `docs/ops/incidents-runbook.md` (a crear). Resumen de
eventos de seguridad:

### 7.1 Severities de seguridad

| Severity | Definición | Tiempo respuesta |
|----------|-----------|------------------|
| SEC-1 | Compromise activo (ej: credentials leak verificado, exfiltración) | < 15 min |
| SEC-2 | Vulnerabilidad explotable detectada | < 1 hora |
| SEC-3 | Secret leak sospechado pero no confirmado | < 4 horas |
| SEC-4 | Reporte de bug bounty (no exploit activo) | < 1 día hábil |

### 7.2 Procedimiento básico

1. **Contener:** desactivar la vulnerabilidad o rotar secrets.
2. **Investigar:** alcance del impacto, qué datos pudieron filtrarse.
3. **Comunicar:** si hay user data afectado, notificar dentro de 72h
   (GDPR) por email + status page.
4. **Remediar:** fix + tests para prevenir recurrencia.
5. **Post-mortem:** documento interno + (opcional) público.

### 7.3 Bug bounty

Post-MVP: programa estructurado en HackerOne o similar.

Pre-MVP: email `security@<dominio>` + acknowledgment público de
reporters legítimos.

---

## 8. Checklist de seguridad para code review

Antes de aprobar un PR que toque seguridad, verificar:

### General

- [ ] No hay secrets hardcoded en código o tests.
- [ ] No se loggean payloads enteros con potencial PII.
- [ ] Inputs validados con Zod en cada boundary.
- [ ] Outputs de LLM validados antes de mostrar al usuario.

### Auth y autorización

- [ ] Endpoints autenticados validan JWT.
- [ ] `userId` del JWT se usa, no el `userId` del payload del cliente.
- [ ] Operaciones admin requieren claim explícito (no solo "logueado").

### DB

- [ ] Queries usan parameterized inputs (no concat).
- [ ] Queries que tocan filas del usuario incluyen `WHERE user_id =
  $user_id_from_jwt`.
- [ ] Operaciones críticas (Sparks, mastery) en transacción con `FOR
  UPDATE`.

### LLM

- [ ] Prompts no incluyen datos de otros usuarios.
- [ ] User input se sanitiza antes de inyectar al prompt.
- [ ] Output validation Zod aplicada.
- [ ] Fallback chain definida.

### Data

- [ ] Eventos no incluyen PII directa (usar `user_id_hash`).
- [ ] Audios subidos a R2 con path generado por backend, no por cliente.
- [ ] Datos sensibles tienen política de retención documentada.

---

## 9. Referencias internas

- [`data-and-events.md`](data-and-events.md) — privacidad en eventos,
  hashing.
- [`../architecture/anti-fraud-system.md`](../architecture/anti-fraud-system.md)
  — defensa contra fraude.
- [`../architecture/authentication-system.md`](../architecture/authentication-system.md)
  — Firebase Auth.
- [`../business/legal-compliance.md`](../business/legal-compliance.md) —
  compliance detallado (a crear).
- [`../ops/incidents-runbook.md`](../ops/incidents-runbook.md) —
  procedimiento de incidentes (a crear).

---

## 10. Recursos externos

- OWASP Top 10: https://owasp.org/www-project-top-ten/
- OWASP API Security Top 10: https://owasp.org/API-Security/
- Firebase Auth security checklist:
  https://firebase.google.com/docs/auth/security
- LLM-specific threats: https://owasp.org/www-project-top-10-for-large-language-model-applications/

---

*Documento vivo. Revisar al menos cada 6 meses, después de cada
incidente, y al introducir un sistema o dependencia nueva.*

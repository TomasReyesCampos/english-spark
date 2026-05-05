# Anti-Fraud and Abuse Prevention System

> Sistema integral para prevenir, detectar y responder a abusos:
> cuentas falsas, farming de Sparks, account sharing, chargebacks
> fraudulentos, manipulación de métricas. Filosofía: prevención >
> detección > castigo.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 establece **filosofía y modelo de amenazas**.
- §2 cubre **boundaries**.
- §3 cubre **prevención por diseño**.
- §4 cubre **detección activa** (fraud scoring).
- §5 cubre **niveles de respuesta**.
- §6 cubre **casos específicos** detallados.
- §7 cubre **schemas Postgres**.
- §8 cubre **API contracts**.
- §9 cubre **eventos emitidos**.
- §10 enumera **edge cases**.
- §11 cubre **observabilidad**.
- §12 cubre **decisiones cerradas**.
- §13 cubre **aspectos legales**.

---

## 1. Por qué importa anti-fraude

### 1.1 El problema invisible

A pequeña escala parece despreciable. A escala, no:

- 100 cuentas falsas farmeando Sparks: pocos dólares de pérdida.
- 10.000 cuentas falsas: miles de dólares + sistema degradado (logros
  pierden valor, leagues contaminadas, datos distorsionados).

### 1.2 Vectores de fraude

**Farming de Sparks:**
- Crear múltiples cuentas para acumular trial Sparks.
- Auto-referidos para ganar Sparks de referral.
- Bots generando actividad falsa para logros.

**Account sharing:**
- Una cuenta paga compartida entre N personas.

**Chargebacks fraudulentos:**
- Pagar, usar el servicio, hacer chargeback.

**Manipulación de métricas:**
- Inflar leagues/leaderboards artificialmente.

**Abuso de soporte:**
- Solicitudes de refund repetitivas.
- Tickets falsos para drenar tiempo.

**Content abuse:**
- Reportes falsos de assets.

### 1.3 Filosofía adoptada

**Prevención > detección > castigo.**

- Mejor diseñar para que el fraude sea costoso/imposible.
- Detectar lo que pase a pesar de la prevención.
- Castigar solo cuando hay evidencia clara.

**Friction proporcional al riesgo.**

- No molestar al 99% honesto por proteger contra el 1% malicioso.
- Aumentar fricción solo con señales sospechosas.

**Sistema invisible para legítimos.**

- Protecciones funcionan en background.
- Usuario legítimo no debe sentir el sistema.

**Iterar con datos.**

- El fraude evoluciona. El sistema también.

### 1.4 Modelo de amenazas

| Tipo de actor | Frecuencia | Daño potencial | Prioridad |
|---------------|-----------|----------------|-----------|
| Casual abuser | Alta (5–10%) | Bajo individual | Baja |
| Self-server | Media (1–3%) | Medio | Media |
| Small operator | Baja (<0.5%) | Alto | Alta |
| Sophisticated attacker | Muy baja | Muy alto | Crítica si aparece |
| Compromised account | Baja-media | Alto | Alta |

---

## 2. Boundaries

### 2.1 Es responsable de

- Generar y mantener `device_fingerprints`.
- Calcular `fraud_score` por usuario (0-100).
- Aplicar restricciones automáticas niveles 1-3.
- Manejar apelaciones (recibir + tracking; resolución es humana).
- Detectar patrones de signups duplicados.
- Detectar self-referrals.
- Emitir eventos cuando se aplica/lifteea restricción.

### 2.2 NO es responsable de

- **Detectar payment fraud:** Stripe Radar lo hace. Solo consumimos
  resultado.
- **Banear cuentas legítimas sin proceso:** todos los niveles 4-5
  requieren revisión humana.
- **Comunicación con el usuario:** delegar a `notifications-system`.
- **Tickets de soporte:** delegar a `customer-support-system`.
- **Borrar datos del usuario:** auth-system maneja deletion.
- **Decidir fraud signals globales:** cada signal tiene definición
  explícita y peso configurable. No magia.

### 2.3 Tensiones

| Tensión | Resolución |
|---------|-----------|
| Usuario con fraud_score alto pero legítimo | Niveles 1-3 son auto, niveles 4-5 requieren humano. Apelaciones rápidas. |
| Familia comparte device | 2 cuentas/device permitido sin flag. 3+: investigación. |
| Detección agresiva genera churn de legítimos | Métrica false positive rate < 5% como guardrail. |
| Atacante aprende a evadir | Pesos de signals rotables; no exponer reglas exactas. |

---

## 3. Prevención por diseño

### 3.1 Trial diseñado para minimizar farming

#### Cap absoluto de Sparks

Free trial otorga **máximo 50 Sparks**, no rolling, no acumulables.
Aunque alguien cree 100 cuentas, no obtiene 100 × 50 utilizables (cada
cuenta es independiente y no se pueden transferir).

#### Limitaciones del trial

Durante el trial:
- No se pueden referir amigos (recompensa solo después de pagar).
- No se pueden acumular Sparks comprados.
- No participan en leagues (evita contaminar leaderboards).

#### Trial expira con consecuencias claras

Después del trial sin convertir:
- Acceso solo a preassets gratis.
- Cuenta queda activa pero limitada.
- Crear nueva cuenta para "re-trial" no es atractivo si: device
  detectado, email pattern, etc.

### 3.2 Fingerprinting de devices

Para detectar múltiples cuentas del mismo device:

**Datos capturados (no PII estricta):**
- Device ID (Apple/Android device identifier).
- Hash de combinación de hardware/software signals.
- IP address y patrón de cambio.
- Timezone y language settings.
- Screen resolution y características.

**Generación:**

```typescript
function generateDeviceFingerprint(deviceInfo: DeviceInfo): string {
  const components = [
    deviceInfo.device_id,
    deviceInfo.platform,
    deviceInfo.os_version,
    deviceInfo.model_name,
    deviceInfo.screen_size,
    deviceInfo.language,
    deviceInfo.timezone,
    hashIpRange(deviceInfo.ip, 24),  // /24 net, no IP exact
  ];
  return sha256(components.join('|'));
}
```

**Uso:**
- Si nuevo signup tiene mismo fingerprint que cuenta existente: flag.
- Si fingerprint matches con 5+ cuentas: restricción automática.

**Compliance:**
- Fingerprinting puede ser regulado (GDPR, CCPA).
- Disclosure en privacy policy.
- Verificar por jurisdicción.

### 3.3 Verificación de email/teléfono

**Email:**
- Verification al registrarse.
- Funcionalidades premium requieren verified.
- Domains de email temporal (10minutemail, etc.) bloqueados via
  blocklist `disposable_email_domains`.

**Phone OTP (cuando se agregue):**
- Verificación de número real.
- Bloqueo de números virtuales (Twilio, etc.) detectables.

### 3.4 Rate limiting agresivo

```typescript
const RATE_LIMITS = {
  per_user: {
    ai_conversations_per_minute: 5,
    signup_per_device_per_month: 2,
    referrals_sent_per_day: 10,
  },
  per_ip: {
    signups_per_hour: 5,
    password_resets_per_hour: 3,
    support_tickets_per_day: 5,
  },
  per_endpoint: {
    sparks_consumption_realtime: 10,  // ops por minuto
  },
};
```

Con captcha si se exceden umbrales.

---

## 4. Detección activa

### 4.1 Fraud scoring algorithm

```typescript
function calculateFraudScore(user: User, signals: FraudSignals): number {
  let score = 0;

  // Device fingerprint matches
  if (signals.matching_fingerprints > 1) score += 30;
  if (signals.matching_fingerprints > 3) score += 50;

  // Email patterns
  if (signals.email_pattern_suspicious) score += 20;

  // IP analysis
  if (signals.ip_signups_24h > 3) score += 25;
  if (signals.ip_signups_24h > 10) score += 50;

  // Behavioral
  if (signals.fast_spend_pattern) score += 15;
  if (signals.signup_to_first_spend_seconds < 60) score += 20;

  // Referrals
  if (signals.self_referral_likely) score += 30;

  // Audio anomalies
  if (signals.audio_never_human) score += 40;

  // Payment patterns
  if (signals.previous_chargebacks > 0) score += 50;

  return Math.min(score, 100);
}
```

### 4.2 Señales por categoría

#### Para farming de Sparks

| Signal | Descripción | Peso |
|--------|-------------|-----:|
| `matching_fingerprints_2_4` | Device fingerprint matches 2-4 cuentas | 30 |
| `matching_fingerprints_5plus` | 5+ cuentas | 80 |
| `email_pattern_sequential` | `user1@x.com, user2@x.com, user3@x.com` | 20 |
| `email_pattern_random` | Random strings sin estructura humana | 15 |
| `ip_signups_3_10_24h` | 3-10 signups desde misma IP/24 en 24h | 25 |
| `ip_signups_10plus_24h` | 10+ signups | 50 |
| `fast_spend` | Gasta todos los Sparks en < 1 día | 15 |
| `signup_to_spend_under_60s` | Spend en < 60s post-signup | 20 |
| `velocity_signup_to_first_attempt_under_5s` | Demasiado rápido para humano | 30 |

#### Para account sharing

| Signal | Peso |
|--------|-----:|
| `simultaneous_sessions_far_regions` | Sesiones simultáneas desde regiones imposibles | 30 |
| `frequent_fingerprint_changes` | Device fingerprint cambia > 3 veces/semana | 20 |
| `inconsistent_skill_patterns` | B1 errores en sesión vs C1 en otra | 25 |
| `nonhuman_24h_pattern` | Actividad continua 24h sin descanso | 30 |

#### Para self-referrals

| Signal | Peso |
|--------|-----:|
| `referrer_referee_same_fingerprint` | Mismo device | 40 |
| `referee_signup_within_5min_of_code` | Signup inmediato post-generación | 20 |
| `referee_never_verifies_email` | Email/phone nunca verificados | 15 |
| `referee_never_completes_onboarding` | No actividad post-signup | 25 |

#### Para chargebacks pre-payment

| Signal | Peso |
|--------|-----:|
| `country_high_chargeback_rate` | País con alta tasa | 10 |
| `card_never_used_with_us` | Tarjeta nueva | 5 |
| `same_card_previous_chargebacks` | Tarjeta con CB previos | 50 |

#### Para bots

| Signal | Peso |
|--------|-----:|
| `temporal_pattern_exact_intervals` | Cada 5 min exactamente | 30 |
| `repetitive_responses` | Mismas respuestas en ejercicios distintos | 25 |
| `audio_never_human_voice` | TTS sintético detectable | 40 |
| `response_speed_under_100ms` | Inhumano | 35 |

### 4.3 Thresholds de score → severity

| Score | Severity | Acción |
|------:|----------|--------|
| 0–30 | `low` | Sin acción, usuario normal |
| 31–60 | `medium` | Monitoreo cercano, posible friction adicional |
| 61–80 | `high` | Restricciones automáticas (no Sparks gratis, no leagues, no referrals) |
| 81–100 | `critical` | Cuenta bloqueada hasta verificación humana |

### 4.4 ML model (post-MVP)

Una vez con suficiente data:

- Random forest o XGBoost simple.
- Features: todas las signals.
- Target: probabilidad de fraude.
- Entrenamiento con casos confirmados (banned + appealed-rejected).

NO necesita ser sofisticado para empezar. Reglas explícitas son más
auditables.

---

## 5. Niveles de respuesta

### 5.1 Estructura de niveles

```
Nivel 1: Soft warning
└── Notificación al usuario, no acción.
    "Detectamos actividad inusual. Si no fuiste vos, contactanos."

Nivel 2: Friction adicional
└── Captcha, verificación adicional, slow throttling.

Nivel 3: Restricciones funcionales
└── No puede acceder a features premium gratis.
    No participa en leagues.
    No puede referir.

Nivel 4: Suspensión temporal (REQUIERE REVISIÓN HUMANA)
└── Cuenta pausada por 7-30 días.
    Email de aviso con opción de apelar.

Nivel 5: Ban permanente (REQUIERE REVISIÓN HUMANA)
└── Cuenta cerrada.
    Device fingerprint bloqueado.
    Email blocklisted.
```

### 5.2 Decisión automática vs manual

**Niveles 1-3:** automáticos con buen tuning. Reduce overhead.

**Niveles 4-5:** humano siempre. Evita falsos positivos costosos.

### 5.3 Mapeo restriction_type → restricciones funcionales

```typescript
const RESTRICTION_BEHAVIOR = {
  no_referrals: {
    can_send_referral_codes: false,
    can_earn_referral_rewards: false,
  },
  no_leagues: {
    can_join_leagues: false,
    can_earn_league_rewards: false,
  },
  no_premium_features: {
    can_use_conversations: false,         // bloquea task 'conversation_turn'
    can_use_personalized_roleplays: false,
  },
  no_trial_sparks: {
    receives_initial_50_sparks: false,
    receives_referee_bonus: false,
  },
  suspended: {
    can_login: true,                      // permite login para apelar
    can_use_features: false,
  },
  banned: {
    can_login: false,                     // hard block
  },
};
```

### 5.4 Apelaciones

```
Cuenta restringida → Email con razón + link de apelación →
Form de apelación → Revisión humana en 48h → Decisión: revertir o
mantener
```

**Importante:** ser justo. Falsos positivos pierden customers
legítimos.

---

## 6. Casos específicos

### 6.1 Multiple accounts del mismo usuario

#### Detección

Combinación de signals:
- Device fingerprint match.
- Pattern de email (mismo prefix con números).
- IP cercana o idéntica.
- Tiempo entre signups corto.

#### Respuesta

**Caso 1: Confirmado (matches fuertes en múltiples signals):**
- Cuenta nueva: `restriction_applied = no_trial_sparks`.
- Usuario notificado: "Ya tienes cuenta con nosotros. Inicia sesión
  con X email."

**Caso 2: Posible duplicado (matches débiles):**
- No bloquear.
- Marcar para monitoreo (`fraud_score = medium`).
- Si comportamiento confirma, escalar.

#### Casos legítimos (edge)

- Familia compartiendo device.
- User que olvidó password y crea cuenta nueva.

**Política:** 2 cuentas/device permitidas (dudas razonables). 3+:
investigación.

### 6.2 Account sharing

#### Por qué es problema

- Reduce conversion (1 cuenta para N personas).
- Inconsistencia métricas.
- Conflictos de progreso.

#### Detección

- Sesiones simultáneas desde regiones imposibles.
- Cambios frecuentes de device fingerprint.
- Patrones de aprendizaje contradictorios (B1 errores en una sesión, C1
  en otra).

#### Respuesta

**Soft enforcement (preferido):**
- Limitar a 1 sesión activa simultánea (Firebase soporta).
- Auto-logout si se detecta nueva.
- Mensaje: "Tu cuenta es para uso individual. Familia y amigos pueden
  registrarse con su propia cuenta gratis."

**Hard enforcement (último recurso):**
- Si patrón persiste, restricciones de plan.

### 6.3 Self-referrals masivos

User A se refiere a sí mismo via:
- Cuenta secundaria con mismo device.
- Cuenta de familiar con mismo device.
- Bot creando cuentas falsas que él controla.

#### Anti-abuse rules

- Recompensa de referral SOLO si referido completa onboarding.
- Recompensa adicional SOLO si referido paga primer mes.
- Cap de Sparks por referrals: 100/mes para nuevos usuarios.

#### Detección automática

- Si referrer y referee tienen mismo fingerprint: no otorgar bonus.
- Si referee nunca paga después de meses: clawback de bonus.

### 6.4 Bots de actividad

#### Detección

- Patrones temporales no humanos (cada 5 min exactamente).
- Respuestas repetitivas.
- Velocidad inhumana (< 100ms).
- Audio sintético detectable.

#### Respuesta

- Captcha si patrón.
- Análisis de audio: si nunca es voz humana, flag.
- Revisión manual.

### 6.5 Chargebacks fraudulentos

#### Prevención

- 3DS authentication para tarjetas nuevas.
- Detección de signals pre-payment.
- Email de bienvenida con disclaimer claro de qué incluye el plan.

#### Detección

- Notificación de chargeback dispute de Stripe/RevenueCat.
- Análisis del comportamiento previo.

#### Respuesta

**Disputa legítima (rare):**
- Aceptar, devolver dinero.
- No banear.

**Disputa fraudulenta:**
- Aportar evidencia a Stripe (logs de uso, screenshots).
- Si se pierde la disputa: ban del usuario.
- Dispute fees absorbidos como costo.

---

## 7. Schemas Postgres

### 7.1 Device fingerprints

```sql
CREATE TABLE device_fingerprints (
  fingerprint     TEXT PRIMARY KEY,
  first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  user_count      INT NOT NULL DEFAULT 1,
  flagged         BOOLEAN NOT NULL DEFAULT false,
  flag_reason     TEXT,
  blocklisted     BOOLEAN NOT NULL DEFAULT false,
  blocklist_reason TEXT
);

CREATE INDEX idx_fingerprints_flagged ON device_fingerprints(flagged) WHERE flagged = true;
CREATE INDEX idx_fingerprints_blocklisted ON device_fingerprints(blocklisted) WHERE blocklisted = true;
```

### 7.2 Asociación user-fingerprint

```sql
CREATE TABLE user_devices (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  fingerprint     TEXT NOT NULL REFERENCES device_fingerprints(fingerprint),
  first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, fingerprint)
);

CREATE INDEX idx_user_devices_fingerprint ON user_devices(fingerprint);
```

### 7.3 Fraud scores

```sql
CREATE TABLE user_fraud_scores (
  user_id         UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  current_score   INT NOT NULL DEFAULT 0 CHECK (current_score BETWEEN 0 AND 100),
  severity        TEXT NOT NULL DEFAULT 'low'
                  CHECK (severity IN ('low', 'medium', 'high', 'critical')),
  signals         JSONB NOT NULL DEFAULT '[]',
  last_calculated TIMESTAMPTZ NOT NULL DEFAULT now(),
  manual_override BOOLEAN NOT NULL DEFAULT false,
  override_reason TEXT,
  override_by     TEXT,                          -- admin id
  override_at     TIMESTAMPTZ
);

CREATE INDEX idx_fraud_scores_severity ON user_fraud_scores(severity)
  WHERE severity IN ('high', 'critical');
```

### 7.4 Fraud events

```sql
CREATE TABLE fraud_events (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID REFERENCES users(id),
  event_type      TEXT NOT NULL,
  severity        TEXT NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),
  signals         JSONB NOT NULL,
  action_taken    TEXT,
  detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  reviewed_by     TEXT,
  reviewed_at     TIMESTAMPTZ,
  resolution      TEXT
);

CREATE INDEX idx_fraud_events_user_date ON fraud_events(user_id, detected_at DESC);
CREATE INDEX idx_fraud_events_unreviewed
  ON fraud_events(detected_at DESC)
  WHERE reviewed_at IS NULL AND severity IN ('high', 'critical');
```

### 7.5 User restrictions

```sql
CREATE TABLE user_restrictions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  restriction_type TEXT NOT NULL CHECK (restriction_type IN (
                    'no_referrals',
                    'no_leagues',
                    'no_premium_features',
                    'no_trial_sparks',
                    'suspended',
                    'banned'
                  )),
  reason          TEXT NOT NULL,
  starts_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  ends_at         TIMESTAMPTZ,                   -- NULL = permanente
  applied_by      TEXT NOT NULL DEFAULT 'system',
  appealed        BOOLEAN NOT NULL DEFAULT false,
  appeal_outcome  TEXT,                           -- 'approved' | 'denied' | NULL
  lifted_at       TIMESTAMPTZ,
  lifted_by       TEXT,
  lifted_reason   TEXT
);

CREATE INDEX idx_restrictions_user_active ON user_restrictions(user_id)
  WHERE lifted_at IS NULL AND (ends_at IS NULL OR ends_at > now());
CREATE INDEX idx_restrictions_active_type ON user_restrictions(restriction_type)
  WHERE lifted_at IS NULL AND (ends_at IS NULL OR ends_at > now());
```

### 7.6 Fraud appeals

```sql
CREATE TABLE fraud_appeals (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  restriction_id  UUID REFERENCES user_restrictions(id),
  appeal_text     TEXT NOT NULL,
  submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending', 'reviewing', 'approved', 'denied')),
  reviewed_by     TEXT,
  reviewed_at     TIMESTAMPTZ,
  decision_notes  TEXT
);

CREATE INDEX idx_appeals_pending ON fraud_appeals(submitted_at DESC)
  WHERE status = 'pending';
```

### 7.7 Disposable email blocklist

```sql
CREATE TABLE disposable_email_domains (
  domain          TEXT PRIMARY KEY,
  added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  source          TEXT
);
-- Seed con lista open source de domains de email temporal
```

---

## 8. API contracts

### 8.1 `evaluateFraudOnSignup`

**Llamado por:** auth-system al `user.signed_up`.

**Request:**

```typescript
interface EvaluateFraudOnSignupRequest {
  user_id: string;
  device_info: DeviceInfo;
  ip: string;
  email: string;
  provider: string;
}
```

**Response:**

```typescript
interface EvaluateFraudOnSignupResponse {
  score: number;
  severity: 'low' | 'medium' | 'high' | 'critical';
  signals: string[];
  restrictions_applied: string[];
}
```

### 8.2 `evaluateFraudOngoing`

**Llamado por:** cron diario o triggers específicos.

```typescript
interface EvaluateFraudOngoingRequest {
  user_id: string;
  trigger_event?: string;
}
```

### 8.3 `applyRestriction`

**Request:**

```typescript
interface ApplyRestrictionRequest {
  user_id: string;
  restriction_type: RestrictionType;
  reason: string;
  duration_days?: number;        // null = permanente
  applied_by: 'system' | `admin:${string}`;
}
```

**Reglas:**
- Niveles 4-5 (`suspended`, `banned`) requieren `applied_by` admin
  no `system`.
- Emite evento `restriction.applied`.

### 8.4 `liftRestriction`

```typescript
interface LiftRestrictionRequest {
  restriction_id: string;
  lifted_by: 'system' | `admin:${string}` | 'appeal_approved';
  reason: string;
}
```

### 8.5 `submitAppeal`

```typescript
interface SubmitAppealRequest {
  restriction_id: string;
  appeal_text: string;
}

interface SubmitAppealResponse {
  appeal_id: string;
  estimated_review_time_hours: number;  // 48
}
```

### 8.6 `getActiveRestrictions`

**Llamado por:** Sparks system, otros sistemas que necesitan saber si
user tiene restricción activa.

```typescript
interface GetActiveRestrictionsRequest {
  user_id: string;
}

interface GetActiveRestrictionsResponse {
  restrictions: Array<{
    id: string;
    type: RestrictionType;
    starts_at: string;
    ends_at: string | null;
    reason: string;
  }>;
}
```

Crítico: cacheable por 60s. Otros sistemas leen este endpoint
frecuentemente.

### 8.7 `reviewAppeal` (admin only)

```typescript
interface ReviewAppealRequest {
  appeal_id: string;
  decision: 'approved' | 'denied';
  decision_notes: string;
  reviewed_by: string;            // admin id
}
```

**Reglas:**
- Si `approved`: lift de la restriction asociada.
- Emite eventos correspondientes.

---

## 9. Eventos emitidos

(Detalle en `cross-cutting/data-and-events.md` §5.9.)

| Evento | Cuándo |
|--------|--------|
| `fraud.score_calculated` | Cada vez que se actualiza el score |
| `restriction.applied` | Aplicación de cualquier restriction |
| `restriction.lifted` | Lift por sistema, admin o apelación |
| `fraud.appeal_submitted` | User envía apelación |
| `fraud.appeal_reviewed` | Admin resuelve apelación |
| `fraud.event_detected` | Nuevo fraud_event creado |

---

## 10. Edge cases (tests obligatorios)

### 10.1 Score calculation

1. **Score llega a 100 con muchas signals:** clamp a 100, severity
   `critical`, restricciones automáticas.
2. **Score baja después de comportamiento normal sostenido (30 días):**
   restricciones nivel 1-3 se liftean automáticamente; niveles 4-5
   requieren manual.

### 10.2 Restrictions

3. **User con `restriction = banned` intenta login:** Auth permite
   token generation pero `applyRestriction.banned` → 403 en cualquier
   feature.
4. **User con `suspended` que expira mientras está logueado:** próximo
   API call ve restriction expired, comportamiento normal.
5. **Admin aplica `banned`, user apela y appeal aprobada:** lift +
   evento; fraud_score se mantiene (no auto-reset).

### 10.3 Fingerprinting

6. **Mismo device, 2 accounts:** OK, no flag.
7. **Mismo device, 3 accounts en < 24h:** flag, fraud_score += 30.
8. **Mismo device, 5+ accounts:** auto-restriction `no_trial_sparks`.
9. **Fingerprint blocklisted:** próximo signup bloqueado en
   `evaluateFraudOnSignup`.

### 10.4 Self-referrals

10. **Referrer y referee comparten device:** referral bonus NO
    otorgado, pero ambos pueden seguir usando la app.
11. **Referee abandona después de signup, sin pagar:** después de 60d,
    clawback de bonus si fue otorgado prematuramente.

### 10.5 Apelaciones

12. **User submite 3 apelaciones consecutivas para misma restricción:**
    rate limit max 1/24h. Las extra se rechazan con mensaje.
13. **Apelación aprobada para `banned`:** restriction lifted, account
    accessible normalmente, fraud_score se evalúa de nuevo.

### 10.6 Compromised accounts

14. **Login desde país X de un user que vive en país Y:** flag suave,
    enviar email "fue tu login?".
15. **Sesión simultánea geo-imposible (CDMX y Madrid en 2h):** force
    logout de la más antigua, requiere re-auth.

### 10.7 Concurrencia

16. **Multiple fraud_score updates concurrentes para mismo user:**
    `FOR UPDATE` lock, último wins; signals se mergean.

---

## 11. Observabilidad

### 11.1 Métricas críticas

| Métrica | Definición | Target |
|---------|-----------|--------|
| Tasa de cuentas flaggeadas | % de signups con fraud_score >= medium | 5–10% (esperado) |
| Distribución de fraud scores | Debería ser piramidal | bimodal extremo = bug |
| **False positive rate** | % de cuentas restringidas que apelan y son aprobadas | < 5% |
| **False negative rate** | % de fraude no detectado (cuando se descubre tarde) | < 10% |
| Time to detection | Tiempo desde fraude hasta detección | < 24h |
| Volumen de apelaciones | # por mes | Bajo y estable |
| Tiempo promedio resolución apelación | | < 48h |
| $ perdidos por chargebacks | mensual | < 1% del revenue |

### 11.2 ROI del sistema

```
ROI = ($ ahorrados estimados - $ pérdidas - $ costos del sistema)
      / $ inversión
```

A 50.000 users: ~$200/mes en infra + tiempo de revisión humana.

### 11.3 Alertas

- False positive rate > 10%: SEV-2 (sistema demasiado agresivo).
- Spike en apelaciones (> 2x baseline): SEV-2.
- Fraud event critical no revisado en 24h: SEV-2.
- Spike en signups desde una IP/country: SEV-3.
- Chargeback rate > 1% de revenue: SEV-2.

---

## 12. Decisiones cerradas

### 12.1 Compartir signals con otras apps de la industria: **NO en MVP** ✓

**Razón:** sin masa crítica para alianzas. Reconsiderar año 2.

### 12.2 Pre-screening agresivo en países con altas tasas: **NO** ✓

**Razón:** discriminación geográfica afecta a usuarios legítimos.
Aplicar mismas reglas a todos.

### 12.3 Soft launch de features anti-fraude (shadow mode): **SÍ** ✓

**Razón:** observar 30 días en shadow antes de aplicar restrictions.
Calibra falsos positivos sin afectar a usuarios.

### 12.4 Identity verification (KYC light) para premium: **NO en MVP** ✓

**Razón:** fricción muy alta para consumer. Reconsiderar si chargebacks
suben significativamente.

### 12.5 Programa "verified user": **NO en MVP** ✓

**Razón:** sobre-ingeniería sin evidencia de necesidad. Considerar año 2
si abuso de cuentas múltiples es problema persistente.

---

## 13. Aspectos legales

### 13.1 Privacidad y compliance

**Fingerprinting puede ser regulado:**
- En EU (GDPR): puede requerir consent.
- En California (CCPA): notice y opt-out.
- En Latam: regulaciones varían.

**Mejor práctica:**
- Disclosure en privacy policy.
- Opt-out posible para usuarios que lo requieran (con limitaciones).
- Datos no vendidos a terceros.

(Ver `business/legal-compliance.md` §3.2 para detalle.)

### 13.2 Terms of Service

T&C debe incluir:
- Cláusulas anti-abuse claras.
- Right to suspend/ban accounts.
- Política sobre múltiples cuentas.
- Política de chargebacks.
- Definición de "uso aceptable".

### 13.3 Comunicación al usuario

Cuando se aplica restricción:
- Email/in-app notification clara.
- Explicación general (NO detallar todos los signals para no enseñar
  cómo evadir).
- Información de cómo apelar.
- Proceso justo y honesto.

---

## 14. Plan de implementación

### 14.1 Fase 1: Foundations (meses 0–3)

**Crítico:**
- Schemas §7.
- Device fingerprinting básico.
- Rate limiting en endpoints sensibles.
- Email verification obligatoria para premium.
- Trial cap de 50 Sparks (sparks-system).

**Importante:**
- Detección de duplicate accounts vía fingerprint.
- Logging de signals sospechosos.

### 14.2 Fase 2: Detección activa (meses 3–9)

**Crítico:**
- Fraud scoring algorithm con reglas claras.
- Restricciones automáticas niveles 1-3.
- Sistema de apelaciones.
- Anti-self-referral logic.

**Importante:**
- Detección de account sharing.
- Patrones de bot detection.
- Shadow mode 30 días antes de enforcement.

### 14.3 Fase 3: Sofisticación (meses 9–18)

- ML model para fraud prediction.
- Calibración basada en datos reales.
- A/B testing de restricciones.
- Detección de chargeback risk pre-payment.

### 14.4 Fase 4: Escala (año 2+)

- Equipo dedicado o partner especializado (Sift).
- Threat intelligence sharing.
- Análisis forense de incidentes grandes.

---

## 15. Herramientas externas

### 15.1 Para fraud prevention

- **Stripe Radar:** built-in con Stripe, payment fraud. Free.
- **Sift:** sofisticado pero caro. Considerar a escala.
- **MaxMind:** IP geolocation y fraud signals. Per API call.
- **Recaptcha:** Google, gratis, anti-bot.

### 15.2 Para email validation

- **ZeroBounce / NeverBounce:** verificación email real.
- Listas open source de disposable email domains.

### 15.3 Para device fingerprinting

- **FingerprintJS:** servicio managed, $200/mes.
- Self-hosted: combinación de signals nativos del SDK móvil + custom.
  Más trabajo pero gratis.

---

## 16. Riesgos del sistema mismo

| Riesgo | Mitigación |
|--------|-----------|
| Falsos positivos altos → churn | Calibración cuidadosa, apelaciones rápidas |
| Sistema visible/molesto para legítimos | Friction proporcional, invisible cuando posible |
| Backlash en redes sociales | Comunicación clara, fairness obvia |
| Compliance issues | Legal review, documentation |
| Sistema crackeable | Reglas en backend, rotación de pesos |
| Costo de mantenimiento | Automatización, foco en ROI |

---

## 17. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`sparks-system.md`](sparks-system.md) | Trial limits y rate limits; lee `user_restrictions`. |
| [`authentication-system.md`](authentication-system.md) | Account validation. |
| [`../business/customer-support-system.md`](../business/customer-support-system.md) | Apelaciones y disputes. |
| [`notifications-system.md`](notifications-system.md) | Notificación de restricciones. |
| [`../business/legal-compliance.md`](../business/legal-compliance.md) §5.3 | T&C de suspensión/ban. |
| [`../cross-cutting/security-threat-model.md`](../cross-cutting/security-threat-model.md) §3.9 | Amenazas anti-fraud system. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.9 | Eventos `fraud.*`, `restriction.*`. |

---

*Documento vivo. Actualizar cuando aparezcan nuevos vectores de fraude
o se descubran patrones nuevos en producción.*

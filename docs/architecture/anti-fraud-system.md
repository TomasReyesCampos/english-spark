# Anti-Fraud and Abuse Prevention System

> Sistema integral para prevenir, detectar y responder a abusos del
> producto: cuentas falsas, farming de Sparks, account sharing,
> chargebacks fraudulentos, y otros vectores de pérdida.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Alcance:** Sistema completo

---

## 1. Por qué importa anti-fraude

### 1.1 El problema invisible

A pequeña escala, el fraude parece despreciable. A escala, no lo es:

- 100 cuentas falsas farmeando Sparks: pocos dólares de pérdida.
- 10.000 cuentas falsas farmeando Sparks: miles de dólares y degrada el
  sistema (logros pierden valor, leagues se contaminan, datos de producto
  se distorsionan).

### 1.2 Vectores de fraude en este producto

**Farming de Sparks:**
- Crear múltiples cuentas para acumular Sparks gratis del trial.
- Auto-referidos para ganar Sparks de referral.
- Bots que generan actividad falsa para ganar logros.

**Account sharing:**
- Una cuenta paga compartida entre 5 personas.
- Reduce conversion potencial.

**Chargebacks fraudulentos:**
- Pagar, usar el servicio, hacer chargeback.
- Costoso (chargeback fee + revenue perdido + posible penalty).

**Manipulación de métricas:**
- Inflar artificialmente métricas (leagues, leaderboards).
- Daña la experiencia de usuarios legítimos.

**Abuso de soporte:**
- Solicitudes de refund repetitivas.
- Crear tickets falsos para drenar tiempo.

**Content abuse:**
- Generar reportes falsos de assets.
- Sabotear contenido para competidores.

### 1.3 Filosofía adoptada

**Prevención > detección > castigo:**
- Mejor diseñar el sistema para que el fraude sea costoso/imposible.
- Detectar lo que pase a pesar de la prevención.
- Castigar solo cuando hay evidencia clara.

**Friction proporcional al riesgo:**
- No molestar a 99% de usuarios legítimos por proteger contra el 1% malicioso.
- Aumentar fricción solo cuando hay señales sospechosas.

**Sistema invisible para usuarios honestos:**
- Las protecciones funcionan en background.
- El usuario legítimo no debe sentir el sistema.

**Iterar con datos:**
- El fraude evoluciona. El sistema también debe evolucionar.

---

## 2. Modelo de amenazas

### 2.1 Tipos de actores

**Casual abuser:** usuario que rompe reglas ocasionalmente, generalmente
sin malicia. Ej: comparte cuenta con un familiar.

**Self-server:** usuario que busca aprovechar el sistema para sí mismo.
Ej: crea 5 cuentas para usar trial varias veces.

**Small operator:** persona o pequeño grupo que sistemáticamente abusa
para beneficio personal (revender cuentas, vender Sparks).

**Sophisticated attacker:** actor con recursos técnicos que usa
automatización (bots, fingerprint spoofing). Más raro pero más dañino.

**Compromised account:** usuario legítimo cuya cuenta fue robada por un
tercero.

### 2.2 Severidad y frecuencia esperada

| Tipo | Frecuencia esperada | Daño potencial | Prioridad |
|------|---------------------|----------------|-----------|
| Casual abuser | Alta (5-10% de usuarios) | Bajo individual | Baja |
| Self-server | Media (1-3%) | Medio | Media |
| Small operator | Baja (<0.5%) | Alto | Alta |
| Sophisticated attacker | Muy baja | Muy alto | Crítica si aparece |
| Compromised account | Baja-media | Alto | Alta |

---

## 3. Prevención por diseño

### 3.1 Trial diseñado para minimizar farming

#### Cap absoluto de Sparks

Free trial otorga **máximo 50 Sparks**, no rolling, no acumulables. Aunque
alguien cree 100 cuentas, no obtiene 100 × 50 Sparks utilizables (cada
cuenta es independiente).

#### Limitaciones del trial

Durante el trial:
- No se pueden referir amigos (recompensa de referral solo después de pagar).
- No se pueden acumular Sparks comprados.
- No participan en leagues competitivas (evita contaminar leaderboards).

#### Trial expira con consecuencias claras

Después del trial sin convertir:
- Acceso solo a preassets gratis.
- Cuenta queda activa pero limitada.
- Crear nueva cuenta para "re-trial" no es atractivo si: misma device
  detectada, mismo email, etc.

### 3.2 Fingerprinting de devices

Para detectar múltiples cuentas del mismo device:

**Datos capturados (no PII estricto):**
- Device ID (Apple/Android device identifier).
- Hash de combinación de hardware/software signals.
- IP address y patrón de cambio.
- Timezone y language settings.
- Screen resolution y características del device.

**Combinación genera fingerprint:**

```typescript
function generateDeviceFingerprint(deviceInfo: DeviceInfo): string {
  const components = [
    deviceInfo.deviceId,
    deviceInfo.platform,
    deviceInfo.osVersion,
    deviceInfo.modelName,
    deviceInfo.screenSize,
    deviceInfo.language,
    deviceInfo.timezone,
    // Hash de IP /24 (no exact IP)
    hashIpRange(deviceInfo.ip, 24)
  ];

  return sha256(components.join('|'));
}
```

**Uso:**
- Si nuevo signup tiene mismo fingerprint que cuenta existente, flag.
- Si fingerprint matches con 5+ cuentas, restricción automática.

**Importante:** fingerprinting tiene limitaciones legales. En Europa
(GDPR) requiere consent en algunos casos. Verificar compliance por
jurisdicción.

### 3.3 Verificación de email/teléfono

**Email verification:**
- Email verification enviado al registrarse.
- Funcionalidades premium requieren email verificado.
- Domains de email temporal (10minutemail, etc.) bloqueados.

**Phone verification (si se implementa Phone OTP):**
- Verificación de número real.
- Bloqueo de números de SMS virtuales (Twilio numbers, etc.) detectables.

### 3.4 Rate limiting agresivo

**Por usuario:**
- Máximo X conversaciones IA por minuto (anti-bot).
- Máximo Y signups por device por mes.
- Máximo Z referrals enviados por día.

**Por IP:**
- Throttling de signups (signup desde misma IP cada N minutos).
- Throttling de password resets.
- Throttling de support tickets.

**Por endpoint:**
- API de Sparks consumption con rate limit estricto.
- Endpoints sensibles con captcha si se exceden umbrales.

---

## 4. Detección activa

### 4.1 Señales de fraude

#### Para farming de Sparks

**Patrones detectables:**
- Múltiples cuentas con mismo device fingerprint.
- Email patterns: `user1@x.com`, `user2@x.com`, `user3@x.com`.
- Names patterns: usuarios con nombres random o muy similares.
- Comportamiento: signup → trial → abandono inmediatamente después de gastar Sparks → otro signup.
- Network IP analysis: signups masivos desde misma IP/range.

**Score de sospecha:**

```typescript
function calculateFraudScore(user: User): number {
  let score = 0;

  // Device fingerprint matches
  const matchingFingerprints = countMatchingFingerprints(user.fingerprint);
  if (matchingFingerprints > 1) score += 30;
  if (matchingFingerprints > 3) score += 50;

  // Email patterns
  if (isSuspiciousEmail(user.email)) score += 20;

  // IP analysis
  const ipSignups = countRecentSignupsFromIp(user.ip, 24); // last 24h
  if (ipSignups > 3) score += 25;
  if (ipSignups > 10) score += 50;

  // Behavioral
  if (user.daysActive < 1 && user.sparksConsumed > 30) score += 15;

  // Velocity
  if (user.signupToFirstSpend < 60) score += 20; // segundos

  return Math.min(score, 100);
}
```

**Thresholds:**
- 0-30: usuario normal, sin acción.
- 31-60: monitoreo cercano, posiblemente friction adicional.
- 61-80: restricciones automáticas (no poder consumir Sparks gratis).
- 81-100: cuenta bloqueada hasta verificación.

#### Para account sharing

**Patrones:**
- Logins simultáneos desde IPs/regions muy distintas en poco tiempo.
- Cambio frecuente de device fingerprints.
- Patrones de uso inconsistentes (matutino + nocturno + tarde, sugiere
  varias personas).

**Acción:**
- Detección suave: notificar al usuario "detectamos múltiples sesiones,
  ¿es esto vos?".
- Si patrón persiste y es claro: limitar a 1 sesión activa.

#### Para self-referrals

**Patrones:**
- Referente y referido con mismo device fingerprint.
- Referido se registra inmediatamente después de generación de código.
- Email y teléfono del referido nunca verificados.
- Referido nunca activa más allá del signup.

**Acción:**
- No otorgar bonus de referral si pattern es claro.
- Investigar manualmente si volumen es alto.

#### Para chargebacks

**Pre-payment signals:**
- País con alta tasa de chargebacks (varía mes a mes).
- Tarjeta nunca usada antes en nuestro sistema.
- Mismo método de pago en cuentas previas con chargebacks.

**Post-payment signals:**
- Uso intensivo seguido de inactividad.
- Cancelación seguida de chargeback dispute.
- Multiple disputes en historial.

**Acción:**
- Para signals altos: requerir 3DS authentication.
- Para chargebacks repetidos: ban permanente.

### 4.2 Detección con ML (post-MVP)

Una vez con suficiente data, modelo ML que:
- Toma todas las señales como features.
- Predice probabilidad de fraude.
- Mejor que reglas hardcoded en patrones complejos.

Implementación: random forest o XGBoost simple. No necesita ser sofisticado
para empezar.

---

## 5. Respuestas al fraude

### 5.1 Niveles de respuesta

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

Nivel 4: Suspensión temporal
└── Cuenta pausada por 7-30 días.
    Email de aviso con opción de apelar.

Nivel 5: Ban permanente
└── Cuenta cerrada.
    Device fingerprint bloqueado.
    Email blocklisted.
```

### 5.2 Decisión automática vs manual

**Automatización segura:**
- Niveles 1-3 pueden ser automáticos con buen tuning.
- Reduce overhead operacional.

**Requiere revisión manual:**
- Nivel 4 (suspensión): humano revisa antes de aplicar.
- Nivel 5 (ban): humano siempre, evitar falsos positivos costosos.

### 5.3 Apelaciones

Sistema de apelación para usuarios que creen ser legítimos:

```
Cuenta restringida → Email al usuario explicando razón →
Link a formulario de apelación → Revisión manual en 48h →
Decisión: revertir o mantener
```

Importante: ser justo. Falsos positivos pierden customers legítimos.

---

## 6. Casos específicos detallados

### 6.1 Multiple accounts del mismo usuario

#### Detección

Combinación de signals:
- Device fingerprint match.
- Pattern de email (mismo prefix con números diferentes).
- IP cercana o idéntica.
- Tiempo entre signups corto.

#### Respuesta

**Caso 1: Confirmado fraude (matches fuertes en múltiples signals):**
- Cuenta nueva: marcada como duplicate.
- Sparks gratuitos del trial: no otorgados.
- Usuario notificado: "Ya tenés cuenta con nosotros. Iniciá sesión con
  X email."

**Caso 2: Posible duplicado (matches débiles):**
- No bloquear.
- Marcar para monitoreo.
- Si comportamiento confirma, escalar.

#### Caso edge: usuarios legítimos

Algunos casos legítimos:
- Familia compartiendo device.
- Usuario que olvidó password y crea nueva cuenta.

Política:
- 2 cuentas por device permitidas (dudas razonables).
- 3+ cuentas: investigación.

### 6.2 Account sharing

#### Por qué es problema

- Reduce conversion (1 cuenta para 5 personas en lugar de 5 cuentas).
- Inconsistencia en métricas (¿quién es realmente el usuario?).
- Conflictos: progreso confuso si varias personas usan misma cuenta.

#### Detección

- Sesiones simultáneas (probablemente OK, pero...).
- Sesiones desde regiones imposibles de cubrir (México y Tokio en 2 horas).
- Patrones de aprendizaje contradictorios (B1 errores en una sesión, C1
  en otra).

#### Respuesta

**Soft enforcement (preferido):**
- Limitar a 1 sesión activa simultánea.
- Auto-logout si se detecta nueva sesión.
- Mensaje: "Tu cuenta es para uso individual. Familia y amigos pueden
  registrarse con su propia cuenta gratis."

**Hard enforcement (último recurso):**
- Si patrón persiste, restricciones de plan.

### 6.3 Self-referrals masivos

#### Detección

User A se refiere a sí mismo via:
- Cuenta secundaria con mismo device.
- Cuenta de familiar con mismo device.
- Bot creando cuentas falsas que él controla.

#### Respuesta

**Anti-abuse rules:**
- Recompensa de referral solo si referido completa onboarding.
- Recompensa adicional solo si referido paga primer mes.
- Cap de Sparks ganados por referrals: 100/mes para nuevos usuarios.

**Detección automática:**
- Si referente y referido tienen mismo fingerprint: no otorgar.
- Si referido nunca paga después de varios meses: clawback de bonus.

### 6.4 Bots de actividad

Bots que automatizan actividad para ganar logros y Sparks.

#### Detección

- Patrones temporales no humanos (siempre exactamente cada 5 minutos).
- Respuestas repetitivas (mismas respuestas en ejercicios diferentes).
- Velocidad inhumana (responder en 100ms).
- Audio sintético detectable (TTS, no humano).

#### Respuesta

- Captcha si se detecta patrón.
- Análisis de audio: si nunca es voz humana, flagging.
- Revisión manual.

### 6.5 Chargebacks fraudulentos

#### Prevención

- 3DS authentication para tarjetas nuevas.
- Detección de signals de fraude pre-payment.
- Email de bienvenida con disclaimer claro de qué incluye el plan.

#### Detección

- Chargeback dispute notification de Stripe/RevenueCat.
- Análisis del comportamiento previo al chargeback.

#### Respuesta

**Disputa legítima (rare):**
- Aceptar, devolver dinero.
- No banear.

**Disputa fraudulenta:**
- Aportar evidencia a Stripe (logs de uso, screenshots).
- Si se pierde la disputa: ban del usuario.
- Dispute fees absorbidos como costo de hacer business.

---

## 7. Schema en Postgres

```sql
-- Fingerprints de devices
CREATE TABLE device_fingerprints (
  fingerprint     TEXT PRIMARY KEY,
  first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  user_count      INT DEFAULT 1,
  flagged         BOOLEAN DEFAULT false,
  flag_reason     TEXT
);

-- Asociación usuarios-fingerprints
CREATE TABLE user_devices (
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  fingerprint     TEXT NOT NULL REFERENCES device_fingerprints(fingerprint),
  first_seen_at   TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (user_id, fingerprint)
);

-- Score de fraude por usuario
CREATE TABLE user_fraud_scores (
  user_id         UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  current_score   INT NOT NULL DEFAULT 0 CHECK (current_score BETWEEN 0 AND 100),
  signals         JSONB DEFAULT '[]',
  last_calculated TIMESTAMPTZ DEFAULT now(),
  manual_override BOOLEAN DEFAULT false,
  override_reason TEXT
);

-- Eventos de fraude detectados
CREATE TABLE fraud_events (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID REFERENCES users(id),
  event_type      TEXT NOT NULL,
  severity        TEXT NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),
  signals         JSONB NOT NULL,
  action_taken    TEXT,
  detected_at     TIMESTAMPTZ DEFAULT now(),
  reviewed_by     TEXT,
  reviewed_at     TIMESTAMPTZ,
  resolution      TEXT
);

-- Restricciones aplicadas
CREATE TABLE user_restrictions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  restriction_type TEXT NOT NULL CHECK (restriction_type IN (
    'no_referrals', 'no_leagues', 'no_premium_features',
    'suspended', 'banned'
  )),
  reason          TEXT NOT NULL,
  starts_at       TIMESTAMPTZ DEFAULT now(),
  ends_at         TIMESTAMPTZ,
  applied_by      TEXT,
  appealed        BOOLEAN DEFAULT false,
  appeal_outcome  TEXT
);

-- Apelaciones
CREATE TABLE fraud_appeals (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  restriction_id  UUID REFERENCES user_restrictions(id),
  appeal_text     TEXT NOT NULL,
  submitted_at    TIMESTAMPTZ DEFAULT now(),
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending', 'reviewing', 'approved', 'denied')),
  reviewed_by     TEXT,
  reviewed_at     TIMESTAMPTZ,
  decision_notes  TEXT
);
```

---

## 8. Métricas de salud del sistema anti-fraude

### 8.1 Volume metrics

- Tasa de cuentas flaggeadas (% del total de signups).
- Distribución de fraud scores (debería ser piramidal: muchos bajo, pocos alto).
- Acciones automáticas tomadas por nivel.

### 8.2 Calidad

- **False positive rate:** % de cuentas restringidas que son legítimas.
  Target: < 5%.
- **False negative rate:** % de fraude no detectado (medido cuando se
  descubre tarde). Target: < 10%.
- **Time to detection:** tiempo desde fraude hasta detección. Target: < 24h.

### 8.3 Apelaciones

- Volumen de apelaciones.
- % de apelaciones aprobadas (alto sugiere falsos positivos).
- Tiempo promedio de resolución.

### 8.4 Pérdidas

- $ perdidos por chargebacks.
- $ perdidos por farming de Sparks.
- $ perdidos por account sharing (estimado).
- $ ahorrados por sistema (estimado).

ROI del sistema: ($ ahorrados - $ pérdidas - $ costos del sistema) / $ inversión.

---

## 9. Plan de implementación

### 9.1 Fase 1: Foundations (meses 0-3)

**Crítico:**
- Schema de fraud detection en Postgres.
- Device fingerprinting básico.
- Rate limiting en endpoints sensibles.
- Email verification obligatoria para premium.
- Trial cap de 50 Sparks (ya en sistema de Sparks).

**Importante:**
- Detección de duplicate accounts via fingerprint.
- Logging de signals sospechosos.

### 9.2 Fase 2: Detección activa (meses 3-9)

**Crítico:**
- Fraud scoring algorithm con reglas claras.
- Restricciones automáticas Niveles 1-3.
- Sistema de apelaciones.
- Anti-self-referral logic.

**Importante:**
- Detección de account sharing.
- Patrones de bot detection.

### 9.3 Fase 3: Sofisticación (meses 9-18)

**Crítico:**
- ML model para fraud prediction.
- Calibración basada en datos reales.
- A/B testing de restricciones (lo que funciona vs no).

**Importante:**
- Detección de chargeback risk pre-payment.
- Behavioral biometrics si se justifica.

### 9.4 Fase 4: Escala (año 2+)

- Equipo dedicado o partner especializado.
- Integración con servicios externos (Sift, Stripe Radar).
- Análisis forense de incidents grandes.
- Threat intelligence sharing con otras apps.

---

## 10. Herramientas y servicios externos

### 10.1 Para fraud prevention

**Stripe Radar:** built-in con Stripe, detecta payment fraud. Free.

**Sift:** anti-fraud sofisticado pero caro. Considerar a escala.

**MaxMind:** IP geolocation y fraud signals. Pricing por API call.

**Recaptcha:** Google, gratis, anti-bot.

### 10.2 Para email validation

**ZeroBounce, NeverBounce:** verificación de emails reales.

**Detección de disposable emails:** listas open source de domains de
emails temporales.

### 10.3 Para device fingerprinting

**FingerprintJS:** servicio managed, $200/mes para volúmenes medios.

**Self-hosted:** combinación de signals nativos del SDK móvil + custom logic.
Más trabajo pero gratis.

---

## 11. Aspectos legales

### 11.1 Privacidad y compliance

**Fingerprinting puede ser regulado:**
- En EU (GDPR): puede requerir consent.
- En California (CCPA): notice y opt-out.
- En Latam: regulaciones varían.

**Mejor práctica:**
- Disclosure en privacy policy.
- Opt-out posible para usuarios que lo requieran (con limitaciones).
- Datos no vendidos a terceros.

### 11.2 Terms of Service

T&C debe incluir:
- Cláusulas anti-abuse claras.
- Right to suspend/ban accounts.
- Política sobre múltiples cuentas.
- Política de chargebacks.
- Definición de "uso aceptable".

### 11.3 Comunicación al usuario

Cuando se aplica restricción:
- Email/in-app notification clara.
- Explicación general (no detallar todos los signals para no enseñar
  cómo evadir).
- Información de cómo apelar.
- Proceso justo y honesto.

---

## 12. Decisiones abiertas

- [ ] ¿Compartir signals de fraude con otras apps de la industria?
  Alianzas anti-fraude.
- [ ] ¿Pre-screening agresivo en países con altas tasas de fraude vs
  experiencia neutral para todos?
- [ ] ¿Soft launch de features anti-fraude (shadow mode antes de
  enforcement)?
- [ ] ¿Investigar identity verification para usuarios premium (KYC light)?
- [ ] ¿Programa "verified user" con badge y privilegios para usuarios
  comprobadamente legítimos a largo plazo?

---

## 13. Riesgos del sistema mismo

| Riesgo | Mitigación |
|--------|-----------|
| Falsos positivos altos → churn | Calibración cuidadosa, apelaciones rápidas |
| Sistema visible/molesto para legítimos | Friction proporcional, invisible cuando posible |
| Backlash en redes sociales | Comunicación clara, fairness obvia |
| Compliance issues | Legal review, documentation |
| Sistema crackeable | Reglas en backend, no cliente |
| Costo de mantenimiento | Automatización, foco en ROI |

---

## 14. Referencias internas

- `docs/architecture/sparks-system.md` — Trial limits y rate limits.
- `docs/architecture/authentication-system.md` — Account validation.
- `docs/product/customer-support-system.md` — Apelaciones y disputes.
- `docs/architecture/notifications-system.md` — Notificación de
  restricciones.

---

*Documento vivo. Actualizar cuando aparezcan nuevos vectores de fraude
o se descubran patrones nuevos en producción.*

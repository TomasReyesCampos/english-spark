# Architecture Overview

> Vista de pájaro de todos los sistemas, sus contratos, sus dependencias
> y el flujo de datos end-to-end. Este documento es el **mapa** que
> permite al agente AI entender cómo encaja cada sistema individual en
> el todo, antes de leer el detalle de cada uno.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador. Leer **antes** de
empezar a implementar cualquier sistema, para entender contratos
externos.

---

## 0. Cómo usar este documento

- §2 lista los sistemas y para qué sirve cada uno (mapa).
- §3 define **boundaries**: qué hace y qué NO hace cada sistema.
- §4 define **data ownership**: cada tabla tiene un único sistema dueño.
- §5 define **contratos** (APIs y eventos) entre sistemas.
- §6 define los **flujos end-to-end** de los user journeys principales.
- §7 define **failure modes**: qué hace cada sistema cuando otro falla.
- §8 define el **orden de implementación** recomendado.

Cuando vayas a implementar el sistema X:
1. Leé §3 (boundaries) para X y los sistemas con los que interactúa.
2. Leé §5 (contratos) para los puntos de contacto.
3. Leé el documento detallado de X (`architecture/<x>.md` o
   `product/<x>.md`).
4. Verificá que tu implementación respeta §4 (data ownership): no
   leer/escribir tablas de otro sistema directamente.

---

## 1. Principio rector de la arquitectura

> **Cada sistema es responsable de un dominio de problema y expone
> contratos explícitos. Otros sistemas consumen contratos, no estado
> interno.**

Implicaciones:

- **No hay tablas compartidas entre sistemas.** Cada tabla tiene un
  dueño. Si otro sistema necesita los datos, los pide vía API o se
  suscribe a eventos.
- **No hay imports entre módulos de sistemas distintos.** Un controller
  de `ai-roadmap-system` no llama directamente a una función de
  `pedagogical-system`. Pasa por el contrato público (HTTP RPC interno
  o evento).
- **Eventos son la vía de notificación cross-system.** APIs son la vía
  de query/comando síncrono. No mezclar.
- **AI Gateway es el único path para LLMs.** Sin excepciones.

---

## 2. Inventario de sistemas

14 sistemas + cross-cutting concerns. Cada uno tiene un documento de
diseño detallado.

### 2.1 Sistemas de plataforma (architecture/)

| Sistema | Doc | Para qué sirve |
|---------|-----|----------------|
| **AI Gateway** | [`ai-gateway-strategy.md`](ai-gateway-strategy.md) | Capa única de acceso a LLMs. Multi-proveedor, fallback, A/B testing, cost tracking. |
| **Authentication** | [`authentication-system.md`](authentication-system.md) | Firebase Auth + sincronización con Postgres. SSO, anonymous, account deletion. |
| **Sparks** | [`sparks-system.md`](sparks-system.md) | Tokens de consumo. Control de costos, gamificación, ajuste dinámico de precio. |
| **Notifications** | [`notifications-system.md`](notifications-system.md) | Push via FCM. Daily reminders, streak alerts, re-engagement, transactional. |
| **Anti-fraud** | [`anti-fraud-system.md`](anti-fraud-system.md) | Fingerprinting, fraud scoring, restricciones automáticas, apelaciones. |
| **Platform** | [`platform-strategy.md`](platform-strategy.md) | Estrategia móvil/web/desktop. Stack por plataforma. |

### 2.2 Sistemas de producto (product/)

| Sistema | Doc | Para qué sirve |
|---------|-----|----------------|
| **AI Roadmap** | [`../product/ai-roadmap-system.md`](../product/ai-roadmap-system.md) | Genera y adapta roadmap personalizado. Decide próximo bloque. |
| **Pedagogical** | [`../product/pedagogical-system.md`](../product/pedagogical-system.md) | Scoring, mastery por sub-skill, estado de bloque, decay, spaced repetition. |
| **Student Profile** | [`../product/student-profile-and-assessment.md`](../product/student-profile-and-assessment.md) | Perfil del usuario. Onboarding, trial 7 días, assessment del día 7. |
| **Motivation** | [`../product/motivation-and-achievements.md`](../product/motivation-and-achievements.md) | Streaks, logros, leagues, sistema social. Detecta perfil motivacional. |
| **Content Creation** | [`../product/content-creation-system.md`](../product/content-creation-system.md) | Pipeline de creación y curaduría de assets. Bibliografía. |

### 2.3 Sistemas de negocio (business/)

| Sistema | Doc | Para qué sirve |
|---------|-----|----------------|
| **Customer Support** | [`../business/customer-support-system.md`](../business/customer-support-system.md) | Self-service, AI Assistant, escalación humana, tickets. |
| **Metrics & Experimentation** | [`../business/metrics-and-experimentation.md`](../business/metrics-and-experimentation.md) | Tracking, NSM (WPU), A/B testing, dashboards. |

### 2.4 Cross-cutting concerns (cross-cutting/, ops/)

A escribir (Ola 3+):

- `cross-cutting/data-and-events.md` — modelo unificado de eventos.
- `cross-cutting/security-threat-model.md` — STRIDE, mitigaciones.
- `cross-cutting/testing-strategy.md` — unit + integration (no E2E).
- `cross-cutting/i18n.md` — estrategia de localización.
- `business/legal-compliance.md` — T&C, privacy, GDPR/LFPDPPP.
- `ops/incidents-runbook.md` — manejo de incidentes operativos.
- `ops/cicd.md` — pipelines de CI/CD.

---

## 3. Boundaries: qué hace y qué NO hace cada sistema

### 3.1 AI Gateway

**Hace:**
- Routing de tareas IA a proveedores (Anthropic, Google, OpenAI, Azure,
  modelos propios).
- Validación de outputs con Zod.
- Retry, fallback chain, A/B testing.
- Logging de cada llamada (costo, latencia, validation result).
- Cobro de Sparks **antes** de invocar el modelo (delegando a Sparks
  system).

**No hace:**
- Lógica de negocio (no sabe qué es un roadmap o un Spark).
- Generación de prompts dinámica (los carga de archivos versionados).
- Decisión de cuándo migrar de modelo (decisión humana, ejecución
  automatizada).

### 3.2 Authentication

**Hace:**
- Wrapper de Firebase Auth (Google, Apple, Email, Anonymous, Phone OTP).
- Sincronización Firebase UID ↔ Postgres `users` table.
- Validación de JWT en cada request autenticada.
- Account deletion con período de gracia 30 días.

**No hace:**
- Autorización / RBAC (no hay roles complejos en MVP, custom claims
  futuros).
- Anti-fraude de cuentas duplicadas (eso es `anti-fraud-system`).
- Account recovery con verificación humana (eso es `customer-support`).

### 3.3 Sparks

**Hace:**
- Mantener balance por usuario (plan + pack + bonus).
- Cobrar operaciones; refund si fallan.
- Asignación mensual del plan; expiración con rollover cap.
- Compra de packs (vía RevenueCat / Stripe webhook).
- Otorgar bonus por gamificación (consumiendo eventos).
- Audit log inmutable.

**No hace:**
- Decidir el costo en Sparks de cada operación dinámicamente (es
  configuración, vive en `sparks_operation_costs`).
- Pricing del plan al usuario (es configuración del producto).
- Detección de abuso de farming (eso es `anti-fraud-system`).

### 3.4 Notifications

**Hace:**
- Envío de push via FCM (Cloudflare Workers + cron).
- Detección de daily reminders timezone-aware.
- Detección de streak en peligro.
- Detección de inactividad y re-engagement.
- Rate limiting por usuario.
- Generación de contenido personalizado (vía AI Gateway, batch nocturno).

**No hace:**
- WhatsApp / SMS / Email transaccional masivo (post-MVP).
- Lógica de logros (los logros vienen de `motivation-and-achievements`).
- Decisión de bloquear notificaciones por fraude (eso es
  `anti-fraud-system` aplicando `user_restrictions`).

### 3.5 Anti-fraud

**Hace:**
- Device fingerprinting.
- Fraud scoring por usuario.
- Aplicación automática de restricciones niveles 1–3.
- Sistema de apelaciones.

**No hace:**
- Pagos (Stripe Radar maneja payment fraud).
- Suspensión de cuentas legítimas (siempre hay path de apelación).
- Compartir signals con terceros (decisión abierta, post-MVP).

### 3.6 AI Roadmap

**Hace:**
- Generación inicial del roadmap (one-shot al onboarding).
- Adaptación nocturna (job que decide qué usuarios necesitan re-análisis).
- Selección del próximo bloque para un usuario en una sesión.
- Generación de insights humanizados (matutino, semanal).
- Consume `getMasteryProfile` de pedagogical-system.

**No hace:**
- Calcular mastery (eso es `pedagogical-system`).
- Generar contenido de bloques (eso es `content-creation-system`).
- Otorgar logros (eso es `motivation-and-achievements`).

### 3.7 Pedagogical

**Hace:**
- Scoring de attempts (pronunciation, fluency, grammar, vocabulary,
  listening).
- Mastery por sub-skill, decay, spaced repetition.
- Estado de bloques (locked → mastered).
- Detección de struggling.
- Test-out voluntario.

**No hace:**
- Decidir qué bloque ofrecer (eso es `ai-roadmap`).
- Generar feedback humanizado al usuario (texto raw del scoring; el
  texto motivador lo agrega `notifications-system` o el cliente).

### 3.8 Student Profile

**Hace:**
- Onboarding inicial (preguntas + mini-test).
- Trial de 7 días con observación.
- Assessment del día 7 (20 min).
- Re-evaluaciones periódicas.
- Persistencia del perfil (demographics, goals, self-assessment,
  observed behavior, assessment results).

**No hace:**
- Generación del roadmap (eso es `ai-roadmap`).
- Scoring del assessment (eso es `pedagogical-system`).
- Conversión al pago (eso es `sparks-system` + UI).

### 3.9 Motivation & Achievements

**Hace:**
- Daily goals, streaks, freeze tokens.
- Engine de detección de logros (consume eventos de otros sistemas).
- Otorgamiento de bonus en Sparks (vía `sparks-system`).
- Visualizaciones de progreso, milestones celebrations.
- Leagues semanales (post-MVP), friend system (post-MVP).
- Detección de perfil motivacional.

**No hace:**
- Mastery scoring (consume desde `pedagogical-system`).
- Push notifications de logros (delega a `notifications-system`).

### 3.10 Content Creation

**Hace:**
- Pipeline de creación de assets asistida por IA + revisión humana.
- Tracking de performance por asset.
- Versionado y retirement de assets.
- Admin panel.

**No hace:**
- Selección del asset al usuario (eso es `ai-roadmap`).
- Scoring de performance del usuario en el asset (eso es
  `pedagogical-system`).

### 3.11 Customer Support

**Hace:**
- Help Center, status page.
- AI Assistant (vía AI Gateway).
- Sistema de tickets, escalación niveles 0–3.
- Categorización y reportes mensuales.

**No hace:**
- Account recovery sin proceso definido (sigue flow de auth).
- Refunds automáticos (revisión humana excepto auto-refund de policy
  estándar).

### 3.12 Metrics & Experimentation

**Hace:**
- Captura de eventos a PostHog, Firebase Analytics, Sentry.
- Cohort analysis.
- A/B testing con feature flags de PostHog.
- Dashboards y rituales de revisión.

**No hace:**
- Decisión de pricing o features basada en métricas (humano decide).
- Tracking de PII (sólo IDs hasheados).

### 3.13 Platform Strategy

No es un sistema de runtime. Es la **estrategia documentada** de qué
plataformas se construyen, cuándo y con qué stack. No tiene tablas ni
contratos.

### 3.14 Authentication-fraud-pricing-content-anti boundaries

Esta tabla resume las **fricciones más comunes** entre sistemas:

| Tensión | Quién decide | Por qué |
|---------|-------------|---------|
| Restringir features por fraud score | `anti-fraud` aplica via `user_restrictions`; otros sistemas leen y respetan | Centralizar política anti-abuso |
| Cobrar Sparks por una operación nueva | `sparks-system` configuración, decisión humana | Pricing es decisión de negocio |
| Ofrecer logro nuevo | `motivation` define en catálogo | Ownership claro de gamificación |
| Asset removido afecta usuarios in-progress | `content-creation` mantiene versión vieja, `pedagogical` usa snapshot | Snapshot inmutable evita inconsistencia |
| Usuario cambia objetivo | `student-profile` actualiza perfil → emite evento → `ai-roadmap` decide regenerar | Profile owner del fact, roadmap owner de la consecuencia |

---

## 4. Data ownership

Cada tabla Postgres tiene **un único sistema dueño**. Otros sistemas
solo pueden leer vía API/eventos, nunca acceso directo a la tabla.

### 4.1 Tablas de Authentication

- `users` (la tabla central, pero "owned" por auth para identity)

### 4.2 Tablas de Sparks

- `user_sparks_balance`
- `sparks_transactions`
- `sparks_packs_purchased`
- `billing_cycles`
- `sparks_operation_costs` (config)

### 4.3 Tablas de Pedagogical

- `subskills_catalog`
- `user_subskill_mastery`
- `exercise_attempts`
- `user_block_status`
- `exercise_dimension_weights` (config)
- `pronunciation_error_patterns` (config)
- `vocabulary_cefr_dictionary` (config)
- `weekly_mastery_snapshots`
- `test_out_attempts`
- `pedagogical_config`

### 4.4 Tablas de AI Roadmap

- `user_roadmaps`
- `user_block_progress` (deprecada, se consolida con `user_block_status`
  de pedagogical en migración)

### 4.5 Tablas de Student Profile

- `student_profiles`
- `user_profiles` (legacy, se consolida con `student_profiles`)

### 4.6 Tablas de Motivation

- `achievements_catalog`
- `user_achievements`
- `achievements_progress`
- `community_events`
- (post-MVP) `friendships`, `leagues`, `league_members`,
  `friend_challenges`

### 4.7 Tablas de Notifications

- `user_fcm_tokens`
- `user_notification_preferences`
- `notifications_log`
- `notifications_scheduled`

### 4.8 Tablas de Anti-fraud

- `device_fingerprints`
- `user_devices`
- `user_fraud_scores`
- `fraud_events`
- `user_restrictions`
- `fraud_appeals`

### 4.9 Tablas de Content Creation

- `media_atomics` (v1.2: pieza individual de media reutilizable)
- `characters` (v1.2: identidad consistent cross-asset)
- `atomic_generation_queue` (v1.2: queue para batch generation)
- `learning_assets` (v1.2: composites con refs a atomics)
- `learning_blocks` (autoritativo aquí)
- `asset_versions`
- `asset_feedback`

### 4.10 Tablas de Customer Support

- `support_tickets`
- `ticket_messages`

### 4.11 Tablas compartidas / sin dueño claro

Idealmente cero. Si algo no encaja, se discute en arch review.

### 4.12 Conflicto con `users`

`users` lo posee Authentication. Toda la información personal del usuario
vive aquí. Datos de aprendizaje y comportamiento van a `student_profiles`
(propiedad de Student Profile system) que referencia `users(id)`.

---

## 5. Contratos entre sistemas

### 5.1 APIs internas (RPC síncrono)

Sistemas exponen funciones llamables como funciones de TypeScript via
Cloudflare Workers + service bindings (o HTTP interno). Convención de
naming: `<verb><Noun>`.

| Owner | API | Llamada por |
|-------|-----|-------------|
| Pedagogical | `submitExerciseAttempt` | Cliente móvil/web |
| Pedagogical | `getMasteryProfile` | AI Roadmap, Motivation, UI |
| Pedagogical | `requestTestOut` | Cliente |
| Pedagogical | `getNextBlocksForUser` | AI Roadmap |
| Pedagogical | `recalculateMasteryGlobal` | Job nocturno |
| AI Gateway | `executeTask` | Todos los demás sistemas que necesitan IA |
| Sparks | `chargeOperation` | AI Gateway, Test-out |
| Sparks | `refundOperation` | AI Gateway |
| Sparks | `getBalance` | UI, Sparks UI |
| Sparks | `awardBonus` | Motivation, system credits |
| Sparks | `processPackPurchase` | RevenueCat/Stripe webhook handlers |
| Anti-fraud | `getFraudScore` | Auth, Sparks, Notifications |
| Anti-fraud | `applyRestriction` | Internal admin |
| Anti-fraud | `submitAppeal` | Cliente |
| AI Roadmap | `generateInitialRoadmap` | Onboarding flow |
| AI Roadmap | `getDailyPlan` | Cliente |
| AI Roadmap | `getRoadmapForUser` | UI |
| Student Profile | `getProfile` | Todos |
| Student Profile | `updateProfile` | Cliente, Onboarding |
| Student Profile | `submitAssessment` | Cliente (al completar día 7) |
| Motivation | `getAchievements` | UI |
| Motivation | `getDailyGoal` | Cliente |
| Notifications | `registerToken` | Cliente |
| Notifications | `updatePreferences` | Cliente |
| Customer Support | `createTicket` | AI Assistant, Cliente |
| Customer Support | `getMyTickets` | Cliente |

### 5.2 Eventos (notificación async)

Bus de eventos: Inngest events. Naming:
`<entity>.<action_past_tense>`. Detalle de cada evento en el documento
del sistema emisor.

#### Eventos clave (resumen)

| Evento | Emisor | Consumidores |
|--------|--------|--------------|
| `user.signed_up` | Auth | Student Profile (crea perfil), Anti-fraud (fingerprint) |
| `user.deleted` | Auth | Todos los sistemas con datos del usuario |
| `block.completed` | Pedagogical | AI Roadmap, Motivation |
| `block.mastered` | Pedagogical | AI Roadmap, Motivation |
| `block.struggling_detected` | Pedagogical | AI Roadmap, Notifications |
| `subskill.level_up` | Pedagogical | Motivation |
| `user.cefr_changed` | Pedagogical | Motivation, Notifications |
| `streak.extended` | Motivation | Notifications |
| `streak.broken` | Motivation | Notifications |
| `streak.at_risk` | Motivation | Notifications |
| `achievement.unlocked` | Motivation | Sparks (bonus), Notifications |
| `sparks.balance_low` | Sparks | Notifications |
| `sparks.depleted` | Sparks | Notifications |
| `payment.succeeded` | RevenueCat/Stripe webhook | Sparks (acreditar pack) |
| `payment.failed` | RevenueCat/Stripe webhook | Notifications |
| `assessment.completed` | Student Profile | AI Roadmap (genera definitivo) |
| `notification.opened` | Notifications | Metrics |
| `roadmap.generated` | AI Roadmap | Notifications (mensaje de bienvenida al plan) |
| `roadmap.updated` | AI Roadmap | Notifications |
| `restriction.applied` | Anti-fraud | Notifications, Sparks (bloquear features), Auth |
| `restriction.lifted` | Anti-fraud | Notifications, Sparks, Auth |

### 5.3 Convenciones de eventos

```typescript
interface DomainEvent<TPayload> {
  event_name: string;             // <entity>.<action_past_tense>
  event_id: string;               // UUID, único por instancia de evento
  emitted_at: string;             // ISO timestamp
  emitter_system: string;         // 'auth' | 'sparks' | ...
  user_id?: string;               // hasheado SHA-256
  payload: TPayload;
  trace_id?: string;              // para debugging cross-system
}
```

- **Idempotencia:** consumidores deben tolerar recibir el mismo
  `event_id` más de una vez.
- **Orden:** no garantizado entre eventos de sistemas distintos. Si
  el orden importa, usar timestamps.
- **Retries:** Inngest maneja retries automáticos. Si un consumidor
  falla, se reintenta hasta 3 veces con backoff exponencial.
- **Dead letter:** eventos que fallan 3 veces van a `event_log_dlq`
  para análisis manual.

---

## 6. User journeys end-to-end

### 6.1 Journey 1: Usuario nuevo, primera sesión

```
[Cliente] tap "Continuar con Google"
    │
    ▼
[Auth] Google Sign-In → idToken
    │
    ▼
[Auth] /auth/sync-user → upsert en `users`
    │ → emite `user.signed_up`
    │
    ├─→ [Student Profile] crea student_profile, trial_started_at = now
    ├─→ [Anti-fraud] genera fingerprint, calcula fraud_score inicial
    └─→ [Sparks] crea balance con 50 trial Sparks
    │
    ▼
[Student Profile] muestra onboarding (5 preguntas + mini-test 3 min)
    │
    ▼
[Pedagogical] vía AI Gateway: scoring del mini-test
    │ tasks: transcribe_user_audio, score_pronunciation, detect_grammar_errors
    │
    ▼
[Student Profile] persiste initial_test_results
    │ → emite `onboarding.completed`
    │
    ▼
[AI Roadmap] generateInitialRoadmap → roadmap inicial (4 semanas, 30 bloques)
    │ → emite `roadmap.generated`
    │
    ▼
[Cliente] muestra roadmap + primer bloque
    │
    ▼
Usuario hace primer ejercicio (pronunciation_drill)
    │
    ▼
[Cliente] uploadea audio a R2 → submitExerciseAttempt
    │
    ▼
[Pedagogical] valida payload
    [Sparks] chargeOperation(score_pronunciation, 0 sparks porque trial)
    [AI Gateway] executeTask("score_pronunciation", audio) → Azure
    │
    ▼
[Pedagogical] aplica delta a sub-skills, actualiza user_block_status
    │ → emite `exercise.attempt_completed`, `subskill.level_up` (si aplica)
    │
    ├─→ [Motivation] verifica logros (`first_steps`, `first_block`...)
    │       │ → emite `achievement.unlocked`
    │       │ → [Sparks] awardBonus(5)
    │       │ → [Notifications] schedule push de logro
    │
    ▼
[Cliente] muestra feedback del ejercicio + "logro desbloqueado"
```

### 6.2 Journey 2: Usuario activo, día 5 del trial

```
07:00 [Cron Cloudflare] checkDailyReminders por timezone
    │
    ▼
[Notifications] query usuarios con preferred_reminder_hour = 19 en su tz
                 que no practicaron hoy
    │
    ▼
Para cada usuario:
    [Notifications] lee `notifications_scheduled` con título/cuerpo pre-generados
    [Notifications] sendFcmNotification → APNs/FCM
    [Notifications] log en `notifications_log`

19:05 Usuario abre push, abre app
    │
    ▼
[Cliente] /home → getDailyPlan
    │
    ▼
[AI Roadmap] consulta UserRoadmap.active_block + bloques pending review
              consume getMasteryProfile (pedagogical) y next_review_at
    │
    ▼
[Cliente] muestra "Hoy: pronunciación de /θ/, 12 min"
    │
    ▼
Usuario hace 3 ejercicios (15 min)
    │
    ▼
[Pedagogical] cada submit → scoring → eventos
    │
    ▼
Bloque completa criterios mastered
    [Pedagogical] emite `block.mastered`
    │
    ├─→ [Motivation] otorga `first_level` si aplica → bonus 25 Sparks
    ├─→ [AI Roadmap] desbloquea siguiente nivel
    └─→ [Notifications] no schedulea nada (usuario activo)

23:00 [Cron] checkDailyGoalCompletion
    [Motivation] usuario cumplió daily goal
    [Motivation] otorga 5 Sparks bonus, extiende streak
    │ → emite `streak.extended`
    │
    ▼
[Notifications] schedulea push matutino del día siguiente vía AI Gateway batch
```

### 6.3 Journey 3: Día 7, assessment + conversión

```
Día 7, 08:00 [Notifications] envía push prominente "Hacé tu assessment"
    │
    ▼
Usuario abre app, ve banner del assessment
    │
    ▼
[Cliente] inicia assessment de 20 min (4 partes, 12 ejercicios)
    │
    ▼
Cada ejercicio:
    [Pedagogical] submitExerciseAttempt → scoring detallado
    [Sparks] cobra (incluido en trial Sparks)
    │
    ▼
Al completar:
    [Student Profile] persiste assessment_results
    [Student Profile] emite `assessment.completed`
    │
    ├─→ [AI Roadmap] generateRoadmapDefinitive (consume profile completo +
    │                  observed_behavior + assessment_results)
    │       │ vía AI Gateway tasks: generate_definitive_roadmap, generate_summary
    │       │
    │       └─→ persiste user_roadmaps, emite `roadmap.generated`
    │
    └─→ [Notifications] envía push "Tu plan está listo"
    │
    ▼
[Cliente] muestra resultados emocionales: B1+, fortalezas, oportunidades
    │
    ▼
[Cliente] muestra paywall con 3 planes
    │
    ▼
Usuario tap "Empezar con Pro"
    │
    ▼
[Cliente] inicia compra in-app vía RevenueCat
    │
    ▼
RevenueCat webhook → POST /webhooks/revenuecat
    │
    ▼
[Sparks] processPackPurchase → balance + 200 Sparks plan
    [Sparks] emite `payment.succeeded`
    │
    ├─→ [Auth] no hace nada (auth ya está)
    ├─→ [Notifications] envía push "Bienvenido a Pro"
    ├─→ [Motivation] verifica logros de payment (referido pago, etc.)
    └─→ [Metrics] track `user_subscribed`
```

### 6.4 Journey 4: Detección de fraude

```
Usuario nuevo se registra desde IP X
    │
    ▼
[Auth] crea user
    │ → emite `user.signed_up`
    │
    ▼
[Anti-fraud] genera fingerprint del device
    [Anti-fraud] consulta cuántos signups del fingerprint existen → 6
    [Anti-fraud] consulta cuántos signups de IP X últimas 24h → 12
    [Anti-fraud] calcula fraud_score = 30 + 50 + 25 = 105 → cap 100
    [Anti-fraud] persiste `user_fraud_scores`, severity 'critical'
    [Anti-fraud] aplica restricción: 'no_referrals', 'no_premium_features'
    [Anti-fraud] crea fraud_event
    [Anti-fraud] emite `restriction.applied`
    │
    ├─→ [Sparks] no acredita los 50 Sparks gratis del trial
    ├─→ [Notifications] envía email de notificación con link de apelación
    └─→ [Auth] marca user como 'restricted' (no impide login pero limita acciones)
    │
    ▼
Usuario apela vía /support/appeal
    │
    ▼
[Customer Support] crea fraud_appeal, status 'pending'
    │
    ▼
[Customer Support] humano revisa en 48h
    │
    ▼
Si legítimo:
    [Anti-fraud] removes restriction
    [Anti-fraud] emite `restriction.lifted`
    [Sparks] acredita los 50 trial Sparks
```

---

## 7. Failure modes y graceful degradation

### 7.1 AI Gateway falla (proveedor LLM caído)

| Sistema afectado | Comportamiento esperado |
|------------------|------------------------|
| Pedagogical (scoring) | Si Azure pronunciation falla: fallback a self-hosted (post-MVP) o devolver attempt como `pending_scoring`, score = null. UI muestra "Estamos procesando, te avisamos". |
| AI Roadmap (generación) | Si Claude Haiku falla en onboarding: fallback a Gemini Flash. Si ambos fallan: usar template del bucket más cercano. |
| AI Roadmap (job nocturno) | Si LLM batch falla: skip esa noche, retry día siguiente. Roadmap del usuario no se actualiza pero no se rompe. |
| Notifications (personalización) | Si LLM falla: enviar template genérico ("Hoy practicamos X, ¿vamos?"). |
| Customer Support (AI Assistant) | Si LLM falla: mostrar "Nuestro asistente está caído, podés contactarnos por email" con link directo. |

### 7.2 Postgres falla

Catastrófico — la mayoría del producto se cae. Mitigaciones:

- Supabase tiene HA y backups.
- Cliente cachea estado básico (balance Sparks, último roadmap) y muestra
  modo offline con audios pre-cacheados.
- Status page actualizada por cron desde monitoreo externo.

### 7.3 Firebase Auth falla

| Sistema | Comportamiento |
|---------|---------------|
| Auth | Login nuevo no funciona. Sesiones existentes siguen porque idToken cachea 1h. |
| Backend | Si JWT validation falla por JWKS unreachable: cachear JWKS por 24h max. |

### 7.4 FCM falla

| Sistema | Comportamiento |
|---------|---------------|
| Notifications | Push no se envía. Tokens marcados con `last_error_at`. Retry con backoff. UI no se afecta. |

### 7.5 RevenueCat / Stripe webhook delay

| Sistema | Comportamiento |
|---------|---------------|
| Sparks | Cliente no debe asumir compra completa hasta que el balance refleje el pack. Polling cada 5s post-compra hasta 60s. Si no llega webhook en 60s: mostrar "Procesando pago, te avisamos en minutos" y delegar a job de reconciliación nocturno que verifica direct con RevenueCat API. |

### 7.6 Inngest (event bus) caído

| Sistema | Comportamiento |
|---------|---------------|
| Todos | Eventos no se entregan. Sistemas críticos (Sparks, Auth) no dependen de eventos para correctness — usan APIs síncronas. Logros se detectan tarde pero se detectan eventualmente. |

### 7.7 Cliente offline

| Sistema | Comportamiento |
|---------|---------------|
| Cliente | Muestra preassets cacheados. Permite ejercicios sin scoring (queue de attempts pending upload). Al volver online, sync. |

---

## 8. Orden de implementación recomendado

### 8.1 Sprint 0 — Infra foundational (semana -1, 0)

Sin runtime aún. Setup de cuentas y proyectos:

1. Firebase project (Auth + FCM).
2. Supabase project (Postgres).
3. Cloudflare account (Workers, R2, KV).
4. PostHog project.
5. Sentry project.
6. RevenueCat account (puede ser placeholder).
7. Stripe account (puede ser placeholder).
8. Apple Developer Program ($99/año), Google Play Developer ($25).
9. Domain + SSL.

### 8.2 Sprint 1 — Backbone (semanas 1–2)

Implementar el **mínimo** que el resto necesita:

1. **Auth básico**: Firebase Auth + Google Sign-In + endpoint
   `/auth/sync-user` + tabla `users` + JWT validation utility.
2. **Schema base**: migración inicial con tablas de Auth + base de
   `student_profiles`.
3. **AI Gateway esqueleto**: LiteLLM + Task Registry inicial con 1 tarea
   (`transcribe_user_audio`).
4. **Sparks foundational**: tablas + `chargeOperation`/`refundOperation`
   + balance UI básico.
5. **Cliente esqueleto**: React Native + Expo dev build + auth flow +
   navegación.

Sin esto, nada más se puede implementar.

### 8.3 Sprint 2 — Onboarding + roadmap inicial (semanas 3–4)

1. **Student Profile**: onboarding (5 preguntas) + mini-test (3 min).
2. **Pedagogical mínimo**: scoring de pronunciation drill via Azure +
   schemas core + `submitExerciseAttempt`.
3. **AI Roadmap**: `generateInitialRoadmap` con biblioteca semilla
   (50 bloques de Job Ready).
4. **Content Creation**: pipeline para crear los 50 primeros bloques.
5. **UI**: pantalla de roadmap + primer ejercicio.

### 8.4 Sprint 3 — Sesiones diarias y notificaciones (semanas 5–6)

1. **Pedagogical completo**: spaced repetition, decay, struggling
   detection.
2. **Notifications**: FCM + daily reminder + cron timezone-aware.
3. **Sparks completo**: rollover, expiración, packs (sin compra real
   aún), webhooks placeholder.
4. **Motivation MVP**: daily goals + streaks + 30 logros core.

### 8.5 Sprint 4 — Assessment día 7 + paywall (semanas 7–8)

1. **Student Profile**: assessment de 20 min completo.
2. **AI Roadmap**: `generateDefinitiveRoadmap`.
3. **Sparks**: integración real con RevenueCat + Stripe (web).
4. **UI**: paywall, plan selection, payment flow.

### 8.6 Sprint 5 — Anti-fraud + soporte (semanas 9–10)

1. **Anti-fraud**: device fingerprinting + scoring + restricciones
   automáticas L1–L3 + apelaciones.
2. **Customer Support**: Help Center + AI Assistant básico + sistema de
   tickets en Notion.
3. **Metrics**: tracking de eventos clave + dashboard en PostHog.

### 8.7 Sprint 6 — Polish + soft launch (semanas 11–12)

1. Bug fixes y polish.
2. Tests de integración para edge cases críticos.
3. Calibración inicial del Pedagogical (50–100 voluntarios).
4. **Soft launch en México**: 4–6 semanas de validación.

### 8.8 Post-soft launch (semanas 13+)

- Hard launch regional.
- Motivation profundizado: leagues, friend system.
- AI Gateway maduro: shadow mode, canary, golden datasets.
- Modelo propio de pronunciation.
- Expansión de biblioteca a 1.500+ assets.

---

## 9. Glosario de términos

| Término | Definición |
|---------|-----------|
| **Asset** | Pieza individual de contenido (drill, roleplay, listening). Vive en biblioteca. |
| **Block** | Secuencia de 3–5 assets que abordan una sub-skill. Unidad de progreso del roadmap. |
| **Sub-skill** | Habilidad concreta evaluable (ej: `pron_th_voiceless`). |
| **Dimension** | Una de las 5: pronunciation, fluency, grammar, vocabulary, listening. |
| **Mastery** | Score 0–100 de dominio de una sub-skill. Niveles: unfamiliar/learning/developing/proficient/mastered. |
| **CEFR** | Common European Framework: A1, A2, B1, B1+, B2, B2+, C1, C2. |
| **Track** | Conjunto de bloques agrupados por objetivo (Job Ready, Travel, Daily). |
| **Roadmap** | Plan personalizado del usuario: niveles → bloques. |
| **Sparks** | Tokens de consumo. Cuestan al usuario (suscripción/pack), se gastan en operaciones premium. |
| **Streak** | Días consecutivos cumpliendo el daily goal. |
| **Daily Goal** | Objetivo diario personalizado (ej: 12 min con foco en /θ/). |
| **WPU** | Weekly Practicing Users — North Star Metric. |
| **Trial** | 7 días gratis con 50 Sparks; termina con assessment al día 7. |
| **Assessment** | Test de 20 min al día 7 que produce el roadmap definitivo. |
| **Test-out** | Mini-test voluntario (5 ejercicios) para saltar una sub-skill o bloque. |
| **AI Gateway** | Capa de abstracción sobre proveedores LLM. Único path para IA. |
| **Task** | Operación registrada en el AI Gateway con schema fijo. |
| **Preassets** | Assets pre-generados (no requieren IA en runtime). Acceso ilimitado en plan. |
| **Pack** | Compra puntual de Sparks (no subscription). |
| **Plan** | Suscripción mensual: Free, Básico, Pro, Premium. |
| **NSM** | North Star Metric. |
| **ADR** | Architecture Decision Record. Documento de decisión arquitectónica importante. |

---

## 10. Referencias internas

Documentos de cada sistema mencionado en este overview. El detalle vive
allí, no acá.

- Architecture: [`ai-gateway-strategy.md`](ai-gateway-strategy.md),
  [`authentication-system.md`](authentication-system.md),
  [`sparks-system.md`](sparks-system.md),
  [`notifications-system.md`](notifications-system.md),
  [`anti-fraud-system.md`](anti-fraud-system.md),
  [`platform-strategy.md`](platform-strategy.md).
- Product: [`../product/ai-roadmap-system.md`](../product/ai-roadmap-system.md),
  [`../product/pedagogical-system.md`](../product/pedagogical-system.md),
  [`../product/student-profile-and-assessment.md`](../product/student-profile-and-assessment.md),
  [`../product/motivation-and-achievements.md`](../product/motivation-and-achievements.md),
  [`../product/content-creation-system.md`](../product/content-creation-system.md).
- Business: [`../business/customer-support-system.md`](../business/customer-support-system.md),
  [`../business/metrics-and-experimentation.md`](../business/metrics-and-experimentation.md).

---

*Documento vivo. Actualizar cuando se agreguen sistemas nuevos, cambien
contratos públicos, o se descubran failure modes no contemplados.*

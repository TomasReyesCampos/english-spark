# Metrics and Experimentation Framework

> Sistema para medir lo que importa, identificar qué funciona y qué no,
> y experimentar de forma rigurosa para mejorar el producto. Diseñado
> para dev solo que necesita decisiones data-driven sin abrumarse con
> dashboards.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 establece **filosofía** y NSM.
- §2 cubre **boundaries**.
- §3 cubre **métricas por categoría**.
- §4 cubre **eventos a trackear** (alineado con
  `cross-cutting/data-and-events.md` §6).
- §5 cubre **stack tecnológico**.
- §6 cubre **framework de experimentación**.
- §7 cubre **A/B testing con PostHog**.
- §8 cubre **rituals** (daily/weekly/monthly/quarterly).
- §9 cubre **cohortes y segmentación**.
- §10 cubre **API contracts**.
- §11 enumera **edge cases**.
- §12 cubre **privacy**.
- §13 cubre **decisiones cerradas**.

---

## 1. Filosofía

### 1.1 Principios

**Pocas métricas críticas, claras y accionables.**

**One North Star metric:** una métrica que captura el éxito del
producto.

**Métricas por categoría:** acquisition, activation, retention, revenue,
referral. Pocas en cada una.

**Cada métrica con threshold y acción:** "si esta métrica cae a X,
hacemos Y".

**Experimentar antes de decidir:** cambios mayores van por A/B test.

### 1.2 NSM: Weekly Practicing Users (WPU)

**Definición:** users que completaron al menos 3 sesiones de práctica
durante la última semana.

#### Por qué WPU es la métrica correcta

- **No es DAU/MAU genérico:** "abrir la app" no es éxito. Practicar es
  éxito.
- **Captura habit formation:** 3 sesiones/semana indica hábito real.
- **Independiente del tamaño:** mide intensidad, no solo crecimiento.
- **Conecta con revenue:** users que practican son users que pagan.
- **Conecta con valor entregado:** user que practica está aprendiendo.

#### Sub-métricas

WPU se descompone en:
- **Acquisition:** nuevos users registrándose.
- **Activation:** % que completan primera sesión.
- **Retention:** % que vuelven D1, D7, D30.
- **Engagement:** sesiones por user/semana.
- **Resurrection:** users inactivos que vuelven.

Cualquier movimiento de WPU se explica por movimientos en estas.

---

## 2. Boundaries

### 2.1 Es responsable de

- Definir taxonomía de events (catálogo en §4).
- Configurar SDKs en cliente y backend (PostHog, Firebase, Sentry).
- Crear y operar dashboards.
- Framework de A/B testing.
- Reportes mensuales y rituales.
- Cohort analysis.

### 2.2 NO es responsable de

- **Emitir events específicos:** los systems dueños del dominio los
  emiten (auth emite `user.signed_up`, sparks emite `sparks.balance_changed`,
  etc.).
- **Decidir features basadas en data:** humano decide.
- **Aplicar restrictions o changes basadas en métricas:** otros systems
  consumen events y deciden.

### 2.3 Tensiones

| Tensión | Resolución |
|---------|-----------|
| Event con PII queremos para analysis | Hashear `user_id`; no enviar PII directa |
| Deduplicación entre cliente y backend events | Convención: cliente emite cuando user lo causa; backend cuando es server-side |
| Mismo concepto en domain event y product event | Separados: domain events para reaction de sistemas; product events para analytics |

---

## 3. Métricas por categoría

### 3.1 Acquisition

**Qué mide:** ¿cuántos users nuevos llegan?

| Métrica | Definición | Target |
|---------|-----------|--------|
| New signups/day | Registros completados | Crecimiento mes a mes |
| CAC | Costo de adquisición/user pago | < $10 USD |
| Source attribution | De dónde vienen | Diversificado |
| Conversion (visitor → signup) | % de visitas que se registran | > 5% |

**Fuentes de data:**
- Firebase Analytics (signups).
- PostHog (funnel completo).
- UTM tracking en marketing.

**Experimentación típica:**
- Mensajes de landing.
- CTAs.
- Social proof.
- Pricing display.

### 3.2 Activation

**Qué mide:** ¿llegan al "aha moment"?

| Métrica | Definición | Target |
|---------|-----------|--------|
| Onboarding completion | % completa onboarding | > 85% |
| First session completion | % completa primera sesión | > 70% |
| Day 1 return | % vuelve al día siguiente | > 50% |
| Mini-test completion (D0) | % hace mini-test | > 80% |
| Assessment completion (D7) | % hace assessment | > 30% |
| Trial → Paid conversion | % paga | > 10% |

**Aha moment:** completar primer ejercicio con feedback positivo. User
piensa "wow, esto funciona".

### 3.3 Retention

**Qué mide:** ¿siguen viniendo?

| Métrica | Definición | Target |
|---------|-----------|--------|
| D1 retention | % activo D1 post-signup | > 50% |
| D7 retention | % activo D7 | > 30% |
| D30 retention | % activo D30 | > 20% |
| D90 retention | % activo D90 | > 15% |
| Churn rate (paid) | % cancela mensualmente | < 10% |
| Streak average | Streak promedio activos | > 5 días |
| Sessions per week | Sesiones/user activo | > 3 |

**Retention curve:** caída fuerte primeras 2 semanas, estabilización
después. "Smile curve" si users habituados se mantienen.

### 3.4 Revenue

| Métrica | Definición | Target |
|---------|-----------|--------|
| MRR | Monthly Recurring Revenue | Crecimiento mes a mes |
| ARPU | Avg Revenue/User pago | > $3 USD/mes |
| LTV | Lifetime Value | > $30 USD |
| LTV/CAC ratio | | > 3 |
| Plan distribution | % en cada plan | Pro > 50% |
| Pack purchases/user | Sparks packs | n/a |
| Conversion (free → paid) | | > 10% |

### 3.5 Referral

| Métrica | Definición | Target |
|---------|-----------|--------|
| Referral rate | % users que refieren al menos 1 | > 15% |
| Avg referrals/referrer | | > 2 |
| Referred user activation | % de referidos que activan | > 70% (mejor que normal) |
| K-factor | (% referrers) × (avg referrals) | > 0.3 |
| Share rate | % que comparte logros | > 5% |

---

## 4. Eventos clave a trackear

### 4.1 Distinción crítica: domain vs product events

(Detalle completo en `cross-cutting/data-and-events.md` §2.)

- **Domain events (Inngest):** comunicación cross-system. Sistemas
  reaccionan.
- **Product events (PostHog):** analytics, funnels, A/B testing.
- **Internal logs (Sentry):** errors, debugging.

### 4.2 Lista de product events para PostHog

Catálogo mínimo del MVP:

```typescript
// Lifecycle
'user_signed_up'              { method, country, ... }
'user_completed_onboarding'   { duration_seconds }
'user_completed_mini_test'    { detected_cefr }
'user_started_trial'          {}
'user_completed_assessment'   { duration_minutes, scores }
'user_subscribed'             { plan, currency, amount, country }
'user_cancelled'              { reason, days_active, plan }

// Engagement
'session_started'             { source: 'notification'|'organic' }
'session_completed'           { duration_seconds, exercises_count }
'exercise_completed'          { type, score, mastery_change }
'block_completed'             { block_id, mastery_score, attempts }
'level_completed'             { level_id, time_to_complete_days }
'daily_goal_met'              { streak_days }
'streak_at_risk'              { streak_days }
'streak_broken'               { previous_streak }

// Achievement & social
'achievement_unlocked'        { achievement_id, rarity }
'sparks_earned'               { amount, source }
'sparks_spent'                { amount, on_what }
'pack_purchased'              { pack_type, amount }
'referral_sent'               { method }
'referral_signed_up'          { referrer_hash }
'logro_shared'                { achievement_id, platform }

// Monetization
'paywall_viewed'              { context, plan_shown }
'plan_selected'               { plan }
'payment_attempted'           { method, amount }
'payment_succeeded'           { method, amount }
'payment_failed'              { reason }

// Support
'help_article_viewed'         { article_id }
'ai_assistant_opened'         { context }
'ticket_created'              { category, priority }

// Notifications
'notification_received'       { notification_id, category }
'notification_opened'         { notification_id, category }
'notification_dismissed'      { notification_id, category }
```

### 4.3 Convenciones

- **Naming:** snake_case, verbo en pasado.
- **`distinct_id`:** hash del `users.id` (SHA-256 con pepper). Nunca
  email u otra PII.
- **Properties:** datos no-PII para análisis. Country sí, IP no.
- **Numerics:** datos numéricos para análisis (no string).
- **Cliente preferentemente:** backend solo si event no es visible al
  cliente (ej: payment webhook).

### 4.4 Anti-patterns

- ❌ Trackear cada UI click sin contexto (`button_clicked`).
- ❌ Eventos genéricos sin properties útiles.
- ❌ PII directa en properties.
- ❌ Eventos duplicados en cliente + backend.

---

## 5. Stack tecnológico

### 5.1 Para tracking

| Componente | Tecnología | Para qué |
|-----------|-----------|---------|
| Product analytics | PostHog | Eventos del producto, funnels, retention, A/B testing, feature flags |
| Mobile analytics | Firebase Analytics | Eventos mobile, attribution |
| Error tracking | Sentry | Crashes, errors |
| Logs | Cloudflare Logs + Better Stack | Logs de Workers |
| Performance | Sentry Performance + Vercel Analytics | API y app performance |
| Revenue | RevenueCat dashboard + Stripe | MRR, churn, etc. |

### 5.2 Para visualización

**MVP:** dashboards nativos de cada herramienta.
**Crecimiento:** Metabase o Grafana para vistas customizadas.
**Escala:** considerar herramienta dedicada (Looker, etc.).

### 5.3 Setup recomendado

**PostHog como hub central:**
- Eventos del producto.
- Funnels custom.
- Retention cohorts.
- A/B testing.
- Feature flags.
- Tier gratuito hasta 1M eventos/mes.

**Firebase Analytics como complemento:**
- Eventos mobile específicos.
- Attribution de adquisición.
- Push notification analytics.

**Sentry para todo lo de errores:**
- Crashes mobile.
- Errors de backend.
- Performance.

Tres servicios. Suficientes.

---

## 6. Framework de experimentación

### 6.1 Cuándo experimentar

**SÍ:**
- Cambios en flujos críticos (onboarding, paywall, pricing).
- Nuevas features con incertidumbre.
- Cambios en algoritmos (recommendations, scoring).
- Mensajes y copy en momentos clave.

**NO:**
- Bug fixes (deploy directo).
- Mejoras obvias de UX.
- Compliance changes.
- Cambios de infrastructure no visibles.

### 6.2 Anatomía de un experimento

```typescript
interface Experiment {
  id: string;
  name: string;
  hypothesis: string;              // "Si hacemos X, esperamos que Y suba/baje"

  variants: {
    control: VariantConfig;
    treatment: VariantConfig;
    // Posibles más variantes
  };

  // Métricas
  primary_metric: string;
  guardrail_metrics: string[];     // métricas que NO deben empeorar
  secondary_metrics: string[];

  // Configuración
  target_audience: AudienceFilter;
  traffic_allocation: number;      // % de users incluidos
  variant_split: Record<string, number>; // % por variant

  // Statistical
  expected_effect_size: number;
  required_sample_size: number;
  expected_duration_days: number;

  // Lifecycle
  status: 'draft' | 'running' | 'completed' | 'rolled_back';
  started_at?: string;
  ended_at?: string;
  results?: ExperimentResults;
}
```

### 6.3 Estadística básica

**Sample size:**
- Mínimo 1.000 users por variant para detectar efectos del 10%+.
- Más users = detectar efectos más pequeños.
- Calculadora online o función built-in de PostHog.

**Significancia estadística:**
- p-value < 0.05 (95% confidence).
- NO detener prematuramente "porque ya tengo significancia".
- Esperar duración planificada salvo casos extremos.

**Effect size:**
- Detectar efectos pequeños requiere mucho más sample.
- Si esperás 1% de mejora: mucho tráfico.
- Si esperás 20%: sample manejable.

**Power:**
- 80% power es estándar.
- Incluir en cálculo de sample.

### 6.4 Tipos de experimentos

| Tipo | Descripción |
|------|-------------|
| A/B Test simple | 1 variable, 2 versiones. Lo más común. |
| Multivariate | Múltiples variables simultáneamente. Requiere más tráfico. |
| Holdout | Un % de users siempre ve "viejo" para baseline a largo plazo. |
| Pre-post | Solo cuando A/B real no posible (cambios que afectan a todos). Menos riguroso. |

### 6.5 Experimentos prioritarios primeros 6 meses

**Trimestre 1:**
1. Onboarding length (5 vs 7 vs 3 preguntas).
2. Trial duration (7 vs 14 días).
3. Free trial Sparks (50 vs 30 vs 100).
4. Assessment timing (Day 5 vs 7 vs 10).

**Trimestre 2:**
1. Paywall positioning (post-assessment vs primera Sparks insuficiente).
2. Plan recommendations (qué plan highlightear).
3. Pricing structure (3 tiers vs 2).
4. Referral rewards (Sparks vs free month).

---

## 7. A/B testing con PostHog

### 7.1 Setup

PostHog tiene feature flags y experiments built-in.

```typescript
// Cliente
import { posthog } from '@posthog/react-native';

const variant = posthog.getFeatureFlag('onboarding-length-test');

if (variant === 'short') {
  showOnboarding3Questions();
} else if (variant === 'long') {
  showOnboarding7Questions();
} else {
  showOnboardingDefault5Questions();
}

// Track conversion
posthog.capture('onboarding_completed', {
  variant,
  duration_seconds: duration,
});
```

PostHog automáticamente analiza resultados con statistical tests.

### 7.2 Reglas de uso

- **Asignación al variant:** `distinct_id` debe ser estable
  (`hash(user_id)`); user siempre ve misma variant.
- **Fallback:** si flag no disponible (sin red), usar default
  conservador.
- **No mid-experiment changes:** si modificás el experimento, archive y
  crear nuevo.
- **Pre-registration:** antes de iniciar, documentar hipótesis y
  guardrails. Evita post-hoc rationalization.

### 7.3 Análisis post-experimento

Antes de declarar ganador:
- p-value < 0.05.
- Sample mínimo alcanzado.
- Guardrails no violados.
- Duration mínima respetada.
- Sanity checks (no bug en setup).

Si guardrail violado: rollback aunque primary mejore.

---

## 8. Rituales

### 8.1 Daily check (5 min)

Cada mañana:
- WPU del día anterior.
- Crashes y errors nuevos.
- Tickets de soporte abiertos.
- Cualquier anomalía evidente.

Tiempo total: 5 min. No es análisis profundo, solo "todo bien?".

### 8.2 Weekly review (30-60 min)

Cada lunes (o domingo noche):
- Tendencia semanal de core metrics.
- Cohort retention de signups recientes.
- Performance de experimentos activos.
- Top categorías de tickets.
- Decisiones para la semana.

### 8.3 Monthly deep dive (2-3 horas)

Mensualmente:
- Análisis completo de funnel.
- Retention cohorts comparados (signups este mes vs anterior).
- Revisión de NSM y sub-metrics.
- Review de costos vs revenue.
- Roadmap de experimentos del mes siguiente.

### 8.4 Quarterly strategic review (1 día)

Trimestralmente:
- ¿Está funcionando la estrategia general?
- ¿Hay que pivotar algo?
- ¿Las decisiones de hace 3 meses fueron correctas?
- Plan estratégico siguiente trimestre.

---

## 9. Cohortes y segmentación

### 9.1 Cohortes a trackear

**Por fecha de signup:**
Cohorts mensuales para retention curves comparativas.

**Por plan:**
Free vs Básico vs Pro vs Premium - comportamiento muy distinto.

**Por país:**
México vs Argentina vs Colombia - métricas pueden divergir.

**Por canal de adquisición:**
Orgánico vs paid vs referral - LTV varía mucho.

**Por nivel CEFR inicial:**
B1 vs B2 - patrones de uso diferentes.

### 9.2 Análisis de cohortes

Las cohortes revelan cosas que las métricas agregadas esconden:

- "Retention general bajó" puede esconder "retention de marzo bajó pero
  abril mejoró".
- "Conversion subió" puede ser solo una cohorte específica que infló
  el promedio.

Mirar cohorts mensualmente, no solo agregados.

---

## 10. API contracts

### 10.1 Cliente: SDK de PostHog

```typescript
// Inicialización en app startup
posthog.init(POSTHOG_API_KEY, {
  host: 'https://app.posthog.com',
  bootstrap: {
    distinctID: hash(userId),
    isIdentifiedID: true,
  },
});

// Track event
posthog.capture('event_name', { property1: 'value', property2: 42 });

// Identify (para vincular pre-signup events)
posthog.identify(hash(userId), { country, plan });

// Feature flag
const variant = posthog.getFeatureFlag('experiment-name');
```

### 10.2 Backend: emit events

Para events que no son visibles al cliente (ej: payment webhook):

```typescript
// apps/workers/src/utils/posthog.ts
export async function trackBackendEvent(
  event: string,
  userId: string | null,
  properties: Record<string, unknown> = {},
) {
  await fetch('https://app.posthog.com/capture/', {
    method: 'POST',
    body: JSON.stringify({
      api_key: POSTHOG_API_KEY,
      event,
      distinct_id: userId ? hash(userId) : 'anonymous',
      properties,
      timestamp: new Date().toISOString(),
    }),
  });
}
```

### 10.3 Internal: `getNsm`

```typescript
async function getNsm(period: 'day' | 'week' | 'month'): Promise<{
  current: number;
  previous: number;
  change_pct: number;
}>
```

Query simple a PostHog API o cálculo desde Postgres directo.

### 10.4 Internal: `getFunnelData`

```typescript
async function getFunnelData(funnelName: string, period: string): Promise<{
  steps: Array<{
    name: string;
    users: number;
    conversion_pct: number;
  }>;
}>
```

### 10.5 Internal: `createExperiment` (admin)

PostHog UI maneja la creación; este endpoint solo para auditoría
interna:

```typescript
async function createExperiment(config: ExperimentConfig): Promise<{
  experiment_id: string;
  posthog_id: string;
}>
```

### 10.6 Internal: `endExperiment`

```typescript
async function endExperiment(
  experimentId: string,
  decision: 'ship_treatment' | 'ship_control' | 'inconclusive',
  notes: string,
): Promise<void>
```

---

## 11. Edge cases (tests obligatorios)

### 11.1 Tracking

1. **User offline durante 24h con events queued:** PostHog SDK
   buffer-flushing automático cuando recupera conexión. Validar.
2. **User cambia device:** mismo `distinct_id` (hash de user_id);
   eventos siguen vinculándose correctamente.
3. **User borra cuenta:** propagar delete a PostHog vía
   `posthog.api/persons/delete`.

### 11.2 Eventos

4. **Evento con property que es PII (bug):** validation pre-emit
   detecta keys prohibidas (`email`, `phone`); rechaza con log.
5. **Evento duplicado en <1s:** PostHog dedupe por
   `distinct_id + event + timestamp`.
6. **Cliente y backend emiten mismo evento:** convención decide quién
   (cliente generalmente). Si ambos: investigar bug.

### 11.3 A/B testing

7. **User en variant A pero feature flag service caído:** SDK retorna
   default; user ve control. Variant assignment puede inconsistente
   este request.
8. **Experiment con guardrail violado pero primary mejorado:**
   rollback automático; humano decide si revivir con cambios.
9. **Sample size no alcanzado pero stakeholder quiere "ya":** rechazar.
   Documentar política.

### 11.4 Cohorts

10. **Cohorte de signups en marzo migra de plan:** retention puede
    medirse por cohorte original (signup) o cohorte actual (plan).
    Documentar cuál se usa.

---

## 12. Privacy y compliance

### 12.1 Datos que tracking

**Permitido (con consent en privacy policy):**
- Eventos de comportamiento.
- Properties anónimas (country, plan, level).
- Performance metrics.
- Crashes y errors.

**Requiere consent explícito:**
- Datos personales (email, nombre).
- Audios capturados.
- Ubicación precisa.

**Nunca:**
- Información de pago.
- Datos sensibles (salud, religión, política).

### 12.2 PII handling

- PostHog y Firebase Analytics aceptan UUIDs hasheados.
- No mandar PII directamente.
- Para análisis cualitativo con PII: tools separadas con permisos
  restringidos.

### 12.3 Right to be forgotten

Account deletion debe propagar a:
- Postgres principal.
- PostHog (`api/persons/delete`).
- Firebase Analytics.
- Sentry (purge user events).
- Cualquier otra tool.

Idealmente automatizado en flow de account deletion.

(Detalle en `architecture/authentication-system.md` §13.2.)

---

## 13. Decisiones cerradas

### 13.1 NSM = WPU (3 sesiones/semana) ✓

(Detalle en §1.2.)

**Razón:** captura habit formation real, no vanity. Conecta con revenue
y retention.

### 13.2 Tracking de costos por user: **agregado, no individual** ✓

**Razón:** trade-off entre precisión y overhead. Agregar `ai_cost_per_user_daily`
es suficiente para alertas; tracking individual de cada operación
inflaría el storage sin justification.

### 13.3 BI tool más sofisticado vs PostHog: **PostHog hasta 50k users
activos** ✓

**Razón:** PostHog cubre product analytics + A/B testing + funnels +
cohorts. Migración a Looker/Metabase solo si necesidades específicas
no cubiertas o si volumen excede tier de PostHog.

### 13.4 Hire de data analyst: **post-100 users pagos** ✓

**Razón:** antes de eso, cantidad de data no justifica. Owner puede
manejar análisis con rituals semanales.

### 13.5 Hacer públicas algunas métricas (radical transparency): **NO en
MVP** ✓

**Razón:** complejidad operativa + riesgo de competidores leverage.
Reconsiderar año 2 si build community fuerte.

---

## 14. Anti-patterns a evitar

### 14.1 Vanity metrics

Métricas que se ven bien pero no aportan:
- Total downloads (no si no usan).
- Total signups (no si no activan).
- Total page views (no si no convierten).
- Followers en redes sociales.

Si una métrica nunca cambia tu decisión, es vanity.

### 14.2 Cherry picking

Mostrar solo las métricas que confirman lo que querías. Honestidad
contigo mismo es esencial.

### 14.3 Correlation = causation

Que A y B subieran juntos no significa A causó B. Solo experimentos
rigurosos establecen causalidad.

### 14.4 Goodhart's Law

"Cuando una métrica se vuelve target, deja de ser buena métrica."

Si optimizás obsesivamente por una métrica, los users la van a
manipular o vos la vas a hackear inconscientemente.

Mitigación: guardrail metrics + revisión cualitativa.

---

## 15. Plan de implementación

### 15.1 Fase 0: Pre-launch

**Crítico:**
- PostHog setup con eventos core.
- Firebase Analytics integrado en mobile.
- Sentry configurado.
- Definición clara de NSM y core metrics.

### 15.2 Fase 1: MVP (meses 0-3)

**Crítico:**
- Tracking de todos los eventos definidos en §4.
- Daily check ritual establecido.
- Weekly review process.

**Importante:**
- Primer dashboard interno.
- Setup de feature flags para A/B testing.

### 15.3 Fase 2: Iteración (meses 3-9)

**Crítico:**
- A/B testing activo en flows críticos.
- Cohort analysis mensual.
- Reportes de performance de experimentos.

**Importante:**
- Migración a herramienta de visualización si necesaria.
- Métricas de calidad de IA.

### 15.4 Fase 3: Madurez (año 1+)

- Predictive analytics (probabilidad de churn por user).
- Real-time dashboards para incidents.
- Métricas de IA propias (calidad de modelos).
- Possibly hire de data analyst part-time.

---

## 16. Métricas operacionales

### 16.1 Performance técnico

| Métrica | Definición | Target |
|---------|-----------|--------|
| API p95 latency | Latencia p95 endpoints | < 300ms |
| App crash rate | Crashes por sesión | < 0.1% |
| API error rate | % requests con error 5xx | < 0.5% |
| Time to interactive | App responsive desde apertura | < 2s |
| AI response time | Tiempo de respuesta IA en chat | < 3s |

### 16.2 Costos

| Métrica | Definición | Target |
|---------|-----------|--------|
| Cost per user/mes | Total infra / users | < $0.50 free, < $2 paid |
| AI cost per user | Costos IA / users | < 30% del ARPU |
| Cost per Spark consumed | Variable / Sparks usados | < $0.005 |

### 16.3 Calidad de IA

| Métrica | Definición | Target |
|---------|-----------|--------|
| AI validation pass rate | % outputs que pasan validation | > 95% |
| Pronunciation accuracy | vs ground truth | > 85% |
| Roadmap quality (rating) | Rating de roadmaps | > 4/5 |
| AI Assistant resolution | % tickets resueltos sin escalar | > 60% |

---

## 17. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) | Definición de domain events vs product events. |
| [`plan_de_negocio.docx`](plan_de_negocio.docx) | Targets de negocio que las métricas validan. |
| [`../architecture/sparks-system.md`](../architecture/sparks-system.md) §15 | Métricas de Sparks. |
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) §13 | Métricas de notifications. |
| [`../product/motivation-and-achievements.md`](../product/motivation-and-achievements.md) §16 | Métricas de engagement. |
| Cada documento de sistema tiene sección §X de Métricas con targets propios. |

---

## 18. Lecturas recomendadas

- "Lean Analytics" de Croll & Yoskovitz.
- "Hooked" de Nir Eyal (engagement metrics).
- "Trustworthy Online Controlled Experiments" (estadística rigurosa).
- Andrew Chen's blog (growth metrics).
- First Round Review: data-driven product.

---

*Documento vivo. Actualizar cuando cambien NSM, se añadan métricas, o
se aprenda de experimentos pasados.*

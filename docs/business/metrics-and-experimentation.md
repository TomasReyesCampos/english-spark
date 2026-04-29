# Metrics and Experimentation Framework

> Sistema para medir lo que importa, identificar qué funciona y qué no,
> y experimentar de forma rigurosa para mejorar el producto continuamente.
> Diseñado para un dev solo que necesita tomar decisiones basadas en datos
> sin abrumarse con dashboards.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Alcance:** Sistema completo

---

## 1. Filosofía

### 1.1 El problema sin métricas

Sin un framework claro de métricas:
- Decisiones se basan en intuición o última conversación con un usuario.
- No sabés si una feature nueva ayudó o empeoró.
- Marketing y desarrollo van en direcciones opuestas.
- Recursos se gastan en cosas que no mueven la aguja.

### 1.2 El problema con demasiadas métricas

El opuesto también es malo:
- 100 métricas tracked, 0 acciones tomadas.
- Análisis paralysis.
- Dashboards bonitos que nadie mira.
- Confusión sobre qué es importante.

### 1.3 Filosofía adoptada

**Pocas métricas críticas, claras y accionables.**

**One North Star metric:** una métrica que captura el éxito del producto.

**Métricas por categoría:** acquisition, activation, retention,
revenue, referral. Pocas en cada una.

**Cada métrica con threshold y acción:** "si esta métrica cae a X,
hacemos Y".

**Experimentar antes de decidir:** cambios mayores van por A/B test
cuando es posible.

---

## 2. North Star Metric

### 2.1 Definición

La North Star Metric (NSM) captura el éxito fundamental del producto.
Cuando esta métrica sube, todo lo demás tiende a mejorar.

**NSM propuesta: Weekly Practicing Users (WPU)**

Definición: usuarios que completaron al menos 3 sesiones de práctica
durante la última semana.

### 2.2 Por qué WPU es la métrica correcta

**No es DAU/MAU genérico:** "abrir la app" no es éxito. Practicar es éxito.

**Captura habit formation:** 3 sesiones por semana indica hábito real.

**Independiente del tamaño del producto:** mide intensidad de uso, no
solo crecimiento bruto.

**Conecta con revenue:** usuarios que practican son usuarios que pagan
(directamente o eventualmente).

**Conecta con valor entregado:** un usuario que practica está aprendiendo,
estamos cumpliendo nuestra promesa.

### 2.3 Sub-métricas que componen WPU

WPU se descompone en:

- **Acquisition:** nuevos usuarios registrándose.
- **Activation:** % de nuevos usuarios que completan primera sesión.
- **Retention:** % que vuelven en días 1, 3, 7, 30.
- **Engagement:** sesiones por usuario por semana.
- **Resurrection:** usuarios inactivos que vuelven.

Cualquier movimiento de WPU se explica por movimientos en estas sub-métricas.

---

## 3. Métricas por categoría

### 3.1 Acquisition (adquisición)

**Qué mide:** ¿cuántos usuarios nuevos llegan al producto?

| Métrica | Definición | Target |
|---------|-----------|--------|
| New signups/day | Registros completados por día | Crecimiento mes a mes |
| CAC | Costo de adquisición por usuario pago | <$10 USD |
| Source attribution | De dónde vienen los usuarios | Diversificado |
| Conversion rate (visitor → signup) | % de visitas que se registran | >5% |

**Fuentes de datos:**
- Firebase Analytics (signups).
- PostHog (funnel completo).
- Plataformas de marketing (UTM tracking).

**Experimentación típica:**
- Mensajes de la landing page.
- Calls to action.
- Social proof.
- Pricing display.

### 3.2 Activation (activación)

**Qué mide:** ¿los usuarios nuevos llegan al "aha moment"?

| Métrica | Definición | Target |
|---------|-----------|--------|
| Onboarding completion rate | % que completa onboarding | >85% |
| First session completion | % que completa primera sesión | >70% |
| Day 1 return | % que vuelve al día siguiente | >50% |
| Mini-test completion (D0) | % que hace mini-test inicial | >80% |
| Assessment completion (D7) | % que hace assessment al día 7 | >30% |
| Trial → Paid conversion | % de trial users que pagan | >10% |

**Aha moment definido:** completar el primer ejercicio con feedback
positivo. El usuario debe pensar "wow, esto funciona".

**Experimentación típica:**
- Estructura del onboarding.
- Primer ejercicio que se le presenta.
- Mensajes durante trial.
- Comunicación pre-assessment.

### 3.3 Retention (retención)

**Qué mide:** ¿los usuarios siguen viniendo?

| Métrica | Definición | Target |
|---------|-----------|--------|
| D1 retention | % activo día después de signup | >50% |
| D7 retention | % activo semana después de signup | >30% |
| D30 retention | % activo mes después de signup | >20% |
| D90 retention | % activo 3 meses después | >15% |
| Churn rate (paid) | % de paid que cancela mensualmente | <10% |
| Streak average | Streak promedio de usuarios activos | >5 días |
| Sessions per week | Sesiones promedio por usuario activo | >3 |

**Retention curve típica para apps de educación:** caída fuerte primeras
2 semanas, estabilización después. Apps exitosas tienen "smile curve":
curva sube después del valle inicial cuando los usuarios habituados se
mantienen.

**Experimentación típica:**
- Frequency y contenido de notificaciones.
- Mecánicas de gamificación.
- Mensajes de re-engagement.
- Daily goal difficulty.

### 3.4 Revenue (monetización)

**Qué mide:** ¿el negocio es sostenible?

| Métrica | Definición | Target |
|---------|-----------|--------|
| MRR | Monthly Recurring Revenue | Crecimiento mes a mes |
| ARR | Annual Recurring Revenue | MRR × 12 |
| ARPU | Avg Revenue Per User (paid) | >$3 USD/mes |
| LTV | Lifetime Value | >$30 USD |
| LTV/CAC ratio | Ratio | >3 |
| Plan distribution | % en cada plan | Pro >50% |
| Pack purchases per user | Sparks packs comprados promedio | n/a |
| Conversion (free → paid) | % que pasa a pagar | >10% |

**Experimentación típica:**
- Pricing.
- Estructura de planes.
- Trial duration.
- Upsell timing.

### 3.5 Referral (viralidad)

**Qué mide:** ¿los usuarios traen más usuarios?

| Métrica | Definición | Target |
|---------|-----------|--------|
| Referral rate | % de usuarios que refieren al menos 1 | >15% |
| Avg referrals per referrer | Cuántos amigos refieren | >2 |
| Referred user activation | % de referidos que activan | >70% (mejor que normal) |
| K-factor | (% referrers) × (avg referrals) | >0.3 |
| Share rate | % que comparte logros externos | >5% |

**Experimentación típica:**
- Recompensas por referido.
- Momento del prompt de referido.
- Mensajes pre-armados para compartir.

---

## 4. Métricas operacionales

### 4.1 Performance técnico

| Métrica | Definición | Target |
|---------|-----------|--------|
| API p95 latency | Latencia p95 endpoints | <300ms |
| App crash rate | Crashes por sesión | <0.1% |
| API error rate | % de requests con error 5xx | <0.5% |
| Time to interactive | App responsive desde apertura | <2s |
| AI response time | Tiempo de respuesta IA en chat | <3s |

### 4.2 Costos

| Métrica | Definición | Target |
|---------|-----------|--------|
| Cost per user per month | Total infra / users | <$0.50 free, <$2 paid |
| AI cost per user | Costos de IA / users | <30% del ARPU |
| Cost per Spark consumed | Variable cost / Sparks usados | <$0.005 |

### 4.3 Calidad de IA

| Métrica | Definición | Target |
|---------|-----------|--------|
| AI validation pass rate | % de outputs que pasan validación | >95% |
| Pronunciation scoring accuracy | vs ground truth | >85% |
| Roadmap quality (user rating) | Rating de roadmaps | >4/5 |
| AI Assistant resolution rate | % de tickets resueltos sin escalar | >60% |

---

## 5. Stack tecnológico

### 5.1 Para tracking

| Componente | Tecnología | Para qué |
|-----------|-----------|---------|
| Product analytics | PostHog | Eventos del producto, funnels, retention |
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

Tres servicios, no más. Suficientes para tomar decisiones.

---

## 6. Eventos clave a trackear

### 6.1 Eventos de usuario

```typescript
// Lifecycle
'user_signed_up'         { method: 'google'|'apple'|'email', country, ... }
'user_completed_onboarding'  { duration_seconds, all_questions_answered }
'user_completed_mini_test'   { detected_cefr }
'user_started_trial'     { day }
'user_completed_assessment'  { duration, scores }
'user_subscribed'        { plan, currency, amount }
'user_cancelled'         { reason, days_active }

// Engagement
'session_started'        { source: 'notification'|'organic' }
'session_completed'      { duration, exercises_count }
'exercise_completed'     { type, score, mastery_change }
'block_completed'        { block_id, attempts, mastery_score }
'level_completed'        { level_id, time_to_complete }
'daily_goal_met'         { streak_days }
'streak_at_risk'         { streak_days }
'streak_broken'          { previous_streak }

// Achievement & social
'achievement_unlocked'   { achievement_id, rarity }
'sparks_earned'          { amount, source }
'sparks_spent'           { amount, on_what }
'pack_purchased'         { pack_type, amount }
'referral_sent'          { method }
'referral_signed_up'     { referrer_id }
'logro_shared'           { achievement_id, platform }

// Monetization
'paywall_viewed'         { context, plan_shown }
'plan_selected'          { plan }
'payment_attempted'      { method, amount }
'payment_succeeded'      { method, amount }
'payment_failed'         { reason }

// Support
'help_article_viewed'    { article_id }
'ai_assistant_opened'    { context }
'ticket_created'         { category, priority }
```

### 6.2 Principios de naming

- Verbo en pasado: `user_signed_up` no `user_signs_up`.
- Snake case consistente.
- Sujeto + acción: `user_X`, `session_X`, `block_X`.
- Properties con datos relevantes para análisis.

### 6.3 Anti-patterns a evitar

- ❌ Eventos genéricos como `button_clicked` (sin contexto = sin valor).
- ❌ Trackear cada UI interaction (ruido).
- ❌ PII en properties (email, nombre completo, etc.).
- ❌ Eventos duplicados (mismo evento en cliente y backend).

---

## 7. Framework de experimentación

### 7.1 Cuándo experimentar

**SÍ experimentar:**
- Cambios en flujos críticos (onboarding, paywall, pricing).
- Nuevas features con incertidumbre.
- Cambios en algoritmos (recommendations, scoring).
- Mensajes y copy en momentos clave.

**NO experimentar:**
- Bug fixes (deploy directo).
- Mejoras obvias de UX (botón más visible, etc.).
- Compliance changes.
- Cambios de infrastructure no visibles al usuario.

### 7.2 Anatomía de un experimento

```typescript
interface Experiment {
  id: string;
  name: string;
  hypothesis: string;          // "Si hacemos X, esperamos que Y suba/baje"

  variants: {
    control: VariantConfig;
    treatment: VariantConfig;
    // Posibles más variantes
  };

  // Métricas
  primary_metric: string;      // métrica principal a optimizar
  guardrail_metrics: string[]; // métricas que NO deben empeorar
  secondary_metrics: string[]; // métricas adicionales a observar

  // Configuración
  target_audience: AudienceFilter;
  traffic_allocation: number;  // % de usuarios incluidos
  variant_split: Record<string, number>; // % por variante

  // Statistical
  expected_effect_size: number;
  required_sample_size: number;
  expected_duration_days: number;

  // Lifecycle
  status: 'draft' | 'running' | 'completed' | 'rolled_back';
  started_at?: Date;
  ended_at?: Date;
  results?: ExperimentResults;
}
```

### 7.3 Estadística básica para dev solo

**Sample size:**
- Mínimo 1.000 usuarios por variante para detectar efectos del 10%+.
- Más usuarios = detectar efectos más pequeños.
- Calculadora: usar online (e.g., Optimizely's calculator) o función
  built-in de PostHog.

**Significancia estadística:**
- p-value < 0.05 es el estándar (95% confidence).
- No detener el experimento prematuramente "porque ya tengo significancia".
- Esperar duración planificada salvo casos extremos.

**Effect size:**
- Detectar efectos pequeños requiere mucho más sample.
- Si esperás 1% de mejora, vas a necesitar mucho tráfico.
- Si esperás 20% de mejora, sample size es manejable.

**Power:**
- 80% power es estándar (probabilidad de detectar el efecto si existe).
- Incluir en cálculo de sample size.

### 7.4 Tipos de experimentos comunes

#### A/B Test simple
Una variable, dos versiones. Lo más común.

Ejemplo: "Botón verde vs azul en paywall"

#### Multivariate
Múltiples variables simultáneamente. Requiere más tráfico.

Ejemplo: combinaciones de copy + color + posición.

#### Holdout
Un % de usuarios siempre ve "viejo" para baseline a largo plazo.

Útil para medir efectos acumulados de muchos cambios.

#### Pre-post (sin control simultáneo)
Solo cuando A/B real no es posible (cambios que afectan a todos).

Menos riguroso, sujeto a confounders.

### 7.5 Experimentos prioritarios para los primeros 6 meses

**Trimestre 1:**
1. Onboarding length (5 preguntas vs 7 vs 3).
2. Trial duration (7 días vs 14 días).
3. Free trial Sparks (50 vs 30 vs 100).
4. Assessment timing (día 5 vs 7 vs 10).

**Trimestre 2:**
1. Paywall positioning (post-assessment vs primera Sparks insuficiente).
2. Plan recommendations (which plan to highlight).
3. Pricing structure (3 tiers vs 2).
4. Referral rewards (Sparks vs free month).

### 7.6 Implementación en PostHog

PostHog tiene feature flags y experiments built-in.

```typescript
// Cliente
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
  duration_seconds: duration
});
```

PostHog automáticamente analiza resultados con statistical tests.

---

## 8. Reportes y rituales

### 8.1 Daily check (5 min)

Cada mañana, mirar:
- WPU del día anterior.
- Crashes y errores nuevos.
- Tickets de soporte abiertos.
- Cualquier anomalía evidente.

Tiempo total: 5 minutos. No es análisis profundo, solo "todo bien?".

### 8.2 Weekly review (30-60 min)

Cada lunes (o domingo noche):
- Tendencia semanal de métricas core.
- Cohort retention de signups recientes.
- Performance de experimentos activos.
- Top categorías de tickets.
- Decisiones para la semana.

### 8.3 Monthly deep dive (2-3 horas)

Mensualmente:
- Análisis completo de funnel.
- Retention cohorts comparados (signups este mes vs mes anterior).
- Revisión de NSM y sub-métricas.
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
- "Conversion subió" puede ser solo una cohorte específica que infló el promedio.

Mirar cohorts mensualmente, no solo agregados.

---

## 10. Privacidad y compliance

### 10.1 Datos que tracking

**Permitido (con consent en privacy policy):**
- Eventos de comportamiento.
- Properties anónimas (país, plan, level).
- Performance metrics.
- Crashes y errores.

**Requiere consent explícito:**
- Datos personales (email, nombre).
- Audios capturados.
- Ubicación precisa.

**Nunca:**
- Información de pago (manejado por Stripe/RevenueCat).
- Datos sensibles (salud, religión, política).

### 10.2 PII handling

PostHog y Firebase Analytics aceptan user IDs (hasheados o UUIDs). No
mandar PII directamente.

Para análisis cualitativo que requiere PII (revisión de tickets):
herramientas separadas con permisos restringidos.

### 10.3 Right to be forgotten

Account deletion debe propagar a:
- Postgres principal.
- PostHog (API de delete user).
- Firebase Analytics.
- Sentry (purge user events).
- Cualquier otra tool.

Idealmente automatizado en el flow de account deletion.

---

## 11. Plan de implementación

### 11.1 Fase 0: Pre-launch

**Crítico:**
- PostHog setup con eventos core.
- Firebase Analytics integrado en mobile.
- Sentry configurado.
- Definición clara de NSM y métricas core.

### 11.2 Fase 1: MVP (meses 0-3)

**Crítico:**
- Tracking de todos los eventos definidos.
- Daily check ritual establecido.
- Weekly review process.

**Importante:**
- Primer dashboard interno.
- Setup de feature flags para A/B testing.

### 11.3 Fase 2: Iteración (meses 3-9)

**Crítico:**
- A/B testing activo en flows críticos.
- Cohort analysis mensual.
- Reportes de performance de experimentos.

**Importante:**
- Migración a herramienta de visualización si necesaria.
- Métricas de calidad de IA.

### 11.4 Fase 3: Madurez (año 1+)

- Predictive analytics (probabilidad de churn por usuario).
- Real-time dashboards para incidents.
- Métricas de IA propias (calidad de modelos).
- Possibly hire de data analyst part-time.

---

## 12. Anti-patrones a evitar

### 12.1 Vanity metrics

Métricas que se ven bien pero no aportan:
- Total downloads (no si no usan).
- Total signups (no si no activan).
- Total page views (no si no convierten).
- Followers en redes sociales.

Si una métrica nunca cambia tu decisión, es vanity.

### 12.2 Cherry picking

Mostrar solo las métricas que confirman lo que querías. Honestidad
contigo mismo es esencial.

### 12.3 Correlation = causation

Que A y B hayan subido juntos no significa que A causó B.
Solo experimentos rigurosos establecen causalidad.

### 12.4 Goodhart's Law

"Cuando una métrica se vuelve target, deja de ser buena métrica."

Si optimizás obsesivamente por una métrica, los usuarios la van a
manipular o vos la vas a hackear inconscientemente.

Mitigación: guardrail metrics + revisión cualitativa.

---

## 13. Decisiones abiertas

- [ ] ¿NSM debería ser WPU o algo más específico (ej: Weekly Speakers,
  usuarios que tuvieron al menos 1 conversación con IA)?
- [ ] ¿Qué tan obsesivamente trackear costos por usuario? Trade-off entre
  precisión y overhead.
- [ ] ¿Cuándo introducir herramienta de BI más sofisticada vs PostHog?
- [ ] ¿Hire de data analyst antes o después de primer 100 usuarios pagos?
- [ ] ¿Hacer públicas algunas métricas (radical transparency) para construir
  trust con usuarios?

---

## 14. Referencias internas

- `docs/business/plan_de_negocio.docx` — Targets de negocio que las
  métricas validan.
- `docs/architecture/sparks-system.md` — Métricas de Sparks system.
- `docs/architecture/notifications-system.md` — Métricas de notificaciones.
- `docs/product/motivation-and-achievements.md` — Métricas de engagement.

---

## 15. Lecturas recomendadas

- "Lean Analytics" de Croll & Yoskovitz.
- "Hooked" de Nir Eyal (engagement metrics).
- "Trustworthy Online Controlled Experiments" (estadística rigurosa).
- Andrew Chen's blog (growth metrics).
- First Round Review: data-driven product.

---

*Documento vivo. Actualizar cuando cambien NSM, se añadan métricas, o
se aprenda de experimentos pasados.*

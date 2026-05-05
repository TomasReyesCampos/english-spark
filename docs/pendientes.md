# Pendientes y Decisiones Abiertas

> Inventario consolidado de **decisiones abiertas**, **trabajos
> pendientes** y **documentos a crear** identificados en cada sistema.
> Cada ítem indica el documento de origen para profundizar.

**Última actualización:** 2026-05
**Cómo usar:** revisar al menos al inicio de cada sprint. Marcar como
resuelto al tomar decisión y mover el detalle al documento correspondiente.

---

## 0. Cambios recientes (ronda 2026-05)

Resumen de lo cerrado en la ronda más reciente (referenciar §1-§5
para detalle):

**Decisiones cerradas:**
- Locale único MVP: mexicano-tuteo (CLAUDE.md §10, reglas.md, i18n.md).
- Assessment timing: 3 niveles (locked / optional / soft / firm) según
  día del trial (§1.1).
- CEFR sampling para assessment: hybrid self-perceived + measured
  (§1.1).
- Assessment broken detection + recovery prompt (§1.1).
- Roadmap inicial: solo 2 niveles + preview borrosa (§1.1).
- PDF assessment: post-MVP (§1.1).
- Versionado de atomics con `_v<n>` (§1.5).
- 6 characters en seed inicial + 12 voces base (§1.5).
- Proveedores generación: ElevenLabs / DALL-E 3 / HeyGen / Freesound
  CC0 (§1.5).
- 5 decisiones de notifications (snooze, smart timing, grupales, rich,
  A/B) cerradas como "no MVP" (§2.4).

**Specs nuevos creados (~10.000 líneas totales):**
- `product/curriculum-by-cefr.md` — plan A1→C2.
- `product/assessment-content-bank.md` — pool items Day 7.
- `product/post-assessment-flow.md` — 8 pantallas reveal.
- `product/onboarding-flow.md` — 16 pantallas Day 0.
- `product/push-notifications-copy-bank.md` — copys para 15 notif_ids +
  80 logros.
- `product/track-job-ready-blocks.md` — 74 bloques B1→B2+.
- `product/track-travel-confident-blocks.md` — 39 bloques B1→B2+.
- `product/track-daily-conversation-blocks.md` — 62 bloques B1→B2+.
- `product/atomics-catalog-seed.md` — 350 atomics MVP.
- `explorations/cultural-annotations.md` — exploration tooltip.
- `explorations/multimedia-assets.md` — promovido parcialmente a v1.2.
- `explorations/sporadic-questions.md` — promovido a spec v1.2.

**Total MVP definido a nivel de spec:**
- 175 bloques pedagógicos con asset_sequence y mastery criteria.
- ~350 media_atomics con generation pipelines.
- ~700 composites estimados (reuse 2x).
- 80 logros con copy literal.
- 15 notification_ids con banco completo de variantes.
- 16 pantallas onboarding + 8 post-assessment con copy literal.

**Próximas prioridades (sin decisiones bloqueantes):**
1. Sprint 0 setup foundational (§5.2).
2. Sprint atomics seed generación (§5.4).
3. Sprint creación de bloques Daily Conversation primero
   (cross-track reuse).

---

## 1. Decisiones de producto

### 1.1 Onboarding y assessment
*(de `product/student-profile-and-assessment.md`)*

- [x] **¿Permitir hacer el assessment antes del día 7?** ✓ CERRADO 2026-05.
  3 niveles según día: Day 1-2 locked, Day 3-4 optional, Day 5-6 obligatorio
  suave (modal con postpone), Day 7+ obligatorio firme (premium gated).
  Excepción transversal: Sparks runout desbloquea aún en Day 1-2. Detalle
  en `student-profile-and-assessment.md` §13.1.
- [x] **¿Re-aplicar assessment cuando cambia objetivo?** ✓ CERRADO 2026-05.
  Sí, si `significant_change = true` (cambio mayor de primary_goals,
  deadline_date >50%, target_english_variant). Prompt al user. Detalle en
  §13.2.
- [x] **¿CEFR para sampling del assessment?** ✓ CERRADO 2026-05. Hybrid
  self-perceived + measured: si match (~50% casos) usar; si difieren 1
  nivel preguntar al user; si difieren 2+ tomar el higher con disclaimer.
  Detalle en §13.2 y `assessment-content-bank.md`.
- [x] **¿Cómo manejar assessments rotos?** ✓ CERRADO 2026-05. Detect +
  recovery prompt: "Notamos que el assessment fue muy rápido, ¿quieres
  rehacerlo?". Si confirma: re-iniciar sin cobrar. Si dice "está bien
  así": persistir con `low_confidence = true`. Detalle en §6.6.
- [x] **¿Mostrar roadmap inicial completo o partial?** ✓ CERRADO. Solo
  primeros 2 niveles + preview borrosa. Mantiene intriga sobre el
  assessment, reduce overwhelm Day 0. Detalle en §13.4.
- [x] **¿PDF descargable del resultado del assessment?** ✓ CERRADO.
  Post-MVP. Reconsiderar si >10% de usuarios lo solicitan via support.
  Detalle en §13.5.

**Trabajo derivado pendiente:**
- [ ] Implementar el flow de hybrid CEFR sampling (UI con choice screen
  cuando difieren 1 nivel).
- [ ] Implementar postpone counter en `student_profiles.assessment_postpone_count`.

### 1.2 Roadmap personalizado
*(de `product/ai-roadmap-system.md`)*

- [ ] ¿Permitir que el usuario edite manualmente su roadmap (skip, reorder)?
  Pros: agency. Contras: rompe coherencia pedagógica.
- [x] **¿Tracks múltiples activos simultáneamente o uno solo?** ✓ CERRADO
  2026-05 por **ADR-007**: un solo track activo a la vez en MVP. Switch
  permitido vía Settings (rate limit 14 días). Sub-skill mastery transfiere
  globalmente; block completion es por-track. Multi-track simultáneo se
  reconsidera año 2 con datos.
- [x] **¿Cómo manejar usuarios que cambian de objetivo mid-roadmap?**
  ✓ CERRADO. Regenerar con confirmación si `significant_change = true`.
  Detalle en `ai-roadmap-system.md` §14.3 y ADR-007.
- [x] **¿Certificados al completar tracks?** ✓ CERRADO. Al completar
  100% Y mastery promedio > 70%. Detalle en `ai-roadmap-system.md`
  §14.4.
- [x] **Política de re-evaluación: mini-test cada N semanas?** ✓ CERRADO.
  Pro/Premium: cada 4 semanas. Básico: cada 8 semanas. Detalle en
  `ai-roadmap-system.md` §14.5 y `student-profile-and-assessment.md` §8.1.

### 1.3 Sistema pedagógico
*(de `product/pedagogical-system.md`)*

- [ ] ¿Cuántas sub-skills inicialmente? (Recomendación: 50 core, expandir).
- [ ] ¿Permitir "test out" voluntario de una sub-skill con mini-test?
- [ ] ¿Mostrar al usuario sus scores en cada dimensión o solo CEFR global?
  (Recomendación: mostrar con explicación).
- [ ] ¿Migrar de Azure Pronunciation Assessment a modelo propio? ¿Cuándo?
  (Recomendación: cuando volumen lo justifique).
- [ ] ¿Cómo manejar dialectos del inglés (British vs American) en scoring?

### 1.4 Motivación y logros
*(de `product/motivation-and-achievements.md`)*

- [ ] ¿Sparks o XP separado para gamificación? (Recomendación: solo Sparks
  para no fragmentar economía).
- [ ] ¿Logros borrables si el usuario hace cheating obvio?
- [ ] ¿Mostrar % de usuarios que tienen cada logro? Pros: hace raridad
  concreta. Contras: puede desmotivar.
- [ ] ¿Permitir que usuarios "donen" Sparks a amigos? Mecánica social
  poderosa pero riesgo de abuso.
- [ ] ¿Sistema de "rivales" amistosos (siempre comparar con un usuario
  específico de nivel similar)? Inspirado en Strava.

### 1.5 Creación de contenido
*(de `product/content-creation-system.md` y `atomics-catalog-seed.md`)*

- [ ] ¿Crowdsource de assets de la comunidad? Pros: escala. Contras: calidad
  variable, moderación.
- [ ] ¿Licenciar contenido existente de proveedores educativos? Cuidado con
  derechos.
- [ ] ¿Generar assets en tiempo real para usuarios con perfiles únicos?
  Posible pero costoso.
- [x] **¿Versionado de atomics?** ✓ CERRADO 2026-05. Semver mayor con
  sufijo `_v<n>` en el `id` del atomic. Permite evolución sin romper
  composites existentes. Detalle en `atomics-catalog-seed.md` §2.
- [x] **¿Cuántos characters en seed inicial?** ✓ CERRADO 2026-05. 6
  characters (alex, grandma, sarah, mike, jamie, emma) con ~30 atomics
  c/u. Más diluye memoria episódica del user. Detalle en §4 del seed.
- [x] **¿Proveedores primarios de generación?** ✓ CERRADO 2026-05.
  ElevenLabs (TTS), DALL-E 3 (imágenes), HeyGen (video talking-head),
  Freesound CC0 (ambient), FFmpeg (distortion overlays). Detalle en §9
  del seed.
- [x] **¿12 voces base en voice pool?** ✓ CERRADO 2026-05. 12 voces
  ElevenLabs cubriendo edades 22-70, M/F, accents US-General/Midwest/
  Northeast. Post-MVP: 4-6 UK/AU. Detalle en §3 del seed.

---

## 2. Decisiones de arquitectura

### 2.1 AI Gateway
*(de `architecture/ai-gateway-strategy.md`)*

- [ ] ¿Construir Gateway propio o usar Portkey/OpenRouter como Gateway managed?
- [ ] ¿Promptfoo o Braintrust para evaluación? (Recomendación: probar ambos).
- [ ] ¿Cómo versionar prompts? Archivos en repo o servicio externo (PromptLayer).
- [ ] Política de retención de logs de llamadas a LLMs (privacy + costo storage).
- [ ] ¿Qué tareas justifican mantener 5% activo en secundario vs solo fallback?

### 2.2 Autenticación
*(de `architecture/authentication-system.md`)*

- [ ] **Phone OTP en MVP o esperar a datos para decidir?** (Recomendación:
  esperar — Opción A en el documento).
- [ ] ¿Permitir "magic link" como método adicional de email login (sin
  password)?
- [ ] ¿MFA para usuarios con > 500 Sparks comprados o > 6 meses activos?
- [ ] ¿Custom claims de Firebase para roles (free vs premium vs admin)?
- [ ] ¿Soporte para login con WhatsApp en lugar de SMS estándar?

### 2.3 Sparks system
*(de `architecture/sparks-system.md`)*

- [ ] ¿Permitir transferencia de Sparks entre usuarios (gifting)? Pros:
  viralidad. Contras: complejidad anti-abuso.
- [ ] ¿Sparks negativos (deuda) para usuarios premium con buen historial?
  (Recomendación: no, agrega complejidad).
- [ ] ¿Programa de fidelidad con multiplicador de Sparks por antigüedad?
- [ ] ¿Sparks comunitarios (eventos, sorteos)? Marketing tool.
- [ ] ¿Sparks comprables como gift card para terceros? Útil en navidad.

### 2.4 Notificaciones
*(de `architecture/notifications-system.md` y `push-notifications-copy-bank.md`)*

- [x] **¿Permitir snooze de notificaciones por 1 hora?** ✓ CERRADO.
  No en MVP. User puede silenciar a nivel SO. Detalle en
  `notifications-system.md` §15.5.
- [x] **¿Smart timing aprender hora real de uso?** ✓ CERRADO.
  Post-MVP. MVP usa hora preferida declarada. §15.6 notifications-system.
- [x] **¿Notificaciones grupales (múltiples eventos en uno)?** ✓ CERRADO.
  No en MVP. Suprimimos extras y user ve todo en su perfil. §15.7.
- [x] **¿Rich notifications con imágenes/audio embebido?** ✓ CERRADO.
  No en MVP. Texto simple alcanza. §15.8.
- [x] **¿A/B test con Firebase Remote Config?** ✓ CERRADO. Post-MVP.
  PostHog feature flags suplen para empezar. §15.9.
- [x] **Banco autoritativo de copys de notificaciones** ✓ CERRADO 2026-05.
  Creado `push-notifications-copy-bank.md` con copy literal mexicano-tuteo
  para los 15 notification_ids + los 80 logros, validation rules, A/B
  tests priorizados, schema Postgres `notifications_copy_bank`.
- [ ] **Cuándo agregar WhatsApp:** validar producto, retention D30 > 30%,
  > 5.000 usuarios, push open rate < 10%.

### 2.5 Anti-fraude
*(de `architecture/anti-fraud-system.md`)*

- [ ] ¿Compartir signals de fraude con otras apps (alianzas anti-fraude)?
- [ ] ¿Pre-screening agresivo en países con altas tasas de fraude vs
  experiencia neutral para todos?
- [ ] ¿Soft launch de features anti-fraude (shadow mode antes de
  enforcement)?
- [ ] ¿Investigar identity verification para usuarios premium (KYC light)?
- [ ] ¿Programa "verified user" con badge y privilegios para usuarios
  comprobadamente legítimos?

### 2.6 Plataformas
*(de `architecture/platform-strategy.md`)*

- [ ] ¿Lanzar en países adicionales fuera de Latam? (España, comunidad
  hispana en EEUU).
- [ ] ¿Versión específica para tablets con UI optimizada o solo responsive?
- [ ] **¿PWA primero o web app independiente cuando llegue la fase 3?**
  (Recomendación: empezar con PWA).
- [ ] ¿Estrategia de pricing diferenciada por país (ajuste por poder
  adquisitivo)?
- [ ] ¿Incluir Apple Watch / Wear OS? (Probablemente no en MVP).

---

## 3. Decisiones de negocio y operación

### 3.1 Customer support
*(de `business/customer-support-system.md`)*

- [ ] ¿Soporte 24/7 con AI o limitado a horario laboral con escalación
  humana? (Recomendación: AI 24/7, humano horario hábil).
- [ ] ¿Foro o Discord comunitario? ¿Cuándo introducirlo?
- [ ] ¿Programa de "power users" que ayudan a otros con incentivos
  (Sparks)?
- [ ] ¿Soporte por WhatsApp? (posible pero requiere setup específico).
- [ ] ¿Política de refunds más generosa para usuarios long-term?

### 3.2 Métricas y experimentación
*(de `business/metrics-and-experimentation.md`)*

- [ ] ¿NSM debería ser WPU o algo más específico (ej: Weekly Speakers,
  usuarios con al menos 1 conversación con IA)?
- [ ] ¿Qué tan obsesivamente trackear costos por usuario? Trade-off entre
  precisión y overhead.
- [ ] ¿Cuándo introducir herramienta de BI más sofisticada vs PostHog?
- [ ] ¿Hire de data analyst antes o después de primeros 100 usuarios pagos?
- [ ] ¿Hacer públicas algunas métricas (radical transparency) para
  construir trust con usuarios?

---

## 4. Documentos a crear (referenciados pero no escritos)

ADRs y otros documentos que están referenciados desde los sistemas
existentes pero aún no se han creado:

- [x] `docs/decisions/ADR-001-ai-gateway.md` — Decisión arquitectónica del
  Gateway. ✓ creado.
- [x] `docs/decisions/ADR-002-platform-strategy.md` — ADR formal de mobile-first.
  ✓ creado.
- [x] `docs/decisions/ADR-003-sparks-system.md` — ADR formal de Sparks.
  ✓ creado.
- [x] `docs/decisions/ADR-004-notifications-fcm.md` — ADR de elección de FCM.
  ✓ creado.
- [x] `docs/decisions/ADR-005-firebase-auth.md` — ADR de Firebase Auth.
  ✓ creado.
- [x] `docs/decisions/ADR-006-trial-assessment.md` — ADR del trial de 7 días.
  ✓ creado.
- [ ] `docs/decisions/ADR-XXX-llm-selection.md` — Decisión sobre proveedores
  LLM por tarea.
- [ ] `docs/decisions/sparks-pricing-changes.md` — Log histórico de ajustes
  de costo en Sparks (vacío hasta primer ajuste).
- [x] `docs/architecture/01-overview.md` — Arquitectura general. ✓ creado.
- [x] `docs/product/learning-blocks-taxonomy.md` — Taxonomía de la
  biblioteca de bloques. **CUBIERTO 2026-05** por los 3 docs de
  desglose por track:
  - `track-job-ready-blocks.md` (74 bloques B1→B2+).
  - `track-travel-confident-blocks.md` (39 bloques B1→B2+).
  - `track-daily-conversation-blocks.md` (62 bloques B1→B2+).
  - **Total MVP: 175 bloques** con sub-skills, prerequisites,
    estimated_minutes, asset_sequence, mastery criteria por bloque.
- [x] `docs/product/curriculum-by-cefr.md` — Plan A1→C2 con vocab,
  grammar, pronunciation, sub-skills, mastery criteria. ✓ creado
  2026-05.
- [x] `docs/product/assessment-content-bank.md` — Pool de items para
  Day 7 assessment con sampling logic. ✓ creado 2026-05.
- [x] `docs/product/post-assessment-flow.md` — 8 pantallas con copy
  literal del reveal post-Day 7. ✓ creado 2026-05.
- [x] `docs/product/onboarding-flow.md` — 16 pantallas con copy literal
  mexicano-tuteo del Day 0. ✓ creado 2026-05.
- [x] `docs/product/push-notifications-copy-bank.md` — Banco
  autoritativo de copys para los 15 notification_ids + 80 logros.
  ✓ creado 2026-05.
- [x] `docs/product/atomics-catalog-seed.md` — Seed inicial de los
  ~350 media_atomics con voice pool, 6 characters, generation
  pipelines, costos. ✓ creado 2026-05.
- [x] `docs/cross-cutting/data-and-events.md` — Modelo unificado de
  eventos. ✓ creado.
- [x] `docs/cross-cutting/security-threat-model.md` — STRIDE por
  sistema. ✓ creado.
- [x] `docs/cross-cutting/testing-strategy.md` — Unit + integration (no
  E2E). ✓ creado.
- [x] `docs/cross-cutting/i18n.md` — Estrategia de localización.
  ✓ creado.
- [x] `docs/business/legal-compliance.md` — T&C, privacy, GDPR/LFPDPPP.
  ✓ creado.
- [x] `docs/ops/incidents-runbook.md` — Manejo de incidentes.
  ✓ creado.
- [x] `docs/ops/cicd.md` — Pipelines de CI/CD. ✓ creado.

### 4.1 Exploraciones (no son spec final aún)

- [ ] **Annotations culturales / coloquiales** — diseño detallado en
  [`docs/explorations/cultural-annotations.md`](explorations/cultural-annotations.md).
  Target: Fase 2 (mes 4–6 post-MVP). Toca `content-creation-system`,
  `pedagogical-system`, `student-profile-and-assessment`, `i18n`,
  `ai-gateway-strategy`. Incluye:
  - Tabla `colloquialism_equivalents` con ~91 entradas seed.
  - Schema `ContentWithAnnotations` mixin para assets relevantes.
  - Pipeline con task IA `extract_cultural_annotations`.
  - UI con marker inline + bottom-sheet.
  - 6 decisiones abiertas por cerrar antes de implementar (ver §9 del
    doc).

- [x] **Estrategia de assets multimedia (audio, imagen, video)** —
  **PARCIALMENTE PROMOVIDO en v1.2 (2026-05).** Diseño detallado en
  [`docs/explorations/multimedia-assets.md`](explorations/multimedia-assets.md).
  Promovido a spec: modelo atomic+composite, sistema de labels,
  schemas (`media_atomics`, `characters`, `atomic_generation_queue`),
  tasks IA foundational (`generate_audio_tts`, `generate_image`,
  `validate_image_no_text`). Sigue en exploración: video generation
  pipelines (HeyGen/Hedra/Runway), voice library final.

- [x] **Sporadic questions durante pre-assessment** — **PROMOVIDO a
  spec en 2026-05.** Diseño detallado en
  [`docs/explorations/sporadic-questions.md`](explorations/sporadic-questions.md).
  Specs tocadas: `student-profile-and-assessment.md` §7.1,
  `ai-roadmap-system.md` §7.4, `pedagogical-system.md` §2.6,
  `assessment-content-bank.md` §1.2, `data-and-events.md` §5.13.
  Decisiones cerradas: lifespan = Day 1 hasta assessment completion
  (NO post-assessment), free path (no cobra Sparks), ML adaptativo
  post-MVP, no consent extra (consent_audio_processing cubre),
  respuestas fake = ignorar + log para ML futuro. Implementación
  Fase 1.5.
  Toca `content-creation-system`, `pedagogical-system`,
  `ai-gateway-strategy`, `sparks-system`. Incluye:
  - Sistema de labels que descompone `AssetType` en dimensiones.
  - Tabla `media_atomics` separada de `learning_assets` para reuse.
  - Tabla `characters` para consistency cross-asset.
  - Strategy 100% TTS con voz "Anita" host cloneada + pool de 8 voces.
  - Strategy 100% AI generation para imágenes (DALL-E 3, fallback Flux).
  - Strategy video con HeyGen/Hedra (talking head 60-70%) + Runway
    Gen-3 (scenes 15-20%) + Lottie (animations 10%) + manual (screen
    recordings 5%).
  - 8 nuevas AI Gateway tasks.
  - Cost model: ~$1.000 USD all-in para MVP con reuse 2x (vs $2.000
    sin reuse).
  - 8 decisiones abiertas (ver §14 del doc).

---

## 5. Trabajo pendiente por sistema

### 5.1 Pre-implementación (todos los sistemas)

- [ ] Definir **owner** para cada documento (todos están con `Owner: —`).
- [ ] Crear los ADRs listados arriba.
- [ ] Validar el plan de negocio y comprometer pricing.
- [ ] Decidir las decisiones marcadas como "Recomendación" arriba.

### 5.2 Sprint 0 (setup foundational)

- [ ] Crear proyecto Firebase y configurar Authentication.
- [ ] Crear proyecto Supabase con schemas iniciales (users, student_profiles,
  learning_blocks).
- [ ] Setup Cloudflare Workers + R2 + KV.
- [ ] Setup PostHog con eventos core definidos.
- [ ] Setup Sentry para error tracking.
- [ ] Apple Developer Program ($99/año) y Google Play Console ($25 único).
- [ ] Configurar Google Sign In en Google Cloud Console.
- [ ] Configurar Apple Sign In en Apple Developer Console.
- [ ] Definir Task Registry inicial del AI Gateway con 3-5 tareas.

### 5.3 Sistema pedagógico — calibración inicial

- [ ] **Recolectar dataset de calibración:** 50–100 usuarios voluntarios
  con nivel CEFR conocido (validado por examen oficial o evaluación
  docente).
- [ ] Hacer que cada uno haga el assessment completo y mapear scores → CEFR.
- [ ] Validar que el sistema clasifique correctamente nuevos usuarios.
- [ ] Contratar consultor en lingüística aplicada (4–8h) para validación
  pedagógica.

### 5.4 Content creation — biblioteca MVP

> **Plan refinado 2026-05.** Total MVP: **175 bloques** con
> `asset_sequence` definida → ~700 composites → ~350 atomics únicos
> (con reuse 2x). Detalle por track en `track-*-blocks.md`. Seed de
> atomics en `atomics-catalog-seed.md`.

- [ ] **Sprint 0 atomics seed (5 semanas content team):**
  - [ ] Sprint 1: voice pool 12 voces + characters infraestructura.
  - [ ] Sprint 2-3: 178 atomics character-bound (6 characters).
  - [ ] Sprint 4: 40 ambient + 70 scene image atomics.
  - [ ] Sprint 5: 60 generic phrase atomics + cleanup.
  - Costo total seed: ~$24 generation + ~$2.750 human time.
- [ ] **Sprints 1-N de creación de bloques (12-15 semanas content team):**
  - [ ] Job Ready 74 bloques (~150h, 6 sprints).
  - [ ] Travel Confident 39 bloques (~78h, 5 sprints).
  - [ ] Daily Conversation 62 bloques (~125h, 7 sprints).
  - **Estrategia:** crear DC primero (genera atomics universales que
    reusan los otros 2 tracks).
- [ ] Garantizar 5+ assets B1, 5+ B2, 3+ C1 para cada sub-skill core
  (verificar al completar tracks).
- [ ] Build admin panel básico para review (Next.js + Tailwind).
- [ ] Pipeline de generación con prompts (ya documentado en
  `atomics-catalog-seed.md` §9 — pendiente código).
- [ ] Pipeline de validación automática (ver
  `content-creation-system.md` §5.3).

### 5.5 Métricas y experimentación

- [ ] Implementar tracking de todos los eventos definidos en sección 6 de
  `business/metrics-and-experimentation.md`.
- [ ] Establecer ritual de **daily check (5 min)**.
- [ ] Establecer ritual de **weekly review (30–60 min)** los lunes.
- [ ] Definir primeros experimentos del trimestre 1:
  - Onboarding length (5 vs 7 vs 3 preguntas).
  - Trial duration (7 vs 14 días).
  - Free trial Sparks (50 vs 30 vs 100).
  - Assessment timing (día 5 vs 7 vs 10).

### 5.6 Anti-fraude

- [ ] Verificar compliance de fingerprinting por jurisdicción (GDPR,
  CCPA, regulaciones latinoamericanas).
- [ ] Definir disclosure en privacy policy.
- [ ] Cláusulas anti-abuse en Terms of Service.

---

## 6. Riesgos a monitorear

Riesgos identificados en los documentos que requieren seguimiento activo:

| Sistema | Riesgo | Mitigación documentada |
|---------|--------|----------------------|
| Anti-fraude | Falsos positivos altos → churn | Calibración cuidadosa, apelaciones rápidas |
| Motivación | Saturación de notificaciones de logros | Calibración cuidadosa, opt-out granular |
| Motivación | Streak addiction tóxica | Vacation mode, freeze tokens |
| AI Gateway | Outage por cambio de proveedor | Multi-proveedor activo, fallback chain |
| Sparks | Cambio de costo de proveedor LLM | Sparks ajustables sin tocar precio al usuario |
| Auth | Desincronización Firebase ↔ Postgres | Endpoint idempotente + estrategia B opcional |
| Content | Contenido que se siente robótico o estereotipado | Revisión humana, especialmente al inicio |
| Pedagogía | Sistema sobrevalúa progreso (mastery falso) | Re-evaluación periódica, validación pedagógica externa |

---

## 7. Cómo se actualiza este documento

- **Al tomar una decisión:** mover el detalle al documento del sistema
  correspondiente y marcar `[x]` aquí (o eliminar si la decisión queda
  documentada en otro lugar).
- **Al descubrir trabajo nuevo:** agregarlo a la sección apropiada con
  referencia al documento que lo motivó.
- **Cada sprint:** revisar al inicio para priorizar y al cierre para
  cerrar lo completado.
- **Mensualmente:** chequeo completo de coherencia con los documentos
  fuente.

---

*Documento vivo. Reflejá la realidad: si hay un pendiente que ya no
aplica, eliminalo en lugar de dejarlo perpetuarse.*

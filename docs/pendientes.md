# Pendientes y Decisiones Abiertas

> Inventario consolidado de **decisiones abiertas**, **trabajos
> pendientes** y **documentos a crear** identificados en cada sistema.
> Cada ítem indica el documento de origen para profundizar.

**Última actualización:** 2026-04
**Cómo usar:** revisar al menos al inicio de cada sprint. Marcar como
resuelto al tomar decisión y mover el detalle al documento correspondiente.

---

## 1. Decisiones de producto

### 1.1 Onboarding y assessment
*(de `product/student-profile-and-assessment.md`)*

- [ ] ¿Permitir hacer el assessment antes del día 7 si el usuario lo pide?
- [ ] ¿Re-aplicar el assessment cuando el usuario cambia significativamente
  su objetivo (ej: "travel" → "job interview")?
- [ ] ¿Cómo manejar usuarios que "rompen" el assessment (responden al azar,
  no graban audio)? Detectar y reintentar.
- [ ] ¿Mostrar el roadmap inicial completo o solo los primeros niveles para
  mantener intriga sobre el assessment?
- [ ] ¿Permitir descargar PDF del resultado del assessment? (uso como prueba
  para empleadores).

### 1.2 Roadmap personalizado
*(de `product/ai-roadmap-system.md`)*

- [ ] ¿Permitir que el usuario edite manualmente su roadmap (skip, reorder)?
  Pros: agency. Contras: rompe coherencia pedagógica.
- [ ] ¿Tracks múltiples activos simultáneamente o uno solo? Empezar con uno.
- [ ] ¿Cómo manejar usuarios que cambian de objetivo a mitad de roadmap?
  ¿Regenerar entero o mergear?
- [ ] ¿Certificados al completar tracks? ¿En qué momento (100% o mastery
  promedio X)?
- [ ] Política de re-evaluación: ¿mini-test cada N semanas para recalibrar?

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
*(de `product/content-creation-system.md`)*

- [ ] ¿Crowdsource de assets de la comunidad? Pros: escala. Contras: calidad
  variable, moderación.
- [ ] ¿Licenciar contenido existente de proveedores educativos? Cuidado con
  derechos.
- [ ] ¿Generar assets en tiempo real para usuarios con perfiles únicos?
  Posible pero costoso.
- [ ] ¿Versionado semántico (semver) o flat para assets? Semver puede ser
  overkill.

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
*(de `architecture/notifications-system.md`)*

- [ ] ¿Permitir snooze de notificaciones por 1 hora desde la propia notif?
- [ ] ¿Smart timing: aprender cuándo cada usuario suele abrir vs hora
  preferida declarada?
- [ ] ¿Notificaciones grupales (múltiples eventos en un solo mensaje)?
- [ ] ¿Rich notifications con imágenes/audio embebido?
- [ ] ¿A/B test de variantes de mensajes con Firebase Remote Config?
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
- [ ] `docs/product/learning-blocks-taxonomy.md` — Taxonomía de la
  biblioteca de bloques.
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

- [ ] Generar **700 assets de calidad** para MVP (~$3.500 USD estimados):
  - [ ] ~150 pronunciation drills.
  - [ ] ~200 roleplays.
  - [ ] ~150 listening exercises.
  - [ ] ~100 free response prompts.
  - [ ] ~100 vocabulary in context.
- [ ] Cobertura: Job Ready (288), Travel Confident (160), Daily
  Conversation (240).
- [ ] Garantizar 5+ assets B1, 5+ B2, 3+ C1 para cada sub-skill core.
- [ ] Build admin panel básico para review (Next.js + Tailwind).
- [ ] Pipeline de generación con prompts iniciales.
- [ ] Pipeline de validación automática.

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

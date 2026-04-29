# Contexto del Proyecto

> Síntesis ejecutiva del producto, su mercado, su arquitectura y las
> decisiones estructurales tomadas. Punto de entrada para cualquier persona
> (o agente) que se incorpore al proyecto.

**Última actualización:** 2026-04
**Estado del proyecto:** Diseño v1.0, pre-implementación

---

## 1. ¿Qué es el producto?

App de aprendizaje de inglés para hispanohablantes latinoamericanos
centrada en **speaking y listening**, asistida por IA. El diferenciador
no es el contenido (otras apps tienen más), sino:

- Personalización profunda con un roadmap adaptativo por usuario.
- Foco en errores típicos del hispanohablante (fonéticos y gramaticales).
- Conversación 1 a 1 con IA en contexto realista (entrevistas laborales,
  trabajo remoto, viajes).
- Feedback inmediato y honesto sobre pronunciación, fluidez y gramática.

### 1.1 Propuesta de valor en una frase

"Hablá inglés con confianza para conseguir el trabajo / viaje / vida que
querés, en menos tiempo del que pensás."

### 1.2 Tracks principales

- **Job Ready:** preparación para entrevistas y trabajo remoto en empresas
  internacionales.
- **Travel Confident:** inglés para viajar.
- **Daily Conversation:** inglés cotidiano.
- (Post-MVP: Business English, Tech Professional, Academic, Customer Service,
  Healthcare).

---

## 2. Mercado y modelo de negocio

### 2.1 Mercado objetivo

- **Geografía:** Latam (México, Argentina, Colombia, Chile, Perú).
- **Demografía:** estudiantes y profesionales jóvenes (~18-44 años).
- **Idioma de la app:** español neutro latinoamericano.

### 2.2 Pricing (en pesos mexicanos como referencia)

| Plan | Precio MXN/mes | Sparks/mes |
|------|---------------:|-----------:|
| Free | $0 | 5 (one-time trial) |
| Básico | $30 | 30 |
| Pro | $100 | 200 |
| Premium | $250 | 600 |

Packs adicionales de Sparks: $50–$350 MXN. Pricing localizado por país.

### 2.3 Free trial

7 días con 50 Sparks gratuitos y acceso completo al Plan Pro. Al día 7,
assessment de 20 minutos que desbloquea el roadmap definitivo y
desencadena la conversión a pago.

---

## 3. Arquitectura técnica

### 3.1 Stack principal

| Capa | Tecnología | Por qué |
|------|-----------|---------|
| App móvil (iOS + Android) | React Native + Expo | Un código, lanzamiento simultáneo |
| Landing / Web app | Next.js + Tailwind, Vercel | Simple, gratis, rápido en Latam |
| Auth | Firebase Authentication | SSO maduro (Google, Apple, email), Phone OTP opcional |
| Base de datos | Postgres (Supabase) | Relacional + JSONB, RLS, sin operar DB |
| Cache | Redis (Upstash) | Biblioteca de bloques, sesiones |
| Storage de assets | Cloudflare R2 | Sin egress fees, CDN global |
| API layer | Cloudflare Workers (TypeScript) | Latencia baja en Latam, serverless |
| Orquestación / cron | Inngest (MVP) → Dagster (escala) | Workflows con steps, observabilidad |
| Push notifications | Firebase Cloud Messaging | Gratis ilimitado |
| LLM (curaduría) | Claude Haiku 4.5 / Gemini 2.0 Flash | Baratos, JSON output confiable |
| LLM (batch nocturno) | Anthropic Batch API (50% off) | Sin urgencia, máximo descuento |
| STT | Whisper / Deepgram | Migración eventual a self-hosted |
| Pronunciation scoring | Azure Pronunciation Assessment → modelo propio | Empezar API, reemplazar al validar |
| Pagos in-app | RevenueCat (Apple + Google) | Orquestador unificado |
| Pagos web | Stripe + MercadoPago / dLocal | Internacional + locales Latam |
| Analytics | PostHog (producto) + Firebase Analytics + Sentry | Tier gratuito generoso |
| Status page | Better Stack | Reduce tickets de soporte |

### 3.2 Sistemas centrales

El producto se compone de los siguientes sistemas, cada uno con su
documento detallado en `docs/architecture/` o `docs/product/`:

**Arquitectura:**
- `ai-gateway-strategy.md` — Capa única de acceso a LLMs con multi-proveedor,
  fallback, A/B testing y observabilidad. **Regla de oro:** ningún código de
  negocio menciona proveedor.
- `authentication-system.md` — Firebase Auth con Google, Apple, email,
  anonymous; sincronización con Postgres vía webhook.
- `sparks-system.md` — Tokens de consumo. Controlan costos de IA, permiten
  ajustar precios sin tocar el plan del usuario, habilitan gamificación.
- `notifications-system.md` — FCM para push, daily reminders timezone-aware
  con contenido personalizado generado en batch nocturno.
- `anti-fraud-system.md` — Prevención > detección > castigo. Fingerprinting,
  scoring de fraude, niveles de respuesta automatizados.
- `platform-strategy.md` — Móvil primero (iOS+Android simultáneo), web como
  complemento, app nativa de Windows fuera de scope.

**Producto:**
- `ai-roadmap-system.md` — Roadmap personalizado generado por IA que cura
  bloques de una biblioteca predefinida ("IA cura, no inventa").
- `pedagogical-system.md` — 5 dimensiones (pronunciación, fluidez, gramática,
  vocabulario, listening) con sub-skills evaluables. Mastery > completion.
- `student-profile-and-assessment.md` — Perfil construido en 3 capas: lo
  declarado (onboarding), lo observado (7 días de trial), lo medido
  (assessment de 20 min al día 7).
- `motivation-and-achievements.md` — 5 capas: hábito diario (streaks),
  progreso visible, logros, social, personalización IA. 80 logros en
  catálogo total.
- `content-creation-system.md` — Pipeline IA-asistida con validación humana
  para construir biblioteca de ~700 assets en MVP.

**Negocio / operación:**
- `plan_de_negocio.docx` — Plan de negocio general (fuente fuera del repo).
- `customer-support-system.md` — Self-service > AI > humano async > humano
  prioritario. Diseñado para dev solo.
- `metrics-and-experimentation.md` — North Star Metric: Weekly Practicing
  Users (WPU). Framework de A/B testing en PostHog.

---

## 4. Principio rector del uso de IA

> **"IA cura, no inventa."**

La IA selecciona, secuencia y personaliza contenido predefinido. No genera
contenido nuevo en tiempo real. Esto garantiza:

- Coherencia pedagógica (validada antes de publicar).
- Costos controlados (caching agresivo, batch nocturno con 50% off).
- Experiencia consistente (sin sorpresas de calidad variable).

Targets de costo de IA por usuario:
- Onboarding (one-time): ~$0.05 USD.
- Mantenimiento del roadmap: ~$0.01 USD/mes.
- Total objetivo primer mes: < $0.10 USD/usuario.
- Meses subsiguientes: < $0.02 USD/usuario.

---

## 5. North Star Metric y métricas críticas

**NSM:** Weekly Practicing Users (WPU) — usuarios que completaron al
menos 3 sesiones de práctica en la última semana.

**Targets clave (MVP):**

| Métrica | Target |
|---------|--------|
| Onboarding completion | >85% |
| Day 1 retention | >50% |
| Day 30 retention | >20% |
| Trial → Paid conversion | >10% |
| Conversion total (registro → pago) | >12% |
| ARPU (paid) | >$3 USD/mes |
| LTV/CAC ratio | >3 |
| Costo IA por usuario | < $0.10/mes |

---

## 6. Roadmap de plataformas

| Fase | Plazo | Alcance |
|------|-------|---------|
| 1 | Mes 1–3 | Landing web (Next.js), captación de waitlist, sesión de prueba sin registro |
| 2 | Mes 3–9 | App móvil iOS + Android (React Native + Expo) — MVP |
| 3 | Mes 9–12 | PWA / web app completa (react-native-web sobre la app móvil) |
| 4 | Mes 12+ | Expansión condicional: tablets, calendarios, B2B portal |

Países en el rollout: México (soft launch), después Argentina, Colombia,
Chile, Perú.

---

## 7. Equipo y operación

**Configuración actual:** **dev solo** trabajando con asistencia de IA.

Implicaciones de diseño:
- Stack favorece servicios managed (Supabase, Cloudflare, Inngest) sobre
  infraestructura propia.
- Soporte: 60–70% self-service, 20–30% AI assistant, <25% escalación humana.
- Anti-fraude prioriza prevención por diseño y automatización segura
  (niveles 1–3 automáticos, 4–5 manuales).
- Content creation: pipeline IA + revisión humana selectiva (50% al inicio,
  10% a escala).

---

## 8. Estado actual y próximos pasos

### 8.1 Estado

Todos los sistemas están en **diseño v1.0**. Nada implementado todavía.
Esta carpeta `docs/` contiene la especificación completa de qué se va a
construir.

### 8.2 Próximos pasos sugeridos (orden propuesto)

1. **Pre-implementación:** definir owner, validar plan de negocio,
   comprometer stack.
2. **Sprint 0 (semana -1):** setup Firebase, Supabase, Cloudflare, R2,
   PostHog. Schema base. Auth básica.
3. **Sprint 1 (semanas 1–4):** Onboarding + mini-test + roadmap inicial
   + biblioteca semilla (~250 bloques de Job Ready).
4. **Sprint 2 (semanas 5–8):** Sparks system + AI Gateway + primeros
   ejercicios + tracking de progreso.
5. **Sprint 3 (semanas 9–12):** Job nocturno + insights humanizados +
   notificaciones push + assessment del día 7.
6. **Soft launch:** México, 4–6 semanas, validar unit economics.
7. **Hard launch:** expansión regional.

Ver `pendientes.md` para la lista de decisiones abiertas y trabajo
pendiente.

---

## 9. Cómo navegar este repo

```
docs/
├── contexto.md              ← este archivo (entrada)
├── pendientes.md            ← decisiones abiertas y TODOs
├── reglas.md                ← principios y convenciones
├── architecture/            ← cómo está construido el sistema
│   ├── ai-gateway-strategy.md
│   ├── anti-fraud-system.md
│   ├── authentication-system.md
│   ├── notifications-system.md
│   ├── platform-strategy.md
│   └── sparks-system.md
├── product/                 ← qué hace el producto y por qué
│   ├── ai-roadmap-system.md
│   ├── content-creation-system.md
│   ├── motivation-and-achievements.md
│   ├── pedagogical-system.md
│   └── student-profile-and-assessment.md
└── business/                ← cómo se mide y opera el negocio
    ├── customer-support-system.md
    ├── metrics-and-experimentation.md
    └── plan_de_negocio.docx
```

**Para una pregunta específica, empezá por:**

| Si necesitás entender... | Leé primero |
|--------------------------|-------------|
| Qué hace el producto | `product/student-profile-and-assessment.md` y `product/ai-roadmap-system.md` |
| Cómo se gana dinero | `business/plan_de_negocio.docx` y `architecture/sparks-system.md` |
| Cómo se construye | `architecture/platform-strategy.md` y `architecture/ai-gateway-strategy.md` |
| Cómo se enseña | `product/pedagogical-system.md` y `product/content-creation-system.md` |
| Cómo se retiene al usuario | `product/motivation-and-achievements.md` y `architecture/notifications-system.md` |
| Cómo se mide el éxito | `business/metrics-and-experimentation.md` |

---

*Documento vivo. Actualizar cuando cambien decisiones estructurales,
sistemas o estado del proyecto.*

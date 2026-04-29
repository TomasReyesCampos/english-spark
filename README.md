# English Spark

> App de aprendizaje de inglés para hispanohablantes latinoamericanos,
> centrada en **speaking y listening**, asistida por IA.

**Estado:** Diseño v1.0 (pre-implementación)
**Plataformas objetivo:** iOS, Android (móvil primero), web (PWA)
**Mercado:** Latam — México, Argentina, Colombia, Chile, Perú

---

## ¿Qué es esto?

Un producto de educación que ayuda a hispanohablantes a hablar inglés
con confianza para conseguir trabajo internacional, viajar, o cualquier
objetivo personal. El diferenciador no es el contenido —otras apps tienen
más— sino:

- **Personalización profunda:** roadmap adaptativo por usuario generado
  por IA.
- **Foco en errores típicos del hispanohablante:** /θ/, /ð/, /v/, vocales
  tensas, past perfect, artículos.
- **Conversación 1 a 1 con IA en contexto realista:** entrevistas, trabajo
  remoto, viajes.
- **Feedback honesto:** mastery > completion. El sistema no premia esfuerzo
  sin resultado.

Principio rector: **"IA cura, no inventa."** La IA selecciona y secuencia
contenido predefinido en lugar de generar desde cero. Costos controlados,
calidad consistente.

---

## Estructura del repo

```
.
├── README.md                ← este archivo
└── docs/
    ├── contexto.md          ← síntesis ejecutiva (empezá acá)
    ├── pendientes.md        ← decisiones abiertas y TODOs
    ├── reglas.md            ← principios y convenciones
    ├── architecture/        ← cómo está construido el sistema
    │   ├── ai-gateway-strategy.md
    │   ├── anti-fraud-system.md
    │   ├── authentication-system.md
    │   ├── notifications-system.md
    │   ├── platform-strategy.md
    │   └── sparks-system.md
    ├── product/             ← qué hace el producto y por qué
    │   ├── ai-roadmap-system.md
    │   ├── content-creation-system.md
    │   ├── motivation-and-achievements.md
    │   ├── pedagogical-system.md
    │   └── student-profile-and-assessment.md
    └── business/            ← cómo se mide y opera el negocio
        ├── customer-support-system.md
        ├── metrics-and-experimentation.md
        └── plan_de_negocio.docx
```

---

## Por dónde empezar

| Si querés entender... | Leé primero |
|-----------------------|-------------|
| El proyecto en general | [docs/contexto.md](docs/contexto.md) |
| Qué falta decidir o construir | [docs/pendientes.md](docs/pendientes.md) |
| Las reglas no negociables del diseño | [docs/reglas.md](docs/reglas.md) |
| Qué hace el producto | [docs/product/student-profile-and-assessment.md](docs/product/student-profile-and-assessment.md) y [docs/product/ai-roadmap-system.md](docs/product/ai-roadmap-system.md) |
| Cómo se gana dinero | [docs/business/plan_de_negocio.docx](docs/business/plan_de_negocio.docx) y [docs/architecture/sparks-system.md](docs/architecture/sparks-system.md) |
| Cómo se construye | [docs/architecture/platform-strategy.md](docs/architecture/platform-strategy.md) y [docs/architecture/ai-gateway-strategy.md](docs/architecture/ai-gateway-strategy.md) |
| Cómo se enseña | [docs/product/pedagogical-system.md](docs/product/pedagogical-system.md) y [docs/product/content-creation-system.md](docs/product/content-creation-system.md) |
| Cómo se retiene al usuario | [docs/product/motivation-and-achievements.md](docs/product/motivation-and-achievements.md) y [docs/architecture/notifications-system.md](docs/architecture/notifications-system.md) |
| Cómo se mide el éxito | [docs/business/metrics-and-experimentation.md](docs/business/metrics-and-experimentation.md) |

---

## Stack en una mirada

| Capa | Tecnología |
|------|-----------|
| App móvil | React Native + Expo |
| Web | Next.js + Tailwind, Vercel |
| Auth | Firebase Authentication (Google, Apple, email, anonymous) |
| Base de datos | Postgres (Supabase) |
| Cache | Redis (Upstash) |
| Storage de assets | Cloudflare R2 |
| API layer | Cloudflare Workers (TypeScript) |
| Orquestación / cron | Inngest |
| Push | Firebase Cloud Messaging |
| LLM (curaduría) | Claude Haiku 4.5 / Gemini 2.0 Flash |
| LLM (batch) | Anthropic Batch API (50% off) |
| STT | Whisper / Deepgram |
| Pronunciation scoring | Azure → modelo propio |
| Pagos in-app | RevenueCat |
| Pagos web | Stripe + MercadoPago / dLocal |
| Analytics | PostHog + Firebase Analytics + Sentry |

Detalle completo y justificación de cada elección en [docs/contexto.md](docs/contexto.md)
y los documentos por sistema.

---

## North Star Metric

**Weekly Practicing Users (WPU):** usuarios que completaron al menos
3 sesiones de práctica en la última semana.

Targets MVP: onboarding completion >85%, D30 retention >20%, trial → paid
>10%, ARPU >$3 USD/mes, costo IA <$0.10/usuario/mes.

---

## Roadmap

| Fase | Plazo | Alcance |
|------|-------|---------|
| 1 | Mes 1–3 | Landing web + waitlist + sesión de prueba sin registro |
| 2 | Mes 3–9 | App móvil iOS + Android (MVP) |
| 3 | Mes 9–12 | PWA / web app completa |
| 4 | Mes 12+ | Expansión condicional |

Soft launch: México (4–6 semanas). Hard launch: Argentina, Colombia,
Chile, Perú.

---

## Equipo

**Configuración actual:** dev solo + asistencia de IA. Esto es una
restricción de diseño, no un detalle: el stack favorece servicios managed
sobre infraestructura propia, automatización sobre procesos manuales,
simple sobre sofisticado.

---

*Todos los documentos en `docs/` son vivos. La fuente de verdad del
diseño es lo que está acá; si la implementación diverge, se actualiza
el documento.*

# ADR-004: Firebase Cloud Messaging para push notifications en MVP

**Status:** Accepted
**Date:** 2026-04
**Author:** —
**Audiencia:** agente AI implementador.

---

## Contexto

El producto vive de la consistencia diaria. Sin notificaciones, los
usuarios olvidan practicar y abandonan en pocos días. Necesitamos:

1. Push notifications nativas a iOS y Android (recordatorios diarios,
   logros, re-engagement).
2. Bajo costo operativo (dev solo).
3. Cobertura completa de carriers en Latam.
4. Integración con el stack ya planeado (Firebase Auth).

## Decisión

**Firebase Cloud Messaging (FCM) directo desde Cloudflare Workers para
todo el push del MVP. Sin WhatsApp, sin SMS, sin email transaccional
masivo (post-MVP).**

### Detalles

- Push delivery: FCM HTTP v1 API, llamada desde Cloudflare Workers con
  service account JWT.
- SDK cliente: `@react-native-firebase/messaging` (requiere Expo dev
  build).
- Orquestación: Cloudflare Workers + Cron Triggers.
- Estado de rate limits: Cloudflare Durable Objects.
- Personalización: contenido pre-generado en batch nocturno por AI
  Gateway, persistido en `notifications_scheduled`.

### Tipos de notificaciones en MVP

- Daily reminder (1/día, hora preferida del usuario).
- Streak at risk (alerta antes de perder racha).
- Achievement unlocked (logros).
- Welcome series (D1, D3 post-registro).
- Re-engagement (D3, D7, D14 sin actividad).
- Transactional: low Sparks, pack expiring, payment failed.

### Lo que NO se incluye en MVP

- WhatsApp Business API.
- SMS.
- Email transaccional masivo (sí recibos via Stripe/RevenueCat).
- In-app messaging avanzado (uso simple, mensaje básico al abrir).

## Alternativas consideradas

### A. Expo Push Service (wrapper de FCM/APNs de Expo)

**Rechazada para MVP, aunque la app usa Expo.** Pros: setup más simple,
abstrae FCM/APNs. Contras:
- Capa intermedia adicional con su propio rate limiting y SLA.
- Menos control sobre features específicas de FCM (topics, conditions,
  analytics).
- Si se migra fuera de Expo en el futuro, hay que reconfigurar.
- Firebase Analytics no integra con Expo Push directamente.

**Decisión:** usar FCM directo. Compatible con Expo (via
`@react-native-firebase/messaging`) y permite máximo control.

### B. WhatsApp Business API desde MVP

**Rechazada para MVP.** Pros: open rate alto en Latam (80%+). Contras:
- Costo: $0.005–0.05 USD por mensaje (variable por país y tipo).
  A 1.000 usuarios activos diarios = $150–1.500 USD/mes solo en mensajes.
- Aprobaciones de Meta para templates: ciclo lento (días).
- Setup complejo (webhooks de Meta, template management).
- Requiere validar producto antes de invertir.

**Reconsiderar cuando:** producto validado (D30 retention >30%), >5.000
usuarios activos, push open rate <10%.

### C. SMS

**Rechazada.** Pros: ubicuo. Contras: costoso ($0.01–0.05 por SMS, aún
peor que WhatsApp), open rate bajo para apps consumer, regulado en
varios países.

### D. Email solo

**Rechazada.** Open rate <20%, no genera urgencia para hábito diario.
Como complemento de push: sí (insights semanales). Como reemplazo: no.

### E. Notificaciones in-app únicamente

**Rechazada.** Solo funcionan si el usuario abre la app, lo cual es
exactamente el problema que estamos tratando de resolver.

### F. OneSignal / Pusher

**Rechazada.** SaaS de notifications agrega capa intermedia con costos
en escala. FCM es gratis e ilimitado.

## Consecuencias

### Positivas

- **Costo cero** para FCM (gratis ilimitado).
- Control total sobre features de FCM.
- Firebase Analytics integra automáticamente métricas de delivery/open.
- Topics y conditions nativas para segmentación.
- Foco en producto durante validación, sin distracción de templates de
  WhatsApp ni costos por mensaje.

### Negativas

- Push tiene open rate moderado (8–15%). Algunos usuarios no engagean.
- iOS APNs requiere certificado Apple ($99/año Apple Developer).
- Algunos usuarios desactivan push y se vuelven inalcanzables.

### Riesgos a monitorear

- Open rate de daily reminder cae <8%: señal de fatiga, ajustar
  frecuencia, contenido o agregar WhatsApp.
- Tokens de FCM se invalidan (cambio de device, reinstall app):
  cleanup periódico necesario.
- Cambios de política de Apple/Google sobre push: revisar trimestralmente.
- Usuarios bloquean push después de spam percibido: rate limiting
  estricto + opt-out granular crítico.

## Referencias

- [`docs/architecture/notifications-system.md`](../architecture/notifications-system.md)
  — diseño detallado.
- [`docs/architecture/authentication-system.md`](../architecture/authentication-system.md)
  — Firebase también para Auth.
- [`docs/architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md)
  — generación de contenido personalizado.
- FCM HTTP v1: https://firebase.google.com/docs/cloud-messaging/migrate-v1.

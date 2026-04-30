# ADR-002: Mobile-first con React Native + Expo, lanzamiento simultáneo iOS + Android

**Status:** Accepted
**Date:** 2026-04
**Author:** —
**Audiencia:** agente AI implementador.

---

## Contexto

El producto es una app de aprendizaje de inglés con foco en **speaking
y listening** para hispanohablantes latinoamericanos. Decisión de
plataforma define stack, esfuerzo de desarrollo, time-to-market y
mercado direccionable.

Restricciones:
- **Dev solo + asistencia de IA.** Capacidad limitada.
- **Casos de uso primariamente móviles:** transporte, gimnasio, momentos
  cortos del día, grabación de audio frecuente.
- **Mercado Latam:** Android ~85% de market share; iOS ~15% pero con
  mayor disposición a pagar.
- **Pagos in-app via stores** reducen fricción para usuarios sin tarjeta
  internacional (común en Latam).

## Decisión

**Móvil primero, iOS y Android simultáneo, con React Native + Expo.
Web como complemento posterior. App nativa de Windows / macOS / Linux
explícitamente fuera de scope (PWA cubre desktop).**

### Roadmap de plataformas

| Fase | Plazo | Plataforma | Stack |
|------|-------|-----------|-------|
| 1 | Mes 1–3 | Landing web | Next.js + Tailwind + Vercel |
| 2 | Mes 3–9 | App móvil iOS + Android (MVP) | React Native + Expo + TypeScript |
| 3 | Mes 9–12 | PWA / web app completa | react-native-web sobre el código móvil |
| 4 | Mes 12+ | Expansión condicional | tablets, B2B portal, etc. |

### Decisiones específicas

- **Lanzamiento iOS + Android simultáneo:** no retrasar uno por el otro.
  React Native + Expo lo hace factible.
- **Expo dev build (no Expo Go):** necesario para Firebase, Apple Sign-In,
  Google Sign-In nativos.
- **react-native-web para PWA:** reusa componentes con ajustes de layout
  responsive. Empezar con Opción A (PWA simple); migrar a Opción B (web
  app independiente con Next.js) solo si hay diferencias de UX
  significativas.
- **App nativa de desktop: NO.** PWA moderna cubre 100% del caso de uso.

## Alternativas consideradas

### A. Flutter (Dart)

**Rechazada.** Pros: excelente performance, UI consistente, lenguaje
limpio. Contras:
- Ecosistema menor para integraciones de IA/audio.
- Lenguaje Dart no se reusa en el resto del stack (Workers, web).
- Comunidad más chica en Latam.
- Reuso para web menos maduro que react-native-web.

### B. Nativo (Swift + Kotlin)

**Rechazada.** Pros: máximo rendimiento, acceso total a APIs. Contras:
inviable para dev solo (dos codebases, doble trabajo, doble
mantenimiento).

### C. Capacitor / Ionic

**Rechazada.** Pros: máximo reuso de código web. Contras: performance
inferior para audio/grabación intensiva (caso de uso central de la app).
Sensación menos nativa.

### D. Web-first (PWA primero, luego app móvil)

**Rechazada.** Casos de uso del producto (caminar, transporte,
notificaciones, audio) son primariamente móviles. PWA-first reduce
mercado direccionable inicial significativamente. Latam usa
mayoritariamente smartphone, no laptop.

### E. iOS-only en MVP, Android después

**Rechazada.** Android es el 85% del mercado en Latam. Lanzar solo iOS
inicialmente excluye al 85% del público objetivo durante meses.

### F. App nativa de Windows

**Rechazada explícitamente.** PWA cubre todos los casos de uso de
desktop con esfuerzo significativamente menor. App nativa agrega
overhead de mantenimiento sin ROI claro. Misma decisión aplica para
macOS y Linux.

### G. Smart TV / Roku / Apple Vision Pro

**Rechazada explícitamente.** Caso de uso no se alinea (no es contenido
que se "vea"); mercado prematuro o pequeño.

## Consecuencias

### Positivas

- Un código → dos plataformas (iOS + Android). Capacidad de dev solo
  factible.
- React Native + Expo es ecosistema muy maduro: bibliotecas para audio
  (expo-av), push (expo-notifications), in-app purchases
  (expo-in-app-purchases), auth (expo-auth-session).
- EAS Build elimina overhead de operación (certificados, builds, deploy).
- EAS Update permite OTA fixes sin pasar por revisión de stores.
- Reuso del código en web vía react-native-web (fase 3).
- Lanzamiento simultáneo maximiza mercado direccionable desde día 1.

### Negativas

- React Native ocasionalmente requiere código nativo para features muy
  específicas (raro para este caso de uso, pero posible).
- Performance ligeramente inferior a nativo puro (no relevante para
  audio/conversación).
- Dependencia del ecosistema Expo: si Expo cambia política, hay que
  reaccionar.
- Apple Developer Program ($99/año) y Google Play Developer ($25 único)
  son costos fijos.

### Riesgos a monitorear

- Cambios de política de App Store / Play Store sobre pagos in-app y
  enlaces externos: pueden afectar estrategia de pricing por canal.
- React Native breaking changes: planificar upgrades trimestrales.
- Diferencias iOS/Android en permisos de micrófono y push: testear en
  devices reales (no simuladores) frecuentemente.

## Referencias

- [`docs/architecture/platform-strategy.md`](../architecture/platform-strategy.md)
  — estrategia detallada de plataformas.
- [`docs/architecture/notifications-system.md`](../architecture/notifications-system.md)
  — usa Expo + FCM para push.
- [`docs/architecture/authentication-system.md`](../architecture/authentication-system.md)
  — Firebase Auth nativo via dev build de Expo.
- Expo: https://docs.expo.dev/.
- React Native Firebase: https://rnfirebase.io/.

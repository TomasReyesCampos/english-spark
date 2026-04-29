# Platform Strategy

> Estrategia de plataformas (móvil, web, desktop) para el desarrollo y
> distribución del producto. Define qué plataformas se construyen, cuándo,
> y con qué stack tecnológico.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —

---

## 1. Decisión central

**Móvil primero (iOS y Android simultáneamente), web como complemento
posterior, app nativa de Windows fuera de scope.**

Esta decisión se basa en:

- El core del producto (speaking y listening) tiene situaciones de uso
  primariamente móviles: transporte, caminatas, gimnasio, momentos
  cortos durante el día.
- El mercado objetivo (Latam, estudiantes y profesionales jóvenes) accede
  a internet primariamente desde el celular.
- Como dev solo, la capacidad de desarrollo y mantenimiento es limitada;
  cada plataforma adicional multiplica el esfuerzo.
- Los pagos in-app via Google Play y App Store reducen fricción para usuarios
  sin tarjeta internacional, común en Latam.

---

## 2. Roadmap de plataformas

### 2.1 Fase 1 (mes 1-3): Landing web

**Objetivo:** captar interés, construir waitlist, permitir prueba gratuita.

**Alcance:**
- Página de aterrizaje con propuesta de valor.
- Formulario de waitlist.
- 1 sesión de prueba interactiva sin registro (validación de mensaje).
- Página de FAQ y planes.

**Stack:** Next.js 15+, deploy en Vercel o Cloudflare Pages, Tailwind CSS.

**Por qué:** validar mensaje y captar usuarios potenciales antes de construir
el producto completo. Permite empezar marketing orgánico sin necesidad de
app instalable.

### 2.2 Fase 2 (mes 3-9): App móvil

**Objetivo:** producto principal funcional en iOS y Android.

**Alcance:** MVP completo según roadmap del producto, lanzamiento simultáneo
en ambas plataformas.

**Stack:** React Native + Expo, TypeScript.

**Por qué:** producto principal del negocio, donde están los usuarios reales.
Lanzamiento simultáneo iOS/Android es factible con React Native y maximiza
mercado direccionable.

### 2.3 Fase 3 (mes 9-12): PWA o web app completa

**Objetivo:** complemento para usuarios que prefieren practicar en escritorio.

**Alcance:** versión web completa de la app, reutilizando componentes mediante
react-native-web. Instalable como PWA en Windows, Mac y Linux.

**Stack:** mismo que app móvil + react-native-web + Next.js como host.

**Por qué:** algunos usuarios prefieren escritorio para sesiones largas de
estudio. La PWA cubre desktop sin requerir app nativa por OS. Decisión
tomada sólo si la demanda real de los primeros 6 meses lo justifica.

### 2.4 Fase 4 (mes 12+): expansión condicional

**Posibles direcciones según métricas:**

- Mejoras específicas de la PWA si tiene tracción.
- Versión optimizada para tablets si hay uso significativo.
- Integraciones con calendarios y herramientas de productividad.
- B2B portal separado para academias y empresas.

**App nativa de Windows:** explícitamente fuera de roadmap. La PWA cubre
todos los casos de uso desktop con esfuerzo significativamente menor.

---

## 3. Análisis de stack móvil

### 3.1 Opciones evaluadas

| Stack | Pros | Contras | Verdict |
|-------|------|---------|---------|
| **React Native + Expo** | Un código para iOS+Android, ecosistema enorme, Expo simplifica deploy y features (audio, push, auth), buena performance para el caso de uso, fácil reuso en web posterior | Requiere código nativo ocasionalmente para features muy específicas | **ELEGIDO** |
| **Flutter** | Excelente performance, UI consistente, lenguaje Dart limpio | Ecosistema menor para integraciones de IA/audio, lenguaje Dart limita reuso, comunidad más chica en Latam | Descartado |
| **Nativo (Swift + Kotlin)** | Máximo rendimiento, acceso total a APIs | Dos codebases, doble trabajo, doble mantenimiento, inviable para dev solo | Descartado |
| **Capacitor / Ionic** | Reuso máximo de código web | Performance inferior para audio/grabación intensiva, sensación menos nativa | Descartado |

### 3.2 Por qué React Native + Expo

Para un desarrollador solo construyendo una app que requiere:

- Grabación de audio frecuente.
- Reproducción de TTS y assets pre-generados.
- Conexión con LLMs en tiempo real para conversación.
- Push notifications para retención.
- Pagos in-app.
- Releases frecuentes con iteración rápida.

React Native + Expo es la combinación con mejor relación esfuerzo/resultado.
Expo en particular elimina la mayor parte del overhead de operación móvil
(certificados, builds, deploy, OTA updates, push, audio APIs).

### 3.3 Features de Expo críticas para el producto

- **expo-av**: grabación y reproducción de audio.
- **expo-notifications**: push notifications.
- **expo-in-app-purchases**: pagos en stores.
- **expo-auth-session**: OAuth con Google/Apple.
- **EAS Build**: builds automáticos para iOS y Android.
- **EAS Update**: OTA updates para corregir bugs sin pasar por revisión de stores.

---

## 4. Estrategia iOS vs Android

### 4.1 Mercado en Latam

- Android: ~85% del market share en Latam.
- iOS: ~15% pero típicamente con mayor disposición a pagar suscripciones.

### 4.2 Decisión

**Lanzar ambos simultáneamente.** No hay razón técnica para retrasar uno
con React Native + Expo, y excluir cualquiera de los dos reduce significativamente
el mercado direccionable.

### 4.3 Diferencias específicas a manejar

| Aspecto | iOS | Android |
|---------|-----|---------|
| Comisión de store | 15-30% | 15-30% |
| Tiempo de revisión | 24-48h típicamente | Horas |
| Política de IA generativa | Más estricta, requerir disclaimers claros | Más laxa |
| Permisos | Más restrictivos, especialmente micrófono | Más flexibles |
| Pagos | Apple Pay y tarjetas | Google Pay, tarjetas, métodos locales |
| Push notifications | APNs, requiere certificado Apple | FCM, más simple |

Expo abstrae la mayoría de estas diferencias, pero conviene tener
checklists específicos por plataforma para review en stores.

---

## 5. Web: alcance y stack

### 5.1 Landing web (fase 1)

**Stack:** Next.js + Tailwind + deploy en Vercel.

**Estructura mínima:**

```
landing/
├── pages/
│   ├── index.tsx         (home con propuesta de valor)
│   ├── pricing.tsx       (planes y Sparks)
│   ├── try.tsx           (sesión de prueba gratis)
│   ├── faq.tsx
│   └── api/
│       └── waitlist.ts   (captura de emails)
└── components/
```

### 5.2 Web app completa (fase 3)

**Estrategia de implementación:**

Opción A: PWA simple usando react-native-web sobre la app móvil. La mayoría
de componentes funcionan sin modificación. Requiere ajustes específicos
para teclado, mouse, layout responsive.

Opción B: Web app independiente con Next.js, reutilizando lógica de negocio
y servicios pero con UI específica para desktop.

**Recomendación inicial:** empezar con Opción A (PWA). Migrar a Opción B
solo si hay diferencias significativas que justifiquen UI separada.

### 5.3 Por qué PWA y no app nativa de desktop

Una PWA moderna, instalada desde el navegador, ofrece:

- Ícono en escritorio, comportamiento de app independiente.
- Notificaciones nativas del sistema operativo.
- Funcionamiento offline para contenido cacheado.
- Acceso a micrófono y audio con APIs estándar.
- Auto-updates sin intervención del usuario.

Lo que NO ofrece vs nativa:

- Integración profunda con el sistema (system tray, hotkeys globales).
- Performance máxima en operaciones muy intensivas.
- Procesamiento de audio en background fuera del navegador.

**Para el caso de uso de la app, ninguna de las desventajas es relevante.**
Una PWA bien hecha cubre el 100% de las necesidades del usuario desktop.

---

## 6. Pagos por plataforma

### 6.1 Comisiones por canal

| Canal | Comisión típica | Cuándo usar |
|-------|----------------|-------------|
| Apple App Store | 15-30% | Suscripciones in-app desde iOS |
| Google Play Store | 15-30% | Suscripciones in-app desde Android |
| Stripe (web) | 3-5% | Pagos desde web app |
| MercadoPago | 4-6% | Pagos locales en Latam (web) |
| dLocal | 4-7% | Métodos locales latinoamericanos |
| Facturación directa | 1-3% | B2B, academias, empresas |

### 6.2 Estrategia de pricing por canal

Las comisiones de las stores impactan significativamente los unit economics.
Estrategia recomendada:

- En la app móvil: pagos in-app como opción principal (frictionless).
- En web: ofrecer descuento por pagar fuera de las stores.
- B2B: siempre facturación directa.

**Ejemplo de presentación al usuario:**

```
Plan Pro:
- $100 MXN/mes (suscripción in-app)
- $85 MXN/mes (pagando desde web con tarjeta)

¡Ahorrá 15% pagando desde web!
```

Esto es legalmente permitido por las políticas vigentes de Apple y Google
en muchas jurisdicciones (verificar políticas actuales antes de implementar,
cambian con frecuencia).

### 6.3 Stack de pagos recomendado

- **RevenueCat**: orquestador unificado de pagos in-app (iOS y Android).
  Maneja webhooks, restauración de compras, sincronización con backend.
- **Stripe**: para pagos web internacionales.
- **MercadoPago**: para pagos locales en Latam (mejor tasa de aprobación).
- **Paddle** (alternativa): merchant of record que maneja impuestos
  internacionales automáticamente.

---

## 7. Distribución por plataforma

### 7.1 App stores

**iOS App Store:**
- Cuenta de desarrollador: $99 USD/año.
- Tiempo típico de aprobación inicial: 24-48h.
- Requisitos específicos: privacy nutrition labels, account deletion in-app.

**Google Play Store:**
- Cuenta de desarrollador: $25 USD único.
- Tiempo típico de aprobación: horas.
- Requisitos específicos: data safety form, target API level actualizado.

**Estrategia de keywords / ASO:**
- Keywords objetivo: "aprender ingles hablar", "speaking ingles", "practica ingles",
  "ingles latinos", "tutor ingles ia", "conversacion ingles".
- Captures localizados para cada país (México, Argentina, Colombia, etc.).
- Reseñas tempranas: pedir activamente a beta testers iniciales.

### 7.2 Web

- Dominio propio (idealmente .com corto y memorable).
- SEO desde el día uno: blog posts sobre aprendizaje de inglés.
- Captación principal mediante contenido en TikTok/Instagram/YouTube
  con creadores hispanohablantes.

### 7.3 Distribución alternativa

- **Sideload Android (APK directo):** considerar para usuarios que no
  pueden o no quieren usar Play Store. Pequeño pero no insignificante en
  algunos países.
- **TestFlight (iOS) y Internal Testing (Android):** para beta cerrada
  antes de lanzamiento público.

---

## 8. Cross-platform considerations

### 8.1 Sincronización de estado

El usuario debe poder:
- Empezar una sesión en móvil y continuarla en web (eventualmente).
- Ver su roadmap actualizado en cualquier plataforma.
- Que sus Sparks reflejen el balance correcto sin importar dónde compre o consuma.

**Implementación:** estado autoritativo en el backend, clientes son thin
clients que sincronizan. Cache local optimista para UX fluida, pero la
fuente de verdad siempre es el backend.

### 8.2 Features específicas por plataforma

Algunas features son nativas de plataforma y no se duplican:

| Feature | Móvil | Web |
|---------|-------|-----|
| Grabación de audio rápida | ✓ (excelente) | ✓ (bueno) |
| Notificaciones push | ✓ (críticas para retención) | ✓ (PWA) |
| Pago in-app | ✓ | ✗ |
| Compartir en redes sociales | ✓ (nativo) | ✓ (web share API) |
| Sesiones offline | ✓ (assets cacheados) | Limitado |
| Sesiones largas (>30 min) | Posible pero no óptimo | ✓ (mejor experiencia) |
| Práctica al caminar/transitar | ✓ | ✗ |

### 8.3 Diseño responsive

Aunque la app móvil es primaria, debe verse bien en tablets sin esfuerzo
adicional. Esto se logra:

- Layouts flexibles desde el inicio (no hardcodear tamaños).
- Componentes que escalan adecuadamente.
- Testing temprano en simuladores de tablet.

---

## 9. Métricas y observabilidad por plataforma

### 9.1 Métricas a separar por plataforma

- Conversion rate del onboarding.
- Retention D1, D7, D30.
- Average session duration.
- Sparks consumidos por sesión.
- Crash rate.
- Tasa de pagos completados.
- ARPU.

### 9.2 Decisiones que estas métricas informan

Comparar performance del producto entre iOS y Android puede revelar:

- Bugs específicos de una plataforma.
- Diferencias de UX que afectan conversion.
- Oportunidades de optimización específica.
- Si una plataforma justifica más inversión que otra.

### 9.3 Stack de analytics

- **PostHog:** product analytics unificado (móvil + web).
- **RevenueCat dashboard:** métricas de monetización.
- **Sentry:** crash reporting y errores en runtime.
- **Better Stack:** uptime y status page.

---

## 10. Plan de testing por plataforma

### 10.1 Beta cerrada (pre-lanzamiento)

- TestFlight (iOS): hasta 10.000 testers, ideal para 100-500 iniciales.
- Internal Testing (Android): grupos de prueba con feedback rápido.
- Versión web: link directo a beta testers manuales.

### 10.2 Soft launch

Lanzamiento limitado a un país (ejemplo: México) por 4-6 semanas antes
de expansión regional. Permite:

- Detectar bugs de producción sin escala completa.
- Validar unit economics con datos reales.
- Iterar en mensajes de marketing y onboarding.

### 10.3 Hard launch

Expansión simultánea a Argentina, Colombia, Chile, Perú una vez validado
el soft launch.

---

## 11. Costos estimados de plataforma

### 11.1 Costos fijos anuales

| Item | Costo |
|------|-------|
| Apple Developer Program | $99 USD/año |
| Google Play Developer | $25 USD único |
| Dominio web | $10-30 USD/año |
| SSL certificate | Gratis (Let's Encrypt) |
| **Total fijo año 1** | **~$140 USD** |

### 11.2 Costos variables relacionados con plataforma

- Hosting web (Vercel/Cloudflare): tier gratuito hasta cierto volumen.
- EAS Build (Expo): tier gratuito limitado, plan pago $99/mes para volumen.
- TestFlight: gratis dentro de Apple Developer Program.
- ASO tools (opcional): $50-200/mes según herramienta.

### 11.3 Comisiones de stores

Aproximadamente 15-30% del revenue mobile, 3-7% del revenue web.
A planificarse en la economía del producto desde el inicio.

---

## 12. Decisiones abiertas

- [ ] ¿Lanzar en países adicionales fuera de Latam? (España, comunidad hispana en EEUU).
- [ ] ¿Versión específica para tablets con UI optimizada o solo responsive?
- [ ] ¿PWA primero o web app independiente cuando llegue la fase 3?
- [ ] ¿Estrategia de pricing diferenciada por país (ajuste por poder adquisitivo)?
- [ ] ¿Incluir Apple Watch / Wear OS? (probablemente no en MVP, evaluar después).

---

## 13. Decisiones que NO se tomaron

Documentación explícita de opciones rechazadas y por qué, para no
revisitarlas innecesariamente:

- **App nativa de Windows:** PWA cubre el caso de uso, app nativa agrega
  overhead sin ROI claro.
- **App de macOS nativa:** mismo razonamiento que Windows.
- **App de Linux:** PWA suficiente, mercado pequeño.
- **Smart TV / Roku:** caso de uso no se alinea con producto (no es un
  contenido que se "vea").
- **Apple Vision Pro / VR:** mercado prematuro, costos de desarrollo no
  justifican.

---

## 14. Referencias internas

- `docs/business/plan_de_negocio.docx` — Plan de negocio general.
- `docs/product/ai-roadmap-system.md` — Sistema de roadmap.
- `docs/architecture/ai-gateway-strategy.md` — Estrategia de proveedores LLM.
- `docs/architecture/sparks-system.md` — Sistema de tokens (a crear).
- `docs/decisions/ADR-002-platform-strategy.md` — ADR formal de esta decisión (a crear).

---

*Documento vivo. Revisar al menos cada 6 meses, especialmente cuando cambien
las políticas de las stores o aparezcan plataformas nuevas relevantes.*

# US-001: Configurar Firebase + Auth providers

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Antes de cualquier auth funcional, necesitamos un proyecto Firebase
configurado con los 4 providers que el producto soporta: Google,
Apple, Email/Password, Anonymous. Esta story es la **fundación
infraestructural** sin la cual ninguna otra story de auth puede
arrancar.

Firebase Auth fue elegido por
[ADR-005](../../decisions/ADR-005-firebase-auth.md) — managed,
multi-provider, ya integrado con FCM (push notifications,
[ADR-004](../../decisions/ADR-004-notifications-fcm.md)).

## Scope

### In

- Crear proyecto Firebase para `english-spark` (3 environments:
  dev, staging, prod).
- Habilitar Cloud Messaging (necesario para US de notifications
  futuras).
- Habilitar Auth con 4 providers:
  - Google Sign In (configurar OAuth client en Google Cloud
    Console).
  - Apple Sign In (configurar en Apple Developer Console + agregar
    `.p8` key).
  - Email/Password.
  - Anonymous.
- Descargar `google-services.json` (Android) y
  `GoogleService-Info.plist` (iOS) para los 3 environments.
- Configurar Firebase service account JSON como secret en
  Cloudflare Workers (`FIREBASE_SERVICE_ACCOUNT`).
- Documentar el setup en `docs/ops/runbook-firebase-setup.md` (a
  crear como parte de esta story).

### Out

- Implementación de los flows de sign in (US-003, US-004, US-005,
  US-006).
- Schema de `users` table (US-002).
- JWT validation middleware (US-007).

## Acceptance criteria

- **Given** un developer con acceso a Firebase Console, **When**
  abre el proyecto `english-spark-dev`, **Then** ve los 4 providers
  habilitados (Google, Apple, Email/Password, Anonymous).
- **Given** los 3 environments configurados (dev, staging, prod),
  **When** se inspecciona cada uno, **Then** cada uno tiene su
  propio bundle id / app id distinto y sus propios `.p8` y OAuth
  clients.
- **Given** el secret `FIREBASE_SERVICE_ACCOUNT` configurado en
  Cloudflare Workers, **When** un Worker llama
  `generateGoogleAccessToken()`, **Then** obtiene un access token
  válido de FCM.
- **Given** un Apple Developer Account sin setup previo, **When**
  un new developer sigue el runbook
  `docs/ops/runbook-firebase-setup.md`, **Then** completa el setup
  iOS en menos de 30 minutos sin escalación.
- **Given** que se ejecuta el sanity check del setup, **When**
  Firebase Auth Console muestra los providers, **Then** un test
  manual de sign in con Google en una app stub retorna un Firebase
  user válido.
- **Given** una equivocación en el setup (ej: typo en bundle id),
  **When** se intenta sign in, **Then** error message claro
  identifica qué config falta o está mal (no error genérico).

## Developer details

### Owning service

Infrastructure (no es un service del producto, sino setup
externo).

### Dependencies

- Apple Developer Program enrollment ($99/año, decisión cerrada
  en `pendientes.md` §5.2).
- Google Play Console enrollment ($25 único, decisión cerrada).
- Acceso admin al Firebase project (typically owner del producto).

### Specs referenciados

- [`architecture/authentication-system.md`](../../architecture/authentication-system.md)
  §3 — providers strategy.
- [`decisions/ADR-005-firebase-auth.md`](../../decisions/ADR-005-firebase-auth.md)
  — decisión arquitectónica.
- [`architecture/notifications-system.md`](../../architecture/notifications-system.md)
  §3.3 — setup FCM compartido.

### Setup steps específicos

1. Crear Firebase project `english-spark-dev` en console.
2. Add iOS app: bundle id `com.englishspark.dev`.
3. Add Android app: package name `com.englishspark.dev`.
4. Enable Authentication → Sign-in method → habilitar 4 providers.
5. Apple Sign In: setup en Apple Developer (Certificates →
   Identifiers → habilitar Sign In with Apple → crear `.p8` key →
   subir a Firebase).
6. Google Sign In: OAuth client en Google Cloud Console (auto-creado
   por Firebase, validar SHA-1 fingerprint Android).
7. Repetir steps 1-6 para staging y prod.
8. Service account: Firebase Console → Project settings → Service
   accounts → Generate new private key → guardar JSON en Cloudflare
   secret `FIREBASE_SERVICE_ACCOUNT`.

### Integration points

- Cloudflare Workers (FCM access token generation).
- Mobile app (`@react-native-firebase/app` package).
- iOS dev environment (Xcode + provisioning profiles).
- Android dev environment (Android Studio + signing keys).

### Notas técnicas

- Firebase project naming convention:
  `english-spark-<env>` para los 3 environments.
- SHA-1 fingerprints Android se obtienen con
  `cd android && ./gradlew signingReport`.
- iOS bundle id no puede cambiar después de submit a App Store.
- Service account JSON es **secret crítico** — nunca commit al
  repo, solo en secret manager.

## Definition of Done

- [ ] 3 Firebase projects creados (dev, staging, prod).
- [ ] 4 providers habilitados en cada uno.
- [ ] `google-services.json` y `GoogleService-Info.plist`
  descargados para los 3 environments y guardados en lugar
  documentado.
- [ ] Apple `.p8` key generada y subida a Firebase para los 3
  environments.
- [ ] Service account JSON guardado en Cloudflare secrets.
- [ ] Sanity check: sign in con Google retorna user válido (test
  manual con app stub).
- [ ] Runbook `docs/ops/runbook-firebase-setup.md` creado con
  steps completos.
- [ ] Tests unit + integration pasan (no aplica para esta story —
  es infra).
- [ ] Validation contra spec referenciado.
- [ ] PR aprobada y mergeada (PR contiene solo el runbook +
  config files mockup).

---

*Esta story es de infraestructura. No produce código de producto pero
desbloquea US-002 a US-009.*

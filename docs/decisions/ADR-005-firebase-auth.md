# ADR-005: Firebase Authentication para identidad de usuario

**Status:** Accepted
**Date:** 2026-04
**Author:** —
**Audiencia:** agente AI implementador.

---

## Contexto

El producto necesita autenticación con:
- SSO multi-proveedor (Google, Apple obligatorio en iOS, email/password).
- Anonymous auth (probar sin registrarse, upgrade después).
- Phone OTP (popular en Latam).
- Account linking (mismo email entre proveedores).
- Account recovery confiable.
- Cumplimiento con políticas de App Store / Play Store.

Restricciones:
- Dev solo. No quiero operar mi propia infra de auth.
- Costo bajo para 0–10.000 usuarios.
- Setup rápido, integración mobile madura.

## Decisión

**Firebase Authentication para todo lo de identidad. Sincronización con
Postgres (Supabase) vía endpoint `/auth/sync-user` al momento del
sign-in.**

### Detalles

- **Proveedores en MVP:** Google Sign-In, Apple Sign-In (obligatorio en
  iOS), Email + Password, Anonymous.
- **Phone OTP:** **NO en MVP.** Reconsiderar mes 3–6 si métricas indican
  >5% de fricción de registro.
- **JWT validation:** todas las requests al backend validan idToken de
  Firebase con JWKS. Cache de JWKS por 24h.
- **Sincronización Firebase → Postgres:** estrategia A (cliente notifica
  al backend) en MVP. Endpoint idempotente. Estrategia B (Cloud Function
  trigger) si observamos desincronización.
- **Account deletion:** soft-delete con período de gracia 30 días, hard
  delete propaga a todos los sistemas.

### Trade-off aceptado

Tener dos identificadores de usuario (Firebase UID y `users.id` UUID
interno). Webhook de sync mantiene consistencia. La principal complejidad
es esa, y se considera aceptable.

## Alternativas consideradas

### A. Supabase Auth

**Considerada seriamente, rechazada.** Pros: integra perfectamente con
Postgres (Row Level Security via JWT), un solo proveedor, schema
compartido. Contras:
- Apple Sign-In en Supabase es más nuevo, menos maduro.
- Phone OTP en Supabase tiene cobertura de carriers en Latam menos
  probada.
- Stack ya incluye Firebase para FCM (notifications). Agregar Auth tiene
  costo marginal cero.
- Si una pieza de Firebase Auth falla, podemos migrar a Supabase Auth
  sin perder mucho (Auth es la capa más fácil de cambiar).

**Decisión:** Firebase Auth gana por madurez de SSO mobile y reuso de
infra Firebase ya planeada.

### B. Auth0

**Rechazada.** Pros: enterprise-grade, features avanzados (MFA, RBAC,
SSO empresarial). Contras: caro a partir de tier gratuito (>1.000 MAU
empieza a costar significativamente). Overkill para MVP consumer.

### C. Roll-your-own auth

**Rechazada.** Pros: control total. Contras: inviable para dev solo.
Auth es uno de los lugares más fáciles de meter bugs de seguridad.
Cualquier producto serio usa managed auth en MVP.

### D. Solo email/password sin SSO

**Rechazada.** Fricción significativa para usuarios mobile que esperan
"continuar con Google/Apple". Apple además **exige** ofrecer Apple
Sign-In si la app tiene otros SSO (política de App Store).

### E. Supabase Auth + Firebase para FCM separado

**Rechazada.** Aunque técnicamente posible, mantener dos proyectos
Firebase + Supabase Auth agrega complejidad sin beneficio claro.

## Consecuencias

### Positivas

- Setup en horas, no semanas.
- Apple Sign-In nativo y maduro.
- Anonymous auth de primera clase (clave para fricción cero del trial).
- Account linking nativo.
- Brute force protection y password breach detection incluidos.
- 50.000 MAU gratis. A 100.000 MAU: ~$275/mes. Buena escalabilidad.
- Reuso de SDK con FCM (notifications).

### Negativas

- Dependencia de Google. Si cambia política o pricing, hay que migrar
  (auth es la capa más fácil pero igual hay trabajo).
- Doble identificador: Firebase UID + `users.id`. Sincronización
  requiere disciplina.
- Custom claims requieren admin SDK (no expuesto a cliente, pero
  manejable).

### Riesgos a monitorear

- Desincronización Firebase ↔ Postgres: monitorear casos donde el JWT
  tiene UID que no existe en `users`. Cron de reconciliación si pasa.
- Phone OTP: cuando se agregue, monitorear tasa de éxito por país (SMS
  en Latam tiene problemas ocasionales con carriers).
- Cambios de política de Firebase Auth (rare): revisar release notes
  trimestralmente.
- Pricing de Firebase Auth post 50k MAU: presupuestar y considerar
  migración si supera presupuesto.

## Referencias

- [`docs/architecture/authentication-system.md`](../architecture/authentication-system.md)
  — diseño detallado.
- [`docs/architecture/notifications-system.md`](../architecture/notifications-system.md)
  — Firebase también para FCM.
- [`docs/architecture/anti-fraud-system.md`](../architecture/anti-fraud-system.md)
  — usa identidad de Auth + fingerprinting.
- Firebase Auth: https://firebase.google.com/docs/auth.
- React Native Firebase: https://rnfirebase.io/auth/usage.

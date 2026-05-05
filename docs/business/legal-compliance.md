# Legal and Compliance

> Plan de cumplimiento legal: T&C, privacy policy, GDPR/LFPDPPP/Ley
> 25.326, requisitos de App Store/Play Store, manejo de datos de
> usuarios, política de fraude.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador + asesor legal cuando se
contrate.
**Alcance:** MVP y primeros 12 meses.

> ⚠️ **Disclaimer:** este documento NO es asesoría legal. Es un plan
> técnico de compliance. Antes del lanzamiento se requiere review por un
> abogado con expertise en privacy + consumer apps en Latam.

---

## 0. Cómo leer este documento

- §1 cubre los **frameworks legales** que aplican al producto.
- §2 cubre **Términos y Condiciones** (estructura mínima).
- §3 cubre **Privacy Policy** (estructura mínima + checklist).
- §4 cubre **derechos del usuario** y cómo implementarlos.
- §5 cubre **políticas internas** (refunds, cancelación, fraude).
- §6 cubre requisitos de **App Store / Play Store**.
- §7 cubre **disclosures específicos**: IA, audios, biometría.
- §8 cubre **menores de edad**.
- §9 cubre **tratamiento de datos para entrenar modelos propios**.

---

## 1. Frameworks legales aplicables

### 1.1 Por jurisdicción

| Jurisdicción | Framework | Aplicabilidad |
|--------------|-----------|---------------|
| México | LFPDPPP (Ley Federal de Protección de Datos Personales en Posesión de los Particulares) | Sí, mercado primario |
| Argentina | Ley 25.326 (Protección de Datos Personales) | Sí |
| Colombia | Ley 1581 / Estatutaria de Habeas Data | Sí |
| Chile | Ley 19.628 (en transición a nueva ley 21.719) | Sí |
| Perú | Ley 29.733 | Sí |
| EU (GDPR) | Regulation (EU) 2016/679 | Sí, si hay usuarios europeos |
| California (CCPA/CPRA) | California Consumer Privacy Act | Sí, si hay usuarios californianos |
| Brasil | LGPD (post-MVP cuando expandamos) | No en MVP |
| EEUU (general) | Sin framework federal unificado; estatales y sectoriales (COPPA si <13) | Sí (COPPA importante) |

### 1.2 Frameworks "by store"

| Framework | Aplica si... |
|-----------|--------------|
| Apple App Store Review Guidelines | App está en App Store |
| Google Play Developer Policy | App está en Play Store |
| Apple Sign In | Si ofrecemos otros SSO en iOS, debemos ofrecer Apple Sign In |

---

## 2. Términos y Condiciones (T&C)

### 2.1 Estructura mínima

```
1. Aceptación de los Términos
2. Descripción del Servicio
3. Cuenta de Usuario
   3.1 Requisitos para crear cuenta
   3.2 Responsabilidad del usuario
   3.3 Suspensión y cierre
4. Planes y Pagos
   4.1 Planes de suscripción
   4.2 Sistema de Sparks
   4.3 Compra de packs
   4.4 Renovación automática
   4.5 Cancelación y reembolsos
5. Uso Aceptable
   5.1 Conductas permitidas
   5.2 Conductas prohibidas
   5.3 Consecuencias de violación
6. Propiedad Intelectual
   6.1 Contenido de la plataforma
   6.2 Contenido del usuario (audios)
   6.3 Licencia que el usuario otorga
7. Servicios de IA
   7.1 Naturaleza de la asistencia con IA
   7.2 Limitaciones y errores posibles
   7.3 Sin garantía de aprendizaje
8. Comunicaciones
   8.1 Notificaciones
   8.2 Marketing (con opt-out)
9. Limitación de Responsabilidad
10. Indemnización
11. Modificaciones a los Términos
12. Resolución de Disputas
   12.1 Ley aplicable
   12.2 Jurisdicción
   12.3 Mediación / arbitraje
13. Disposiciones Generales
14. Contacto
```

### 2.2 Cláusulas clave a incluir

#### Sparks no son moneda

**Crítico** (de `architecture/sparks-system.md`):

- Sparks no son moneda de curso legal.
- Sparks no son transferibles entre usuarios.
- Sparks no se pueden convertir a dinero.
- La empresa puede ajustar el costo en Sparks de las operaciones con
  notice previo de 30 días.

#### Política de cuentas múltiples

- Una cuenta por persona.
- Detección automática puede aplicar restricciones (ver
  `anti-fraud-system`).
- Usuarios sospechados tienen derecho de apelación.

#### Cancelación

- El usuario puede cancelar en cualquier momento.
- La cancelación es **efectiva al final del ciclo actual** (no
  pro-rateo).
- Sparks comprados (packs) no se reembolsan al cancelar la suscripción.

#### Refunds

- Dentro de 14 días desde la compra: refund completo, sin preguntas.
- Después de 14 días: caso por caso.
- Sparks **consumidos** nunca son reembolsables (servicio prestado).
- En App Store / Play Store: refund según política de la store.

#### IA

- El producto usa IA (LLMs y modelos acústicos) para personalizar
  aprendizaje.
- La IA puede cometer errores en feedback o scoring.
- El usuario es responsable de su aprendizaje; la app es una
  herramienta.

#### Audios

- El usuario otorga licencia limitada para que procesemos sus audios
  con el propósito de evaluar su aprendizaje.
- Audios crudos se retienen máximo 30 días.
- El usuario puede optar (o no) por compartir audios anonimizados para
  mejorar nuestros modelos. Default: SÍ (con disclosure claro y
  opción de revocar).

### 2.3 Ley aplicable y jurisdicción

Para una empresa con sede en (asumiendo) México:

- **Ley aplicable:** leyes de México (sin perjuicio de leyes locales
  del consumidor que sean de orden público en otros países).
- **Jurisdicción:** tribunales de Ciudad de México (con excepción de
  derechos de consumidor que las leyes locales otorgan).

Si la empresa tiene sede diferente, ajustar.

### 2.4 Cuando cambian los T&C

- Notificar a usuarios activos por email + in-app banner con 30 días
  de notice.
- Cambios sustanciales requieren re-aceptación explícita.
- Cambios menores (correcciones de redacción): no requieren
  re-aceptación, pero se notifica.

---

## 3. Privacy Policy

### 3.1 Estructura mínima

```
1. Introducción
   1.1 Quiénes somos (datos del responsable del tratamiento)
   1.2 Cómo contactarnos (email privacy@<dominio>)
2. Datos que recolectamos
   2.1 Datos que tú nos das directamente
   2.2 Datos que recolectamos automáticamente
   2.3 Datos de terceros (Google, Apple OAuth)
3. Cómo usamos tus datos
   3.1 Para proveerte el servicio
   3.2 Para personalizar tu experiencia
   3.3 Para mejorar nuestros modelos de IA
   3.4 Para comunicarnos contigo
4. Cómo compartimos tus datos
   4.1 Proveedores de servicios (sub-procesadores)
   4.2 Requisitos legales
   4.3 Transferencias de negocio
5. Tus derechos
   5.1 Acceso, rectificación, cancelación, oposición (ARCO)
   5.2 Portabilidad
   5.3 Borrado (right to erasure)
   5.4 Cómo ejercer estos derechos
6. Retención de datos
7. Seguridad
8. Transferencias internacionales
9. Datos de menores
10. Cookies y tecnologías similares
11. Cambios a esta política
12. Contacto y autoridad de protección de datos
```

### 3.2 Datos que declaramos

#### Datos que el usuario nos da directamente

- Email.
- Nombre (opcional).
- Foto de perfil (opcional).
- Respuestas del onboarding (objetivos, contexto profesional, nivel
  autopercibido, etc.).
- Audios grabados en ejercicios.
- Mensajes a soporte.
- Información de pago: **NO** la procesamos directamente, va a
  Stripe/RevenueCat.

#### Datos que recolectamos automáticamente

- Identificadores de device (Firebase UID, device fingerprint).
- IP address y país detectado.
- Timezone, idioma del device.
- Versión del OS, modelo del device, app version.
- Eventos de uso (qué bloques completa, scores, tiempo en la app).
- Cookies / local storage (en web).

#### Datos de terceros

- Si el usuario usa Google Sign In: email, nombre, foto (según permisos
  otorgados a Google).
- Si usa Apple Sign In: email (real o relay), nombre.

### 3.3 Sub-procesadores que declaramos

Lista pública de quién más procesa los datos del usuario:

| Sub-procesador | Para qué | Donde procesa |
|----------------|----------|---------------|
| Google (Firebase) | Auth, FCM | EEUU/EU |
| Supabase (Postgres) | Storage de datos del usuario | US-East |
| Cloudflare | Workers, R2, KV | Global edge |
| Anthropic | LLM para feedback de aprendizaje | EEUU |
| Google (Gemini) | LLM fallback | EEUU |
| OpenAI | LLM fallback | EEUU |
| Microsoft (Azure) | Pronunciation scoring | EEUU |
| Stripe | Procesamiento de pagos web | EEUU |
| RevenueCat | Procesamiento de pagos in-app | EEUU |
| MercadoPago | Pagos locales Latam | Latam |
| PostHog | Analytics de producto | EEUU/EU |
| Sentry | Error tracking | EEUU |
| Inngest | Event bus | EEUU |
| ElevenLabs (post-MVP) | TTS para contenido pre-generado | EEUU |
| Deepgram (post-MVP) | STT alternativo | EEUU |

Mantener esta lista **actualizada** y publicada en el sitio web.

### 3.4 Cookies (web)

Para la landing y eventual web app:

- **Estrictamente necesarias:** sesión, CSRF. Sin consent banner por
  estas.
- **Analytics:** PostHog. Requiere consent en EU.
- **Marketing:** ninguna en MVP.

Banner de cookies activable por geolocalización (EU = mostrar; Latam
= no obligatorio pero recomendable como buena práctica).

---

## 4. Derechos del usuario y su implementación

### 4.1 Derechos garantizados (mínimo común GDPR + LFPDPPP + Ley 25.326)

#### Acceso

**Mecanismo:** export en JSON desde Settings → "Mis datos".

**Implementación:**
- Endpoint `GET /me/data-export` que genera un ZIP con:
  - `profile.json`: datos de `users`, `student_profiles`.
  - `mastery.json`: scores actuales y histórico.
  - `attempts.json`: ejercicios realizados (sin audios crudos —
    referencias).
  - `sparks.json`: balance e historial.
  - `tickets.json`: tickets de soporte.
  - `audios/`: signed URLs (TTL 7 días) para descargar audios crudos.
- Generación asíncrona si > 100MB; email con link cuando esté listo.

#### Rectificación

**Mecanismo:** UI directa para campos editables (nombre, foto, país,
preferencias de notificación, idioma target). Para campos no editables
(email del proveedor SSO): contactar a soporte.

#### Cancelación / Borrado

**Mecanismo:** Settings → "Eliminar cuenta".

Flujo (de `architecture/authentication-system.md` §8.3):
1. Confirmación con explicación de qué se borra.
2. Período de gracia 30 días (con opción de cancelar).
3. Hard delete: cascade en Postgres + delete en Firebase + R2 +
   PostHog + Sentry.

#### Oposición

**Mecanismo:** opt-outs específicos en Settings:

- Notificaciones de marketing (default off, opt-in).
- Personalización por behavior (default on, opt-out impacta calidad
  del producto).
- Compartir datos anonimizados para investigación lingüística (default
  off, opt-in).
- Procesamiento de audios para entrenar modelos propios (default on,
  opt-out limita mejora futura).

#### Portabilidad

Mismo endpoint que Acceso (§4.1.1): JSON export.

### 4.2 Cómo se ejerce cada derecho

| Derecho | Cómo se ejerce | Tiempo de respuesta |
|---------|---------------|---------------------|
| Acceso | Self-service en Settings | Inmediato (export en < 5 min) |
| Rectificación campos editables | Self-service | Inmediato |
| Rectificación campos no editables | Email a `privacy@<dominio>` | < 30 días |
| Borrado | Self-service en Settings | 30 días grace + 24h hard delete |
| Oposición | Self-service en Settings | Inmediato |
| Portabilidad | Self-service en Settings | Inmediato (mismo que Acceso) |

### 4.3 Privacy email

`privacy@<dominio>` para:
- Solicitudes que no se pueden completar self-service.
- Reclamos formales.
- Inquietudes generales sobre privacy.

Debe tener proceso documentado de respuesta:
1. Acuse de recibo en 24h.
2. Verificación de identidad (responder desde el email registrado).
3. Resolución en < 30 días.
4. Log interno con resolución para auditoría.

---

## 5. Políticas internas

### 5.1 Refunds

(De `business/customer-support-system.md` §9.1):

- 14 días desde la compra: refund completo, sin preguntas.
- Después de 14 días: caso por caso, default = no.
- Sparks consumidos: nunca reembolsables.
- En App Store / Play Store: las stores manejan según sus políticas
  (usualmente más generosas); no podemos override.

### 5.2 Cancelación

- Self-service en Settings.
- Efectiva al final del ciclo de facturación actual.
- Sin pro-rateo del mes en curso.
- El usuario mantiene acceso hasta el fin del ciclo pagado.
- Sparks comprados (packs) no se ven afectados; siguen vigentes hasta
  6 meses.

### 5.3 Suspensión / Ban (anti-fraud)

(De `architecture/anti-fraud-system.md` §5):

- Niveles 1–3 (warning, friction, restricciones funcionales) son
  automáticos.
- Niveles 4–5 (suspensión temporal, ban permanente) requieren revisión
  humana.
- Todos los casos tienen path de apelación.
- Decisión final del ban: documentada con razón en `user_restrictions`.

### 5.4 Comunicación al usuario

Cuando se aplica una restricción/suspensión:

```
Asunto: Actividad inusual detectada en tu cuenta

Hola,

Detectamos actividad que viola nuestros Términos de Uso (sección 5.2).
Como resultado, hemos aplicado [restricción específica].

Si crees que esto es un error, puedes apelar respondiendo a este email
o desde [link a apelación].

Equipo de [Producto]
```

Tono: factual, sin acusación pública, claro sobre cómo apelar.

---

## 6. Requisitos de App Store / Play Store

### 6.1 Apple App Store

**Requisitos críticos:**

- [ ] **Privacy Nutrition Labels:** declaración detallada en App Store
  Connect de qué datos se recolectan, cómo se usan, si se linkean a
  identidad.
- [ ] **Account deletion in-app:** obligatorio (implementado en
  `authentication-system.md` §8.3).
- [ ] **Sign in with Apple:** obligatorio si ofrecemos otros SSO.
- [ ] **App Tracking Transparency (ATT):** prompt si trackeamos al
  usuario para fines de publicidad cross-app. **No aplica** porque no
  hacemos advertising tracking.
- [ ] **Política de IA generativa:** disclosure claro de que usamos IA;
  filtrar contenido objetable; permitir reportar contenido inapropiado.
- [ ] **In-app purchases vía Apple:** todas las suscripciones in-app
  van por Apple. Pueden coexistir con web payments (15% descuento web
  permitido en muchas jurisdicciones — verificar al implementar).

### 6.2 Google Play Store

**Requisitos críticos:**

- [ ] **Data safety form:** declaración equivalente a Apple labels.
- [ ] **Account deletion in-app:** obligatorio.
- [ ] **Target API level actualizado:** compilar contra el último
  Android API obligatorio.
- [ ] **In-app purchases vía Google Play Billing:** suscripciones
  in-app vía Google.
- [ ] **Política de IA generativa:** similar a Apple.

### 6.3 Apple ATT (App Tracking Transparency)

No usamos tracking cross-app. **No** mostraremos prompt ATT salvo que
introduzcamos features que requieran tracking de terceros (no planeado).

### 6.4 Reviews y respuestas

- Responder a reseñas de App Store / Play Store en español.
- Tono: profesional, no defensivo.
- Para reseñas con bug claro: ofrecer ayuda + crear ticket de soporte.
- Para reseñas con queja de pricing: explicar valor sin disculparse del
  precio.

---

## 7. Disclosures específicos

### 7.1 Uso de IA

Visible en onboarding y privacy policy:

> Usamos inteligencia artificial (modelos de lenguaje y de procesamiento
> de audio) para personalizar tu plan de aprendizaje, evaluar tu
> pronunciación y generar feedback. La IA puede cometer errores; usa tu
> juicio. No reemplaza un profesor humano si estás preparándote para un
> examen oficial.

### 7.2 Captura de audios

Antes del primer ejercicio que requiere micrófono, prompt explícito:

> Para evaluar tu pronunciación, necesitamos grabar audios cortos cuando
> hablas en la app. Estos audios:
>
> - Se almacenan encriptados.
> - Se borran automáticamente después de 30 días.
> - Pueden usarse para mejorar nuestros modelos de IA, en forma
>   anonimizada (puedes desactivar esto en Settings).
>
> [Aceptar y continuar] [No, gracias]

### 7.3 Biometric data

En algunas jurisdicciones, audios de voz se consideran datos biométricos
con tratamiento especial. Privacy policy debe mencionar:

> Reconocemos que tu voz es información biométrica sensible. Por eso:
>
> - Audios crudos se retienen máximo 30 días.
> - No vendemos ni compartimos tus audios con terceros (salvo
>   sub-procesadores explícitamente listados en §X).
> - No usamos tus audios para identificarte (no hay "voice ID").
> - El propósito único es evaluar tu aprendizaje del inglés.

### 7.4 Modelos propios entrenados con audios

Si el usuario optó in (`opt_in_training` = true en preferences):

> Tus audios anonimizados pueden incluirse en datasets para entrenar
> nuestros propios modelos de evaluación. Esto significa:
>
> - Removemos tu identidad de los audios antes de incluirlos.
> - Los datasets se mantienen internos; no se publican.
> - Podés revocar este consentimiento en Settings; los audios futuros
>   no se incluirán (los ya incluidos no se pueden retirar de modelos
>   ya entrenados, pero sí se pueden remover de datasets futuros).

---

## 8. Menores de edad

### 8.1 Edad mínima

**Decisión:** edad mínima **13 años**, con verificación de mayoría de
edad legal según jurisdicción para consentimiento de tratamiento de
datos:

- COPPA (EEUU): < 13 prohibido sin consent parental documentado.
- GDPR (EU): 13–16 según país (default 16, algunos países permiten 13).
- México (LFPDPPP): mayoría de edad 18, pero consent parental hasta
  entonces es estándar.

**Decisión MVP:** edad mínima 13. Onboarding pregunta fecha de
nacimiento con validación. Si < 13: rechazo de signup.

### 8.2 Disclosure en stores

- Apple App Store: rating "4+" (apto para todos pero con guidance
  parental sugerida).
- Google Play: "Everyone" o "Teen" según contenido.

### 8.3 Consent parental (post-MVP)

Para versión "kids" o expansión a 9–12 años:
- Doble opt-in con email parental verificado.
- COPPA compliance: no recolectar más datos que los necesarios.
- Sin advertising ni tracking.

---

## 9. Tratamiento de datos para entrenar modelos propios

### 9.1 Plan futuro

`pedagogical-system.md` §3.1 y `content-creation-system.md` mencionan
que entrenaremos modelos propios de pronunciation scoring con audios
anonimizados.

### 9.2 Anonimización

Antes de incluir un audio en dataset de entrenamiento:

- Remover `user_id` y cualquier identificador directo.
- Etiquetar con metadata mínima necesaria: dialecto del español del
  hablante (si declarado), edad range (si declarado), país, género
  (si declarado), variante de inglés target.
- Audio crudo se mantiene; transcripción anonimizada de cualquier dato
  personal (nombres propios, direcciones, teléfonos).

### 9.3 Consent

Default: **opt-in al onboarding** (preguntando explícitamente). Si el
usuario rechaza, no se usan sus audios para entrenamiento (sí para
servicio inmediato).

### 9.4 Retiro de consent

- Cambio de preferencia en Settings: efectivo desde ese momento.
- Audios ya incluidos en datasets de entrenamiento permanecen (no se
  pueden retirar de modelos ya entrenados, pero se excluyen de futuros
  datasets).
- Esta limitación se declara explícitamente en la pantalla de consent.

---

## 10. Plan de implementación legal

### 10.1 Pre-MVP (Sprint 0–2)

- [ ] Contratar revisión legal de T&C y Privacy Policy.
- [ ] Publicar T&C y Privacy Policy en website.
- [ ] Implementar account deletion in-app.
- [ ] Implementar export de datos.
- [ ] Configurar `privacy@<dominio>`.
- [ ] Configurar `security@<dominio>` para reportes.
- [ ] Privacy Nutrition Labels (Apple) + Data Safety (Google) en
  preparación.

### 10.2 MVP (Sprint 3–6)

- [ ] Privacy Nutrition Labels publicadas.
- [ ] Cookie banner en web (si hay tracking analytics).
- [ ] Onboarding con consents granulares.
- [ ] Disclosure de IA visible.
- [ ] Opt-ins editables en Settings.
- [ ] Proceso documentado de respuesta a privacy@.

### 10.3 Post-MVP

- [ ] Registro ante AAIP (Argentina) si aplicable.
- [ ] Revisión legal anual.
- [ ] Bug bounty público.

---

## 11. Costos y recursos

| Item | Costo estimado |
|------|----------------|
| Revisión legal inicial T&C + Privacy | $1.500–$5.000 USD (one-time) |
| Revisión anual | $500–$1.500 USD/año |
| Proceso de notificación de breach (si ocurre) | Variable |
| Asesoría continua | Retainer mensual o por hora |

---

## 12. Riesgos legales

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|-----------|
| Demanda por uso indebido de audios | Baja | Alto | Disclosure claro, retención corta, opt-out fácil |
| Breach de datos con notificación tardía | Baja | Crítico | Runbook de incident response, tooling de detección |
| Reclamo por modelo IA generando contenido inapropiado | Media | Medio | Output filtering, system prompts con guardrails, reportar fácil |
| Regulación nueva (ej: AI Act EU) | Media | Medio | Monitoreo trimestral, asesoría legal |
| Usuario menor que mintió sobre edad | Alta | Bajo–Medio | Validación al signup, ban si se descubre |
| Cobro a usuario después de cancelación | Baja | Medio | Tests de integración del flow de cancellation |
| Datos compartidos con sub-procesador no listado | Baja | Crítico | Lista pública mantenida, code review de cualquier nueva integración externa |

---

## 13. Referencias internas

- [`../architecture/authentication-system.md`](../architecture/authentication-system.md) — account deletion flow.
- [`../architecture/sparks-system.md`](../architecture/sparks-system.md) — refund policy de Sparks.
- [`../architecture/anti-fraud-system.md`](../architecture/anti-fraud-system.md) — políticas de ban + apelación.
- [`../cross-cutting/security-threat-model.md`](../cross-cutting/security-threat-model.md) §6 — compliance overview.
- [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §7 — retención y privacidad.
- [`customer-support-system.md`](customer-support-system.md) §9 — refunds, account deletion, GDPR requests.

---

## 14. Recursos externos

- LFPDPPP México: https://www.diputados.gob.mx/LeyesBiblio/pdf/LFPDPPP.pdf
- Ley 25.326 Argentina: https://www.argentina.gob.ar/normativa/nacional/ley-25326-64790
- GDPR: https://gdpr.eu/
- COPPA: https://www.ftc.gov/business-guidance/privacy-security/childrens-privacy
- Apple App Review Guidelines: https://developer.apple.com/app-store/review/guidelines/
- Google Play Developer Policy: https://play.google.com/about/developer-content-policy/

---

*Documento vivo. Revisar al menos cada 12 meses, después de cualquier
incidente de privacidad, y al introducir features que toquen datos
sensibles. **No es asesoría legal.**.*

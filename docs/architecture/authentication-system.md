# Authentication System

> Sistema de autenticación basado en Firebase Auth con SSO multi-proveedor.
> Diseñado para minimizar fricción en onboarding mientras mantiene
> seguridad robusta y recovery confiable.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Alcance:** MVP y primeros 18 meses

---

## 1. Objetivos del sistema

### 1.1 Objetivos primarios

- Reducir el tiempo entre "abrir la app" y "primer ejercicio" a menos
  de 30 segundos para la mayoría de usuarios.
- Cumplir requisitos de App Store (Apple Sign In si hay otros SSO).
- Cubrir el 95%+ de usuarios potenciales con métodos disponibles.
- Permitir account recovery confiable (los usuarios tienen Sparks
  comprados y progreso valioso).

### 1.2 Objetivos secundarios

- Permitir prueba sin registro ("anonymous auth") para reducir fricción
  de exploración inicial.
- Soporte multi-device desde el día uno.
- Account linking cuando un usuario se autentica con varios métodos.
- Cumplimiento con regulaciones de privacidad (GDPR, Ley Federal de
  Protección de Datos en México, Ley 25.326 en Argentina).

---

## 2. Decisión: Firebase Authentication

### 2.1 Por qué Firebase Auth

| Criterio | Firebase Auth | Supabase Auth | Auth0 | Roll your own |
|----------|--------------|---------------|-------|---------------|
| Costo MVP | Gratis | Gratis | Caro | "Gratis" pero alto costo en bugs |
| SSO móvil maduro | Excelente | Bueno | Excelente | Inviable |
| Apple Sign In | Nativo | Nativo (más nuevo) | Nativo | Implementación compleja |
| Phone OTP | Maduro | Bueno | Maduro | Inviable |
| Anonymous auth | Sí | Limitado | Sí | Inviable |
| Account linking | Excelente | Bueno | Excelente | Difícil |
| Integración con stack | Aprovecha FCM existente | Aprovecha Postgres | Independiente | N/A |
| **Recomendación** | **✅ Elegido** | Alternativa válida | Overkill para MVP | Mala idea |

### 2.2 Por qué no Supabase Auth aunque ya tenemos Supabase

Tensión real considerada: Supabase Auth se integra perfectamente con
Postgres y tiene Row Level Security (RLS) basada en JWT.

Pero las ventajas de Firebase Auth en SSO móvil pesan más:

- Apple Sign In en Firebase está significativamente más maduro.
- Phone OTP en Firebase tiene mejor cobertura de carriers en Latam.
- El stack ya incluye Firebase para FCM, agregar Auth tiene costo marginal cero.
- La sincronización Firebase → Postgres es relativamente simple (webhook).

### 2.3 Trade-off aceptado

La principal desventaja: tener dos identificadores de usuario (Firebase UID
y un internal user_id en Postgres) y mantener sincronización entre ambos.

Mitigación: webhook de Firebase Auth (Cloud Function o Cloudflare Worker)
que crea/actualiza el registro en Postgres cuando hay cambios. Sección 7
detalla la implementación.

---

## 3. Proveedores de identidad

### 3.1 Proveedores soportados en MVP

| Proveedor | Plataformas | Prioridad | % esperado de usuarios |
|-----------|------------|-----------|------------------------|
| Google Sign In | iOS + Android | Alta | 50-60% |
| Apple Sign In | iOS (obligatorio) | Alta | 15-25% |
| Email + Password | Todas | Alta | 20-30% |
| Phone OTP | Todas | Opcional MVP | 5-15% si se incluye |

### 3.2 Justificación por proveedor

**Google Sign In (crítico):**
- Cubre el método de login más usado en Android.
- Funciona también en iOS para usuarios que prefieren cuenta Google.
- Onboarding más rápido (1 tap si ya logueado en device).
- Datos básicos pre-verificados (email, nombre).

**Apple Sign In (obligatorio en iOS):**
- Política de App Store: si la app ofrece otros SSO, debe ofrecer Apple Sign In.
- Usuarios iOS conscientes de privacidad lo prefieren.
- Permite ocultar email real (Hide My Email).
- Onboarding en 1 tap con FaceID/TouchID.

**Email + Password (necesario como fallback):**
- Cubre casos donde SSO falla o usuario prefiere desacoplarse de Google/Apple.
- Importante para usuarios que cambian de teléfono frecuentemente.
- Único método si no hay smartphone moderno (web app futura).

**Phone OTP (opcional MVP):**
- Muy aceptado en Latam por la cultura WhatsApp.
- Onboarding simple (no necesita recordar contraseña).
- Pero: tiene costo por SMS (~$0.01-0.05 por mensaje en Latam).
- A escala puede sumar significativamente.
- **Decisión:** evaluar incluirlo en MVP o agregar en mes 3-6 según fricción
  observada en métricas reales.

### 3.3 Proveedores explícitamente descartados para MVP

- **Facebook Login:** target no se identifica con Facebook para apps de
  educación, complejidad sin valor.
- **Twitter/X, LinkedIn, GitHub:** no relevantes para mercado consumer.
- **Microsoft:** posible para B2B año 2+, no MVP.
- **Magic links** (login solo por email sin password): UX confuso para
  algunos usuarios, agrega fricción a recovery.

---

## 4. Estrategia de UX de autenticación

### 4.1 Pantalla principal de login

Diseño recomendado, en orden vertical:

```
┌─────────────────────────────────────┐
│           [Logo de la app]          │
│                                     │
│   "Empezá a hablar inglés hoy"      │
│                                     │
│   ┌───────────────────────────────┐ │
│   │  🔍 Continuar con Google      │ │  <- destacado (botón principal)
│   └───────────────────────────────┘ │
│                                     │
│   ┌───────────────────────────────┐ │
│   │   Continuar con Apple         │ │  <- iOS only
│   └───────────────────────────────┘ │
│                                     │
│   ──────────  o  ──────────         │
│                                     │
│   📧  Continuar con email           │  <- secundario
│                                     │
│   📱  Continuar con teléfono        │  <- si phone OTP habilitado
│                                     │
│   "¿Querés probar primero?"         │
│   [Empezar sin registrarme]          │  <- anonymous auth
│                                     │
└─────────────────────────────────────┘
```

### 4.2 Anonymous auth como feature de conversión

Permitir que el usuario use la app sin registro durante un período
limitado:

- Anonymous auth desde primer tap (Firebase soporta nativamente).
- Acceso completo a la app durante 7 días o 5 sesiones.
- Progreso, Sparks gratuitos y datos quedan asociados al usuario anónimo.
- Después del límite o al primer pago, prompt de "guardá tu progreso"
  que upgrade a usuario permanente sin perder datos.

Beneficios:

- Reduce fricción de "tengo que registrarme antes de saber si me sirve".
- Mejora conversion al primer uso.
- Cuando el usuario está convencido, hace upgrade a permanente sin trauma.

Riesgos a mitigar:

- Anti-abuso: rate limit por device/IP para evitar farming de pruebas.
- Cleanup: usuarios anónimos que nunca convierten se eliminan después
  de 30 días para no inflar la base.

### 4.3 Account linking

Manejo de casos donde un usuario:

- Se registra con Google.
- Después intenta logear con email/password usando el mismo email.

Comportamiento:

- Sistema detecta email duplicado.
- Pregunta al usuario: "Ya tenés cuenta con este email, vinculada a Google.
  ¿Querés agregar email/password como método adicional de acceso?"
- Si confirma, linkea las cuentas (Firebase soporta nativamente).
- Resultado: una sola cuenta con dos métodos de acceso.

### 4.4 Account recovery

Flujos de recuperación según método:

**Email + password:**
- Reset por email con magic link de un solo uso.
- Token expira en 1 hora.
- Mensaje claro: "verificá spam si no lo ves".

**Google/Apple SSO:**
- Recovery se delega al proveedor (Google/Apple).
- Mensaje: "ingresá con Google/Apple para recuperar acceso".

**Phone OTP:**
- Reset por SMS al mismo número.
- Si el usuario perdió el número, fallback a "contactanos por email".

**Pérdida total de acceso:**
- Email a soporte con verificación manual.
- Importante para usuarios con Sparks comprados o progreso significativo.
- Considerar verificación de pagos previos (RevenueCat, Stripe) como prueba
  de propiedad de cuenta.

---

## 5. Flujo técnico de autenticación

### 5.1 Setup en cliente (React Native + Expo)

Dependencias:

```json
{
  "@react-native-firebase/app": "...",
  "@react-native-firebase/auth": "...",
  "@react-native-google-signin/google-signin": "...",
  "@invertase/react-native-apple-authentication": "..."
}
```

Importante: requiere development build con Expo, no funciona con Expo Go.

### 5.2 Flujo de Google Sign In

```typescript
import auth from '@react-native-firebase/auth';
import { GoogleSignin } from '@react-native-google-signin/google-signin';

GoogleSignin.configure({
  webClientId: 'xxx.apps.googleusercontent.com',
  offlineAccess: false
});

async function signInWithGoogle() {
  try {
    // 1. Obtener idToken de Google
    await GoogleSignin.hasPlayServices();
    const { idToken } = await GoogleSignin.signIn();

    // 2. Crear credencial de Firebase
    const googleCredential = auth.GoogleAuthProvider.credential(idToken);

    // 3. Autenticar con Firebase
    const userCredential = await auth().signInWithCredential(googleCredential);

    // 4. Si es usuario nuevo, sincronizar a Postgres
    if (userCredential.additionalUserInfo?.isNewUser) {
      await api.post('/auth/sync-user', {
        firebaseUid: userCredential.user.uid,
        email: userCredential.user.email,
        displayName: userCredential.user.displayName,
        provider: 'google'
      });
    }

    return userCredential.user;
  } catch (error) {
    handleAuthError(error);
  }
}
```

### 5.3 Flujo de Apple Sign In

```typescript
import { appleAuth } from '@invertase/react-native-apple-authentication';

async function signInWithApple() {
  try {
    // 1. Solicitar credenciales de Apple
    const appleAuthRequestResponse = await appleAuth.performRequest({
      requestedOperation: appleAuth.Operation.LOGIN,
      requestedScopes: [appleAuth.Scope.EMAIL, appleAuth.Scope.FULL_NAME]
    });

    // 2. Obtener identityToken
    const { identityToken, nonce } = appleAuthRequestResponse;
    if (!identityToken) throw new Error('Apple Sign-In failed');

    // 3. Crear credencial Firebase
    const appleCredential = auth.AppleAuthProvider.credential(
      identityToken,
      nonce
    );

    // 4. Autenticar
    const userCredential = await auth().signInWithCredential(appleCredential);

    // 5. Sincronizar (Apple solo da fullName la primera vez)
    if (userCredential.additionalUserInfo?.isNewUser) {
      await api.post('/auth/sync-user', {
        firebaseUid: userCredential.user.uid,
        email: userCredential.user.email,
        displayName: appleAuthRequestResponse.fullName?.givenName,
        provider: 'apple'
      });
    }

    return userCredential.user;
  } catch (error) {
    handleAuthError(error);
  }
}
```

### 5.4 Flujo de Email + Password

```typescript
// Registro
async function signUpWithEmail(email: string, password: string, name: string) {
  const userCredential = await auth().createUserWithEmailAndPassword(email, password);

  // Actualizar perfil con nombre
  await userCredential.user.updateProfile({ displayName: name });

  // Enviar email de verificación
  await userCredential.user.sendEmailVerification();

  // Sincronizar a Postgres
  await api.post('/auth/sync-user', {
    firebaseUid: userCredential.user.uid,
    email: email,
    displayName: name,
    provider: 'email',
    emailVerified: false
  });
}

// Login
async function signInWithEmail(email: string, password: string) {
  const userCredential = await auth().signInWithEmailAndPassword(email, password);
  return userCredential.user;
}
```

### 5.5 Anonymous auth

```typescript
async function signInAnonymously() {
  const userCredential = await auth().signInAnonymously();

  await api.post('/auth/sync-user', {
    firebaseUid: userCredential.user.uid,
    provider: 'anonymous',
    isAnonymous: true
  });
}

// Upgrade a permanente
async function upgradeAnonymousAccount(email: string, password: string) {
  const credential = auth.EmailAuthProvider.credential(email, password);
  const userCredential = await auth().currentUser?.linkWithCredential(credential);

  // Actualizar registro en Postgres: ya no es anónimo
  await api.patch('/auth/upgrade-anonymous', {
    firebaseUid: userCredential.user.uid,
    email: email,
    provider: 'email'
  });
}
```

---

## 6. Sincronización Firebase ↔ Postgres

### 6.1 Schema en Postgres

```sql
CREATE TABLE users (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  firebase_uid      TEXT NOT NULL UNIQUE,
  email             TEXT,
  email_verified    BOOLEAN NOT NULL DEFAULT false,
  display_name      TEXT,
  photo_url         TEXT,
  primary_provider  TEXT NOT NULL CHECK (primary_provider IN (
    'google', 'apple', 'email', 'phone', 'anonymous'
  )),
  is_anonymous      BOOLEAN NOT NULL DEFAULT false,
  linked_providers  TEXT[] NOT NULL DEFAULT '{}',
  created_at        TIMESTAMPTZ DEFAULT now(),
  updated_at        TIMESTAMPTZ DEFAULT now(),
  last_login_at     TIMESTAMPTZ DEFAULT now(),
  deleted_at        TIMESTAMPTZ  -- soft delete
);

CREATE INDEX idx_users_firebase_uid ON users(firebase_uid) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_anonymous ON users(is_anonymous, created_at)
  WHERE is_anonymous = true AND deleted_at IS NULL;
```

### 6.2 Estrategias de sincronización

**Estrategia A: Cliente notifica al backend (recomendada para MVP)**

Después de cada autenticación exitosa, el cliente llama a un endpoint
del backend que crea o actualiza el registro en Postgres.

Ventajas: simple, no requiere setup adicional, control completo.
Desventajas: depende de que el cliente haga la llamada (si falla, queda
desincronizado).

Mitigación: idempotencia en el endpoint de sync. Si se llama múltiples
veces con los mismos datos, no causa problemas.

```typescript
// Endpoint /auth/sync-user
export async function syncUser(req: Request) {
  const { firebaseUid, email, displayName, provider } = await req.json();

  // Verificar que el JWT corresponde al firebaseUid
  const decodedToken = await verifyFirebaseToken(req.headers.authorization);
  if (decodedToken.uid !== firebaseUid) {
    return new Response('Unauthorized', { status: 401 });
  }

  // Upsert en Postgres
  await db.query(`
    INSERT INTO users (firebase_uid, email, display_name, primary_provider, is_anonymous)
    VALUES ($1, $2, $3, $4, $5)
    ON CONFLICT (firebase_uid) DO UPDATE
      SET email = EXCLUDED.email,
          display_name = COALESCE(EXCLUDED.display_name, users.display_name),
          last_login_at = now(),
          linked_providers = array_append_unique(users.linked_providers, EXCLUDED.primary_provider),
          updated_at = now()
  `, [firebaseUid, email, displayName, provider, provider === 'anonymous']);

  return new Response('OK');
}
```

**Estrategia B: Firebase Function trigger (opcional para robustez)**

Cloud Function que se dispara en evento `onCreate` de Firebase Auth.
Crea automáticamente el registro en Postgres sin depender del cliente.

Ventajas: garantía de sincronización aunque el cliente falle.
Desventajas: complejidad adicional, costo mínimo de Cloud Functions.

**Recomendación:** Estrategia A para MVP, evaluar agregar B si se observan
casos de desincronización.

### 6.3 Validación de tokens en backend

Cualquier llamada autenticada al backend debe validar el JWT de Firebase:

```typescript
// utils/verifyFirebaseToken.ts
import { jwtVerify, createRemoteJWKSet } from 'jose';

const JWKS = createRemoteJWKSet(
  new URL('https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com')
);

export async function verifyFirebaseToken(authHeader: string | null) {
  if (!authHeader?.startsWith('Bearer ')) throw new Error('No token');

  const token = authHeader.slice(7);
  const { payload } = await jwtVerify(token, JWKS, {
    issuer: `https://securetoken.google.com/${FIREBASE_PROJECT_ID}`,
    audience: FIREBASE_PROJECT_ID
  });

  return payload;  // contiene uid, email, etc.
}
```

Esto se puede ejecutar eficientemente en Cloudflare Workers (jose es
edge-compatible).

---

## 7. Sesiones y manejo de tokens

### 7.1 Flujo de tokens

Firebase Auth maneja automáticamente:

- **idToken**: JWT corto (1 hora de validez), se envía en cada request.
- **refreshToken**: token largo, usado para renovar idToken.

El SDK de Firebase refresca automáticamente el idToken antes de expirar.

### 7.2 Cliente

```typescript
// Obtener token actual para llamadas API
const idToken = await auth().currentUser?.getIdToken();

// En interceptor de Axios o fetch wrapper
const apiClient = axios.create({
  baseURL: API_URL
});

apiClient.interceptors.request.use(async (config) => {
  const token = await auth().currentUser?.getIdToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

### 7.3 Logout

```typescript
async function signOut() {
  // 1. Sign out de proveedores externos
  if (await GoogleSignin.isSignedIn()) {
    await GoogleSignin.signOut();
  }

  // 2. Sign out de Firebase
  await auth().signOut();

  // 3. Limpiar estado local de la app
  await clearLocalState();
}
```

### 7.4 Sesiones multi-device

Firebase soporta sesiones simultáneas en múltiples devices nativamente.
Cada device tiene su propio refresh token.

Política de sesiones:

- Sin límite de sesiones simultáneas para usuarios standard.
- Cambio de password invalida todas las sesiones (security best practice).
- Posibilidad futura de "ver sesiones activas" y cerrar individualmente.

---

## 8. Privacidad y datos

### 8.1 Datos mínimos guardados

Filosofía: pedir solo lo estrictamente necesario.

| Dato | ¿Se guarda? | Justificación |
|------|------------|---------------|
| Email | Sí | Identificador, recovery, comunicación |
| Nombre | Sí (opcional) | Personalización en notificaciones y UI |
| Foto de perfil URL | Sí (opcional) | UI de perfil |
| Firebase UID | Sí | Sincronización con Auth |
| Provider usado | Sí | UX de "ingresá con el método con el que te registraste" |
| Fecha de nacimiento | NO | No necesario para el producto |
| Género | NO | No necesario |
| Teléfono | Solo si usa Phone OTP | Necesario para método de auth |
| Dirección | NO | No aplica |

### 8.2 Cumplimiento con regulaciones

**GDPR (si tenés usuarios europeos):**
- Consentimiento explícito en privacy policy.
- Derecho al borrado (account deletion completa).
- Portabilidad de datos (export en JSON).

**Ley Federal de Protección de Datos (México):**
- Aviso de privacidad accesible.
- Mecanismos para ejercer derechos ARCO.

**Ley 25.326 (Argentina):**
- Registro ante AAIP si se procesan datos personales.
- Consentimiento informado.

### 8.3 Account deletion

Funcionalidad obligatoria por políticas de App Store y Play Store:
permitir al usuario eliminar su cuenta desde la app misma.

Flujo:

1. Usuario solicita deletion en settings.
2. Confirmación con explicación de qué se borra (Sparks, progreso, certificados).
3. Período de gracia de 30 días donde la cuenta queda en soft-delete.
4. Usuario puede cancelar deletion durante el período de gracia.
5. Después de 30 días: hard delete de Firebase + Postgres + S3 (audios).

```typescript
async function requestAccountDeletion(userId: string) {
  // 1. Marcar como pending deletion en Postgres
  await db.query(`
    UPDATE users
    SET deleted_at = now() + interval '30 days'
    WHERE id = $1
  `, [userId]);

  // 2. Notificar al usuario por email
  await sendEmail({
    to: user.email,
    subject: 'Tu cuenta será eliminada en 30 días',
    body: `Si fue un error, podés cancelar desde la app o respondiendo este email.`
  });

  // 3. Logout de Firebase
  await auth().signOut();
}

// Cron diario que ejecuta hard delete después de 30 días
async function executeScheduledDeletions() {
  const usersToDelete = await db.query(`
    SELECT id, firebase_uid FROM users
    WHERE deleted_at IS NOT NULL AND deleted_at < now()
  `);

  for (const user of usersToDelete) {
    // 1. Borrar de Firebase
    await admin.auth().deleteUser(user.firebase_uid);

    // 2. Borrar audios de S3
    await deleteUserAudios(user.id);

    // 3. Borrar de Postgres (cascade borra todo)
    await db.query('DELETE FROM users WHERE id = $1', [user.id]);
  }
}
```

### 8.4 Política de datos para usuarios anónimos

Los usuarios anónimos generan datos durante su período de prueba. Política:

- Datos del anónimo se mantienen 30 días.
- Si convierte a permanente: datos pasan al usuario nuevo (mismo Firebase UID).
- Si no convierte en 30 días: cleanup automático.

---

## 9. Seguridad

### 9.1 Validaciones críticas

**En backend (siempre):**
- Validar Firebase JWT en cada request autenticada.
- Verificar que el `firebase_uid` del token corresponde al usuario que se quiere modificar.
- Rate limiting por IP y por usuario.

**En cliente (UX):**
- Validación de inputs antes de enviar.
- Manejo de errores con mensajes amigables (sin exponer detalles internos).

### 9.2 Política de passwords

Para email/password:

- Mínimo 8 caracteres.
- Recomendado: una mayúscula, un número, un símbolo.
- NO hasheo manual: Firebase maneja hashing internamente con scrypt.
- Detección de passwords comprometidos: Firebase tiene "password breach
  detection" que rechaza passwords de leaks conocidos.

### 9.3 Email verification

- Email verification es obligatoria para acceder a features pagas.
- Email no verificado puede usar app gratuita.
- Notificación periódica para verificar si está pendiente (max 1 por día).

### 9.4 Brute force protection

Firebase incluye protecciones nativas:

- Throttling automático después de varios intentos fallidos.
- Lockout temporal de la cuenta.
- Captcha en web (Recaptcha invisible).

A nivel de aplicación, agregar:

- Rate limit en endpoints sensibles (login, password reset).
- Logging de intentos fallidos para análisis de patrones.
- Alertas si una cuenta tiene múltiples logins desde IPs muy diferentes
  en corto tiempo.

### 9.5 MFA (Multi-Factor Authentication)

No requerido para MVP. Considerar para usuarios premium o con saldo
significativo de Sparks comprados, en post-MVP.

Firebase Auth soporta SMS-based MFA y TOTP nativamente cuando se decida
implementar.

---

## 10. Phone OTP (decisión a tomar)

### 10.1 Pros y contras

**Pros:**
- Aceptación cultural alta en Latam.
- Login sin recordar password.
- Pre-verificación del teléfono (útil si después se quiere usar WhatsApp).

**Contras:**
- Costo: $0.01-0.05 USD por SMS según país y carrier.
- A 1.000 logins diarios = $300-1.500 USD/mes solo en SMS.
- Requiere captcha invisible para prevenir abuso.
- Carrier issues ocasionales (SMS no llega).
- Algunos países tienen regulaciones específicas para SMS marketing.

### 10.2 Recomendación

**Opción A (conservadora):** No incluir Phone OTP en MVP. Lanzar con Google
+ Apple + Email. Si se observa que > 5% de intentos de registro fallan
porque usuarios no quieren ninguno de los tres, evaluar agregar Phone OTP.

**Opción B (agresiva):** Incluir Phone OTP desde MVP con rate limiting
estricto (1 SMS por número cada 5 minutos, máximo 3 SMS por día por número).
Asumir el costo como inversión en conversion.

**Decisión sugerida:** Opción A. Los datos del MVP determinarán si vale la
pena agregarlo. Mejor ahorrar $200-500/mes durante validación que pagar
sin saber si aporta valor.

### 10.3 Setup técnico (cuando se agregue)

```typescript
import auth from '@react-native-firebase/auth';

async function sendOtp(phoneNumber: string) {
  // Firebase enviará SMS automáticamente
  const confirmation = await auth().signInWithPhoneNumber(phoneNumber);
  // Guardar confirmation para verificar después
  return confirmation;
}

async function verifyOtp(confirmation: any, code: string) {
  const credential = await confirmation.confirm(code);
  return credential;
}
```

---

## 11. Plan de implementación

### 11.1 Sprint 1 (semana 1): Setup foundational

- Crear proyecto Firebase y configurar Authentication.
- Configurar Google Sign In (consoles de Google Cloud).
- Configurar Apple Sign In (Apple Developer Console).
- Schema de tabla `users` en Postgres.
- Endpoint `/auth/sync-user` con validación de JWT.

### 11.2 Sprint 2 (semana 2): SSO en cliente

- Integrar SDK de Firebase en la app (development build).
- UI de pantalla de login con todos los proveedores.
- Implementación de Google Sign In flow.
- Implementación de Apple Sign In flow.
- Testing en devices reales (importante: SSO no funciona bien en simuladores).

### 11.3 Sprint 3 (semana 3): Email + recovery

- Email + password flow (signup, login).
- Email verification flow.
- Password reset por email.
- Manejo de errores con mensajes amigables.

### 11.4 Sprint 4 (semana 4): Anonymous + linking

- Anonymous auth en pantalla de login.
- Upgrade flow de anonymous a permanente.
- Account linking (detección de email duplicado).
- Cleanup cron de usuarios anónimos viejos.

### 11.5 Sprint 5 (semana 5): Account management

- UI de settings con métodos de auth conectados.
- Account deletion flow (con período de gracia).
- Cron diario de hard deletion.
- Logs de auth events para auditoría.

### 11.6 Sprint 6 (semana 6): Polish y observabilidad

- Métricas de conversion por método (Firebase Analytics).
- Alertas de errores de auth.
- Tests end-to-end.
- Documentación de troubleshooting común.

---

## 12. Costos

### 12.1 Firebase Auth pricing

- Hasta 50.000 usuarios mensuales activos: **gratis**.
- Después: $0.0055 USD por usuario adicional/mes.

A 100.000 usuarios activos: ~$275/mes solo por Auth (excelente escalabilidad).

### 12.2 SMS para Phone OTP (si se incluye)

- Variable por país, ~$0.01-0.05 por SMS.
- Volumen típico: 1 SMS por registro + 1 SMS por recovery + login ocasionales.

### 12.3 Otros costos relacionados

- Apple Developer Program: $99 USD/año (necesario para Apple Sign In).
- Google Cloud Console: gratis para uso normal de Auth.
- Recaptcha: gratis hasta volúmenes muy altos.

### 12.4 Total estimado

Para 10.000 usuarios activos: $0/mes (dentro del tier gratuito).
Para 100.000 usuarios activos: ~$275/mes.

---

## 13. Métricas y observabilidad

### 13.1 Métricas críticas

**Conversion del onboarding:**
- % de usuarios que abren la app y completan auth.
- Conversion por método (Google vs Apple vs Email).
- Tiempo promedio desde "abrir app" hasta "primer ejercicio".

**Anonymous to permanent:**
- % de usuarios anónimos que convierten a permanente.
- Tiempo promedio antes de convertir.
- Evento que dispara la conversion (paywall, primer logro, etc.).

**Account recovery:**
- Tasa de éxito de password reset.
- Volumen de tickets de soporte por pérdida de cuenta.

**Errores:**
- Tasa de fallos por método.
- Tipos de errores más comunes.

### 13.2 Stack de observabilidad

- **Firebase Analytics:** métricas nativas de Auth events.
- **PostHog:** funnel de onboarding, conversion analytics.
- **Sentry:** errores de runtime en flows de auth.
- **Cloudflare Logs:** validación de tokens y endpoints de sync.

### 13.3 Alertas

- Tasa de fallos de auth > 5% (algo está roto).
- Spike en password resets (posible breach).
- Caída en conversion de un método específico.

---

## 14. Decisiones abiertas

- [ ] ¿Phone OTP en MVP o esperar a datos del MVP para decidir?
- [ ] ¿Permitir "magic link" como método adicional de email login (sin password)?
- [ ] ¿MFA para usuarios con > 500 Sparks comprados o > 6 meses activos?
- [ ] ¿Custom claims de Firebase para roles (free vs premium vs admin)?
- [ ] ¿Soporte para login con número de WhatsApp en lugar de SMS estándar?

---

## 15. Referencias internas

- `docs/business/plan_de_negocio.docx` — Plan de negocio.
- `docs/architecture/notifications-system.md` — Sistema de notificaciones (usa Firebase también).
- `docs/architecture/sparks-system.md` — Sistema de Sparks (atado a user_id).
- `docs/architecture/platform-strategy.md` — Estrategia de plataformas.
- `docs/decisions/ADR-005-firebase-auth.md` — ADR formal (a crear).

---

## 16. Recursos externos

- [Firebase Authentication docs](https://firebase.google.com/docs/auth)
- [Apple Sign In requirements](https://developer.apple.com/sign-in-with-apple/)
- [React Native Firebase](https://rnfirebase.io/)
- [Google Sign-In for iOS and Android](https://developers.google.com/identity/sign-in)

---

*Documento vivo. Actualizar cuando se agreguen métodos de auth, cambien
políticas de proveedores, o se observen patrones de uso que sugieran ajustes.*

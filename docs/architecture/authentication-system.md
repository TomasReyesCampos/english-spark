# Authentication System

> Sistema de autenticación basado en Firebase Auth con SSO multi-proveedor.
> Sincronización con Postgres. Anonymous auth + upgrade. Account
> deletion con período de gracia.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador. Crítico para
correctness de identidad. Antes de modificar JWT validation o sync
flow: leer §6, §7, §8 completos.
**Alcance:** MVP y primeros 18 meses

---

## 0. Cómo leer este documento

- §1 establece **objetivos**.
- §2 cubre **boundaries**.
- §3 lista **proveedores soportados** (decisión cerrada por proveedor).
- §4 cubre **UX** (pantalla principal, anonymous, linking, recovery).
- §5 cubre **flujo técnico** del cliente (RN + Expo).
- §6 cubre **sincronización Firebase ↔ Postgres** (autoritativo).
- §7 cubre **sesiones y tokens**.
- §8 cubre **API contracts** (endpoints expuestos).
- §9 cubre **schemas Postgres**.
- §10 cubre **eventos emitidos**.
- §11 enumera **edge cases**.
- §12 cubre **seguridad y validación de tokens**.
- §13 cubre **deletion con período de gracia**.
- §14 cubre **decisiones cerradas**.

---

## 1. Objetivos

### 1.1 Primarios

- Reducir tiempo entre "abrir la app" y "primer ejercicio" a **< 30s**
  para mayoría de usuarios.
- Cumplir requisitos de App Store (Apple Sign-In si hay otros SSO).
- Cubrir 95%+ de usuarios potenciales con métodos disponibles.
- Permitir account recovery confiable.

### 1.2 Secundarios

- Permitir prueba sin registro (**anonymous auth**) para fricción cero.
- Soporte multi-device desde día 1.
- Account linking entre proveedores.
- Cumplimiento con regulaciones (GDPR, LFPDPPP, Ley 25.326).

---

## 2. Boundaries

### 2.1 Es responsable de

- Wrapper de Firebase Auth (Google, Apple, Email, Anonymous; Phone OTP
  diferido).
- Sincronización Firebase UID ↔ Postgres `users` table.
- Validación de JWT en cada request autenticada del backend.
- Account deletion con período de gracia 30 días + hard delete.
- Account linking (email duplicado entre providers).
- Anonymous → permanent upgrade.
- Email verification flow.

### 2.2 NO es responsable de

- **Autorización / RBAC** (no hay roles complejos en MVP; custom
  claims minimal).
- **Anti-fraud de cuentas duplicadas** (eso es `anti-fraud-system`;
  este sistema solo provee la identidad, fraud-system la analiza).
- **Account recovery con verificación humana** cuando todos los
  métodos fallan (eso es `customer-support-system`).
- **Pricing o suscripciones** (eso es `sparks-system` + producto).
- **Datos de aprendizaje** (esos viven en `student-profile-and-assessment`).
- **Gestión de tokens FCM** para notifications (eso es
  `notifications-system`).

### 2.3 Tensiones

| Tensión | Resolución |
|---------|-----------|
| Usuario tiene Firebase UID pero falló sync a Postgres | Sync endpoint es idempotente; client retry en cada request hasta succeed |
| Email duplicado entre providers | Account linking via Firebase nativo (§4.3) |
| Usuario logueado con cuenta suspended por anti-fraud | Auth permite login, pero `user_restrictions` limita features |
| User borra cuenta y vuelve | Mismos email/Firebase: cuenta nueva (no recovery automático del soft-deleted) |

---

## 3. Proveedores de identidad

### 3.1 Soportados en MVP

| Proveedor | Plataformas | Prioridad | % esperado |
|-----------|------------|-----------|------------|
| Google Sign In | iOS + Android + Web | Alta | 50–60% |
| Apple Sign In | iOS (obligatorio) | Alta | 15–25% |
| Email + Password | Todas | Alta | 20–30% |
| Anonymous | Todas | Crítico (trial flow) | 100% pre-signup |

### 3.2 NO en MVP (decisiones cerradas §14)

- **Phone OTP:** evaluar post-MVP con datos. Costo SMS y rate de SMS
  fail en Latam justifica posponer.
- **Magic link:** UX confuso para algunos usuarios.
- **Facebook, Twitter, LinkedIn, GitHub, Microsoft:** no relevantes
  para target.

### 3.3 Variant de inglés default por proveedor

No aplica directamente. La variant viene de `student_profiles.country`
+ `target_english_variant` (ver `student-profile-and-assessment.md`
§8.1).

---

## 4. UX de autenticación

### 4.1 Pantalla principal

```
┌─────────────────────────────────────┐
│           [Logo de la app]          │
│                                     │
│   "Empezá a hablar inglés hoy"      │
│                                     │
│   ┌───────────────────────────────┐ │
│   │  🔍 Continuar con Google      │ │
│   └───────────────────────────────┘ │
│                                     │
│   ┌───────────────────────────────┐ │
│   │   Continuar con Apple         │ │  <- iOS only
│   └───────────────────────────────┘ │
│                                     │
│   ──────────  o  ──────────         │
│                                     │
│   📧  Continuar con email           │
│                                     │
│   "¿Querés probar primero?"         │
│   [Empezar sin registrarme]          │
└─────────────────────────────────────┘
```

### 4.2 Anonymous auth

Permite uso sin registro durante 7 días o hasta primer pago.

```typescript
const ANONYMOUS_CONFIG = {
  trial_duration_days: 7,
  cleanup_after_days: 30,           // si no convierte a permanent
  share_progress_with_permanent: true, // Firebase mantiene UID en upgrade
};
```

**Comportamiento:**
- Anonymous user signs in → Firebase crea Anonymous User → recibe
  Firebase UID.
- Cliente llama `/auth/sync-user` para crear row en `users` con
  `is_anonymous = true`.
- Trial de 50 Sparks (`sparks-system`).
- Acceso completo durante 7 días.
- Después de 7 días: banner para "guardar tu progreso" (upgrade).
- Si no upgrade en 30 días: cleanup automático.

### 4.3 Account linking

Cuando un usuario:
1. Registra con Google (email `juan@example.com`).
2. Después intenta logear con email/password usando `juan@example.com`.

**Comportamiento:**
- Sistema detecta email duplicado en `users.email`.
- Pregunta: "Ya tienes cuenta con este email vinculada a Google.
  ¿Quieres agregar email/password como método adicional?"
- Si confirma: `auth().currentUser.linkWithCredential(emailCred)` →
  Firebase linkea las cuentas (mantiene UID).
- `linked_providers` se actualiza en Postgres.

### 4.4 Account recovery

| Método original | Flujo de recovery |
|----------------|-------------------|
| Email + password | Reset por email con magic link de un solo uso, expira en 1h |
| Google SSO | Recovery delegado a Google ("ingresá con Google") |
| Apple SSO | Recovery delegado a Apple |
| Phone OTP (futuro) | SMS al mismo número; si número perdido → soporte manual |
| Pérdida total | Email a soporte con verificación de pagos previos (RevenueCat/Stripe) como prueba de propiedad |

---

## 5. Flujo técnico del cliente

### 5.1 Setup

```typescript
// app/notifications/setup.ts
import auth from '@react-native-firebase/auth';
import { GoogleSignin } from '@react-native-google-signin/google-signin';
import { appleAuth } from '@invertase/react-native-apple-authentication';

GoogleSignin.configure({
  webClientId: 'xxx.apps.googleusercontent.com',
  offlineAccess: false,
});
```

Requiere **Expo dev build** (no funciona con Expo Go).

### 5.2 Google Sign In

```typescript
async function signInWithGoogle(): Promise<FirebaseUser> {
  await GoogleSignin.hasPlayServices();
  const { idToken } = await GoogleSignin.signIn();

  const credential = auth.GoogleAuthProvider.credential(idToken);
  const userCredential = await auth().signInWithCredential(credential);

  // Sync a Postgres (idempotente)
  await api.post('/auth/sync-user', {
    firebase_uid: userCredential.user.uid,
    email: userCredential.user.email,
    display_name: userCredential.user.displayName,
    photo_url: userCredential.user.photoURL,
    provider: 'google',
    email_verified: userCredential.user.emailVerified,
    is_new_user: userCredential.additionalUserInfo?.isNewUser ?? false,
  });

  return userCredential.user;
}
```

### 5.3 Apple Sign In (iOS only)

```typescript
async function signInWithApple(): Promise<FirebaseUser> {
  const appleResp = await appleAuth.performRequest({
    requestedOperation: appleAuth.Operation.LOGIN,
    requestedScopes: [appleAuth.Scope.EMAIL, appleAuth.Scope.FULL_NAME],
  });

  const { identityToken, nonce } = appleResp;
  if (!identityToken) throw new Error('Apple Sign-In failed');

  const credential = auth.AppleAuthProvider.credential(identityToken, nonce);
  const userCredential = await auth().signInWithCredential(credential);

  // Apple solo da fullName la PRIMERA vez. Persistirlo si existe.
  await api.post('/auth/sync-user', {
    firebase_uid: userCredential.user.uid,
    email: userCredential.user.email,
    display_name: appleResp.fullName?.givenName,  // first time only
    provider: 'apple',
    is_new_user: userCredential.additionalUserInfo?.isNewUser ?? false,
  });

  return userCredential.user;
}
```

### 5.4 Email + Password

```typescript
// Sign up
async function signUpWithEmail(email: string, password: string, name: string) {
  const userCredential = await auth().createUserWithEmailAndPassword(email, password);
  await userCredential.user.updateProfile({ displayName: name });
  await userCredential.user.sendEmailVerification();

  await api.post('/auth/sync-user', {
    firebase_uid: userCredential.user.uid,
    email,
    display_name: name,
    provider: 'email',
    email_verified: false,
    is_new_user: true,
  });
}

// Sign in
async function signInWithEmail(email: string, password: string) {
  const userCredential = await auth().signInWithEmailAndPassword(email, password);
  await api.post('/auth/sync-user', {
    firebase_uid: userCredential.user.uid,
    email,
    provider: 'email',
    is_new_user: false,
  });
  return userCredential.user;
}
```

### 5.5 Anonymous auth

```typescript
async function signInAnonymously() {
  const userCredential = await auth().signInAnonymously();

  await api.post('/auth/sync-user', {
    firebase_uid: userCredential.user.uid,
    provider: 'anonymous',
    is_anonymous: true,
    is_new_user: true,
  });
}

// Upgrade anonymous → permanent (mantiene UID y datos)
async function upgradeAnonymousToEmail(email: string, password: string, name: string) {
  const credential = auth.EmailAuthProvider.credential(email, password);
  const userCredential = await auth().currentUser?.linkWithCredential(credential);
  await userCredential.user.updateProfile({ displayName: name });

  await api.patch('/auth/upgrade-anonymous', {
    firebase_uid: userCredential.user.uid,
    email,
    display_name: name,
    provider: 'email',
  });
}

async function upgradeAnonymousToGoogle() {
  const { idToken } = await GoogleSignin.signIn();
  const credential = auth.GoogleAuthProvider.credential(idToken);
  const userCredential = await auth().currentUser?.linkWithCredential(credential);

  await api.patch('/auth/upgrade-anonymous', {
    firebase_uid: userCredential.user.uid,
    email: userCredential.user.email,
    display_name: userCredential.user.displayName,
    photo_url: userCredential.user.photoURL,
    provider: 'google',
  });
}
```

---

## 6. Sincronización Firebase ↔ Postgres

### 6.1 Estrategia A: Cliente notifica al backend (MVP)

Después de cada autenticación exitosa, el cliente llama
`/auth/sync-user`. Endpoint **idempotente**: llamadas múltiples con
mismos datos no causan problemas.

**Ventajas:**
- Simple, sin setup adicional.
- Control completo del flow.

**Desventajas:**
- Depende de que el cliente haga la llamada.

**Mitigación:**
- Idempotency en el endpoint.
- Cliente reintenta con backoff si falla.
- Validar `firebase_uid` con Firebase JWT en cada request: si no existe
  en Postgres, sync transparente desde el handler.

### 6.2 Estrategia B: Firebase trigger (post-MVP, opcional)

Cloud Function que se dispara en `onCreate` de Firebase Auth y crea
row en Postgres automáticamente.

**Cuándo agregarla:** si observamos > 0.5% de casos de
desincronización en producción.

### 6.3 Recovery de desincronización

Si JWT válido apunta a `firebase_uid` que NO existe en Postgres:

```typescript
async function verifyTokenAndSync(authHeader: string): Promise<User> {
  const token = await verifyFirebaseToken(authHeader);

  let user = await db.users.findByFirebaseUid(token.uid);

  if (!user) {
    // Auto-sync transparente
    user = await db.users.create({
      firebase_uid: token.uid,
      email: token.email,
      email_verified: token.email_verified ?? false,
      primary_provider: detectProviderFromToken(token),
      is_anonymous: token.firebase?.sign_in_provider === 'anonymous',
    });

    // Emitir evento como si fuera signup nuevo
    await emitDomainEvent('user.signed_up', { /* ... */ });
  }

  return user;
}
```

Esto **garantiza** consistency aunque el cliente falle el sync inicial.

---

## 7. Sesiones y manejo de tokens

### 7.1 Flujo de tokens

Firebase Auth maneja automáticamente:

- **idToken:** JWT corto (1h validez), enviado en `Authorization`
  header.
- **refreshToken:** token largo, usado por SDK para renovar idToken.

El SDK refresca automáticamente antes de expirar.

### 7.2 Cliente: interceptor

```typescript
// apps/mobile/src/api/client.ts
import auth from '@react-native-firebase/auth';
import axios from 'axios';

const apiClient = axios.create({
  baseURL: API_URL,
  timeout: 10000,
});

apiClient.interceptors.request.use(async (config) => {
  const user = auth().currentUser;
  if (user) {
    const token = await user.getIdToken(/* forceRefresh? */ false);
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // Token expirado o inválido. Refrescar y reintentar UNA vez.
      const user = auth().currentUser;
      if (user && !error.config._retried) {
        error.config._retried = true;
        const newToken = await user.getIdToken(/* forceRefresh */ true);
        error.config.headers.Authorization = `Bearer ${newToken}`;
        return apiClient.request(error.config);
      }
    }
    throw error;
  },
);
```

### 7.3 Backend: validación de JWT

```typescript
// apps/workers/src/utils/verify-firebase-token.ts
import { jwtVerify, createRemoteJWKSet } from 'jose';

const JWKS = createRemoteJWKSet(
  new URL('https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com'),
  {
    cooldownDuration: 24 * 60 * 60 * 1000,  // 24h cache
  },
);

export interface FirebaseTokenPayload {
  uid: string;
  email?: string;
  email_verified?: boolean;
  firebase: { sign_in_provider: string };
  iat: number;
  exp: number;
}

export async function verifyFirebaseToken(
  authHeader: string | null,
  projectId: string,
): Promise<FirebaseTokenPayload> {
  if (!authHeader?.startsWith('Bearer ')) {
    throw new UnauthorizedError('No token provided');
  }

  const token = authHeader.slice(7);
  const { payload } = await jwtVerify(token, JWKS, {
    issuer: `https://securetoken.google.com/${projectId}`,
    audience: projectId,
  });

  return payload as FirebaseTokenPayload;
}
```

Eficiente en Cloudflare Workers (jose es edge-compatible).

### 7.4 Logout

```typescript
async function signOut() {
  // Sign out de proveedores externos
  if (await GoogleSignin.isSignedIn()) {
    await GoogleSignin.signOut();
  }

  // Sign out de Firebase
  await auth().signOut();

  // Limpiar estado local
  await clearLocalState();
  await clearAsyncStorage();
}
```

### 7.5 Sesiones multi-device

Firebase soporta nativamente. Cada device tiene su propio
refreshToken.

**Política:**
- Sin límite de sesiones simultáneas (MVP).
- Cambio de password invalida todas las sesiones (Firebase comportamiento
  default).
- Posible feature futura: "ver sesiones activas" + cerrar individualmente.

---

## 8. API contracts

### 8.1 `POST /auth/sync-user`

**Llamado por:** cliente después de cualquier login/signup exitoso de
Firebase.

**Request:**

```typescript
interface SyncUserRequest {
  firebase_uid: string;
  email?: string;
  display_name?: string;
  photo_url?: string;
  provider: 'google' | 'apple' | 'email' | 'phone' | 'anonymous';
  email_verified?: boolean;
  is_anonymous?: boolean;
  is_new_user: boolean;             // del cliente: signInResult.additionalUserInfo
}
```

**Headers:** `Authorization: Bearer <firebase_idToken>`.

**Response:**

```typescript
interface SyncUserResponse {
  user_id: string;                  // UUID interno
  is_new: boolean;                  // si se creó la row ahora
  email_verified: boolean;
  primary_provider: string;
  linked_providers: string[];
  is_anonymous: boolean;
}
```

**Reglas:**
- **Idempotente:** mismos datos no causan problemas.
- Verificar JWT: `firebase_uid` del request debe matchear JWT.uid.
  Si no, `401 UNAUTHORIZED`.
- Upsert en `users`:
  ```sql
  INSERT INTO users (firebase_uid, email, display_name, primary_provider, is_anonymous)
  VALUES ($1, $2, $3, $4, $5)
  ON CONFLICT (firebase_uid) DO UPDATE
    SET email = COALESCE(EXCLUDED.email, users.email),
        display_name = COALESCE(EXCLUDED.display_name, users.display_name),
        last_login_at = now(),
        linked_providers = array_append_unique(users.linked_providers, EXCLUDED.primary_provider),
        updated_at = now();
  ```
- Si `is_new = true`: emitir `user.signed_up`.

### 8.2 `PATCH /auth/upgrade-anonymous`

**Llamado por:** cliente cuando anonymous → permanent.

**Request:**

```typescript
interface UpgradeAnonymousRequest {
  firebase_uid: string;
  email: string;
  display_name?: string;
  photo_url?: string;
  provider: 'google' | 'apple' | 'email';
}
```

**Response:**

```typescript
interface UpgradeAnonymousResponse {
  user_id: string;
  upgraded_at: string;
}
```

**Reglas:**
- Verificar JWT corresponde al user.
- Verificar el user tenía `is_anonymous = true` antes.
- Update fields:
  ```sql
  UPDATE users SET
    email = $1,
    display_name = COALESCE($2, display_name),
    photo_url = COALESCE($3, photo_url),
    primary_provider = $4,
    is_anonymous = false,
    upgraded_at = now()
  WHERE firebase_uid = $5;
  ```
- Emitir `user.upgraded_from_anonymous`.

### 8.3 `POST /auth/request-deletion`

**Llamado por:** cliente cuando user solicita account deletion.

**Request:**

```typescript
interface RequestDeletionRequest {
  reason?: string;                  // opcional, para analytics
}
```

**Response:**

```typescript
interface RequestDeletionResponse {
  user_id: string;
  deletion_scheduled_for: string;   // 30 días desde ahora
  cancellation_token: string;       // permite cancel sin login
}
```

**Reglas:**
- Verificar JWT.
- Set `users.deleted_at = now() + interval '30 days'`.
- Generar `cancellation_token` único, persistir en
  `users.deletion_cancellation_token`.
- Emitir `user.deletion_requested`.
- Enviar email de confirmación con link de cancel.

### 8.4 `POST /auth/cancel-deletion`

**Llamado por:** link en email o desde Settings.

**Request:**

```typescript
interface CancelDeletionRequest {
  cancellation_token: string;
}
```

**Response:** `200 OK` o `404 NOT_FOUND`.

**Reglas:**
- Lookup user por `cancellation_token`.
- Si encontrado y `deleted_at > now()`: clear `deleted_at` y token.
- Emitir `user.deletion_cancelled`.

### 8.5 `GET /auth/me`

**Llamado por:** cliente para verificar quién está logueado.

**Response:**

```typescript
interface MeResponse {
  user_id: string;
  email?: string;
  display_name?: string;
  photo_url?: string;
  primary_provider: string;
  linked_providers: string[];
  is_anonymous: boolean;
  email_verified: boolean;
  created_at: string;
  trial_status: 'active' | 'expired' | 'converted' | 'never_started';
}
```

### 8.6 `POST /auth/resend-verification-email`

**Llamado por:** cliente si user con `email_verified = false` quiere
re-recibir el email.

**Reglas:**
- Rate limit: max 1 cada 5 minutos por user.
- Llama Firebase Admin SDK.

---

## 9. Schemas Postgres

### 9.1 `users` (tabla central)

```sql
CREATE TABLE users (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  firebase_uid        TEXT NOT NULL UNIQUE,
  email               TEXT,
  email_verified      BOOLEAN NOT NULL DEFAULT false,
  display_name        TEXT,
  photo_url           TEXT,
  primary_provider    TEXT NOT NULL CHECK (primary_provider IN (
                        'google', 'apple', 'email', 'phone', 'anonymous'
                      )),
  is_anonymous        BOOLEAN NOT NULL DEFAULT false,
  linked_providers    TEXT[] NOT NULL DEFAULT '{}',
  custom_claims       JSONB NOT NULL DEFAULT '{}',  -- {is_admin, is_premium, ...}
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_login_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  upgraded_at         TIMESTAMPTZ,                    -- cuando anonymous → permanent
  deleted_at          TIMESTAMPTZ,                    -- soft delete (gracia 30d)
  deletion_cancellation_token TEXT
);

CREATE UNIQUE INDEX idx_users_firebase_uid ON users(firebase_uid)
  WHERE deleted_at IS NULL OR deleted_at > now();
CREATE INDEX idx_users_email ON users(email)
  WHERE email IS NOT NULL AND (deleted_at IS NULL OR deleted_at > now());
CREATE INDEX idx_users_anonymous ON users(is_anonymous, created_at)
  WHERE is_anonymous = true AND (deleted_at IS NULL OR deleted_at > now());
CREATE INDEX idx_users_pending_deletion ON users(deleted_at)
  WHERE deleted_at IS NOT NULL AND deleted_at < now() + interval '30 days';
```

### 9.2 `array_append_unique` helper

```sql
CREATE OR REPLACE FUNCTION array_append_unique(arr TEXT[], new_val TEXT)
RETURNS TEXT[] AS $$
BEGIN
  IF new_val = ANY(arr) THEN
    RETURN arr;
  ELSE
    RETURN array_append(arr, new_val);
  END IF;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

### 9.3 Cron de cleanup

```sql
-- Hard delete de cuentas con período de gracia expirado
-- Cron diario 04:00 UTC

WITH deleted AS (
  DELETE FROM users
  WHERE deleted_at IS NOT NULL AND deleted_at < now()
  RETURNING id, firebase_uid
)
-- Cascade automático borra: sparks_balance, student_profiles,
-- exercise_attempts, etc.
SELECT id, firebase_uid FROM deleted;

-- Cleanup de anonymous users no convertidos en 30 días
DELETE FROM users
WHERE is_anonymous = true
  AND created_at < now() - interval '30 days'
  AND upgraded_at IS NULL;
```

Después de hard delete: borrar de Firebase, R2 y analytics
externos (PostHog, Sentry).

---

## 10. Eventos emitidos

Detalle del shape en `cross-cutting/data-and-events.md` §5.1.

| Evento | Cuándo |
|--------|--------|
| `user.signed_up` | Nueva row en `users` (primera vez) |
| `user.signed_in` | Login exitoso (no signup) |
| `user.email_verified` | Click en link de verification email |
| `user.linked_provider` | Account linking |
| `user.upgraded_from_anonymous` | Anonymous → permanent |
| `user.deletion_requested` | User solicita deletion |
| `user.deletion_cancelled` | User cancela durante gracia |
| `user.deleted` | Hard delete ejecutado |

---

## 11. Edge cases (tests obligatorios)

### 11.1 Sync

1. **Sync con `firebase_uid` que no existe + JWT válido:** crea row,
   emite signed_up.
2. **Sync con `firebase_uid` ya existente:** update fields, emite
   signed_in (no signed_up).
3. **JWT inválido en sync:** `401 UNAUTHORIZED` sin tocar DB.
4. **JWT válido pero `firebase_uid` del body NO matchea JWT.uid:**
   `401 MISMATCH`.
5. **Sync con email que ya existe en otra row:** account linking
   prompt en cliente; si no aceptado, error `409 EMAIL_CONFLICT`.

### 11.2 Anonymous

6. **Anonymous user no llama upgrade y pasan 30 días:** cleanup
   borra row + Firebase user.
7. **Anonymous user upgrade a Google con email que ya existe en otra
   row:** linkWithCredential falla; cliente debe ofrecer "linkear con
   esa cuenta y descartar progreso del anonymous" o mantener anonymous.

### 11.3 Account linking

8. **User con email/password agrega Google:** `linked_providers`
   contiene ambos. JWT desde cualquier provider trae mismo
   `firebase_uid`.
9. **User pierde acceso al provider primario:** linkear secundario
   ofrece path alternativo.

### 11.4 Account deletion

10. **User solicita deletion + cancela en día 5:** `deleted_at` se
    limpia. Cuenta vuelve a normal.
11. **User solicita deletion y se loguea en día 15:** ve banner "tu
    cuenta se borrará en 15 días, ¿cancelar?". Login funciona normal
    (no bloqueamos).
12. **Hard delete falla a media (Firebase OK pero R2 timeout):** mark
    `users.id` con flag `deletion_partial`; cron retry de partes
    pendientes; alerta a humano si no se completa en 24h.
13. **User borra cuenta y se registra mismo email después:** account
    nueva. Sin recovery automático del soft-deleted.

### 11.5 Tokens

14. **idToken expirado en request:** interceptor del cliente refresh
    + retry. Backend ya debería rechazar con 401.
15. **JWT de proyecto Firebase distinto:** rechazado por audience
    mismatch.
16. **JWT con `firebase.sign_in_provider = 'anonymous'`:** verificar
    que `users.is_anonymous = true`. Si false: `409 STATE_CONFLICT`.

### 11.6 Email verification

17. **User con email no verificado intenta feature paga:** UI muestra
    prompt "verificá tu email primero". Backend NO bloquea
    automáticamente (es responsabilidad del producto decidir).

### 11.7 Concurrencia

18. **Sync + sync concurrentes del mismo user en <1s:** ON CONFLICT
    resuelve correctamente; ambos retornan mismos datos.

---

## 12. Seguridad

### 12.1 Validaciones críticas

**En backend (siempre):**
- Validar Firebase JWT en cada request autenticada.
- Verificar `firebase_uid` del JWT corresponde al usuario que se
  quiere modificar.
- Rate limiting por IP y por user (Cloudflare + Durable Objects).

**En cliente:**
- Validación de inputs antes de enviar.
- Manejo de errors con mensajes amigables (no expone detalles internos).

### 12.2 Política de passwords

- Mínimo 8 caracteres.
- Recomendado: 1 mayúscula, 1 número, 1 símbolo (no enforced en MVP).
- **No hashing manual:** Firebase usa scrypt internamente.
- Password breach detection: Firebase rechaza passwords de leaks
  conocidos automáticamente.

### 12.3 Email verification

- Email verification es **obligatoria para features pagas** (Sparks
  packs, suscripciones).
- Email no verificado puede usar app gratuita y trial.
- Notificación periódica: max 1 por día.

### 12.4 Brute force protection

Firebase incluye:
- Throttling automático tras múltiples intentos fallidos.
- Lockout temporal.
- Recaptcha invisible en web.

A nivel app:
- Rate limit en endpoints sensibles (login, password reset).
- Logging de intentos fallidos.
- Alerta si una cuenta tiene logins desde IPs muy distintas en corto
  tiempo (cross con anti-fraud).

### 12.5 MFA

**No requerido para MVP.** Considerar para:
- Usuarios premium con saldo > $100 USD en Sparks comprados.
- Usuarios con > 6 meses activos.

Firebase Auth soporta SMS MFA y TOTP nativamente.

### 12.6 Custom claims

Mínimos en MVP:
```typescript
interface CustomClaims {
  is_admin?: boolean;       // para admin panel
  is_premium?: boolean;     // si tiene plan Premium activo
}
```

Set vía Firebase Admin SDK desde Workers cuando cambia plan o se
otorga admin.

---

## 13. Account deletion (detalle)

### 13.1 Flow completo

```
[User]: tap "Eliminar cuenta" en Settings
   ↓
[Cliente]: muestra confirmación con explicación de qué se borra
   ↓
[Cliente]: POST /auth/request-deletion { reason }
   ↓
[Backend]:
   - users.deleted_at = now() + interval '30 days'
   - users.deletion_cancellation_token = uuid
   - emit user.deletion_requested
   - send email con link de cancel
   ↓
[Cliente]: logout local, mostrar mensaje "Tu cuenta será eliminada en 30 días"
   ↓
30 días pasan...
   ↓
[Cron diario]: hardDeleteScheduled()
   ↓
[Backend]:
   - admin.auth().deleteUser(firebase_uid)
   - delete from R2: users/<user_id>/*
   - delete from PostHog: api/persons/delete
   - delete from Sentry
   - DELETE FROM users WHERE id = ... (cascade en Postgres)
   - emit user.deleted
```

### 13.2 Qué se borra

**Postgres (cascade desde `users`):**
- `student_profiles`, `user_subskill_mastery`, `exercise_attempts`,
  `user_block_status`, `user_sparks_balance`, `sparks_packs_purchased`,
  `billing_cycles`, `user_achievements`, `achievements_progress`,
  `user_fcm_tokens`, `user_notification_preferences`, `notifications_log`,
  `device_fingerprints` (vía `user_devices`), `user_fraud_scores`,
  `support_tickets`, `weekly_mastery_snapshots`, `test_out_attempts`.

**Otros:**
- Firebase Auth user.
- R2: todos los audios bajo `users/<user_id>/`.
- PostHog: `distinct_id = sha256(user_id)`.
- Sentry: events bajo `user_id_hash`.

### 13.3 Lo que NO se borra

- `sparks_transactions` con `user_id = NULL` (audit financiero).
  Anonimizar en lugar de borrar.
- `event_log` (anonimizado a 90 días automático).

### 13.4 Reglas

- Si hard delete falla parcialmente: flag `deletion_partial`, alerta a
  humano, retry hasta 24h máx.
- Si pasa 24h sin completarse: escalation a soporte para investigación
  manual.

---

## 14. Decisiones cerradas

### 14.1 Phone OTP en MVP: **NO** ✓

**Razón:** costo SMS ($0.01–0.05 USD/SMS), tasa de fail en algunos
carriers en Latam, complejidad de setup. Re-evaluar mes 3-6 si > 5%
de signups fallan porque user no quiere ninguno de los 3 métodos.

### 14.2 Magic link como método: **NO** ✓

**Razón:** UX confuso para algunos usuarios. Mantenemos email + password
estándar.

### 14.3 MFA en MVP: **NO** ✓

**Razón:** target consumer no espera MFA. Considerar para usuarios
premium con > $100 USD en Sparks comprados o > 6 meses activos
(post-soft-launch).

### 14.4 Custom claims: **mínimos (is_admin, is_premium)** ✓

**Razón:** evitar over-engineering. RBAC complejo no se justifica para
consumer app de MVP.

### 14.5 Login con WhatsApp: **NO** ✓

**Razón:** WhatsApp Login Embedded Sign-In tiene cobertura limitada.
Phone OTP genérico es mejor opción si decidimos auth por teléfono
post-MVP.

---

## 15. Plan de implementación

### 15.1 Sprint 1 (semana 1)

- Crear proyecto Firebase, configurar Auth.
- Configurar Google Sign-In (Google Cloud Console).
- Configurar Apple Sign-In (Apple Developer Console).
- Schema `users` en Postgres.
- Endpoint `/auth/sync-user` con JWT validation.
- Tests unit de `verifyFirebaseToken`.

### 15.2 Sprint 2 (semana 2)

- SDK Firebase en cliente (Expo dev build).
- UI pantalla de login.
- Google Sign-In flow.
- Apple Sign-In flow.
- Testing en devices reales (no simuladores).

### 15.3 Sprint 3 (semana 3)

- Email + password (signup, login).
- Email verification.
- Password reset.
- Manejo de errors.

### 15.4 Sprint 4 (semana 4)

- Anonymous auth + UI de "Empezar sin registrarme".
- Upgrade flow anonymous → permanent.
- Account linking (email duplicado).
- Cleanup cron de anonymous viejos.

### 15.5 Sprint 5 (semana 5)

- UI Settings con métodos de auth conectados.
- Account deletion flow.
- Cron de hard delete.
- Logs de auth events para auditoría.

### 15.6 Sprint 6 (semana 6)

- Métricas en Firebase Analytics.
- Alertas de errors.
- Tests de integración.
- Documentación de troubleshooting.

---

## 16. Costos

### 16.1 Firebase Auth

- Hasta 50.000 MAU: gratis.
- > 50.000 MAU: $0.0055 USD/usuario adicional/mes.

A 100.000 MAU: ~$275 USD/mes solo por Auth.

### 16.2 SMS (si se agrega Phone OTP)

- $0.01–0.05 USD/SMS según país.
- Volumen típico: 1 SMS por registro + 1 por recovery + login
  ocasionales.

### 16.3 Otros

- Apple Developer Program: $99 USD/año.
- Google Play Developer: $25 USD único.
- Recaptcha (web): gratis hasta volúmenes altos.

### 16.4 Total estimado

- 10.000 MAU: $0/mes (tier gratuito).
- 100.000 MAU: ~$275 USD/mes.

---

## 17. Métricas y observabilidad

### 17.1 Métricas críticas

**Conversion del onboarding:**
- % de usuarios que abren la app y completan auth.
- Conversion por método (Google vs Apple vs Email).
- Tiempo desde "abrir app" hasta "primer ejercicio".

**Anonymous to permanent:**
- % de anonymous que convierten.
- Tiempo promedio antes de convertir.
- Evento que dispara conversion (paywall, primer logro).

**Account recovery:**
- Tasa de éxito de password reset.
- Volumen de tickets de soporte por pérdida de cuenta.

**Errores:**
- Tasa de fallos por método.
- Tipos de errors más comunes.

### 17.2 Alertas

- Tasa de fallos de auth > 5%: SEV-2.
- Spike en password resets: posible breach, SEV-2.
- Caída en conversion de un método específico > 30% día contra día:
  SEV-2.
- JWT verify failures > 1% del tráfico: posible ataque, SEV-2.

---

## 18. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`../decisions/ADR-005-firebase-auth.md`](../decisions/ADR-005-firebase-auth.md) | Decisión arquitectónica. |
| [`sparks-system.md`](sparks-system.md) | Trial Sparks dependen de auth. |
| [`anti-fraud-system.md`](anti-fraud-system.md) | Lee `users` para fingerprinting. |
| [`notifications-system.md`](notifications-system.md) | Firebase también para FCM. |
| [`../product/student-profile-and-assessment.md`](../product/student-profile-and-assessment.md) | Crea `student_profile` al `user.signed_up`. |
| [`../cross-cutting/security-threat-model.md`](../cross-cutting/security-threat-model.md) §3.1 | Amenazas de auth. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.1 | Eventos emitidos. |
| [`../business/legal-compliance.md`](../business/legal-compliance.md) §4 | Account deletion como GDPR right. |

---

## 19. Recursos externos

- Firebase Authentication: https://firebase.google.com/docs/auth
- Apple Sign In: https://developer.apple.com/sign-in-with-apple/
- React Native Firebase: https://rnfirebase.io/
- Google Sign-In: https://developers.google.com/identity/sign-in
- jose (JWT verification): https://github.com/panva/jose

---

*Documento vivo. Actualizar cuando se agreguen métodos de auth, cambien
políticas de proveedores, o se observen patrones que sugieran ajustes.*

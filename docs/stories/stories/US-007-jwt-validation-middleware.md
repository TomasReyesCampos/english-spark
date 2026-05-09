# US-007: JWT validation middleware en Workers

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Admin
**Owner:** —

---

## Contexto

Toda request autenticada al backend (Cloudflare Workers) debe
validar el JWT de Firebase antes de procesar. Sin esto, el backend
es completamente abierto.

Validación correcta de JWT incluye:
- Verificar **firma** contra public keys de Firebase (rotadas
  diariamente).
- Verificar **claims** (issuer, audience, expiration).
- Cachear public keys para performance.
- Extraer `firebase_uid` y exponerlo a handlers downstream.

Esta story es **bloqueante para US-008** y todos los endpoints
autenticados futuros.

Backend en
[`authentication-system.md`](../../architecture/authentication-system.md)
§4 (JWT validation strategy).

## Scope

### In

- Middleware `validateFirebaseJwt` para Cloudflare Workers que:
  - Lee header `Authorization: Bearer <jwt>`.
  - Fetcha public keys de Firebase (con cache).
  - Verifica firma RS256.
  - Verifica claims (`iss`, `aud`, `exp`, `auth_time`).
  - Inyecta `request.user = { firebase_uid, email, ... }` para
    downstream handlers.
- Cache de public keys en KV con TTL alineado a Cache-Control
  header de Firebase (~1 hora típicamente).
- Manejo de errors: token missing, invalid, expired, malformed,
  signature failed.
- Tests unit con tokens mockeados (válidos, expirados, mal
  firmados).
- Tests integration con un token real de Firebase Auth emulator.

### Out

- Endpoints específicos que usan el middleware (US-008 y futuras).
- Sesión refresh (Firebase SDK del cliente lo maneja
  automáticamente).
- Custom claims (post-MVP, ver `pendientes.md` §2.2).

## Acceptance criteria

- **Given** una request sin header `Authorization`, **When** llega
  al middleware, **Then** retorna 401 con body
  `{ error: "missing_auth_header" }`.
- **Given** una request con `Authorization: Bearer <token-válido>`,
  **When** el middleware valida, **Then** la request continúa al
  handler con `request.user.firebase_uid` poblado.
- **Given** un token con firma inválida, **When** el middleware
  intenta validar, **Then** retorna 401 con
  `{ error: "invalid_signature" }`.
- **Given** un token expirado (exp < now), **When** se valida,
  **Then** retorna 401 con `{ error: "token_expired" }`.
- **Given** un token con `aud` distinto al project_id de Firebase,
  **When** se valida, **Then** retorna 401 con
  `{ error: "invalid_audience" }`.
- **Given** las public keys de Firebase ya están en cache KV,
  **When** llega una nueva request en menos de TTL, **Then** NO se
  hace fetch a Firebase (cache hit).
- **Given** las public keys expiraron en cache, **When** llega una
  request, **Then** se hace fetch fresh y se actualiza cache.
- **Given** Firebase está down (no se pueden fetch keys), **When**
  llega una request y cache también expiró, **Then** retorna 503
  `{ error: "auth_service_unavailable" }` y se loguea SEV-2.

## Developer details

### Owning service

`apps/workers/auth-middleware` (compartido por todos los Workers).

### Dependencies

- US-001: Firebase project con Auth habilitado.
- KV namespace en Cloudflare configurado (variable
  `FIREBASE_PUBLIC_KEYS`).
- Library de JWT verification para Workers runtime: `jose` (works
  in Cloudflare Workers).

### Specs referenciados

- [`authentication-system.md`](../../architecture/authentication-system.md)
  §4 — JWT validation strategy.
- [`decisions/ADR-005-firebase-auth.md`](../../decisions/ADR-005-firebase-auth.md)
  — decisión.

### Implementación esperada

```typescript
// apps/workers/shared/auth-middleware.ts
import { jwtVerify, createRemoteJWKSet } from 'jose';

const FIREBASE_PROJECT_ID = 'english-spark-dev';
const FIREBASE_JWKS_URL =
  'https://www.googleapis.com/service_accounts/v1/jwk/securetoken@system.gserviceaccount.com';

interface AuthedRequest extends Request {
  user?: {
    firebase_uid: string;
    email: string | null;
    email_verified: boolean;
    is_anonymous: boolean;
  };
}

export async function validateFirebaseJwt(
  request: AuthedRequest,
  env: Env
): Promise<Response | null> {
  const authHeader = request.headers.get('Authorization');
  if (!authHeader?.startsWith('Bearer ')) {
    return new Response(JSON.stringify({ error: 'missing_auth_header' }), {
      status: 401, headers: { 'Content-Type': 'application/json' },
    });
  }

  const token = authHeader.substring(7);

  try {
    const jwks = await getCachedJwks(env);
    const { payload } = await jwtVerify(token, jwks, {
      issuer: `https://securetoken.google.com/${FIREBASE_PROJECT_ID}`,
      audience: FIREBASE_PROJECT_ID,
    });

    request.user = {
      firebase_uid: payload.sub as string,
      email: (payload.email as string) ?? null,
      email_verified: (payload.email_verified as boolean) ?? false,
      is_anonymous: (payload.firebase as any)?.sign_in_provider === 'anonymous',
    };
    return null; // continue to handler
  } catch (error) {
    if (error.code === 'ERR_JWT_EXPIRED') {
      return new Response(JSON.stringify({ error: 'token_expired' }),
        { status: 401, headers: { 'Content-Type': 'application/json' } });
    }
    if (error.code === 'ERR_JWS_SIGNATURE_VERIFICATION_FAILED') {
      return new Response(JSON.stringify({ error: 'invalid_signature' }),
        { status: 401, headers: { 'Content-Type': 'application/json' } });
    }
    return new Response(JSON.stringify({ error: 'invalid_token' }),
      { status: 401, headers: { 'Content-Type': 'application/json' } });
  }
}

async function getCachedJwks(env: Env) {
  const cached = await env.AUTH_KV.get('firebase_jwks');
  if (cached) return createRemoteJWKSet(new URL(FIREBASE_JWKS_URL));

  // Fetch + store
  const response = await fetch(FIREBASE_JWKS_URL);
  if (!response.ok) throw new Error('jwks_fetch_failed');

  const cacheControl = response.headers.get('cache-control');
  const maxAge = parseMaxAge(cacheControl) ?? 3600;
  const jwks = await response.text();

  await env.AUTH_KV.put('firebase_jwks', jwks, { expirationTtl: maxAge });
  return createRemoteJWKSet(new URL(FIREBASE_JWKS_URL));
}
```

### Uso en handlers

```typescript
// apps/workers/api/handlers/profile.ts
export async function handleGetProfile(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const profile = await db.getProfile(request.user!.firebase_uid);
  return new Response(JSON.stringify(profile), {
    headers: { 'Content-Type': 'application/json' },
  });
}
```

### Integration points

- Todos los Workers que expongan endpoints autenticados.
- Cloudflare KV (cache de JWKS).
- Firebase Auth (source of public keys).

### Notas técnicas

- `jose` library funciona en Workers runtime (no Node.js APIs).
- JWKS endpoint de Firebase rota cada ~1h; cache respecta el
  Cache-Control header.
- En desarrollo con emulator: usar
  `http://localhost:9099/...` con flag `EMULATOR=true` para
  bypassear validación strict (solo en dev).
- Token de Firebase tiene exp típicamente de 1h.

## Definition of Done

- [ ] Middleware `validateFirebaseJwt` implementado.
- [ ] Cache de JWKS funcionando con KV.
- [ ] 8 errors del AC manejados.
- [ ] Tests unit con tokens mockeados (valid, expired, invalid
  sig, missing).
- [ ] Test integration con Firebase Auth emulator.
- [ ] Performance: JWT validation < 50ms con cache hit, < 500ms
  con cache miss.
- [ ] Documentación en `docs/architecture/auth-middleware.md` (a
  crear).
- [ ] Validation contra spec `authentication-system.md` §4.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-008 y todos los endpoints futuros autenticados.*

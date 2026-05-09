# US-015: Persistir respuestas del onboarding en student_profile

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 2
**Persona:** Admin (consumido por todas las US 010-014)
**Owner:** —

---

## Contexto

Las stories US-010 a US-014 capturan respuestas del onboarding y
las persisten **incrementalmente** en `student_profiles` mediante
llamadas a `/profile/update`. Esta story define el endpoint
unificado, su validación, idempotency, y manejo de session resume
(user cierra app mid-flow).

Sin esta story, cada pantalla del onboarding tendría su propio
endpoint custom — over-engineered.

## Scope

### In

- Endpoint `PATCH /profile/update`:
  - Valida JWT.
  - Recibe partial update body (campos opcionales).
  - Valida cada campo con Zod schema (matchea student_profile
    schema).
  - Aplica UPDATE SQL solo a campos presentes en body.
  - Updates `updated_at` automáticamente (trigger).
  - Retorna profile actualizado completo.
- Endpoint `GET /profile`:
  - Valida JWT.
  - Retorna profile completo del user actual.
  - Si user no tiene profile: retorna 404 (no debería pasar
    post-US-008 pero defensive).
- Soporte session resume:
  - Cliente al reabrir app mid-onboarding consulta GET /profile.
  - Si campos básicos están null/empty: retoma desde la primera
    pregunta no respondida.
  - Si todos los campos están: continúa al mini-test.
- Validation con Zod:
  - country: enum ['MX', 'AR', 'CO', 'CL', 'PE', 'OT'].
  - primary_goals: array de enum strings, max 3 items.
  - has_deadline: boolean; si true, deadline_date debe estar.
  - speaking_confidence: int 1-5.
  - daily_minutes_available: enum [5, 10, 15, 20, 30, 45, 60].
  - professional_field: enum o null.
  - self_perceived_level: enum CEFR.
  - language_anxiety: int 1-5.
- Telemetry: `profile.updated` event con campos cambiados.

### Out

- UI para editar profile post-onboarding (Settings) — story
  propia.
- Lógica `significant_change` que regenera roadmap — story propia
  para post-onboarding.
- Validación cross-field compleja (ej: si professional_field='student',
  primary_goals no debería incluir 'remote_work'). MVP no enforce.

## Acceptance criteria

- **Given** user logged in, **When** PATCH `/profile/update` con
  body `{ country: 'MX' }`, **Then** retorna 200 con profile
  completo y solo `country` se actualiza en BD.
- **Given** user envía body `{ country: 'XX' }` (inválido), **When**
  endpoint valida con Zod, **Then** retorna 400 con
  `{ error: 'invalid_country' }`.
- **Given** user envía `{ primary_goals: ['a','b','c','d'] }` (4
  items), **When** endpoint valida, **Then** retorna 400 con
  `{ error: 'primary_goals_max_3' }`.
- **Given** user envía 2 campos válidos, **When** endpoint procesa,
  **Then** ambos se actualizan en una sola transacción.
- **Given** user envía body vacío `{}`, **When** PATCH, **Then**
  retorna 200 sin cambios (es no-op válido pero loguea warning).
- **Given** user reabre app mid-onboarding después de responder
  pantallas 3 y 4, **When** cliente llama GET `/profile`, **Then**
  retorna profile con `country` y `primary_goals` poblados,
  `professional_field` null. Cliente navega a US-013.
- **Given** request sin JWT, **When** PATCH `/profile/update`,
  **Then** retorna 401.
- **Given** PATCH con `has_deadline: true` pero sin `deadline_date`,
  **When** endpoint valida, **Then** retorna 400 con
  `{ error: 'deadline_date_required' }`.

## Developer details

### Owning service

`apps/workers/api`.

### Dependencies

- US-007: JWT middleware.
- US-008: user existe (`/auth/sync` lo crea).
- US-002: schema `student_profiles`.

### Specs referenciados

- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §3.2 — schema autoritativo.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §10.5, §10.6 — API contracts `getProfile`, `updateProfile`.

### Implementación esperada

```typescript
// apps/workers/api/handlers/profile.ts
import { z } from 'zod';

const ProfileUpdateSchema = z.object({
  country: z.enum(['MX', 'AR', 'CO', 'CL', 'PE', 'OT']).optional(),
  primary_goals: z.array(z.enum([
    'job_interview', 'remote_work', 'business_communication',
    'travel', 'studies', 'personal_growth', 'exam_prep',
    'communication_with_family', 'entertainment',
  ])).max(3).optional(),
  has_deadline: z.boolean().optional(),
  deadline_date: z.string().datetime().optional(),
  professional_field: z.enum([
    'tech', 'health', 'finance', 'sales', 'marketing',
    'education', 'legal', 'creative', 'service', 'student', 'other',
  ]).nullable().optional(),
  speaking_confidence: z.number().int().min(1).max(5).optional(),
  language_anxiety: z.number().int().min(1).max(5).optional(),
  self_perceived_level: z.enum(['A2','B1','B1+','B2','B2+','C1']).optional(),
  daily_minutes_available: z.enum([5, 10, 15, 20, 30, 45, 60]).optional(),
  active_track: z.enum([
    'job_ready', 'travel_confident', 'daily_conversation',
  ]).optional(),
}).refine(
  (data) => !data.has_deadline || data.deadline_date,
  { message: 'deadline_date_required' }
);

export async function handleUpdateProfile(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const body = await request.json();
  const parsed = ProfileUpdateSchema.safeParse(body);
  if (!parsed.success) {
    return jsonResponse({
      error: parsed.error.issues[0].message ?? 'invalid_input',
    }, 400);
  }

  const updates = parsed.data;
  if (Object.keys(updates).length === 0) {
    console.warn('Empty profile update', { uid: request.user!.firebase_uid });
    const profile = await db.getProfile(request.user!.firebase_uid);
    return jsonResponse(profile);
  }

  const updated = await db.updateProfile(request.user!.firebase_uid, updates);

  await emitEvent('profile.updated', {
    user_id: updated.user_id,
    fields_changed: Object.keys(updates),
  });

  return jsonResponse(updated);
}

export async function handleGetProfile(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const profile = await db.getProfile(request.user!.firebase_uid);
  if (!profile) return jsonResponse({ error: 'profile_not_found' }, 404);

  return jsonResponse(profile);
}
```

### SQL update dinámico

```typescript
// apps/workers/api/lib/db-profile.ts
export async function updateProfile(firebaseUid: string, updates: any) {
  const fields = Object.keys(updates);
  if (fields.length === 0) return getProfile(firebaseUid);

  const setClause = fields.map((f, i) => `${f} = $${i + 2}`).join(', ');
  const values = [firebaseUid, ...fields.map(f => updates[f])];

  const result = await env.DB.execute(`
    UPDATE student_profiles
    SET ${setClause}
    WHERE user_id = (SELECT id FROM users WHERE firebase_uid = $1 AND deleted_at IS NULL)
    RETURNING *
  `, values);

  return result[0];
}
```

### Integration points

- US-010 a US-014 (consumers).
- Event bus (`profile.updated` event).
- AI Roadmap engine (US-021) lee profile completo antes de generar.

### Notas técnicas

- SQL injection prevention: el dynamic SET clause usa positional
  args ($1, $2, ...) — seguro.
- `fields_changed` event payload permite a otros consumers (ej:
  AI Roadmap) detectar `significant_change`.
- Idempotency: PATCH con mismo body 2x produce mismo resultado.

## Definition of Done

- [ ] 2 endpoints implementados (PATCH /profile/update, GET /profile).
- [ ] Zod schema completo y validation testeado.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit de schema validation.
- [ ] Test integration end-to-end con todas las US-010 a US-014.
- [ ] Telemetry events.
- [ ] Validation contra spec
  `student-profile-and-assessment.md` §10.5, §10.6.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para US-010 a US-014.*

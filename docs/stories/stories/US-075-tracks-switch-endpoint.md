# US-075: POST /tracks/switch endpoint con rate limit

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 2
**Story points:** 5
**Persona:** Estudiante (cambia track)
**Owner:** —

---

## Contexto

Implementa la decisión clave de ADR-007: user puede switch track
activo (Job Ready ↔ Travel Confident ↔ Daily Conversation) con:
- Confirmación required.
- Rate limit 1 switch / 14 días.
- Hard block 3 switches / 60 días.
- Sub-skill mastery transfiere globalmente.
- Block completion NO transfiere (start fresh en nuevo track).
- Roadmap se regenera para el nuevo track.

Backend en
[`decisions/ADR-007-multi-track-strategy.md`](../../decisions/ADR-007-multi-track-strategy.md)
+
[`ai-roadmap-system.md`](../../product/ai-roadmap-system.md).

## Scope

### In

- Endpoint `POST /tracks/switch`:
  - Valida JWT.
  - Body: `{ to_track, confirmation: true }`.
  - Validation:
    - `to_track` enum (job_ready / travel_confident /
      daily_conversation).
    - `confirmation` debe ser true (anti-accidental).
    - to_track ≠ current active_track.
  - Rate limit check:
    - `last_track_switch_at > now() - 14 days` → 429.
    - `track_switched_count >= 3` en 60 días → 429 hard
      block.
  - Atomic operation:
    - Archive current roadmap: `is_active = false`,
      `archived_reason = 'track_switched'`.
    - Update `student_profiles.active_track`,
      `track_switched_count++`, `last_track_switch_at = now()`.
    - Trigger Inngest function para generate new roadmap async
      (reuse US-021 pattern con new track).
  - Emit event `user.track_switched`.
  - Retorna `{ new_active_track, message }`.
  - Cache invalidation (roadmap_active key).
- Telemetry: `track.switch_requested`, `.switch_succeeded`,
  `.switch_rate_limited`.

### Out

- Track switch UI (US-076).
- Roadmap regeneration logic (reuse US-021 pattern).

## Acceptance criteria

- **Given** user con active_track='job_ready', **When** POST
  switch con `to_track: 'travel_confident'`, **Then**
  active_track actualiza, current roadmap archived, new roadmap
  triggered.
- **Given** user con `last_track_switch_at = hace 10 días`,
  **When** intent switch, **Then** 429 con `{ error:
  'rate_limited', next_allowed_at }`.
- **Given** user con switched_count=3 en últimos 60 días, **When**
  4to intento, **Then** 429 hard block con
  `{ error: 'switch_count_exceeded' }`.
- **Given** `to_track = current_track`, **When** valida, **Then**
  400 `{ error: 'same_track' }`.
- **Given** `confirmation: false` o missing, **When** valida,
  **Then** 400 `{ error: 'confirmation_required' }`.
- **Given** switch exitoso, **When** completa, **Then** sub-skill
  mastery se preserva (verify via query post-switch).
- **Given** switch exitoso, **When** próximo GET /roadmap/active,
  **Then** retorna roadmap nuevo (cache invalidated).
- **Given** Inngest function de regeneration falla, **When** falla,
  **Then** user queda con current_track updated pero sin roadmap
  activo → endpoint `/roadmap/active` retorna pending state hasta
  retry.

## Developer details

### Owning service

`apps/workers/api/handlers/tracks-switch.ts`.

### Dependencies

- US-072 schemas.
- US-021 pattern de generación.
- ADR-007 logic.

### Specs referenciados

- [`decisions/ADR-007-multi-track-strategy.md`](../../decisions/ADR-007-multi-track-strategy.md)
  — reglas autoritativas.
- [`ai-roadmap-system.md`](../../product/ai-roadmap-system.md) §14.

### Implementación esperada

```typescript
// apps/workers/api/handlers/tracks-switch.ts
const SwitchSchema = z.object({
  to_track: z.enum(['job_ready', 'travel_confident', 'daily_conversation']),
  confirmation: z.literal(true),
});

const COOLDOWN_DAYS = 14;
const MAX_SWITCHES_IN = 60;
const MAX_SWITCHES_COUNT = 3;

export async function handleTrackSwitch(request: AuthedRequest, env: Env) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const body = await request.json();
  const parsed = SwitchSchema.safeParse(body);
  if (!parsed.success) {
    return jsonResponse({
      error: parsed.error.issues[0].message ?? 'confirmation_required',
    }, 400);
  }

  const userId = await getUserIdFromFirebaseUid(request.user!.firebase_uid, env);

  const profile = await env.DB.query(`
    SELECT active_track, track_switched_count, last_track_switch_at
    FROM student_profiles WHERE user_id = $1
  `, [userId]);

  const current = profile[0];
  if (current.active_track === parsed.data.to_track) {
    return jsonResponse({ error: 'same_track' }, 400);
  }

  // Rate limit check 1: cooldown 14 días
  if (current.last_track_switch_at) {
    const elapsed = Date.now() - new Date(current.last_track_switch_at).getTime();
    const cooldownMs = COOLDOWN_DAYS * 24 * 3600 * 1000;
    if (elapsed < cooldownMs) {
      track('track.switch_rate_limited', { reason: 'cooldown' });
      return jsonResponse({
        error: 'rate_limited',
        reason: 'cooldown_active',
        next_allowed_at: new Date(new Date(current.last_track_switch_at).getTime() + cooldownMs).toISOString(),
      }, 429);
    }
  }

  // Rate limit check 2: max switches in 60 days
  const recentSwitches = await env.DB.query(`
    SELECT COUNT(*) AS count FROM domain_events
    WHERE event_name = 'user.track_switched'
      AND payload->>'user_id' = $1::text
      AND created_at > now() - interval '${MAX_SWITCHES_IN} days'
  `, [userId]);

  if (Number(recentSwitches[0].count) >= MAX_SWITCHES_COUNT) {
    track('track.switch_rate_limited', { reason: 'max_count' });
    return jsonResponse({
      error: 'switch_count_exceeded',
      max: MAX_SWITCHES_COUNT, days: MAX_SWITCHES_IN,
    }, 429);
  }

  track('track.switch_requested', {
    from: current.active_track,
    to: parsed.data.to_track,
  });

  // Atomic transition
  await env.DB.transaction(async (tx) => {
    // Archive current roadmap
    await tx.execute(`
      UPDATE roadmaps
      SET is_active = false, archived_at = now(),
          archived_reason = 'track_switched'
      WHERE user_id = $1 AND is_active = true
    `, [userId]);

    // Update profile
    await tx.execute(`
      UPDATE student_profiles
      SET active_track = $1,
          track_switched_count = track_switched_count + 1,
          last_track_switch_at = now()
      WHERE user_id = $2
    `, [parsed.data.to_track, userId]);
  });

  // Invalidate cache
  await env.ROADMAP_KV.delete(`roadmap_active:${userId}`);

  // Trigger regeneration async
  await emitEvent('user.track_switched', {
    user_id: userId,
    from: current.active_track,
    to: parsed.data.to_track,
    switched_at: new Date().toISOString(),
  });

  track('track.switch_succeeded', {
    from: current.active_track,
    to: parsed.data.to_track,
  });

  return jsonResponse({
    new_active_track: parsed.data.to_track,
    message: 'Track switched. Your new plan is being generated.',
  });
}

// Inngest function consumes user.track_switched event
// and calls the generate_initial_roadmap pattern with new track
```

### Integration points

- US-021/032 pattern reused for new roadmap generation.
- US-073 cache invalidated.
- Roadmap reveal screen (mobile shows pending state until ready).

### Notas técnicas

- Atomicidad: archive old + update profile en single transaction.
- Roadmap generation es **async** (Inngest event): UI shows
  loading hasta que termine.
- Rate limit count via query a domain_events (assumes event bus
  with audit trail).

## Definition of Done

- [ ] Endpoint implementado.
- [ ] Rate limit logic correcto (cooldown + count).
- [ ] Atomic transition.
- [ ] Roadmap regeneration triggered.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Validation contra ADR-007.
- [ ] PR aprobada y mergeada.

---

*Depende de US-072. Bloqueante para US-076 UI.*

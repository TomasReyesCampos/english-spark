# US-070: Rate limiter Durable Object

**Estado:** Draft
**Epic:** EPIC-06-notifications-system
**Sprint target:** Sprint 3
**Story points:** 3
**Persona:** Admin (anti-spam)
**Owner:** —

---

## Contexto

Anti-spam: cada user tiene caps de notifs por categoría/día/semana
(de `push-notifications-copy-bank.md` §1.5):
- reminder: 1/día, 7/semana.
- achievement: 3/día, 15/semana.
- onboarding: 1/día, 4/semana.
- reengagement: 1/día, 3/semana.
- transactional: 5/día, 20/semana.

Durable Object por user mantiene contadores atomic, consistent.

Backend en
[`notifications-system.md`](../../architecture/notifications-system.md)
§9.

## Scope

### In

- Durable Object `UserNotifRateLimiter`:
  - Storage por user_id.
  - Methods:
    - `checkAndIncrement(category)`: atomic check + increment.
      Returns `{ allowed: bool, current_count, cap }`.
    - `getCounts()`: snapshot de counts actuales.
    - `reset()`: para admin debug.
- Buckets:
  - `daily_${category}_${YYYY-MM-DD}`: count del día.
  - `weekly_${category}_${YYYY-WW}`: count de la semana ISO.
  - Cleanup automático via DO alarm cuando bucket envejece.
- Caps por categoría hardcoded:
  ```typescript
  const RATE_LIMITS = {
    reminder:      { daily: 1, weekly: 7 },
    achievement:   { daily: 3, weekly: 15 },
    onboarding:    { daily: 1, weekly: 4 },
    reengagement:  { daily: 1, weekly: 3 },
    transactional: { daily: 5, weekly: 20 },
  };
  ```
- Integration en `sendFcmNotification` (US-061):
  - Antes de FCM call: `checkAndIncrement(category)`.
  - Si `allowed = false`: skip + telemetry suppress.
- Telemetry: `notif.rate_limit_check`,
  `notif.rate_limit_rejected`.

### Out

- Per-user customizable caps (post-MVP).
- Override caps para Premium (post-MVP).
- Sliding window precise (MVP usa daily/weekly buckets — OK
  approximation).

## Acceptance criteria

- **Given** user sin notifs hoy, **When** checkAndIncrement
  `reminder`, **Then** `allowed: true, current_count: 1, cap: 1`.
- **Given** user ya recibió 1 reminder hoy, **When** check 2do,
  **Then** `allowed: false, current_count: 1, cap: 1`.
- **Given** user con 7 reminders esta semana (semana llena),
  **When** intent 8th, **Then** rechaza por weekly cap aún si
  daily cap available.
- **Given** transactional category, **When** check, **Then**
  permite hasta 5/día independent de otros.
- **Given** day rollover (medianoche UTC), **When** check primera
  notif del nuevo día, **Then** counter daily reset a 0.
- **Given** DO alarm dispara después de 8 días, **When** ejecuta,
  **Then** cleanup buckets antiguos (no hoarding storage).
- **Given** 10 concurrent checks para mismo user mismo category,
  **When** Promise.all, **Then** exactly cap número de allows
  (atomic).
- **Given** sendFcmNotification con rate_limit_exceeded, **When**
  procesa, **Then** log + skip + count para `notif.rate_limit_rejected`.

## Developer details

### Owning service

`apps/workers/notifications/src/utils/rate-limiter-do.ts` +
`apps/workers/notifications/src/utils/rate-limit-helper.ts`.

### Dependencies

- US-061 (consumer principal).

### Specs referenciados

- [`notifications-system.md`](../../architecture/notifications-system.md)
  §9.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §1.5.

### Implementación esperada

```typescript
// apps/workers/notifications/src/utils/rate-limiter-do.ts
const RATE_LIMITS: Record<string, { daily: number; weekly: number }> = {
  reminder:      { daily: 1, weekly: 7 },
  achievement:   { daily: 3, weekly: 15 },
  onboarding:    { daily: 1, weekly: 4 },
  reengagement:  { daily: 1, weekly: 3 },
  transactional: { daily: 5, weekly: 20 },
};

export class UserNotifRateLimiter implements DurableObject {
  state: DurableObjectState;

  constructor(state: DurableObjectState) {
    this.state = state;
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);
    if (url.pathname === '/check') {
      const category = url.searchParams.get('category')!;
      return this.checkAndIncrement(category);
    }
    if (url.pathname === '/snapshot') {
      return this.getCounts();
    }
    return new Response('Not Found', { status: 404 });
  }

  async checkAndIncrement(category: string): Promise<Response> {
    const limits = RATE_LIMITS[category];
    if (!limits) {
      return jsonResponse({ allowed: false, error: 'unknown_category' }, 400);
    }

    const today = new Date().toISOString().slice(0, 10);
    const weekISO = getISOWeek(new Date());
    const dailyKey = `daily_${category}_${today}`;
    const weeklyKey = `weekly_${category}_${weekISO}`;

    const dailyCount = (await this.state.storage.get<number>(dailyKey)) ?? 0;
    const weeklyCount = (await this.state.storage.get<number>(weeklyKey)) ?? 0;

    if (dailyCount >= limits.daily || weeklyCount >= limits.weekly) {
      return jsonResponse({
        allowed: false,
        current_count: { daily: dailyCount, weekly: weeklyCount },
        cap: limits,
      });
    }

    await this.state.storage.put(dailyKey, dailyCount + 1);
    await this.state.storage.put(weeklyKey, weeklyCount + 1);

    // Schedule cleanup alarm
    await this.state.storage.setAlarm(Date.now() + 8 * 24 * 60 * 60 * 1000);

    return jsonResponse({
      allowed: true,
      current_count: { daily: dailyCount + 1, weekly: weeklyCount + 1 },
      cap: limits,
    });
  }

  async alarm(): Promise<void> {
    // Cleanup buckets older than 8 days
    const today = new Date();
    const cutoff = new Date(today.getTime() - 8 * 24 * 60 * 60 * 1000);
    const keys = await this.state.storage.list({ prefix: 'daily_' });

    for (const [key] of keys) {
      const dateStr = key.split('_').pop();
      if (dateStr && new Date(dateStr) < cutoff) {
        await this.state.storage.delete(key);
      }
    }
    // Weekly cleanup similar
  }
}

// Helper for sendFcmNotification
export async function checkRateLimit(
  userId: string, category: string, env: Env
): Promise<boolean> {
  const id = env.NOTIF_RATE_LIMITER.idFromName(userId);
  const stub = env.NOTIF_RATE_LIMITER.get(id);
  const response = await stub.fetch(
    `https://internal/check?category=${category}`
  );
  const { allowed } = await response.json();
  if (!allowed) {
    track('notif.rate_limit_rejected', { user_id: userId, category });
  }
  track('notif.rate_limit_check', { allowed });
  return allowed;
}
```

### Wrangler binding

```toml
[[durable_objects.bindings]]
name = "NOTIF_RATE_LIMITER"
class_name = "UserNotifRateLimiter"
```

### Integration con US-061

```typescript
// En sendFcmNotification, before FCM call:
const allowed = await checkRateLimit(job.user_id, job.category, env);
if (!allowed) {
  return { delivered: 0, failed: 0, suppressed: true };
}
```

### Integration points

- US-061 (consumer principal).
- Durable Object storage.

### Notas técnicas

- ISO week calculation: estándar Lun-Dom o Dom-Sab depende de
  locale. MVP usa ISO 8601 (Lun-Dom).
- Atomicidad: DO garantiza single-threaded por user_id — no race.
- Storage cost: ~10 keys per user max → despreciable.

## Definition of Done

- [ ] Durable Object implementado.
- [ ] Helper `checkRateLimit` exportado.
- [ ] Integration con US-061 (sendFcmNotification).
- [ ] Cleanup alarm funcional.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration (concurrent checks).
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Bloqueante para production (sin esto, spam risk alto).*

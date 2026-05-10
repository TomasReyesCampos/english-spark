# EPIC-06: Notifications system orchestrator

**Status:** Draft
**Sprint target:** Sprint 2-3
**Owner:** —
**Estimación total:** ~38 puntos (13 stories aprox.)

---

## Resumen

Implementar el orchestrator de notificaciones push del MVP usando
Firebase Cloud Messaging (FCM) directo. Componentes:

1. Schema de tokens + preferences + log + scheduled.
2. FCM HTTP v1 sender con OAuth access token caching.
3. Crons timezone-aware:
   - Daily reminder (cada hora chequea quién está en su hora
     preferida).
   - Streak at risk (4h antes de medianoche TZ del user).
   - Inactivity detector (D3, D7, D14).
   - Welcome series (D1, D3, D5, D7 post-signup).
4. Event-driven handlers:
   - Achievement unlocked.
   - Sparks events (low/depleted/expiring).
   - Payment events (failed, restriction).
5. Rate limiter via Durable Object.
6. Circuit breaker para users low-engagement.

Este epic **consume los events** que EPIC-01, EPIC-04 y future
EPIC-03 emiten. Sin él, esos events quedan al vacío.

---

## Por qué este epic ahora

- Aprendizaje de idiomas vive de **consistencia diaria**. Push
  notifications son el mecanismo principal de retention.
- Múltiples events ya emitidos (US-008, US-046, US-050, US-051,
  US-055, US-057) esperan consumer.
- Sin notifs, retention D2-D7 caería significativamente.
- Pre-production checklist: imposible go-live sin notifs.

---

## Specs autoritativos referenciados

| Spec | Cubre |
|------|-------|
| [`architecture/notifications-system.md`](../../architecture/notifications-system.md) | Sistema completo FCM + crons + handlers |
| [`product/push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md) | Copys literales mexicano-tuteo para los 15 notification_ids + 80 logros |
| [`decisions/ADR-004-notifications-fcm.md`](../../decisions/ADR-004-notifications-fcm.md) | Decisión FCM directo (no Expo Push) |

---

## Stories del epic

### Slice 1: Foundation (~9 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-059 | Schema notifications (tokens + preferences + log + scheduled) | 3 |
| US-060 | Token registration + preferences endpoints | 3 |
| US-061 | FCM HTTP v1 sender + OAuth token caching | 3 |

### Slice 2: Daily flows (~9 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-062 | Cron daily reminders timezone-aware | 5 |
| US-063 | AI batch nocturno content generation | 2 |
| US-064 | Streak at risk cron | 2 |

### Slice 3: Event-driven handlers (~10 pts, 4 stories)

| US | Título | Pts |
|----|--------|----:|
| US-065 | Achievement unlocked notification handler | 3 |
| US-066 | Sparks events handler (low/depleted/expiring) | 3 |
| US-067 | Payment events handler (failed/restriction) | 2 |
| US-068 | Welcome series cron (D1/D3/D5/D7) | 2 |

### Slice 4: Resilience + ops (~10 pts, 3 stories)

| US | Título | Pts |
|----|--------|----:|
| US-069 | Inactivity detection (D3/D7/D14) | 3 |
| US-070 | Rate limiter Durable Object | 3 |
| US-071 | Circuit breaker para low-engagement users | 2 |

---

## Total estimado

~38 puntos en 13 stories. Asumiendo velocity 20-25 pts/sprint:
**~2 sprints**, paralelo a EPIC-02/03.

---

## Dependencies externas

- US-001: Firebase Cloud Messaging habilitado (ya en EPIC-01).
- `FIREBASE_SERVICE_ACCOUNT` secret configurado.
- Cloudflare Cron Triggers.

---

## Dependencies inter-EPIC

- **EPIC-01:** US-008 emit `user.signed_up` → US-068 welcome
  series. US-023 registra token FCM.
- **EPIC-04:** US-051 emit `sparks.balance_low` → US-066.
  US-057 emit `sparks.expiring_soon` → US-066. US-053/054
  emit `payment.failed` → US-067.
- **EPIC-05:** US-036 task `generate_notification_content` →
  US-063 batch nocturno.

---

## Definition of Done del epic

- [ ] Push notifications llegan a iOS + Android en device real.
- [ ] Token registration funcional.
- [ ] Preferences endpoint con opt-out por categoría.
- [ ] Daily reminders enviados en hora preferida TZ del user.
- [ ] Streak at risk envía 4h antes del corte.
- [ ] Welcome series D1/D3/D5/D7 funcional.
- [ ] Achievement unlocked dispara push.
- [ ] Sparks/payment events disparan transactional notifs.
- [ ] Inactivity D3/D7/D14 con tono diferenciado.
- [ ] Rate limiting respeta los caps de `push-notifications-copy-bank.md` §1.5.
- [ ] Circuit breaker pausa users con bajo engagement.

---

## Riesgos identificados

| Riesgo | Mitigación |
|--------|------------|
| Token inválido en FCM | Mark `is_active=false` automáticamente |
| Spam de notifs durante development | Rate limit + manual kill switch |
| Drift de copy IA vs templates | Validation pipeline US-036 fallback |
| Timezone wrong de user | Default America/Mexico_City + override en US-060 |
| FCM downtime | Retry budget + alerts |

---

## A/B tests post-MVP

(Coordinar con
[`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
§11.)

1. Emoji en título daily_reminder vs sin.
2. Mencionar focus_today vs solo "tu plan".
3. Streak_at_risk 4h vs 2h antes.
4. inactivity_d14 con 20 sparks vs sin oferta.
5. AI-generated daily vs template.

---

*Documento vivo. Actualizar cuando se agreguen nuevos
notification_ids o crons.*

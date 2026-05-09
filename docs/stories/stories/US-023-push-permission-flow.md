# US-023: Push permission flow

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 2
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Pantalla 15 del onboarding (post-roadmap reveal): solicitar push
permission. Decisión cerrada en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §19.4:
"Permisos push DESPUÉS del reveal: Sí" — user ya vio valor (su
plan), ahora se pide push como herramienta para mantenerlo.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §16.

## Scope

### In

- Pantalla con copy literal:
  - Hero: "¿Te aviso cada día?"
  - Body: "El inglés vive de la consistencia. Te aviso una vez al
    día a la hora que elijas."
  - Selector de hora preferida (time picker, default 19:00 hora
    local).
  - Primary CTA: "Activar recordatorios"
  - Secondary CTA: "Después" (tertiary visual).
- Tap "Activar recordatorios":
  - Persiste hora preferida en
    `user_notification_preferences.preferred_reminder_hour`.
  - Trigger native push permission prompt.
  - Si granted: registra FCM token (US futura, `notifications-system.md`
    §6.1) — para esta story es OK invocar la función pero no
    bloquear si falla.
  - Navega a US-024.
- Tap "Después":
  - `push_enabled = false` persistido.
  - Navega a US-024 (no obligatorio para continuar).
- Si user **deniega** native prompt:
  - Persiste `push_enabled = false`.
  - Navega a US-024 sin error.
- Telemetry: `onboarding.push_permission_seen`,
  `.push_hour_selected`, `.push_permission_granted`,
  `.push_permission_denied`, `.push_postponed`.

### Out

- FCM token registration completa (story propia,
  `notifications-system.md`).
- Banco de copys de notifications (ya cubierto por
  `push-notifications-copy-bank.md`).
- Recovery flow si user deniega y luego quiere activar (Settings,
  post-MVP).

## Acceptance criteria

- **Given** user llega de US-022, **When** se renderiza, **Then**
  ve hero, body, time picker default 19:00, y 2 CTAs.
- **Given** user cambia hora a 21:00 y tapea "Activar
  recordatorios", **When** native prompt aparece, **Then**
  `preferred_reminder_hour = 21` se persiste antes del prompt.
- **Given** user permite push, **When** OS retorna granted, **Then**
  `push_enabled = true` y telemetry
  `push_permission_granted` se emite.
- **Given** user deniega push, **When** OS retorna denied, **Then**
  `push_enabled = false` se persiste y user navega a US-024 sin
  error.
- **Given** user tapea "Después", **When** confirma, **Then**
  `push_enabled = false`, telemetry
  `push_postponed` se emite, navega a US-024.
- **Given** user en Android con notifications globalmente
  desactivadas en SO, **When** intenta activar, **Then** se
  trata como denied y se persiste `push_enabled = false`.
- **Given** persistencia de preferences falla, **When** request
  falla, **Then** user ve toast error pero puede seguir (no se
  bloquea el onboarding).

## Developer details

### Owning service

Mobile app + endpoint
`PATCH /notifications/preferences`.

### Dependencies

- US-022: precede.
- US-008: profile sync (asume user existe).
- Schema `user_notification_preferences` (parte de
  `notifications-system.md` §5.2; story propia o asumido en
  US-002).

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §16.
- [`notifications-system.md`](../../architecture/notifications-system.md)
  §6.1 — setupNotifications client function.
- [`notifications-system.md`](../../architecture/notifications-system.md)
  §10.2 — endpoint `/notifications/preferences`.

### Implementación esperada

```typescript
// app/screens/onboarding/PushPermission.tsx
import messaging from '@react-native-firebase/messaging';

export function PushPermissionScreen() {
  const [hour, setHour] = useState(19);

  useEffect(() => { track('onboarding.push_permission_seen'); }, []);

  const handleEnable = async () => {
    track('onboarding.push_hour_selected', { hour });

    // Persist preferences first
    await api.patch('/notifications/preferences', {
      preferred_reminder_hour: hour,
    });

    // Request OS permission
    const authStatus = await messaging().requestPermission();
    const enabled =
      authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
      authStatus === messaging.AuthorizationStatus.PROVISIONAL;

    if (enabled) {
      track('onboarding.push_permission_granted');
      await api.patch('/notifications/preferences', { push_enabled: true });

      // Try to register token (best-effort, no blocker)
      try {
        const token = await messaging().getToken();
        await api.post('/notifications/register-token', {
          token,
          platform: Platform.OS,
        });
      } catch (e) {
        console.warn('Token register failed (non-blocking)', e);
      }
    } else {
      track('onboarding.push_permission_denied');
      await api.patch('/notifications/preferences', { push_enabled: false });
    }

    navigation.navigate('FirstExerciseIntro');
  };

  const handleSkip = async () => {
    track('onboarding.push_postponed');
    await api.patch('/notifications/preferences', { push_enabled: false });
    navigation.navigate('FirstExerciseIntro');
  };

  return <UI ... />;
}
```

### Integration points

- Firebase Cloud Messaging (request permission, getToken).
- `/notifications/preferences` endpoint (parcialmente cubierto en
  story de notifications-system; puede requerir story propia para
  el endpoint si no está lista).
- US-024 (siguiente).

### Notas técnicas

- iOS: `Info.plist` no necesita config extra (handled by Firebase
  Messaging package).
- Android 13+: native permission prompt obligatorio (antes era
  auto-granted).
- Token register failure NO debe bloquear el onboarding — best
  effort.

## Definition of Done

- [ ] Pantalla con time picker + 2 CTAs.
- [ ] Native permission prompt funcional iOS + Android.
- [ ] Persistencia de preferences.
- [ ] FCM token register best-effort.
- [ ] 7 acceptance criteria pasan.
- [ ] Tests unit del componente.
- [ ] Test integration con mocks.
- [ ] QA manual iOS + Android.
- [ ] Validation contra spec `onboarding-flow.md` §16.
- [ ] PR aprobada y mergeada.

---

*Depende de US-022. Precede US-024.*

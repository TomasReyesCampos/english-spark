# US-016: Mic permission flow + recovery

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante
**Owner:** —

---

## Contexto

El producto es **speaking-first** — sin micrófono no hay producto.
Esta pantalla solicita permission de mic **antes** del mini-test
(que requiere grabar audio).

iOS y Android tienen flows distintos para mic permission. Si user
deniega: necesitamos recovery flow elegante (deep link a Settings
del SO + explicación clara).

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §9.

## Scope

### In

- Pantalla con explicación pre-prompt (§9.3 spec):
  - Hero: "Necesitamos tu micrófono"
  - Body: "Para evaluar tu pronunciación grabamos audios cortos
    cuando hablas. Se borran automáticamente después de 30 días."
  - 3 bullet points: privacy + retention + opt-out.
  - CTA: "Permitir acceso al micrófono"
- Tap CTA → trigger native permission prompt (iOS/Android).
- Si user **permite:** persiste
  `student_profiles.consent_audio_processing = true` y navega a
  US-017 (primer ejercicio mini-test).
- Si user **deniega:** recovery flow:
  - Pantalla "Sin micrófono no podemos evaluar tu inglés."
  - Botón primario "Abrir Settings" (deep link iOS / Android).
  - Botón secundario "Continuar sin mini-test" (skip; se asume
    `self_perceived_level` como CEFR inicial).
- Detección de status de permission al volver de Settings:
  - Si concedido: continúa al mini-test.
  - Si sigue denied: vuelve a recovery o continúa con skip.
- Telemetry: `onboarding.mic_explanation_seen`,
  `.mic_permission_requested`, `.mic_permission_granted`,
  `.mic_permission_denied`, `.mic_recovery_settings_opened`,
  `.mic_skip_mini_test`.

### Out

- Mini-test ejercicios en sí (US-017 a US-019).
- Settings UI para revocar permission post-onboarding (post-MVP).
- Retroactive recovery (user que negó pero quiere usar mic
  después).

## Acceptance criteria

- **Given** user llega a pantalla mic permission, **When** se
  renderiza, **Then** ve hero, body, bullets, y CTA "Permitir
  acceso al micrófono".
- **Given** user tapea CTA, **When** OS muestra prompt nativo,
  **Then** user ve el prompt iOS/Android estándar.
- **Given** user permite mic, **When** OS retorna granted, **Then**
  `consent_audio_processing = true`, telemetry
  `mic_permission_granted` se emite, navega a US-017.
- **Given** user deniega, **When** OS retorna denied, **Then** ve
  pantalla de recovery con copy explicativo + 2 botones.
- **Given** user en recovery tapea "Abrir Settings", **When**
  ejecuta deep link, **Then** abre Settings de la app en SO.
- **Given** user vuelve de Settings con permission concedido,
  **When** la app retoma foco, **Then** detecta el cambio y navega
  a US-017 automáticamente.
- **Given** user vuelve de Settings sin cambiar permission, **When**
  retoma la app, **Then** ve recovery screen otra vez.
- **Given** user tapea "Continuar sin mini-test", **When** confirma,
  **Then** `consent_audio_processing = false`, mini-test se skip,
  navega a US-021 (procesamiento) con `initial_test_results = null`
  y `initial_test_results.cefr_estimate` derivado de
  `self_perceived_level`.
- **Given** user en Android (donde permission "Don't ask again" es
  posible), **When** seleccionan eso, **Then** flow recovery se
  comporta igual (botón "Abrir Settings" sigue siendo el camino).

## Developer details

### Owning service

Mobile app (React Native).

### Dependencies

- US-014: precede en el flow.
- US-015: endpoint para persistir consent.
- Native packages: `react-native-permissions` para detección
  uniforme cross-platform.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §9 —
  pantalla mic permission + recovery flow.
- [`legal-compliance.md`](../../business/legal-compliance.md) §7.2 —
  consent audio processing requirements.

### Implementación esperada

```typescript
// app/screens/onboarding/MicPermission.tsx
import { check, request, RESULTS, openSettings } from 'react-native-permissions';
import { Platform } from 'react-native';

const MIC_PERMISSION = Platform.OS === 'ios'
  ? PERMISSIONS.IOS.MICROPHONE
  : PERMISSIONS.ANDROID.RECORD_AUDIO;

export function MicPermissionScreen() {
  const [phase, setPhase] = useState<'explain'|'recovery'>('explain');

  useEffect(() => { track('onboarding.mic_explanation_seen'); }, []);

  const handleRequest = async () => {
    track('onboarding.mic_permission_requested');
    const result = await request(MIC_PERMISSION);

    if (result === RESULTS.GRANTED) {
      track('onboarding.mic_permission_granted');
      await api.patch('/profile/update', { consent_audio_processing: true });
      navigation.navigate('MiniTest1');
    } else {
      track('onboarding.mic_permission_denied');
      setPhase('recovery');
    }
  };

  const handleOpenSettings = () => {
    track('onboarding.mic_recovery_settings_opened');
    openSettings();
  };

  const handleSkip = async () => {
    track('onboarding.mic_skip_mini_test');
    await api.patch('/profile/update', { consent_audio_processing: false });
    navigation.navigate('Processing', { skipped_mini_test: true });
  };

  // Re-check permission cuando app retoma foco
  useEffect(() => {
    const subscription = AppState.addEventListener('change', async (state) => {
      if (state === 'active' && phase === 'recovery') {
        const status = await check(MIC_PERMISSION);
        if (status === RESULTS.GRANTED) {
          await api.patch('/profile/update', { consent_audio_processing: true });
          navigation.navigate('MiniTest1');
        }
      }
    });
    return () => subscription.remove();
  }, [phase]);

  return phase === 'explain' ? <ExplainView /> : <RecoveryView />;
}
```

### Copy de recovery (mexicano-tuteo)

```
Hero: "Sin micrófono no podemos evaluar tu inglés."

Body: "Te perdés la parte más importante: cómo suenas hablando.
Puedes activar el micrófono en Settings y volver, o continuar
sin el mini-test (tu nivel se asume del que dijiste antes)."

Primary CTA: "Abrir Settings"
Secondary CTA: "Continuar sin mini-test"
```

(Nota: copy revisado para tuteo en spec onboarding-flow.md §9.4.)

### Integration points

- Native OS permission system.
- `/profile/update` para persistir consent.
- US-017 to US-019 (mini-test ejercicios).
- US-021 (procesamiento screen).

### Notas técnicas

- iOS: `Info.plist` requires `NSMicrophoneUsageDescription` con
  copy en español: "Necesitamos el micrófono para evaluar tu
  pronunciación durante los ejercicios."
- Android: `AndroidManifest.xml` requires
  `android.permission.RECORD_AUDIO`.
- Deep link a Settings es estándar
  (`react-native-permissions.openSettings()`).
- Detección post-Settings vía `AppState` change listener.

## Definition of Done

- [ ] Pantalla explain + recovery implementadas.
- [ ] Native permission prompt iOS + Android funcionando.
- [ ] Deep link a Settings funcional.
- [ ] Detección de cambio post-Settings (AppState listener).
- [ ] Skip flow funciona y propaga `skipped_mini_test` al
  procesamiento.
- [ ] 9 acceptance criteria pasan.
- [ ] Tests unit del componente.
- [ ] Test integration end-to-end (granted, denied → recovery,
  recovery → settings → granted, skip).
- [ ] QA manual iOS device + Android device.
- [ ] Screenshot/video.
- [ ] Validation contra spec `onboarding-flow.md` §9 +
  `legal-compliance.md` §7.2.
- [ ] PR aprobada y mergeada.

---

*Depende de US-014, US-015. Bloqueante para US-017.*

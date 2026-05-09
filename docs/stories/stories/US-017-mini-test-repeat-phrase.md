# US-017: Mini-test ejercicio 1 (repetir frase)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Primer ejercicio del mini-test del onboarding (Pantalla 11 de
spec). User escucha una frase nativa y la repite. Mide
**pronunciation base** sin frustración (frase es B1 amistosa).

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §11.
Backend en
[`pedagogical-system.md`](../../product/pedagogical-system.md) §3.1
y
[`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
§4.2 (mini-test).

## Scope

### In

- Pantalla con UI:
  - Indicator "Ejercicio 1 de 3".
  - Frase target visible: "I would love to travel to Japan
    someday."
  - Botón "Escuchar" (reproduce TTS audio).
  - Botón "Grabar" (record user audio).
  - Indicador de grabación (waveform o pulse).
  - Botón "Siguiente" (disabled hasta que user grabe ≥ 1.5s).
- Audio TTS atomic referenciado:
  `audio_minitest_phrase_japan_v1` (a generar con voice neutral
  US — US-001 voice pool).
- User puede escuchar la frase múltiples veces (sin penalización).
- User puede regrabar antes de tapear "Siguiente".
- Audio uploaded a R2 con storage key
  `users/{user_id}/onboarding/minitest_1.mp3`.
- Llamada a backend
  `POST /pedagogical/submit-minitest-attempt`:
  - Recibe audio_url + exercise_index = 1.
  - Trigger Azure Pronunciation Assessment.
  - Persiste partial result en `student_profile.initial_test_results.exercise_1`.
- Telemetry: `minitest.ex1_seen`, `.ex1_phrase_listened`,
  `.ex1_recording_started`, `.ex1_recording_completed`,
  `.ex1_submitted`.

### Out

- Ejercicios 2 y 3 (US-018, US-019).
- Cálculo del CEFR estimate combinado (US-020 procesamiento).
- Streaming/realtime feedback durante la grabación (post-MVP).

## Acceptance criteria

- **Given** user llega a Mini-Test ejercicio 1 con mic permission
  granted, **When** se renderiza, **Then** ve la frase, botón
  "Escuchar" y botón "Grabar".
- **Given** user tapea "Escuchar", **When** TTS audio se reproduce,
  **Then** scrubbing visual sincroniza con el audio (~3-4 segundos).
- **Given** user tapea "Grabar" y habla, **When** completa
  ≥1.5 segundos de audio, **Then** botón "Siguiente" se habilita.
- **Given** user tapea "Grabar" y silence (< 0.5s útil), **When**
  termina, **Then** ve "No detectamos tu voz. Acercá el teléfono e
  intenta de nuevo." y "Siguiente" sigue disabled.
- **Given** user terminó la primera grabación pero tapea "Grabar"
  otra vez, **When** confirma, **Then** descarta la primera y
  graba nueva.
- **Given** user tapea "Siguiente", **When** request a
  `/pedagogical/submit-minitest-attempt` se inicia, **Then** ve
  loading state breve (max 2s) y navega a US-018.
- **Given** request falla (network o backend), **When** loading,
  **Then** retry automático 1x; si falla 2x: muestra error con
  retry manual.
- **Given** user cierra la app mid-grabación, **When** vuelve a
  abrir, **Then** retoma desde el ejercicio 1 sin la grabación
  parcial (audio se descarta).

## Developer details

### Owning service

Mobile app + backend
`/pedagogical/submit-minitest-attempt`.

### Dependencies

- US-016: mic permission granted.
- US-015: endpoint profile.
- Atomic `audio_minitest_phrase_japan_v1` generado y aprobado en
  catalog (parte del seed de US-001 setup).
- Cloudflare R2 bucket configurado para audios.
- AI Gateway task `score_pronunciation` registrada (Azure
  Pronunciation Assessment).

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §11.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §4.2 — mini-test estructura.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §3.1 — pronunciation scoring.
- [`atomics-catalog-seed.md`](../../product/atomics-catalog-seed.md)
  §3 — voice pool TTS.

### Implementación esperada

```typescript
// app/screens/onboarding/MiniTest1.tsx
import { Audio } from 'expo-av';

const TARGET_PHRASE = "I would love to travel to Japan someday.";
const TARGET_AUDIO_ATOMIC = 'audio_minitest_phrase_japan_v1';

export function MiniTest1Screen() {
  const [recording, setRecording] = useState<Audio.Recording | null>(null);
  const [recordedUri, setRecordedUri] = useState<string | null>(null);
  const [recordedDuration, setRecordedDuration] = useState(0);

  useEffect(() => { track('minitest.ex1_seen'); }, []);

  const startRecording = async () => {
    const { recording } = await Audio.Recording.createAsync(
      Audio.RecordingOptionsPresets.HIGH_QUALITY
    );
    setRecording(recording);
    track('minitest.ex1_recording_started');
  };

  const stopRecording = async () => {
    if (!recording) return;
    await recording.stopAndUnloadAsync();
    const uri = recording.getURI()!;
    const status = await recording.getStatusAsync();
    setRecordedUri(uri);
    setRecordedDuration(status.durationMillis ?? 0);
    setRecording(null);
    track('minitest.ex1_recording_completed', { duration_ms: status.durationMillis });
  };

  const handleSubmit = async () => {
    if (recordedDuration < 1500) {
      showToast('No detectamos tu voz. Acércate al teléfono e intenta de nuevo.');
      return;
    }
    track('minitest.ex1_submitted');

    // Upload to R2 via signed URL
    const uploadUrl = await api.post('/storage/sign-upload', {
      key: `users/me/onboarding/minitest_1.mp3`,
      content_type: 'audio/mpeg',
    });
    await fetch(uploadUrl.url, {
      method: 'PUT',
      body: await fetch(recordedUri!).then(r => r.blob()),
      headers: { 'Content-Type': 'audio/mpeg' },
    });

    // Submit to scoring
    await api.post('/pedagogical/submit-minitest-attempt', {
      exercise_index: 1,
      target_text: TARGET_PHRASE,
      target_audio_atomic_id: TARGET_AUDIO_ATOMIC,
      audio_storage_key: `users/me/onboarding/minitest_1.mp3`,
      duration_ms: recordedDuration,
    });

    navigation.navigate('MiniTest2');
  };

  return <UI ... />;
}
```

### Endpoint backend

```typescript
// apps/workers/api/handlers/pedagogical-minitest.ts
export async function handleSubmitMiniTestAttempt(
  request: AuthedRequest, env: Env
) {
  const authError = await validateFirebaseJwt(request, env);
  if (authError) return authError;

  const body = await request.json();
  const { exercise_index, target_text, audio_storage_key, duration_ms } = body;

  // Trigger pronunciation scoring via AI Gateway
  const score = await env.AI_GATEWAY.invoke('score_pronunciation', {
    audio_url: getR2SignedUrl(audio_storage_key, env),
    target_text,
  });

  // Persist partial result
  await db.updateProfile(request.user!.firebase_uid, {
    [`initial_test_results.exercise_${exercise_index}`]: {
      audio_storage_key,
      duration_ms,
      pronunciation_score: score.overall,
      phoneme_scores: score.phonemes,
      submitted_at: new Date().toISOString(),
    },
  });

  return jsonResponse({ ok: true });
}
```

### Integration points

- Audio recording: `expo-av` o
  `react-native-audio-recorder-player`.
- R2 upload: signed URLs generated by Worker.
- AI Gateway: `score_pronunciation` task.
- Profile update.

### Notas técnicas

- Audio format: MP3 22050Hz mono (matchea TTS atomics del catalog).
- Duración mínima útil: 1.5s. Si menor, prompt de retry.
- Audio se sube **sync** antes de navegar — UX simple, max 2s en
  conexión razonable.
- Atomic `audio_minitest_phrase_japan_v1` debe estar pre-generado y
  aprobado en `media_atomics` antes del release.

## Definition of Done

- [ ] Pantalla con audio playback + recording funcional.
- [ ] Atomic `audio_minitest_phrase_japan_v1` creado y aprobado.
- [ ] R2 upload via signed URL.
- [ ] Backend endpoint
  `/pedagogical/submit-minitest-attempt` con scoring.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit del componente + handler.
- [ ] Test integration end-to-end (record → upload → score
  → next).
- [ ] QA manual iOS + Android.
- [ ] Validation contra spec `onboarding-flow.md` §11 +
  `pedagogical-system.md` §3.1.
- [ ] PR aprobada y mergeada.

---

*Depende de US-016. Bloqueante para US-018.*

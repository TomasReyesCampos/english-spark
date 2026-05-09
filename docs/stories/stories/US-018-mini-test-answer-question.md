# US-018: Mini-test ejercicio 2 (responder pregunta)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Segundo ejercicio del mini-test (Pantalla 12). User responde libre
a una pregunta abierta tipo "How are you doing today?" durante 30
segundos. Mide **fluidez espontánea** (WPM, pausas, fillers,
longitud de respuesta).

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §12.
Backend en
[`pedagogical-system.md`](../../product/pedagogical-system.md) §3.2
(fluency scoring).

## Scope

### In

- Pantalla con UI:
  - Indicator "Ejercicio 2 de 3".
  - Pregunta visible: "How are you doing today?"
  - Audio TTS opcional reproducible una vez (atomic
    `audio_minitest_question_how_are_you_v1`).
  - Timer visible con cuenta regresiva 30 segundos.
  - Botón "Empezar a hablar" (start recording).
  - Indicador de grabación en vivo + tiempo restante.
  - Auto-stop al llegar a 30s.
  - Botón "Listo" para terminar antes de los 30s (min 5s).
  - Botón "Siguiente" disabled hasta que user grabe ≥ 5s.
- Audio uploaded a R2:
  `users/{user_id}/onboarding/minitest_2.mp3`.
- Llamada `POST /pedagogical/submit-minitest-attempt` con
  `exercise_index = 2`. Backend trigger:
  - Whisper transcription.
  - Fluency scoring (WPM, pausas, fillers).
  - Coherence light (verifica que la respuesta es relevante a la
    pregunta).
- Telemetry: `minitest.ex2_seen`, `.ex2_question_listened`,
  `.ex2_recording_started`, `.ex2_recording_completed`,
  `.ex2_submitted`.

### Out

- Análisis semántico profundo (post-MVP).
- Hint si user no sabe qué decir (post-MVP).

## Acceptance criteria

- **Given** user llega al ejercicio 2, **When** se renderiza,
  **Then** ve la pregunta, botón opcional "Escuchar pregunta" y
  botón "Empezar a hablar".
- **Given** user tapea "Empezar a hablar", **When** comienza la
  grabación, **Then** timer arranca cuenta regresiva 30s y
  indicador visual muestra grabación activa.
- **Given** grabación en curso, **When** llega a 30s, **Then**
  auto-stop ocurre y "Siguiente" se habilita.
- **Given** user tapea "Listo" antes de 30s con ≥ 5s grabados,
  **When** confirma, **Then** grabación se detiene y "Siguiente"
  se habilita.
- **Given** user tapea "Listo" con < 5s, **When** intenta, **Then**
  ve "Hablá un poco más para que podamos evaluarte." y botón
  permanece pressing-only para >5s.
- **Given** user grabó pero quiere regrabar, **When** tapea
  "Empezar de nuevo", **Then** descarta el audio y permite nueva
  grabación desde 30s.
- **Given** "Siguiente" tapado, **When** upload + submit terminan,
  **Then** navega a US-019 con loading <2s.
- **Given** user habla algo no inglés (ej: en español), **When**
  Whisper detecta, **Then** se persiste el resultado pero con flag
  `language_mismatch = true` (informa al procesamiento sin bloquear).

## Developer details

### Owning service

Mobile app + backend mismo endpoint que US-017.

### Dependencies

- US-017: precede.
- Atomic `audio_minitest_question_how_are_you_v1`.
- AI Gateway tasks: `transcribe_user_audio` (Whisper) +
  `score_fluency`.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §12.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §3.2 — fluency scoring.

### Implementación esperada

(Pattern similar a US-017 pero con timer y free-form recording.)

```typescript
// app/screens/onboarding/MiniTest2.tsx
const QUESTION_TEXT = "How are you doing today?";
const QUESTION_AUDIO = 'audio_minitest_question_how_are_you_v1';
const MAX_DURATION_MS = 30_000;
const MIN_DURATION_MS = 5_000;

export function MiniTest2Screen() {
  const [timer, setTimer] = useState(MAX_DURATION_MS);
  const [recording, setRecording] = useState<Audio.Recording | null>(null);
  const intervalRef = useRef<any>();

  const startRecording = async () => {
    const { recording } = await Audio.Recording.createAsync(...);
    setRecording(recording);
    track('minitest.ex2_recording_started');

    intervalRef.current = setInterval(() => {
      setTimer(t => {
        if (t <= 100) {
          stopRecording(); // auto-stop
          return 0;
        }
        return t - 100;
      });
    }, 100);
  };

  // ... resto similar a US-017 con duration_ms validation 5_000-30_000
}
```

### Backend extension

El endpoint
`/pedagogical/submit-minitest-attempt` para `exercise_index = 2`:

```typescript
// Para ejercicio 2 (free response):
const transcript = await env.AI_GATEWAY.invoke('transcribe_user_audio', {
  audio_url: getR2SignedUrl(audio_storage_key, env),
});

const fluencyScore = await env.AI_GATEWAY.invoke('score_fluency', {
  transcript: transcript.text,
  audio_url: getR2SignedUrl(audio_storage_key, env),
});

await db.updateProfile(uid, {
  'initial_test_results.exercise_2': {
    audio_storage_key,
    duration_ms,
    transcript: transcript.text,
    detected_language: transcript.language,
    fluency_score: fluencyScore.overall,
    wpm: fluencyScore.wpm,
    filler_count: fluencyScore.fillers,
    language_mismatch: transcript.language !== 'en',
    submitted_at: new Date().toISOString(),
  },
});
```

### Integration points

- Whisper via AI Gateway.
- Fluency scoring task.
- Profile update.

## Definition of Done

- [ ] Pantalla con timer 30s + recording funcional.
- [ ] Atomic `audio_minitest_question_how_are_you_v1` aprobado.
- [ ] Backend extiende endpoint para `exercise_index = 2`.
- [ ] Whisper + fluency scoring integrados.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] QA manual.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-017.*

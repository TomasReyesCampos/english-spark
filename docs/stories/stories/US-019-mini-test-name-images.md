# US-019: Mini-test ejercicio 3 (nombrar imágenes)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Tercer y último ejercicio del mini-test (Pantalla 13). User ve 3
imágenes simples y nombra cada una en inglés. Mide **vocabulario
activo** + pronunciation a nivel single-word.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §13.

## Scope

### In

- Pantalla con 3 imágenes secuenciales:
  - Imagen 1: una manzana (`image_minitest_apple_v1`).
  - Imagen 2: un avión (`image_minitest_airplane_v1`).
  - Imagen 3: una computadora portátil (`image_minitest_laptop_v1`).
- Por cada imagen:
  - "¿Qué ves?" como prompt.
  - Botón "Grabar" (record short audio, max 5s).
  - User dice una palabra → audio grabado.
  - Auto-advance al siguiente o al "Siguiente" final.
- Indicator "Imagen 1 de 3", "2 de 3", "3 de 3".
- Audios uploaded a R2:
  `users/{user_id}/onboarding/minitest_3_{1|2|3}.mp3`.
- Llamada
  `POST /pedagogical/submit-minitest-attempt` con
  `exercise_index = 3` (después de las 3 imágenes; un solo submit
  con array de audios).
- Backend trigger:
  - Whisper transcription por cada audio.
  - Match contra expected vocabulary
    (`apple` / `airplane`/`plane` / `laptop`/`computer`).
  - Pronunciation scoring por palabra.
  - Vocabulary score derivado: % palabras correctamente identificadas.
- Telemetry: `minitest.ex3_image_seen` (por imagen, con
  `image_index`), `.ex3_recording_completed`, `.ex3_submitted`.

### Out

- Imágenes adicionales o aleatorias (post-MVP A/B).
- Acepta sinónimos múltiples flexibles (post-MVP); MVP acepta lista
  predefinida.

## Acceptance criteria

- **Given** user llega al ejercicio 3, **When** se renderiza,
  **Then** ve "Imagen 1 de 3", la imagen de la manzana y botón
  "Grabar".
- **Given** user graba "apple" y tapea Listo, **When** procesa,
  **Then** auto-advance a Imagen 2.
- **Given** user graba "manzana" (en español) y tapea Listo,
  **When** procesa, **Then** auto-advance pero el match falla
  (`vocabulary_match = false` para imagen 1).
- **Given** user completa las 3 imágenes, **When** la última
  procesa, **Then** botón "Siguiente" navega a US-020.
- **Given** user no graba nada (silence), **When** tapea Listo,
  **Then** se considera audio inválido y no advanza, mostrando
  "Decí una palabra para continuar".
- **Given** user dice "plane" en imagen 2 (sinónimo aceptado),
  **When** Whisper transcribe, **Then** matchea contra
  `['airplane', 'plane']` y `vocabulary_match = true`.
- **Given** user dice "computer" en imagen 3, **When** matchea,
  **Then** `vocabulary_match = true` (lista acepta laptop, computer,
  notebook).
- **Given** all 3 imágenes completadas, **When** request submit,
  **Then** persiste array de 3 results con scores agregados.

## Developer details

### Owning service

Mobile app + backend mismo endpoint con `exercise_index = 3`.

### Dependencies

- US-018: precede.
- 3 image atomics: `image_minitest_apple_v1`,
  `image_minitest_airplane_v1`, `image_minitest_laptop_v1`.
- Whisper task vía AI Gateway.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §13.
- [`pedagogical-system.md`](../../product/pedagogical-system.md)
  §3.4 — vocabulary scoring.

### Vocabulary acceptance lists

```typescript
const ACCEPTED_VOCABULARY = {
  image_minitest_apple_v1: ['apple', 'an apple'],
  image_minitest_airplane_v1: ['airplane', 'plane', 'aeroplane'],
  image_minitest_laptop_v1: ['laptop', 'computer', 'notebook'],
};

function matchVocabulary(transcript: string, imageId: string): boolean {
  const accepted = ACCEPTED_VOCABULARY[imageId];
  const normalized = transcript.trim().toLowerCase();
  return accepted.some(word => normalized.includes(word));
}
```

### Backend extension

```typescript
// Para exercise_index = 3:
const results = [];
for (const audioEntry of body.audios) {
  const transcript = await env.AI_GATEWAY.invoke('transcribe_user_audio', {
    audio_url: getR2SignedUrl(audioEntry.storage_key, env),
  });
  const matched = matchVocabulary(transcript.text, audioEntry.image_atomic_id);
  results.push({
    image_atomic_id: audioEntry.image_atomic_id,
    transcript: transcript.text,
    vocabulary_match: matched,
    pronunciation_score: matched
      ? await scorePronunciation(transcript, audioEntry.storage_key, env)
      : null,
  });
}

await db.updateProfile(uid, {
  'initial_test_results.exercise_3': {
    images: results,
    vocabulary_score: (results.filter(r => r.vocabulary_match).length / 3) * 100,
    submitted_at: new Date().toISOString(),
  },
});
```

### Integration points

- 3 image atomics aprobados en
  `media_atomics`.
- AI Gateway: Whisper + pronunciation scoring.
- Profile update.

## Definition of Done

- [ ] Pantalla con 3 imágenes secuenciales + grabaciones.
- [ ] 3 image atomics aprobados.
- [ ] Backend extiende endpoint con vocabulary matching.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] QA manual.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-018. Bloqueante para US-020.*

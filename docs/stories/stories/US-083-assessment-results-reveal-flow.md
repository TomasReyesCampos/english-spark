# US-083: Assessment results 8-screen reveal flow

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3
**Story points:** 3
**Persona:** Estudiante (post-assessment)
**Owner:** —

---

## Contexto

UI mobile que muestra los 8 pantallas del reveal post-assessment
(de `post-assessment-flow.md` §2-§9):
1. Procesamiento (5-15s).
2. Reveal CEFR (emocional).
3. Scores por dimensión.
4. Strengths & weaknesses.
5. Roadmap reveal.
6. Paywall.
7. Payment.
8. Welcome to plan.

Spec UX completo en
[`post-assessment-flow.md`](../../product/post-assessment-flow.md).

## Scope

### In

- Pantallas 1-5 (post-finalize, pre-paywall):
  - §1 Procesamiento: animation while definitivo roadmap genera
    (polling `/roadmap/active` hasta `roadmap_type='definitive'`).
  - §2 Reveal CEFR: hero "Tu nivel es **B1+**", animación.
  - §3 Scores: 5 dimension scores con bars.
  - §4 Strengths/Weaknesses: top 3 cada uno.
  - §5 Roadmap reveal: primer level visible + preview borrosa.
- Pantallas 6-7 (paywall + payment): delegado a EPIC-04 +
  post-assessment-flow.md §7-§8 (story propia separada o
  integrada).
- Pantalla 8 Welcome to plan: shown post-payment, links a US-024
  primer ejercicio.
- Telemetry: `assessment_reveal.screen_N_seen`,
  `.cta_tapped`, `.dropped_at`.
- Manejo del broken assessment prompt (de US-082): si
  `low_confidence: true`, mostrar modal "Notamos que..." con
  opción retry.

### Out

- Paywall + payment UI específico (post-MVP story propia o
  cubierta por EPIC-04 stories de UI).
- A/B test variants (post-MVP).

## Acceptance criteria

- **Given** finalize completed, **When** UI carga, **Then**
  pantalla 1 procesamiento aparece con animation.
- **Given** definitive roadmap se genera (~10s), **When** polling
  detecta, **Then** navega a pantalla 2 reveal CEFR.
- **Given** measured_cefr = B1+, **When** pantalla 2 renderiza,
  **Then** muestra hero "Tu nivel es **B1+**" con animation
  number-up.
- **Given** pantalla 3 scores, **When** renderiza, **Then** ve
  5 dimensions con bars y valores numéricos.
- **Given** pantalla 4 strengths/weaknesses, **When** renderiza,
  **Then** ve top 3 de cada (de assessment_results).
- **Given** pantalla 5 roadmap reveal, **When** renderiza, **Then**
  primer level + 1 con preview borrosa (decisión §13.4).
- **Given** broken assessment detected (low_confidence), **When**
  pantalla 2 carga, **Then** modal "Notamos que..." con retry
  CTA antes de continuar.
- **Given** user drops off mid-reveal, **When** vuelve, **Then**
  resume desde pantalla donde quedó (state preservado).

## Developer details

### Owning service

Mobile app (React Native).

### Dependencies

- US-082 finalize endpoint.
- US-073 polling /roadmap/active.
- post-assessment-flow.md spec con copy literal.

### Specs referenciados

- [`post-assessment-flow.md`](../../product/post-assessment-flow.md)
  §2-§9 — 8 pantallas completas con copy.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §6.4.

### Implementación esperada

```tsx
// app/screens/assessment-reveal/index.tsx
const SCREENS = [
  'Processing', 'CefrReveal', 'Scores', 'Strengths',
  'RoadmapReveal', 'Paywall', 'Payment', 'Welcome',
];

export function AssessmentRevealFlow() {
  const [currentIndex, setCurrentIndex] = useState(0);
  const { results } = useAssessmentResults();

  // Pantalla 1: Processing
  if (currentIndex === 0) {
    return <ProcessingScreen onReady={() => setCurrentIndex(1)} />;
  }

  // Pantalla 2: CEFR Reveal + broken prompt
  if (currentIndex === 1 && results.low_confidence) {
    return <BrokenAssessmentPrompt
      onRetry={() => navigation.navigate('AssessmentRestart')}
      onContinue={() => setCurrentIndex(2)} />;
  }

  if (currentIndex === 1) {
    return <CefrRevealScreen cefr={results.measured_cefr_level}
      onNext={() => setCurrentIndex(2)} />;
  }

  if (currentIndex === 2) {
    return <ScoresScreen scores={results.scores}
      onNext={() => setCurrentIndex(3)} />;
  }

  // ... etc
}

function ProcessingScreen({ onReady }: { onReady: () => void }) {
  useEffect(() => {
    track('assessment_reveal.screen_1_seen');
    const interval = setInterval(async () => {
      const { roadmap } = await api.get('/roadmap/active');
      if (roadmap?.roadmap_type === 'definitive') {
        clearInterval(interval);
        onReady();
      }
    }, 2000);
    return () => clearInterval(interval);
  }, []);

  return (
    <Screen>
      <Animation source="processing.lottie" />
      <Text>Analizando tu assessment...</Text>
    </Screen>
  );
}
```

### Copy literal (mexicano-tuteo, del spec)

Ver `post-assessment-flow.md` §2.4-§9.3 para copy exhaustivo de
cada pantalla.

### Integration points

- US-082 finalize results.
- US-073 polling.
- EPIC-04 UI paywall/payment (pantalla 6-7).

### Notas técnicas

- Animation en pantalla 1: lottie o equivalente RN.
- Polling cada 2s, timeout 30s. Si timeout: error
  "Esto está tardando más de lo esperado..." con retry.
- State preservation: usar Zustand para mantener current screen
  across app backgrounding.

## Definition of Done

- [ ] 5 pantallas implementadas (1-5).
- [ ] Broken assessment prompt.
- [ ] Polling + state preservation.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit + integration.
- [ ] Screenshots/video.
- [ ] Validation contra spec post-assessment-flow.md.
- [ ] PR aprobada y mergeada.

---

*Depende de US-082. Hand-off a paywall (EPIC-04 UI).*

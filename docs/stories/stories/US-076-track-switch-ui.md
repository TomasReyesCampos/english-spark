# US-076: Track switch UI + confirmation flow

**Estado:** Draft
**Epic:** EPIC-07-roadmap-engine
**Sprint target:** Sprint 3
**Story points:** 2
**Persona:** Estudiante
**Owner:** —

---

## Contexto

UI mobile para que user pueda cambiar de track activo desde
Settings. Incluye:
- Lista de 3 tracks disponibles (con preview de bloques + tiempo).
- Confirmación clara (copy del ADR-007).
- Manejo de rate limit (mostrar countdown si está en cooldown).
- Estado "regenerating" mientras nuevo roadmap se genera.

UX en
[`decisions/ADR-007-multi-track-strategy.md`](../../decisions/ADR-007-multi-track-strategy.md)
+
[`onboarding-flow.md`](../../product/onboarding-flow.md) (para
coherencia con el tono).

## Scope

### In

- Pantalla Settings → "Cambiar plan / track":
  - Lista de 3 tracks con cards:
    - Track name (mexicano-tuteo: "Listo para tu trabajo",
      "Viaja con confianza", "Conversación diaria").
    - Description + emoji.
    - Total bloques + tiempo estimado.
    - Badge "Actual" en el track activo.
  - Tap un track distinto → modal de confirmación.
- Modal de confirmación:
  - Hero: "¿Cambiar a [nuevo track]?"
  - Body con copy literal: "Vas a cambiar de [track actual] a
    [track nuevo]. Tu progreso de sub-skills se mantiene, pero
    empezarás desde el principio del nuevo track."
  - Primary CTA: "Sí, cambiar"
  - Secondary CTA: "Cancelar"
- Estado loading post-confirm:
  - "Estamos armando tu nuevo plan..."
  - Polling cada 2s a /roadmap/active hasta que retorne nuevo.
- Estado rate limited:
  - Si error 429 con `cooldown_active`: "Cambiaste de plan hace
    poco. Podés cambiar de nuevo el {next_allowed_at}."
  - Si `switch_count_exceeded`: "Llegaste al máximo de cambios.
    Contacta soporte si necesitas otro cambio."
- Telemetry: `track.switch_screen_seen`,
  `.switch_modal_opened`, `.switch_confirmed`, `.switch_cancelled`.

### Out

- Animación de transición (post-MVP polish).
- Diff visual entre tracks (post-MVP).

## Acceptance criteria

- **Given** user con active_track='job_ready' entra a Settings,
  **When** ve track switcher, **Then** ve 3 cards y "Actual" en
  Job Ready.
- **Given** user tapea otra card, **When** procesa, **Then** modal
  confirm aparece con copy adaptado.
- **Given** modal abierto, **When** tapea "Cancelar", **Then**
  modal cierra sin cambios.
- **Given** modal "Sí, cambiar", **When** request POST switch
  succeeds, **Then** UI muestra loading "Armando tu plan..." y
  poll cada 2s.
- **Given** polling detecta nuevo roadmap activo, **When** detect,
  **Then** UI navega a home con toast "¡Listo! Tu nuevo plan ya
  está disponible."
- **Given** request retorna 429 cooldown, **When** error, **Then**
  muestra mensaje user-friendly con date de próximo cambio.
- **Given** request retorna 429 hard block, **When** error, **Then**
  muestra "Llegaste al máximo de cambios..." con CTA contactar
  soporte.
- **Given** network error mid-request, **When** falla, **Then**
  toast retry "No pudimos cambiar. Intenta de nuevo."

## Developer details

### Owning service

Mobile app (React Native).

### Dependencies

- US-075 endpoint.
- US-073 polling de roadmap.

### Specs referenciados

- [`decisions/ADR-007-multi-track-strategy.md`](../../decisions/ADR-007-multi-track-strategy.md).
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §1.1 (tono mexicano-tuteo para consistency).

### Implementación esperada

```tsx
// app/screens/settings/TrackSwitcher.tsx
const TRACKS = [
  { id: 'job_ready', name: 'Listo para tu trabajo', emoji: '💼', blocks: 74, weeks: 24 },
  { id: 'travel_confident', name: 'Viaja con confianza', emoji: '✈️', blocks: 39, weeks: 14 },
  { id: 'daily_conversation', name: 'Conversación diaria', emoji: '💬', blocks: 62, weeks: 20 },
];

export function TrackSwitcherScreen() {
  const { profile } = useAuth();
  const [pending, setPending] = useState<TrackId | null>(null);

  useEffect(() => { track('track.switch_screen_seen'); }, []);

  return (
    <Screen>
      <Hero>¿Cuál es tu plan?</Hero>
      {TRACKS.map(t => (
        <TrackCard
          key={t.id}
          track={t}
          isActive={profile.active_track === t.id}
          onPress={() => {
            if (t.id !== profile.active_track) setPending(t.id);
          }}
        />
      ))}
      {pending && (
        <ConfirmSwitchModal
          to={pending}
          from={profile.active_track}
          onCancel={() => setPending(null)}
          onConfirm={async () => {
            await handleSwitch(pending);
            setPending(null);
          }}
        />
      )}
    </Screen>
  );
}

async function handleSwitch(toTrack: string) {
  track('track.switch_confirmed', { to_track: toTrack });
  try {
    await api.post('/tracks/switch', { to_track: toTrack, confirmation: true });

    // Show loading + poll
    navigation.navigate('RegenerationLoading');
    const newRoadmap = await pollUntilRoadmapReady();

    showToast('¡Listo! Tu nuevo plan ya está disponible.');
    navigation.navigate('Home');
  } catch (error: any) {
    if (error.status === 429) {
      const reason = error.body?.reason;
      if (reason === 'cooldown_active') {
        showAlert(`Cambiaste de plan hace poco. Podés cambiar de nuevo el ${formatDate(error.body.next_allowed_at)}.`);
      } else {
        showAlert('Llegaste al máximo de cambios. Contacta soporte si necesitas otro cambio.');
      }
    } else {
      showToast('No pudimos cambiar. Intenta de nuevo.');
    }
  }
}

async function pollUntilRoadmapReady(timeout = 30000) {
  const start = Date.now();
  while (Date.now() - start < timeout) {
    await new Promise(r => setTimeout(r, 2000));
    const { roadmap } = await api.get('/roadmap/active');
    if (roadmap?.is_active && roadmap.active_track === expectedTrack) {
      return roadmap;
    }
  }
  throw new Error('regeneration_timeout');
}
```

### Copy literal (mexicano-tuteo)

| Elemento | Copy |
|----------|------|
| Hero | `¿Cuál es tu plan?` |
| Card description | `{blocks} bloques · ~{weeks} semanas` |
| Modal hero | `¿Cambiar a {nuevo track}?` |
| Modal body | `Vas a cambiar de {actual} a {nuevo}. Tu progreso de sub-skills se mantiene, pero empezarás desde el principio del nuevo track.` |
| Primary CTA | `Sí, cambiar` |
| Secondary CTA | `Cancelar` |
| Loading | `Estamos armando tu nuevo plan...` |
| Success toast | `¡Listo! Tu nuevo plan ya está disponible.` |
| Cooldown alert | `Cambiaste de plan hace poco. Podés cambiar de nuevo el {date}.` |
| Hard block | `Llegaste al máximo de cambios. Contacta soporte si necesitas otro cambio.` |

### Integration points

- US-075 endpoint.
- US-073 polling.
- Mobile Settings screen.

## Definition of Done

- [ ] Pantalla + modal implementadas.
- [ ] Copy literal mexicano-tuteo correcto.
- [ ] Polling post-switch.
- [ ] Error states (429 cooldown + hard block + network).
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit del componente.
- [ ] QA manual: completar switch end-to-end.
- [ ] Screenshots/video.
- [ ] PR aprobada y mergeada.

---

*Depende de US-075. Mobile-only UI.*

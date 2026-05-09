# US-011: Pregunta ubicación (Pantalla 3)

**Estado:** Draft
**Epic:** EPIC-01-auth-onboarding
**Sprint target:** Sprint 0
**Story points:** 3
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Primera pregunta del onboarding: detectar país del user. Es la base
para:
- Pricing localizado (`i18n.md` §4).
- Logros culturales por país.
- Variantes de inglés target default.
- Methods de pago disponibles.

UX prioriza **detección automática vía IP + confirmación user**
para minimizar fricción.

UX en
[`onboarding-flow.md`](../../product/onboarding-flow.md) §4.

## Scope

### In

- Pantalla con pregunta "¿Dónde estás?" (copy literal §4.3 spec).
- 6 botones grandes con bandera + país: México, Argentina, Colombia,
  Chile, Perú, "Otro país de Latam".
- Pre-selección automática:
  - Cliente lee header `cf-ipcountry` (Cloudflare envía en cada
    request).
  - Si país detectado coincide con uno de los 5 prioritarios:
    botón aparece pre-seleccionado (border highlight).
  - Si no coincide: pre-selecciona "Otro país de Latam".
- User puede cambiar la selección antes de confirmar.
- Persiste en `student_profiles.country` (ISO 3166-1 alpha-2: MX,
  AR, CO, CL, PE, OT).
- Telemetry: `onboarding.location_seen`, `onboarding.location_selected`,
  `onboarding.location_changed_from_detected`.

### Out

- Selección granular de region/city (post-MVP).
- Detección de timezone (parte de US-014).

## Acceptance criteria

- **Given** user llega a la pantalla y su IP es de México, **When**
  se renderiza, **Then** el botón "🇲🇽 México" aparece
  pre-seleccionado.
- **Given** user con IP no-Latam (ej: USA, España), **When** se
  renderiza, **Then** "🌎 Otro país de Latam" aparece pre-seleccionado.
- **Given** la detección de IP falla (cf-ipcountry no llega), **When**
  user llega, **Then** ningún botón está pre-seleccionado y el CTA
  está disabled hasta que user elija.
- **Given** user con detección México pre-seleccionada, **When**
  cambia a Argentina, **Then** Argentina queda highlighted y México
  pierde highlight; al tap CTA se persiste 'AR' en
  `student_profiles.country` y se emite
  `onboarding.location_changed_from_detected` con `from: 'MX', to: 'AR'`.
- **Given** user selecciona un país y tapea CTA, **When** la
  request a `/profile/update` es exitosa, **Then** navega a US-012
  (Pantalla objetivo).
- **Given** request `/profile/update` falla, **When** user tapea
  CTA, **Then** ve toast "No pudimos guardar. Intenta de nuevo." y
  permanece en la pantalla.

## Developer details

### Owning service

Mobile app + backend `/profile/update`.

### Dependencies

- US-008: endpoint para persistir profile.
- US-010: pantalla welcome (precede).

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §4 —
  Pantalla 3 mockup + copy.
- [`student-profile-and-assessment.md`](../../product/student-profile-and-assessment.md)
  §3.1 — campo `country`.
- [`i18n.md`](../../cross-cutting/i18n.md) §6.1 — detección de país.

### Implementación esperada

```tsx
// app/screens/onboarding/Location.tsx
const COUNTRIES = [
  { code: 'MX', flag: '🇲🇽', name: 'México' },
  { code: 'AR', flag: '🇦🇷', name: 'Argentina' },
  { code: 'CO', flag: '🇨🇴', name: 'Colombia' },
  { code: 'CL', flag: '🇨🇱', name: 'Chile' },
  { code: 'PE', flag: '🇵🇪', name: 'Perú' },
  { code: 'OT', flag: '🌎', name: 'Otro país de Latam' },
];

export function LocationScreen() {
  const detected = useDetectedCountry(); // de cf-ipcountry header
  const [selected, setSelected] = useState<string | null>(detected);
  const detectedAtMount = useRef(detected);

  useEffect(() => { track('onboarding.location_seen'); }, []);

  const handleConfirm = async () => {
    if (!selected) return;

    if (detectedAtMount.current && detectedAtMount.current !== selected) {
      track('onboarding.location_changed_from_detected', {
        from: detectedAtMount.current, to: selected,
      });
    }
    track('onboarding.location_selected', { country: selected });

    try {
      await api.post('/profile/update', { country: selected });
      navigation.navigate('Goal');
    } catch (error) {
      showToast('No pudimos guardar. Intenta de nuevo.');
    }
  };

  return (
    <Screen>
      <Hero>¿Dónde estás?</Hero>
      <ButtonGrid columns={2}>
        {COUNTRIES.map(c => (
          <CountryButton
            key={c.code}
            country={c}
            selected={selected === c.code}
            onPress={() => setSelected(c.code)}
          />
        ))}
      </ButtonGrid>
      <PrimaryButton disabled={!selected} onPress={handleConfirm}>
        Continuar
      </PrimaryButton>
    </Screen>
  );
}
```

### Detección de país (en cliente)

```typescript
// app/lib/detect-country.ts
export function useDetectedCountry(): string | null {
  // El cliente RN no tiene acceso directo a cf-ipcountry.
  // El backend lo expone via endpoint dedicado al cargar la app.
  const { data } = useSWR('/geo', () => api.get('/geo'));
  const detected = data?.country;
  if (!detected) return null;

  const PRIORITY = ['MX', 'AR', 'CO', 'CL', 'PE'];
  return PRIORITY.includes(detected) ? detected : 'OT';
}
```

```typescript
// apps/workers/api/handlers/geo.ts
export async function handleGeo(request: Request, env: Env) {
  const country = request.headers.get('cf-ipcountry') ?? null;
  return jsonResponse({ country });
}
```

### Integration points

- Cloudflare Workers (endpoint `/geo` que lee header).
- Backend `/profile/update` (US-008 extension; o nueva story para
  endpoints CRUD de profile).
- Telemetry.

## Definition of Done

- [ ] Pantalla renderiza con 6 botones país.
- [ ] Detección IP funcional (test con VPN simulando MX, AR, USA).
- [ ] Pre-selección correcta por país detectado.
- [ ] Persistencia en `student_profiles.country`.
- [ ] 6 acceptance criteria cumplidos.
- [ ] Tests unit del componente.
- [ ] Test integration con `/geo` mockeado.
- [ ] QA manual.
- [ ] Screenshot.
- [ ] Validation contra spec `onboarding-flow.md` §4.
- [ ] PR aprobada y mergeada.

---

*Depende de US-008 + US-010. Precede US-012.*

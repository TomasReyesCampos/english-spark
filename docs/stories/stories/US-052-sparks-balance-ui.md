# US-052: UI de balance display en home + paywall

**Estado:** Draft
**Epic:** EPIC-04-sparks-system
**Sprint target:** Sprint 2
**Story points:** 1
**Persona:** Estudiante
**Owner:** —

---

## Contexto

Componente reusable de UI que muestra el balance de Sparks con
tratamiento visual distinto según contexto (home, exercise pre-
charge, paywall, low balance state).

UX implícito en spec
[`onboarding-flow.md`](../../product/onboarding-flow.md) §17 (badge
"⚡50 Sparks gratis para tu trial") y
[`welcome-flow-post-paywall.md`](../../product/welcome-flow-post-paywall.md)
§2.1.

## Scope

### In

- Componente React Native `<SparksBadge variant />`:
  - `variant: 'compact' | 'full' | 'paywall' | 'low'`.
  - Compact: solo número + ⚡ icono ("⚡50").
  - Full: número + label + cycle info ("⚡200 / 200 Sparks · Plan
    Pro").
  - Paywall: highlight con animación si bajo cap del plan.
  - Low: rojo + CTA "Comprar más" si current < 20%.
- Consume `useSWR('/sparks/balance')` con polling cada 60s en
  background.
- Real-time refresh cuando triggered events de cambio (cache
  invalidation no llega al cliente; refresh manual).
- `<SparksBadge />` en:
  - Home header (variant=compact).
  - Pre-exercise screen (variant=full).
  - Paywall (variant=paywall).
- Animación discreta cuando balance cambia (count up/down).

### Out

- Animation de "+5 Sparks" como confetti (post-MVP).
- Push real-time de balance changes (post-MVP, requires
  websockets).

## Acceptance criteria

- **Given** user con balance 50, **When** SparksBadge renderiza
  con variant=compact, **Then** muestra "⚡50".
- **Given** user con balance 200/200 Plan Pro, **When** variant=full,
  **Then** muestra "⚡200 / 200 Sparks · Plan Pro".
- **Given** user con balance 30 de allotment 200 (15% < 20%),
  **When** variant=low, **Then** color rojo + CTA "Comprar más".
- **Given** balance cambia de 50 a 40, **When** componente recibe
  update, **Then** count down animation 50 → 40 (1s).
- **Given** background polling 60s, **When** balance cambió (otra
  device hizo charge), **Then** UI actualiza sin user action.
- **Given** SparksBadge en paywall con balance 0, **When**
  variant=paywall, **Then** muestra estilo "agotado" sin alarma
  visual (no rojo escandaloso).
- **Given** balance fetch falla (network), **When** muestra,
  **Then** placeholder "⚡--" sin crash.

## Developer details

### Owning service

Mobile app (React Native).

### Dependencies

- US-045: endpoint `/sparks/balance`.

### Specs referenciados

- [`onboarding-flow.md`](../../product/onboarding-flow.md) §17.
- [`welcome-flow-post-paywall.md`](../../product/welcome-flow-post-paywall.md) §2.1.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §8.1 (low balance copy).

### Implementación esperada

```tsx
// app/components/SparksBadge.tsx
type Variant = 'compact' | 'full' | 'paywall' | 'low';

export function SparksBadge({ variant = 'compact' }: { variant?: Variant }) {
  const { data: balance } = useSWR('/sparks/balance', {
    refreshInterval: 60_000,
  });

  if (!balance) return <Placeholder text="⚡--" />;

  const ratio = balance.cycle_allotment > 0
    ? balance.current / balance.cycle_allotment
    : 1;
  const isLow = ratio < 0.20;
  const effectiveVariant = isLow && variant !== 'paywall' ? 'low' : variant;

  return (
    <AnimatedCount value={balance.current}>
      {(displayValue) => {
        switch (effectiveVariant) {
          case 'compact':
            return <Text>⚡{displayValue}</Text>;
          case 'full':
            return (
              <Row>
                <Text>⚡{displayValue} / {balance.cycle_allotment} Sparks</Text>
                <Text variant="muted"> · Plan {balance.plan_name ?? 'Trial'}</Text>
              </Row>
            );
          case 'paywall':
            return (
              <PaywallBadge>
                <Text>⚡{displayValue}</Text>
                <ProgressBar value={ratio} />
              </PaywallBadge>
            );
          case 'low':
            return (
              <LowBadge>
                <Text color="red">⚡{displayValue}</Text>
                <Button onPress={() => navigation.navigate('SparksShop')}>
                  Comprar más
                </Button>
              </LowBadge>
            );
        }
      }}
    </AnimatedCount>
  );
}
```

### Integration points

- US-045 endpoint.
- Welcome flow post-paywall (US-024 muestra inicial 50).
- Pre-exercise screens (post-MVP, EPIC-03).
- Paywall screens (post-MVP).

### Notas técnicas

- Polling 60s es OK para visualización; transactions importantes
  invalidan cache server-side (próximo poll trae fresh data).
- AnimatedCount wrapping reduce flicker visual.
- "⚡--" placeholder explícito en lugar de skeleton evita layout
  shifts.

## Definition of Done

- [ ] Componente con 4 variants.
- [ ] Polling 60s.
- [ ] Animation de count change.
- [ ] Low state visual.
- [ ] 7 acceptance criteria pasan.
- [ ] Tests unit del componente con varios states.
- [ ] Screenshots de cada variant.
- [ ] Validation contra spec.
- [ ] PR aprobada y mergeada.

---

*Depende de US-045. Story de UI pequeña.*

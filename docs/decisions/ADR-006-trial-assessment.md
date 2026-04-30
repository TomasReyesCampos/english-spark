# ADR-006: Free trial 7 días con assessment al día 7

**Status:** Accepted
**Date:** 2026-04
**Author:** —
**Audiencia:** agente AI implementador.

---

## Contexto

Apps de educación tienen alto churn temprano (80% abandona en <60 días).
Necesitamos un **funnel de activación** que:

1. Reduzca fricción de "tengo que pagar para saber si me sirve".
2. Construya hábito durante los primeros 7 días (críticos para
   retención).
3. Capture data del usuario para personalizar el roadmap definitivo.
4. Convierta al pago en un momento de alta motivación e inversión
   psicológica.

## Decisión

**Trial de 7 días con 50 Sparks gratuitos + assessment opcional pero
fuertemente incentivado al día 7. La conversión al pago se ofrece
inmediatamente después del assessment.**

### Estructura

| Día | Qué pasa |
|-----|----------|
| 0 | Onboarding: 5 preguntas + mini-test 3 min. Roadmap inicial provisional generado. 50 Sparks otorgados. |
| 1–6 | Usuario practica con acceso completo a Plan Pro. Sistema observa comportamiento. Notificaciones progresivas construyen expectativa del assessment. |
| 7 | Assessment de 20 min (4 partes, 12 ejercicios). Genera roadmap definitivo. Paywall presentado. |
| 8+ | Si convirtió: plan activo. Si no: acceso continuo a preassets gratis, Sparks agotados, banner persistente para hacer assessment cuando esté listo. |

### Por qué 7 días específicamente

- **5 días no alcanza** para construir hábito.
- **14 días es demasiado** — extiende el trial más allá del momento de
  pico de motivación.
- **7 días = una semana completa** de patrones (incluye fines de semana,
  rutinas distintas).

### Por qué 50 Sparks

- Permite ~3–5 conversaciones (10 min cada una) + ejercicios de
  pronunciación.
- Si el usuario es muy intensivo, los 50 duran ~5 días — convierte en
  oportunidad de "early conversion".
- Si es casual, los 50 alcanzan los 7 días.
- Costo aproximado para nosotros: $1.50–2.50 USD por usuario en trial.

### Por qué assessment opcional, no gate obligatorio

- Forzar el assessment como gate genera fricción y churn.
- Hacerlo opcional + atractivo permite que usuarios listos lo hagan y
  obtengan plan superior.
- Usuarios no listos siguen usando la versión limitada con incentivo
  permanente a hacerlo cuando estén listos.

### Asymmetric incentive

El assessment **desbloquea** el roadmap definitivo (no solo "actualiza"
el provisional). El usuario percibe:

- Antes del assessment: "tu plan inicial - se actualizará al día 7".
- Después del assessment: "tu plan personalizado, hecho para vos".

Esto crea expectativa positiva sin forzar.

## Alternativas consideradas

### A. Trial sin tiempo limit pero con limit de uso

**Rechazada.** Sin tiempo limit, no hay urgencia para convertir. Los
power users gastan los Sparks rápido y abandonan; los casual users no
sienten que el trial "expire".

### B. Trial de 14 días

**Considerada, rechazada.** Más datos, pero:
- Pico de motivación cae después de 7–10 días.
- Costo de Sparks gratis se duplica.
- Window of conversion se diluye.

A/B test prioritario: 7 vs 14 días una vez tengamos volumen.

### C. Trial con paywall inmediato post-onboarding

**Rechazada.** Maximiza conversion-blocked-by-payment, mata
exploración. Usuarios prefieren probar antes de pagar; sin try-before-buy
muchos no ingresan al funnel.

### D. Sin trial, freemium puro

**Rechazada.** Consumer apps con freemium puro tienen tasas de conversion
muy bajas (<1%). Trial focalizado en assessment crea momentum hacia el
pago.

### E. Assessment al día 0 (parte del onboarding)

**Rechazada.** Demasiado largo para day 0 (20 min); usuarios abandonan
el onboarding. El mini-test de 3 min al día 0 captura data inicial sin
fatigar.

### F. Assessment como gate obligatorio

**Rechazada.** Genera fricción y reduce engagement. El sistema funciona
mejor con incentivos que con obligaciones.

### G. Trial sin Sparks limit (todo libre)

**Rechazada.** Costo descontrolado para usuarios power. Cap de 50 Sparks
da control predecible.

## Consecuencias

### Positivas

- Reduce fricción de "registro → pago" significativamente.
- Construye hábito durante la ventana crítica de 7 días.
- Captura data de comportamiento real (no solo lo declarado).
- Assessment al día 7 alcanza al usuario en pico de motivación.
- Usuarios que no convierten quedan en la base con preassets gratis →
  oportunidad futura.

### Negativas

- Costo de adquisición incluye los Sparks gratis ($1.50–2.50 por usuario).
- Complejidad: hay que distinguir trial users vs paid users en muchos
  lugares del código.
- Si el assessment no convence, el usuario tiene un plan personalizado
  que no usa.

### Riesgos a monitorear

- Tasa de completion del onboarding <85%: simplificar preguntas.
- Return día 2 <60%: revisar primer ejercicio (debe ser épico).
- Return día 7 <40%: revisar comunicación durante trial.
- Completion del assessment <30%: assessment muy largo o no se comunicó
  bien su valor.
- Conversion post-assessment <40%: revisar paywall y propuesta.
- Conversion total registro → pago <12%: optimizar funnel.

### A/B tests prioritarios (post-MVP)

1. Trial duration: 7 vs 14 días.
2. Sparks iniciales: 50 vs 30 vs 100.
3. Assessment timing: día 5 vs 7 vs 10.
4. Onboarding length: 5 vs 7 vs 3 preguntas.
5. Paywall positioning: post-assessment vs primera Sparks-insuficiente.

## Referencias

- [`docs/product/student-profile-and-assessment.md`](../product/student-profile-and-assessment.md)
  — diseño detallado del trial y assessment.
- [`docs/product/ai-roadmap-system.md`](../product/ai-roadmap-system.md)
  — generación del roadmap (inicial vs definitivo).
- [`docs/architecture/sparks-system.md`](../architecture/sparks-system.md)
  — economía del trial (50 Sparks).
- [`docs/business/metrics-and-experimentation.md`](../business/metrics-and-experimentation.md)
  — métricas del funnel.

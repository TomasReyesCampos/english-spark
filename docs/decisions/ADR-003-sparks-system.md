# ADR-003: Sistema de Sparks como unidad de consumo

**Status:** Accepted
**Date:** 2026-04
**Author:** —
**Audiencia:** agente AI implementador.

---

## Contexto

El producto consume IA en operaciones premium (conversación 1 a 1,
roleplays personalizados, scoring de pronunciación, etc.). Costos
variables de IA pueden destrozar los unit economics si no se controlan:

- Un usuario power que use IA intensivamente puede costar $5–10 USD/mes
  contra un revenue de $5 USD del plan Pro → margen negativo.
- Costos de proveedores LLM cambian frecuentemente (subidas y bajadas).
  Si el precio al usuario se ata directamente a costo, hay que hacer
  cambios visibles al usuario cada vez que cambia un proveedor.

Necesitamos un mecanismo que:
1. **Limite el costo de IA por usuario** dentro de su plan.
2. **Permita ajustar el costo en backend** sin tocar el precio al
   usuario.
3. **Habilite gamificación** (recompensas que se sienten valiosas).
4. **Sea entendible** para el usuario (no fricción).

## Decisión

Implementar un sistema de **Sparks** — una unidad interna de consumo —
que controla el acceso a operaciones premium.

### Definición

- 1 Spark ≈ **1 minuto de conversación 1 a 1 con IA** o ~5 ejercicios
  cortos con análisis IA.
- La equivalencia exacta es ajustable internamente sin afectar la
  experiencia del usuario.

### Origen y flujo

Los Sparks ingresan por:
1. Asignación mensual del plan (Free 5 trial, Básico 30, Pro 200,
   Premium 600).
2. Compra de packs adicionales (100 / 300 / 1.000 Sparks).
3. Recompensas de gamificación (streaks, referidos, logros).
4. Compensaciones del sistema (créditos por bugs).

Se consumen al ejecutar operaciones costosas (cobro previo, refund si
falla).

### Expiración

- Sparks del plan: rollover hasta 2x el plan, lo demás expira al fin
  de ciclo.
- Sparks de packs: 6 meses desde la compra.
- Sparks de gamificación: mismas reglas que los del plan.

### Acceso a preassets siempre, aún con 0 Sparks

Crítico: aún con balance de 0, el usuario tiene **acceso completo a la
biblioteca de preassets**. Los Sparks son para **operaciones premium en
tiempo real**, no para acceso básico.

### Ajustes de costo

El costo en Sparks de una operación es **configuración**, no constante.
Si el costo del proveedor cambia >20%, se ajusta el costo en Sparks. El
precio del plan al usuario **no cambia**.

Comunicación al usuario con 30 días de notice; aplica solo a operaciones
futuras; documentado en `docs/decisions/sparks-pricing-changes.md`.

## Alternativas consideradas

### A. Sin sistema de tokens, costo absorbido por el plan mensual

**Rechazada.** Imposible de sostener. Un power user puede hacer
explotar costos sin que el plan lo cubra. Sin mecanismo de control,
los unit economics dependen de que usuarios "se autolimite", lo cual
no pasa.

### B. Pricing usage-based puro (pago por minuto)

**Rechazada.** Funciona para B2B, no para consumer. Genera ansiedad
de uso ("estoy gastando dinero ahora mismo"), reduce engagement,
fricciona conversion.

### C. Limit hard de operaciones por plan (ej: "Plan Pro: 50 conversaciones/mes")

**Considerada.** Más simple pero menos flexible. No permite que el
usuario decida si gastar su "asignación" en conversaciones largas o
ejercicios cortos. No habilita compra de "más" sin upgrade de plan.

**Decisión:** Sparks es estrictamente más flexible y permite compras
puntuales sin cambiar plan.

### D. Créditos de dinero (USD/MXN)

**Rechazada.** Implica regulación financiera mucho más estricta
(monederos electrónicos en algunas jurisdicciones). Sparks es
"unidad arbitraria del producto" — no es moneda.

### E. Tokens de proveedor LLM expuestos al usuario

**Rechazada.** Demasiado técnico, varía por modelo, no es la unidad
relevante para el usuario.

## Consecuencias

### Positivas

- Control de costos predictivo: cap por usuario por mes.
- Flexibilidad de pricing interno sin afectar al usuario.
- Habilita gamificación natural ("ganaste 10 Sparks").
- Permite upselling de packs a power users sin upgrade de plan.
- Proteje márgenes contra cambios de precio de LLM.

### Negativas

- Concepto adicional para el usuario aprender ("¿qué es un Spark?").
  Mitigación: tooltips, indicación previa de costo en cada operación.
- Implementación con audit log inmutable, transactions, rollover,
  expiración requiere código robusto y bien testeado.
- Contabilidad: Sparks comprados son pasivo hasta consumir
  (revenue recognition diferido).

### Riesgos a monitorear

- Usuario percibe Sparks como restricción molesta más que como límite
  razonable: medir CSAT post-paywall, NPS, comentarios de soporte.
- Power users frustrados con Plan Pro porque corren los Sparks rápido:
  monitorear tasa de compra de packs vs upgrade a Premium.
- Confusión entre Sparks "del plan" vs "comprados" vs "bonus":
  desglose claro en la UI.
- Anti-fraud de farming de Sparks gratis (cuentas múltiples): cubierto
  en `anti-fraud-system`.

## Referencias

- [`docs/architecture/sparks-system.md`](../architecture/sparks-system.md)
  — diseño detallado.
- [`docs/architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md)
  — Gateway cobra Sparks antes de cada operación.
- [`docs/architecture/anti-fraud-system.md`](../architecture/anti-fraud-system.md)
  — protección anti-farming.
- [`docs/business/plan_de_negocio.docx`](../business/plan_de_negocio.docx)
  — pricing y planes.

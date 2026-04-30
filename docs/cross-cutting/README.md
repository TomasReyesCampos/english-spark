# Cross-Cutting Concerns

> Temas que atraviesan múltiples sistemas y no encajan limpiamente en
> un solo dominio. Centralizados acá para evitar duplicación o
> divergencia entre documentos de sistema.

**Audiencia:** agente AI implementador.

---

## Documentos planificados

| Doc | Status | Para qué |
|-----|--------|----------|
| `data-and-events.md` | Pendiente | Modelo unificado de eventos (naming, shape, idempotencia, dead-letter), tracking taxonomy, retención de datos |
| `security-threat-model.md` | Pendiente | STRIDE por sistema, mitigaciones, secrets management, rotation policy, audit log |
| `testing-strategy.md` | Pendiente | Unit + integration tests (E2E fuera de scope), tooling (Vitest, Playwright NO), targets de coverage, CI gates |
| `i18n.md` | Pendiente | Estrategia de localización: tone, español neutro, formato de fechas/números/moneda, currency conversion en runtime |

---

## Cuándo agregar un doc cross-cutting

- Cuando 3+ sistemas necesitarían describir lo mismo.
- Cuando una decisión técnica tiene impacto transversal (ej: librería de
  logging usada por todos).
- Cuando un riesgo (security, privacy, compliance) requiere visión
  unificada.

Si solo aplica a un sistema, vive en el doc de ese sistema.

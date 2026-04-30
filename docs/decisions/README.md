# Architecture Decision Records (ADRs)

> Registro de decisiones arquitectónicas significativas. Cada ADR
> documenta **una** decisión: contexto, alternativas, decisión tomada,
> consecuencias.

**Audiencia:** agente AI implementador. Antes de cuestionar una decisión
arquitectónica, leé el ADR correspondiente para entender por qué se
tomó.

---

## Cuándo escribir un ADR

Escribir un ADR cuando:

- Se elige una tecnología o servicio externo (vs construirlo o usar
  alternativa).
- Se define un principio arquitectónico (ej: monolito modular vs
  microservicios).
- Se decide algo difícil de revertir (esquemas centrales, contratos
  públicos).
- Se rechaza explícitamente una opción que vuelve a aparecer en
  discusiones.

NO escribir ADR para:

- Detalles de implementación dentro de un sistema (van en el doc del
  sistema).
- Configuraciones que cambian frecuentemente.
- Decisiones de UX o copy.

---

## Formato

```markdown
# ADR-XXX: <Título>

**Status:** Proposed | Accepted | Superseded by ADR-YYY | Deprecated
**Date:** YYYY-MM-DD
**Author:** —
**Audiencia:** agente AI implementador.

## Contexto
<Situación que requiere decisión.>

## Decisión
<Lo que decidimos.>

## Alternativas consideradas
<Opciones rechazadas y por qué.>

## Consecuencias
### Positivas
### Negativas
### Riesgos a monitorear

## Referencias
```

---

## ADRs activos

| ID | Título | Status |
|----|--------|--------|
| [ADR-001](ADR-001-ai-gateway.md) | Capa de abstracción AI Gateway | Accepted |
| [ADR-002](ADR-002-platform-strategy.md) | Mobile-first con React Native + Expo | Accepted |
| ADR-003 | Sistema de Sparks como unidad de consumo | Pendiente |
| ADR-004 | Firebase Cloud Messaging para push | Pendiente |
| ADR-005 | Firebase Authentication | Pendiente |
| ADR-006 | Free trial 7 días + assessment día 7 | Pendiente |

---

## Convenciones

- ID secuencial: `ADR-001`, `ADR-002`, ...
- Filename: `ADR-XXX-<slug-kebab-case>.md`.
- Una decisión por ADR. Si una propuesta tiene 3 sub-decisiones,
  hacer 3 ADRs.
- ADRs no se borran. Si se deprecan, se marca `Status: Deprecated` o
  `Superseded by ADR-YYY` con el reemplazo.

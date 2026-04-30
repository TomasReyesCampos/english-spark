# Operations

> Documentos operativos: cómo correr el sistema en producción, manejar
> incidentes, hacer deploys, monitorear.

**Audiencia:** agente AI implementador + operador humano.

---

## Documentos planificados

| Doc | Status | Para qué |
|-----|--------|----------|
| `incidents-runbook.md` | Pendiente | Cómo detectar, responder y postmortear incidentes. Severities, escalation, comms con usuarios. |
| `cicd.md` | Pendiente | Pipelines de build/test/deploy. GitHub Actions, EAS Build, Cloudflare Workers deploy. |
| `monitoring.md` | Pendiente (opcional) | Dashboards consolidados, alertas críticas, métricas de SLO. |

---

## Estado del entorno productivo

**No hay aún.** El proyecto está en pre-Sprint 0 (diseño). Los docs en
esta carpeta se completan antes del soft launch.

## Convención de severities

(Anticipo, se formaliza en `incidents-runbook.md`.)

| Severity | Definición | Tiempo de respuesta |
|----------|-----------|---------------------|
| SEV-1 | Outage total o pérdida de datos | < 15 min |
| SEV-2 | Feature crítica caída para >10% usuarios | < 1 hora |
| SEV-3 | Feature secundaria caída o degradación | < 4 horas |
| SEV-4 | Bug menor, sin impacto operativo | Siguiente día hábil |

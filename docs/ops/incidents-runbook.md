# Incidents Runbook

> Cómo detectar, responder, comunicar y postmortear incidentes
> operativos. Severities, escalation, comunicación al usuario, recovery
> procedures.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** operador humano + agente AI implementador para
crear los hooks/alertas necesarios.
**Alcance:** Operación productiva (post-soft launch).

---

## 0. Cómo leer este documento

- §1 define **severities** y SLAs.
- §2 cubre **detección** (alertas, monitoring).
- §3 cubre **respuesta inicial**: contener, evaluar.
- §4 cubre **playbooks específicos** para incidentes comunes.
- §5 cubre **comunicación**: status page, banner in-app, emails.
- §6 cubre **recovery**: restaurar normalidad.
- §7 cubre **post-mortem**.
- §8 cubre **incidentes de seguridad** (link a security-threat-model).

---

## 1. Severity y SLA

### 1.1 Definiciones

| Severity | Definición | Ejemplos |
|----------|-----------|----------|
| **SEV-1** | Outage total o pérdida de datos confirmada | Postgres caído; webhook de pagos no procesa; bug que borra datos de usuarios |
| **SEV-2** | Feature crítica caída para >10% usuarios | Conversación 1:1 con IA falla; login de Google roto; push notifications no llegan |
| **SEV-3** | Feature secundaria caída o degradación notable | Roleplays específicos no cargan; latencia 3x lo normal |
| **SEV-4** | Bug menor sin impacto operativo | Tooltip mal posicionado; logro no se desbloquea para 1 usuario |
| **SEC-1** | Compromise activo o exfiltración | Credentials leaked usados; SQL injection explotado |
| **SEC-2** | Vulnerabilidad explotable detectada | Endpoint sin auth; secret en logs |

### 1.2 SLAs

| Severity | Tiempo a primera respuesta | Tiempo a mitigación | Tiempo a resolución |
|----------|---------------------------|---------------------|---------------------|
| SEV-1 | < 15 min | < 1 hora | < 4 horas |
| SEV-2 | < 1 hora | < 4 horas | < 24 horas |
| SEV-3 | < 4 horas (hábil) | < 24 horas | < 72 horas |
| SEV-4 | < 24 horas hábil | Próximo release | Próximo release |
| SEC-1 | < 15 min | < 1 hora | Variable |
| SEC-2 | < 1 hora | < 4 horas | < 24 horas |

### 1.3 Mientras seamos dev solo

- **Horario hábil:** lunes a viernes, 9–19 hora local. SEV-1 / SEC-1
  requieren respuesta 24/7 incluso en este modo.
- **SEV-1/SEC-1 fuera de horario:** atender ASAP. Si dormíamos, el
  status page debe estar actualizado en cuanto se note.
- **Sin oncall rotation:** no hay equipo aún. Cuando exista, formalizar
  rotación.

---

## 2. Detección

### 2.1 Fuentes de alertas

| Fuente | Qué detecta | Severity típica |
|--------|------------|----------------|
| **Sentry** | Errors de runtime (Workers, cliente) | SEV-2 a SEV-4 según volumen |
| **Better Stack (Uptime)** | Endpoints públicos abajo | SEV-1 / SEV-2 |
| **Cloudflare Logs** | Picos de errors 5xx | SEV-1 a SEV-3 |
| **Supabase dashboard** | DB CPU > 80%, conexiones agotadas | SEV-1 / SEV-2 |
| **PostHog alertas** | NSM cae > 30% día contra día | SEV-2 / SEV-3 |
| **Stripe / RevenueCat alertas** | Webhooks fallando consistentemente | SEV-1 (es revenue!) |
| **Firebase Console** | Auth issues, FCM delivery rate caído | SEV-2 |
| **Internal cron de health check** | Sparks audit reconciliation con divergencia | SEV-2 |
| **Dead letter queue (`event_log_dlq`)** | Eventos fallando consistentemente | SEV-2 / SEV-3 |
| **Reportes de usuarios** | Tickets de soporte con bugs reproducibles | Variable |

### 2.2 Alertas automáticas críticas

A configurar al inicio del MVP:

```yaml
# Alertas que despiertan en mitad de la noche (SEV-1)
- name: api_5xx_rate_high
  condition: 5xx errors > 5% en últimos 10 min sostenido
  severity: SEV-1
  notify: email + sms

- name: postgres_unreachable
  condition: connection pool exhausted o timeout
  severity: SEV-1
  notify: email + sms

- name: auth_login_success_rate_low
  condition: success rate < 90% en últimos 10 min
  severity: SEV-1
  notify: email + sms

- name: payment_webhook_failures
  condition: > 3 fallos consecutivos en webhook de Stripe/RevenueCat
  severity: SEV-1
  notify: email + sms

- name: sparks_balance_inconsistency
  condition: audit job detecta SUM(transactions) != balance en cualquier user
  severity: SEV-1
  notify: email + sms

# Alertas hábiles (SEV-2/3, no despiertan)
- name: latency_p95_high
  condition: p95 endpoint latency > 2x baseline durante 30 min
  severity: SEV-2

- name: ai_validation_rate_low
  condition: < 95% de outputs LLM pasan validación
  severity: SEV-2

- name: notifications_delivery_low
  condition: FCM delivery rate < 90% durante 1 hora
  severity: SEV-2

- name: dlq_size_growing
  condition: event_log_dlq size > 100 unresolved
  severity: SEV-2

- name: cost_spike
  condition: AI cost diario > 2x promedio histórico
  severity: SEV-2
```

### 2.3 Sentry alerts

Configuración recomendada en Sentry:

- **Issue alerts:** error nuevo (no visto antes) → email.
- **Error frequency:** > 100 errors/hora del mismo issue → email + page.
- **Performance:** p95 transaction > 3x baseline → email.

### 2.4 Detección manual (fuera de alertas)

- **Daily check:** revisar Sentry inbox + PostHog dashboard 5 min/día.
- **Weekly review:** análisis de DLQ, costos, anomalías.
- **Reportes de usuarios:** monitor `support@` + reseñas de stores.

---

## 3. Respuesta inicial

### 3.1 Primeros 15 minutos de un SEV-1

1. **Acknowledge** la alerta (cualquier canal).
2. **Confirmar el incidente** (¿es real o falso positivo?):
   - Reproducir si posible.
   - Verificar varias fuentes (Sentry + Cloudflare + status de
     proveedores).
3. **Crear "war room"** (incluso si dev solo, equivale a doc nuevo
   en Notion/Markdown):
   - `incidents/<YYYY-MM-DD>-<short-name>.md`.
   - Timestamps de cada evento mientras transcurre.
4. **Status page:** marcar como "Investigating".
5. **Contener:** primer instinto debe ser **detener el daño**, no
   resolver la causa.

### 3.2 Patrones de containment

| Tipo de incidente | Containment inmediato |
|-------------------|----------------------|
| Deploy reciente rompió algo | **Rollback** el deploy (Cloudflare Workers tiene rollback nativo) |
| Endpoint puntual fallando | **Rate limit a 0** ese endpoint o desactivar feature flag |
| Cobro de Sparks errando | **Disable feature flag** que causa el bug; los usuarios pierden uso pero no pierden Sparks |
| Webhook de pago no procesa | Implementar reconciliación manual con API del proveedor |
| Token de proveedor LLM revocado | Switch a fallback en AI Gateway (debería ser automático) |
| DB con CPU al 100% | Identificar query culpable, mata; escalar instancia si necesario |
| Audio uploads fallando | Acceptar attempts sin audio temporalmente; informar al usuario |
| Push notifications no llegan | Verificar FCM access token; regenerar si necesario |

### 3.3 Quién hace qué (para dev solo)

Como dev solo, todo cae en una persona. Cuando crezca el equipo:

| Rol | Función durante incidente |
|-----|--------------------------|
| **Incident Commander** | Coordina, decide. Es la voz que comunica al usuario. |
| **Technical Lead** | Investiga causa raíz, implementa fix. |
| **Comms Lead** | Actualiza status page, redacta emails, posts. |

Para dev solo: **comms primero, fix después**. Un mensaje de "estamos
trabajando" cada 30 min reduce ansiedad de los usuarios mientras
trabajás.

---

## 4. Playbooks específicos

### 4.1 Postgres (Supabase) caído

**Síntomas:**
- 5xx en endpoints que tocan DB.
- Sentry inundado de `connection refused` o `timeout`.

**Containment:**
1. Status page → "Investigating" con mensaje genérico.
2. Verificar Supabase status page (https://status.supabase.com).
3. Si Supabase dice OK: nuestra config (pool exhausted? wrong creds?).
4. Si Supabase dice incident: esperar + comunicar.

**Mitigación:**
- Pool exhausted: identificar query lenta en `pg_stat_activity`, kill,
  refactor.
- Migración fallida: rollback de la migración (preparar `DOWN.sql`
  siempre).
- Suspendido por billing: pagar.

**Recovery:**
- Verificar que todos los workers reconecten.
- Verificar audit jobs (Sparks reconciliation) no perdieron data.

### 4.2 AI provider down (Anthropic, OpenAI, etc.)

**Síntomas:**
- Tasks específicos del AI Gateway fallan.
- Sentry: `503 Service Unavailable` o timeout de cliente.

**Containment:**
- AI Gateway debería **automáticamente** caer a fallback. Si lo está
  haciendo: la "incidencia" es invisible al usuario excepto por
  potencial latencia mayor.
- Si NO está cayendo a fallback: hay un bug en el Gateway. Disable la
  task afectada vía feature flag para evitar loops.

**Mitigación:**
- Verificar status del proveedor.
- Si extendido: declarar canary del fallback como primary temporalmente.

**Recovery:**
- Cuando primary vuelve, restablecer routing original.
- Verificar cost tracking durante el incidente (se gastó más en
  fallback?).

### 4.3 Webhook de Stripe / RevenueCat fallando

**Síntomas:**
- Usuarios reportan pago hecho pero plan no activado.
- Endpoint `/webhooks/stripe` retorna 5xx en Cloudflare logs.

**Containment:**
- **NO desactivar el endpoint** — necesitamos seguir recibiendo, aunque
  fallen.
- Stripe / RevenueCat reintentán automáticamente; si nuestro endpoint
  retorna 5xx, eventualmente entran a su DLQ.

**Mitigación:**
- Identificar bug del handler (Sentry).
- Hotfix + deploy.
- Mientras: usar endpoint de reconciliación manual con la API del
  proveedor para acreditar usuarios afectados.

**Recovery:**
- Reprocessar webhooks que fallaron desde el dashboard de Stripe /
  RevenueCat.
- Auditar usuarios afectados: balance correcto?

### 4.4 FCM push notifications no llegan

**Síntomas:**
- Open rate cae a 0 después de un cambio.
- Logs del Worker de notifications muestran 401/403 al llamar FCM.

**Containment:**
- Verificar que `FIREBASE_SERVICE_ACCOUNT` es válido (no rotó).
- Verificar que el access token del Worker está cacheado correctamente.

**Mitigación:**
- Regenerar service account key en Firebase Console.
- Setear nuevo secret en Cloudflare.
- Revocar viejo.

**Recovery:**
- Disparar daily reminders perdidos (manual run de cron).
- Verificar delivery con un test push a tu propio device.

### 4.5 Pérdida de audios en R2

**Síntomas:**
- Usuarios reportan que ejercicio enviado no procesó.
- Logs muestran `404 NoSuchKey` al intentar leer audios.

**Containment:**
- Alertar a usuarios afectados que reintentan el ejercicio.
- Refund Sparks de los attempts fallidos.

**Mitigación:**
- Identificar root cause: cleanup cron borró antes de tiempo? Bug en
  upload? R2 outage?

**Recovery:**
- R2 NO tiene undelete. Si los audios se borraron, se perdieron.
- Comunicación honesta: "perdimos audios de X período. Tu progreso no
  se afectó (los scores ya estaban calculados); solo no puedes re-oír
  esas grabaciones."

### 4.6 Cost spike de IA

**Síntomas:**
- Alerta de `cost_spike`.
- Costo diario > 2x promedio histórico.

**Containment:**
- Identificar usuario individual con consumo > $1/día (alerta separada).
  Si existe: probable bug o abuso.
- Identificar task con costo desproporcionado.

**Mitigación:**
- Si es bug (loop): disable task vía feature flag.
- Si es abuso (un usuario): aplicar restriction temporal vía
  anti-fraud.
- Si es uso legítimo aumentado: re-evaluar pricing en Sparks (no urgente).

**Recovery:**
- Ajustar alertas si es nueva baseline esperada.

### 4.7 Sparks balance inconsistency

**Síntomas:**
- Audit job nocturno detecta `SUM(transactions) != balance` en N
  usuarios.

**Containment:**
- **NO** ejecutar charges nuevos sobre los usuarios afectados hasta
  resolver.
- Disable feature flag de cobros si el N es grande.

**Mitigación:**
- Identificar root cause:
  - Race condition en cobro?
  - Webhook no procesado?
  - Bug en función de balance update?

**Recovery:**
- Reconciliar balance contra transactions:
  ```sql
  -- Para cada usuario afectado
  UPDATE user_sparks_balance
  SET plan_sparks = (calc), pack_sparks = (calc), bonus_sparks = (calc)
  WHERE user_id = $user_id;
  ```
- Insert de transaction "system_credit" / "correction" con razón
  documentada.
- Notificar a usuarios afectados si discrepancia fue en su contra.

### 4.8 Deploy rompe producción

**Síntomas:**
- Errores empiezan inmediatamente después de un deploy.

**Containment (PRIMERA ACCIÓN):**
- **Rollback** el deploy. No diagnosticar primero.
  - Cloudflare Workers: `wrangler rollback`.
  - Cliente móvil: hard a través de stores; en su lugar usar EAS Update
    para OTA fix si es bug del cliente.

**Mitigación:**
- Una vez en producción estable, investigar en branch:
  - Reproducir bug.
  - Escribir test de regresión.
  - Fix.

**Recovery:**
- Redeploy con fix.
- Post-mortem: por qué pasó CI? por qué no se detectó en staging?

### 4.9 DDoS / spike de tráfico

**Síntomas:**
- Spike masivo de requests, latencia subiendo.

**Containment:**
- Cloudflare automatic DDoS mitigation debería actuar.
- Activar "Under Attack Mode" en Cloudflare si se sospecha ataque.
- Rate limiting más agresivo.

**Mitigación:**
- Identificar si es ataque o pico legítimo (lanzamiento, viral).
- Si legítimo: escalar Postgres si necesario.
- Si ataque: `cf.country` filter; bot detection.

### 4.10 Account compromise (un usuario)

**Síntomas:**
- Usuario reporta actividad no reconocida.

**Containment:**
- Force logout de todas sesiones del usuario.
- Disable la cuenta temporal.

**Mitigación:**
- Force password reset (si email/password).
- Audit log: qué pasó durante el período sospechoso.
- Reverse Sparks transactions sospechosas.

**Recovery:**
- Re-enable cuenta.
- Ofrecer MFA setup.
- Documentar en `fraud_events`.

---

## 5. Comunicación

### 5.1 Canales

| Canal | Audiencia | Cuándo se usa |
|-------|-----------|--------------|
| **Status page** (Better Stack o Atlassian Statuspage) | Todos | Cualquier SEV-1/2 |
| **Banner in-app** | Usuarios actuales | SEV-1/2 que afecta UX |
| **Email a usuarios afectados** | Específico | Pérdida de datos, breach, incidente largo |
| **Twitter / redes sociales** | Público amplio | SEV-1 con visibility (opcional según marca) |
| **Email a equipo / stakeholders** | Internal | Cualquier incidente |
| **Post-mortem público** | Comunidad técnica | Incidentes serios resueltos |

### 5.2 Templates

#### Status page: Investigating

```
[Incident] Estamos investigando problemas con [feature]

Estamos detectando [síntoma observable]. Nuestro equipo está
investigando. Actualizaciones en 30 minutos.
```

#### Status page: Identified

```
[Incident] Identificamos el problema con [feature]

Identificamos la causa: [causa breve, no técnica]. Estamos trabajando
en el fix. ETA: [tiempo].
```

#### Status page: Monitoring

```
[Incident] Fix desplegado, monitoreando

Aplicamos un fix. El sistema está volviendo a la normalidad. Vamos a
monitorear durante las próximas [N] horas.
```

#### Status page: Resolved

```
[Resolved] [Feature] está operativa

Resolvimos el incidente. El sistema funciona normalmente. Pedimos
disculpas por las molestias. Próximamente publicaremos un postmortem
detallado.
```

#### Email a usuarios afectados (data loss)

```
Asunto: Información importante sobre tu cuenta en [Producto]

Hola,

Te escribimos para informarte sobre un incidente que ocurrió el
[fecha] entre las [hora_inicio] y [hora_fin] que afectó a tu cuenta.

Qué pasó:
[Descripción honesta y clara, sin tecnicismos.]

Qué se afectó:
[Lista específica.]

Qué hicimos:
[Acciones tomadas.]

Qué recibís como compensación:
[X Sparks bonus / mes gratis / etc., según severity.]

Estamos trabajando para que esto no vuelva a pasar y publicaremos un
postmortem público en [fecha estimada].

Si tienes alguna pregunta, escríbenos a soporte@[dominio].

Equipo de [Producto]
```

### 5.3 Reglas de comunicación

- **Honestidad:** decir qué pasó, no minimizar.
- **Sin culpar:** "tuvimos un problema" no "el proveedor X falló".
- **Concrete:** decir cuándo, qué, qué hacemos.
- **Compensación apropiada:** SEV-1 con impacto al usuario merece
  algo (Sparks bonus, mes gratis).
- **No prometer SLAs futuros:** evitar "esto nunca volverá a pasar".
  Decir "estamos trabajando para reducir la probabilidad".

---

## 6. Recovery

### 6.1 Verificaciones post-mitigación

Antes de declarar "Resolved":

- [ ] Métrica primaria del incidente vuelta a baseline.
- [ ] Métricas relacionadas no afectadas.
- [ ] Smoke test manual del feature afectado.
- [ ] Logs limpios (no errors residuales).
- [ ] Si se hizo rollback: plan de re-deploy con fix.

### 6.2 Restaurar datos perdidos

Si hubo pérdida de datos:

- **Postgres:** restaurar desde backup de Supabase (puntual o
  point-in-time según retención).
- **R2:** no recuperable. Comunicar al usuario.
- **Inngest events DLQ:** replay manual desde dashboard.

### 6.3 Reverso de cobros

Si se cobró Sparks indebidamente o se aplicó restriction errónea:

- Insert `sparks_transactions` con `source = 'refund'` o `'correction'`.
- Lift `user_restrictions` con resolución documentada.
- Email al usuario con explicación.

---

## 7. Post-mortem

### 7.1 Cuándo hacer post-mortem

- **SEV-1 / SEC-1:** siempre. Público para SEV-1 si afectó a muchos
  usuarios.
- **SEV-2:** internal mínimo, público si tiene visibility.
- **SEV-3:** internal solo si hay aprendizaje.
- **Near-miss:** si algo casi se vuelve SEV-1 pero no, vale la pena
  postmortem internal.

### 7.2 Estructura

```markdown
# Post-mortem: [título corto del incidente]

**Date:** YYYY-MM-DD
**Duration:** X hours
**Severity:** SEV-N
**Authors:** —
**Status:** Final

## TL;DR
[1-2 oraciones de qué pasó y qué hicimos.]

## Impact
- Cuántos usuarios afectados
- Por cuánto tiempo
- Pérdida monetaria estimada (si aplica)
- Pérdida de datos (si aplica)

## Timeline
| Hora (UTC) | Evento |
|------------|--------|
| ... | ... |

## Root cause
[Análisis técnico de qué causó el incidente.]

## Detection
[Cómo se detectó. Cuánto tardamos en detectarlo.]

## Resolution
[Cómo se mitigó y resolvió.]

## What went well
- ...

## What went wrong
- ...

## Action items

| # | Item | Owner | Due |
|---|------|-------|-----|
| 1 | ... | — | YYYY-MM-DD |

## Lessons learned
[Aprendizajes generales aplicables a futuro.]
```

### 7.3 Sin culpa (blameless)

Postmortem se enfoca en **el sistema**, no en la persona:

- ✅ "El sistema permitió que esa migración se desplegara sin
  detectar el problema en CI".
- ❌ "Tomás se olvidó de testear la migración".

Cuando hay culpa al sistema, hay aprendizaje accionable. Cuando hay
culpa a la persona, solo hay miedo.

### 7.4 Action items

Cada action item debe tener:

- **Owner** (persona o "system" si es config).
- **Due date.**
- **Tracking** en pendientes.md o backlog.

Action items que no se ejecutan en 30 días: re-priorizar o cerrar
explícitamente con razón.

---

## 8. Incidentes de seguridad

Detalle en `cross-cutting/security-threat-model.md` §7.

Diferencias clave con incidentes operativos:

- **Containment más agresivo:** rotar secrets, revocar sessions,
  bloquear IPs.
- **Comunicación con notice legal:** notificar dentro de 72h si hay
  data breach (GDPR).
- **Forense:** preservar logs, no borrar evidencia.
- **Coordinar con legal** antes de comunicación pública.

---

## 9. Contactos de emergencia

(A poblar cuando crezca el equipo y se contraten servicios.)

| Servicio | Contacto | Para qué |
|----------|----------|----------|
| Supabase support | — | DB issues |
| Cloudflare support | — | Workers / network |
| Stripe support | — | Payment issues |
| RevenueCat support | — | Mobile payment issues |
| Firebase support | — | Auth / FCM |
| Anthropic support | — | LLM issues |
| Asesor legal | — | Breach notification, regulatory |

---

## 10. Métricas de operación

### 10.1 SLO targets

| Métrica | Target |
|---------|--------|
| Uptime API público | 99.5% (≈ 3.6h downtime/mes) |
| p95 latency API | < 300ms |
| Crash rate cliente | < 0.1% |
| FCM delivery rate | > 95% |
| Webhooks success rate | > 99% |

### 10.2 Métricas de incident response

- **MTTD (Mean Time to Detect):** desde inicio del incidente hasta
  alerta accionable. Target: < 5 min para SEV-1.
- **MTTR (Mean Time to Resolve):** desde alerta hasta resolved. Target:
  < 4h para SEV-1.
- **Recurrencia:** % de incidentes que repiten root cause de uno previo.
  Target: < 10%.

---

## 11. Plan de implementación

### 11.1 Pre-soft-launch

- [ ] Better Stack uptime configurado para endpoints públicos.
- [ ] Sentry alerts críticas configuradas.
- [ ] Status page pública creada.
- [ ] Templates de comunicación listos.
- [ ] Procedimiento de rollback de Cloudflare Workers documentado y
  probado.
- [ ] Cron de Sparks audit reconciliation activo.

### 11.2 Post-soft-launch

- [ ] Primer postmortem (incluso de un near-miss) para validar el
  proceso.
- [ ] Refinement de severities según incidentes reales.
- [ ] Cuando crezca el equipo: oncall rotation formal.

---

## 12. Referencias internas

- [`../cross-cutting/security-threat-model.md`](../cross-cutting/security-threat-model.md) — incidentes de seguridad.
- [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §4.5 — DLQ y retries.
- [`cicd.md`](cicd.md) — proceso de rollback (a crear).

---

## 13. Recursos externos

- Atlassian Incident Handbook: https://www.atlassian.com/incident-management/handbook
- Google SRE Book — Postmortems: https://sre.google/sre-book/postmortem-culture/
- Better Stack: https://betterstack.com/

---

*Documento vivo. Actualizar después de cada incidente con aprendizajes,
y al menos una vez al año con nuevas dependencias y cambios al stack.*

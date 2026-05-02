# Customer Support System

> Sistema de soporte al usuario diseñado para escalar con equipo
> mínimo. Combina self-service, IA, y escalación humana solo cuando
> necesaria.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 establece **filosofía**.
- §2 cubre **boundaries**.
- §3 cubre **niveles de soporte** (escalación).
- §4 cubre **Nivel 0: Self-Service**.
- §5 cubre **Nivel 1: AI Assistant**.
- §6 cubre **Nivel 2: Human Async**.
- §7 cubre **Nivel 3: Critical/Priority**.
- §8 cubre **schemas Postgres**.
- §9 cubre **API contracts**.
- §10 enumera **edge cases**.
- §11 cubre **eventos**.
- §12 cubre **casos especiales** (refunds, account deletion, GDPR).
- §13 cubre **feedback as product input**.
- §14 cubre **decisiones cerradas**.

---

## 1. Filosofía

### 1.1 Principios

**Self-service por default:** la mayoría prefiere resolver solo si la
documentación está disponible.

**IA como first responder:** chatbot inteligente con contexto del user
resuelve 60-80% de consultas.

**Humano solo en casos críticos:** problemas técnicos complejos,
disputas de pagos, casos sensibles.

**Prevención > resolución:** invertir en evitar problemas (UX clara,
mensajes proactivos) reduce volumen.

**Cada ticket es feedback:** no es costo, es fuente de mejora.

### 1.2 Por qué este modelo funciona para dev solo

- Niveles 0 y 1 escalan infinitamente: sin trabajo humano marginal.
- Nivel 2 absorbe casos manejables: revisión asíncrona compatible con
  dev solo.
- Nivel 3 es excepcional: pocos casos, máxima atención.
- Sistema aprende: patrones del Nivel 2 informan mejoras del Nivel 1,
  reduciendo escalación en el tiempo.

---

## 2. Boundaries

### 2.1 Es responsable de

- Help Center (artículos, FAQs).
- Status page público.
- AI Assistant con contexto del user.
- Sistema de tickets (creación, tracking, resolución).
- Categorización y reportes.
- Apelaciones de fraude (recibir; resolución es admin).
- Refunds dentro de policy auto, manuales fuera de policy.

### 2.2 NO es responsable de

- **Account recovery completo:** delega a `auth-system`.
- **Aplicación de restricciones de fraude:** delega a `anti-fraud-system`.
- **Refunds de pagos directos:** Stripe/RevenueCat manejan refunds
  según su política; este sistema solicita.
- **Publicación de incidentes:** delega a `ops/incidents-runbook`.

### 2.3 Tensiones

| Tensión | Resolución |
|---------|-----------|
| AI Assistant promete refund que no se puede dar | Prompt explícito "nunca prometas refunds sin verificar"; escalation_needed flag |
| User quiere account deletion via support | Delega a auth-system flow; support solo guía |
| Bug crítico afecta a muchos | Comms proactiva via status page + email masivo (no via tickets individuales) |
| User abusivo que consume tiempo de soporte | Anti-fraud aplica restriction; soporte cierra tickets |

---

## 3. Niveles de soporte

### 3.1 Estructura

```
NIVEL 0: Self-Service (60-70% objetivo)
├── Help Center con artículos
├── In-app FAQ contextual
├── Tooltips y explicaciones inline
└── Status page público

         ↓ (si no resuelve)

NIVEL 1: AI Assistant (20-30% objetivo)
├── Chatbot con contexto del user
├── Respuestas a consultas comunes
├── Triage para escalación
└── Manejo de problemas estandarizados

         ↓ (si no resuelve)

NIVEL 2: Human Async (5-10%)
├── Email/in-app messaging
├── Respuesta en 24-48h
├── Casos complejos pero no urgentes
└── Revisión de feedback estructurado

         ↓ (si es crítico)

NIVEL 3: Human Priority (<2%)
├── Premium customers
├── Bugs críticos / pérdida de datos
├── Disputas de pagos
└── Issues de seguridad/privacidad
```

---

## 4. Nivel 0: Self-Service

### 4.1 Help Center

**Plataforma:** Notion público para MVP. Migrar a GitBook si volumen lo
justifica.

#### Estructura inicial

```
Help Center
├── Empezando
│   ├── ¿Cómo me registro?
│   ├── ¿Cómo funciona el free trial?
│   ├── ¿Qué es el assessment?
│   └── Mi primer día en la app
├── Mi cuenta
│   ├── Cambiar mi email
│   ├── Cambiar mi contraseña
│   ├── Eliminar mi cuenta
│   └── Recuperar mi cuenta
├── Sparks y planes
│   ├── ¿Qué son los Sparks?
│   ├── ¿Cuántos Sparks tengo?
│   ├── ¿Cómo compro packs?
│   ├── ¿Los Sparks expiran?
│   └── Cambiar mi plan
├── Aprendizaje
│   ├── ¿Cómo funciona mi roadmap?
│   ├── ¿Qué son los logros?
│   ├── ¿Cómo se calcula mi nivel?
│   ├── Re-evaluación periódica
│   └── Streaks y rachas
├── Pagos
│   ├── Métodos de pago aceptados
│   ├── ¿Cómo cancelo mi suscripción?
│   ├── Reembolsos
│   └── Facturación
├── Problemas técnicos
│   ├── La app no abre
│   ├── No me llegan notificaciones
│   ├── Problemas con el audio
│   └── La conversación con IA no funciona
└── Privacidad y datos
    ├── ¿Qué datos guardamos?
    ├── ¿Cómo accedo a mis datos?
    └── ¿Cómo borro mis datos?
```

### 4.2 Cobertura inicial

**Prioridad alta (crítico desde Day 1):**
- Cómo funciona el free trial.
- Cómo cancelar suscripción (legalmente requerido).
- Cómo eliminar cuenta (App Store requirement).
- Métodos de pago.
- Recuperación de cuenta.

**Prioridad media (semanas 1-4):**
- Sistema de Sparks.
- Roadmap y assessment.
- Logros.

**Prioridad baja (post feedback real):**
- Edge cases que aparezcan.

### 4.3 In-app FAQ contextual

Cada pantalla principal tiene "?" que abre FAQ relevante. Reduce
escalación porque user obtiene respuesta donde tiene la duda.

### 4.4 Status page

URL pública (ej: `status.<dominio>`). Stack: Better Stack o Atlassian
Statuspage (tier gratuito).

Muestra:
- Status actual (app, API, IA, pagos).
- Incidentes pasados.
- Mantenimientos programados.

Reduce tickets cuando hay outages.

---

## 5. Nivel 1: AI Assistant

### 5.1 Concepto

Chatbot accesible desde la app y Help Center que:
- Tiene contexto completo del user.
- Conoce el producto a fondo.
- Resuelve consultas comunes con respuesta inmediata.
- Escala a humano solo cuando no puede resolver.

### 5.2 Arquitectura

```
User tiene una pregunta
         ↓
Abre AI Assistant (in-app o web)
         ↓
Sistema captura contexto:
- User ID, plan, historial reciente
- Pantalla actual cuando abrió chat
- Errores recientes en su sesión
         ↓
Pregunta + contexto → AI Gateway task 'ai_assistant_response'
         ↓
LLM clasifica:
- ¿Puedo responder con knowledge base? → Responde
- ¿Es problema técnico identificable? → Diagnóstico + sugiere acción
- ¿Requiere humano? → Crea ticket + asegura al user
- ¿Es crítico? → Escala con prioridad alta
         ↓
Respuesta al user o ticket creado
```

### 5.3 Knowledge base del AI

El AI Assistant accede a:

**Datos del user (en context):**
- Perfil, plan, balance Sparks.
- Historial de pagos, status de suscripción.
- Roadmap actual, progreso.
- Tickets previos.

**Documentación del producto:**
- Todos los artículos del Help Center.
- FAQs.
- Políticas (privacy, cancelación, refunds).

**Datos operacionales:**
- Status actual de servicios.
- Incidentes recientes.
- Bugs conocidos con workarounds.

### 5.4 Prompt del AI Assistant

(Task `ai_assistant_response` en AI Gateway, ver §4.2.8.)

```
Sos un agente de soporte de [App Name], una app de aprendizaje de
inglés para hispanohablantes.

DATOS DEL USER:
- Plan: {{user.plan}}
- Sparks balance: {{user.sparks_balance}}
- Trial status: {{user.trial_status}}
- Días en la app: {{user.days_active}}
- Último issue reportado: {{user.last_ticket}}

CONTEXTO DEL MOMENTO:
- Pantalla actual: {{current_screen}}
- Última acción: {{last_action}}
- Errores recientes: {{recent_errors}}

KNOWLEDGE BASE DISPONIBLE:
{{relevant_articles_from_search}}

INSTRUCCIONES:
1. Sé cálido pero conciso (max 3 párrafos por respuesta).
2. Usá el nombre del user si lo tenés.
3. Si la pregunta tiene respuesta en knowledge base, respondé directo.
4. Si requiere acción del user en la app, pasos numerados claros.
5. Si requiere acción solo nosotros podemos tomar (refund, edit
   cuenta), creá ticket con CREATE_TICKET tag.
6. Si es bug crítico (pérdida de datos, no puede acceder, pagó pero no
   recibió), escalá con PRIORITY_HIGH tag.
7. NUNCA prometas cosas que no podés cumplir (descuentos, refunds sin
   verificar).
8. Si no estás seguro, decí "déjame escalar esto a nuestro equipo".

Respondé en español neutro latinoamericano.

DEVOLVÉ JSON:
{
  "response": "...",
  "escalation_needed": boolean,
  "escalation_reason": "...",  // si escalation_needed
  "create_ticket": boolean,
  "ticket_priority": "low" | "normal" | "high" | "critical"  // si create_ticket
}
```

### 5.5 Escalación inteligente

**Inmediata (Nivel 3, priority `critical`):**
- Pérdida de datos.
- User no puede acceder a su cuenta.
- Pago realizado pero plan no activado.
- Mención de "vulneración", "hackeo", "datos robados".
- Issues legales o regulatorios.

**Normal (Nivel 2, priority `high` o `normal`):**
- Solicitud de refund parcial o total.
- Cambio de plan complejo.
- Bug que el AI no puede explicar.
- Quejas sobre contenido específico.
- Casos no resueltos en 3 turnos del AI.

**Sin escalación (resueltos por AI):**
- Preguntas de "cómo hacer X".
- Explicaciones del producto.
- Problemas con workarounds conocidos.
- Información sobre planes y pricing.

### 5.6 Costos

Por conversación promedio:
- 5-10 mensajes intercambiados.
- ~2.000 tokens en context, ~500 en response.
- Costo con Haiku: ~$0.003 por conversación.

A 1.000 conversaciones diarias = $90/mes. Despreciable comparado con
costo de personal humano equivalente.

---

## 6. Nivel 2: Human Async

### 6.1 Cuándo se usa

- AI Assistant escaló el caso.
- User solicita explícitamente "hablar con persona".
- Casos que requieren acción manual (refund, cambio de cuenta).

### 6.2 Stack

**MVP (dev solo):**
- Email forwarding a tu email personal.
- Tracking en Notion o Linear.
- Respuesta en 24-48h.

**Crecimiento (con primer hire o freelancer):**
- Helpscout o Intercom (~$25-50/mes).
- Respuesta en 12-24h.

**Escala (>1.000 tickets/mes):**
- Helpscout o Zendesk full.
- Equipo dedicado.
- Respuesta en horas.

### 6.3 Templates de respuesta

**Refund Request - dentro de política:**

```
Hola [Nombre],

Recibimos tu solicitud de reembolso. Verifiqué tu cuenta y todo está
en orden para procesarlo.

[Detalles del reembolso]

Vas a ver el reembolso en tu método de pago en X-Y días hábiles.

Si volvés en el futuro, tu progreso queda guardado.

Saludos,
[Nombre]
```

**Refund Request - fuera de política:**

```
Hola [Nombre],

Gracias por escribirnos.

[Razón de la negativa, ej: "El reembolso solo es posible dentro de los
primeros 14 días" o "Los Sparks consumidos no son reembolsables"]

Lo que sí puedo ofrecerte:
- [Alternativa, ej: "Pausar tu suscripción 1 mes gratis"]

Saludos,
[Nombre]
```

**Bug Report:**

```
Hola [Nombre],

Gracias por reportarlo. Reproduje el problema y ya está en nuestra
lista de fixes.

Mientras tanto, podés:
- [Workaround si existe]

Te aviso cuando esté solucionado, probablemente en X días.

Saludos,
[Nombre]
```

### 6.4 Métricas Nivel 2

- Tiempo de primera respuesta: < 24h.
- Tiempo de resolución: < 72h promedio.
- Customer Satisfaction Score (CSAT) post-resolución.
- % de tickets resueltos en primera respuesta.

---

## 7. Nivel 3: Critical / Priority

### 7.1 Definición de "crítico"

- Pérdida de datos del user.
- Cuenta comprometida (security).
- Pagó pero no recibió plan.
- No puede acceder a su cuenta.
- Issues legales/regulatorios.
- Premium customer con cualquier problema.

### 7.2 SLA

- Primera respuesta: < 2h en horario hábil.
- Resolución: depende del caso, comunicación cada 24h mínimo.

### 7.3 Procesos especiales

**Para data loss:**
- Backup recovery process documentado.
- Comunicación honesta con el user.
- Compensación apropiada (Sparks bonus, mes gratis).

**Para payment issues:**
- Verificación con RevenueCat/Stripe directamente.
- Sin demoras: si pagó, dar plan inmediato, debug después.

**Para security:**
- Forzar password reset.
- Notificar al user sobre actividad sospechosa.
- Auditoría de accesos.
- Posible engagement de servicios externos si hay breach.

---

## 8. Schemas Postgres

### 8.1 Tabla `support_tickets`

```sql
CREATE TABLE support_tickets (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
  email           TEXT,                            -- por si no es user
  subject         TEXT NOT NULL,
  description     TEXT NOT NULL,
  category        TEXT NOT NULL CHECK (category IN (
                    'billing', 'technical', 'account', 'product_feedback',
                    'bug_report', 'feature_request', 'fraud_appeal', 'other'
                  )),
  subcategory     TEXT,
  priority        TEXT NOT NULL DEFAULT 'normal'
                  CHECK (priority IN ('low', 'normal', 'high', 'critical')),
  status          TEXT NOT NULL DEFAULT 'open'
                  CHECK (status IN (
                    'open', 'in_progress', 'waiting_user', 'resolved', 'closed'
                  )),
  source          TEXT NOT NULL CHECK (source IN (
                    'ai_escalation', 'direct_email', 'in_app', 'web_form'
                  )),
  assigned_to     TEXT,                            -- staff id
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at     TIMESTAMPTZ,
  closed_at       TIMESTAMPTZ,
  resolution_notes TEXT,
  csat_score      INT CHECK (csat_score BETWEEN 1 AND 5),
  csat_comment    TEXT
);

CREATE INDEX idx_tickets_user_status
  ON support_tickets(user_id, status)
  WHERE user_id IS NOT NULL;
CREATE INDEX idx_tickets_open
  ON support_tickets(priority, created_at)
  WHERE status IN ('open', 'in_progress');
CREATE INDEX idx_tickets_category
  ON support_tickets(category, created_at DESC);
```

### 8.2 Tabla `ticket_messages`

```sql
CREATE TABLE ticket_messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ticket_id       UUID NOT NULL REFERENCES support_tickets(id) ON DELETE CASCADE,
  author          TEXT NOT NULL,                   -- user_id, 'ai', 'staff:<name>'
  author_type     TEXT NOT NULL CHECK (author_type IN ('user', 'ai', 'staff')),
  content         TEXT NOT NULL,
  internal        BOOLEAN NOT NULL DEFAULT false,  -- nota interna no visible al user
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ticket_messages_ticket
  ON ticket_messages(ticket_id, created_at);
```

### 8.3 Tabla `ai_assistant_conversations`

```sql
CREATE TABLE ai_assistant_conversations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  ended_at        TIMESTAMPTZ,
  message_count   INT NOT NULL DEFAULT 0,
  resolved_by_ai  BOOLEAN NOT NULL DEFAULT false,
  escalated_to_ticket_id UUID REFERENCES support_tickets(id),
  total_tokens    INT,
  total_cost_usd  NUMERIC(10,6),
  user_satisfaction INT CHECK (user_satisfaction BETWEEN 1 AND 5)
);

CREATE INDEX idx_ai_convs_user
  ON ai_assistant_conversations(user_id, started_at DESC);
```

### 8.4 Tabla `ai_assistant_messages`

```sql
CREATE TABLE ai_assistant_messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES ai_assistant_conversations(id) ON DELETE CASCADE,
  role            TEXT NOT NULL CHECK (role IN ('user', 'assistant')),
  content         TEXT NOT NULL,
  metadata        JSONB DEFAULT '{}',              -- escalation_needed, create_ticket, etc.
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 9. API contracts

### 9.1 `startAiAssistantConversation`

**Llamado por:** cliente cuando user abre el AI Assistant.

```typescript
interface StartAiAssistantConversationRequest {
  user_id: string;
  context: {
    current_screen?: string;
    last_action?: string;
    recent_errors?: string[];
  };
}

interface StartAiAssistantConversationResponse {
  conversation_id: string;
  greeting_message: string;        // ej: "Hola María, ¿en qué puedo ayudarte?"
}
```

### 9.2 `sendAiAssistantMessage`

```typescript
interface SendAiAssistantMessageRequest {
  conversation_id: string;
  message: string;
}

interface SendAiAssistantMessageResponse {
  reply: string;
  escalation_needed: boolean;
  ticket_id?: string;              // si se creó ticket
  ticket_priority?: 'low' | 'normal' | 'high' | 'critical';
  message: 'awaiting_user' | 'escalated' | 'resolved';
}
```

**Reglas:**
- Llama AI Gateway task `ai_assistant_response`.
- Si `create_ticket = true` en respuesta del LLM: crear ticket y
  retornar `ticket_id`.
- Si `escalation_needed = true` sin ticket: marcar conversación como
  `pending_human_review`.

### 9.3 `endAiAssistantConversation`

```typescript
interface EndAiAssistantConversationRequest {
  conversation_id: string;
  user_satisfaction?: number;      // 1-5, opcional CSAT
  resolved?: boolean;
}
```

### 9.4 `createTicket`

**Llamado por:** AI Assistant escaladas, formularios manuales,
auth/anti-fraud para casos especiales.

```typescript
interface CreateTicketRequest {
  user_id?: string;
  email?: string;
  subject: string;
  description: string;
  category: TicketCategory;
  subcategory?: string;
  priority?: 'low' | 'normal' | 'high' | 'critical';
  source: 'ai_escalation' | 'direct_email' | 'in_app' | 'web_form';
  initial_messages?: Array<{
    author: string;
    author_type: 'user' | 'ai';
    content: string;
  }>;
}

interface CreateTicketResponse {
  ticket_id: string;
  estimated_response_time_hours: number;
}
```

### 9.5 `getMyTickets`

```typescript
interface GetMyTicketsRequest {
  user_id: string;
  status_filter?: TicketStatus;
}

interface GetMyTicketsResponse {
  tickets: Array<{
    id: string;
    subject: string;
    category: string;
    status: TicketStatus;
    priority: string;
    created_at: string;
    last_message_at: string;
    last_message_excerpt: string;
  }>;
}
```

### 9.6 `getTicketDetail`

```typescript
interface GetTicketDetailRequest {
  ticket_id: string;
  user_id: string;                  // verificar ownership
}

interface GetTicketDetailResponse {
  ticket: SupportTicket;
  messages: TicketMessage[];        // sin internal=true
}
```

### 9.7 `addUserMessageToTicket`

```typescript
interface AddUserMessageRequest {
  ticket_id: string;
  user_id: string;
  content: string;
}
```

### 9.8 `submitCsat`

```typescript
interface SubmitCsatRequest {
  ticket_id: string;
  user_id: string;
  score: number;                    // 1-5
  comment?: string;
}
```

Reglas: solo si ticket `status = resolved`.

### 9.9 Internal: admin endpoints

- `assignTicket(ticket_id, staff_id)`.
- `addStaffMessage(ticket_id, content, internal)`.
- `updateTicketStatus(ticket_id, new_status, resolution_notes)`.
- `bulkClassify(ticket_ids[], category)`.

---

## 10. Edge cases (tests obligatorios)

### 10.1 AI Assistant

1. **User en mitad de conversación con AI cierra app:** conversation
   se cierra automáticamente después de 30 min sin actividad.
2. **AI da respuesta que valida promesa de refund:** prompt explícito
   evita pero edge: detectar palabras "te devuelvo", "garantizamos
   refund" en validation post-LLM; rechazar respuesta.
3. **User envía mensaje en idioma distinto a español:** AI responde en
   español neutro, ofrece "¿prefieres responderme en otro idioma?"
   pero sin servicio formal en otros idiomas.
4. **AI falla 3 veces consecutivas en una conversación:** auto-escalar
   a Nivel 2 con mensaje "voy a pasar tu caso a una persona del
   equipo".

### 10.2 Tickets

5. **Ticket creado por user borrado:** mantener con `user_id = NULL`,
   email persiste para correspondencia.
6. **User crea 5 tickets en 1 hora:** rate limit 3/hora; 4to+ rechazado
   con mensaje "ya tenés N tickets abiertos, esperá nuestra
   respuesta".
7. **Ticket en `waiting_user` por > 14 días:** auto-cierre con flag
   `auto_closed = true`. User puede reabrir.

### 10.3 Refunds

8. **User solicita refund Día 13 post-compra:** dentro de policy 14d,
   procesar.
9. **User solicita refund Día 15:** fuera de policy. Templates de
   respuesta con alternativas.
10. **User compró pack hace 2 meses, consumió 50%, pide refund:** caso
    por caso. Default: no.

### 10.4 Account issues

11. **User report account compromised:** Nivel 3 inmediato. Force
    password reset + audit log.
12. **User no puede recuperar email/SSO:** verificación de pagos
    previos como prueba; manual recovery via support.

### 10.5 Concurrencia

13. **Staff y user editan ticket simultáneamente:** optimistic locking
    via `updated_at`.

---

## 11. Eventos

### 11.1 Emitidos

(Detalle en `cross-cutting/data-and-events.md` §5.10.)

| Evento | Cuándo |
|--------|--------|
| `ticket.created` | Nuevo ticket |
| `ticket.assigned` | Staff toma el ticket |
| `ticket.message_added` | Nueva mensaje en conversación |
| `ticket.resolved` | Marcado resolved |
| `ticket.csat_submitted` | User dio CSAT |
| `ai_assistant.conversation_started` | User abrió chat |
| `ai_assistant.conversation_resolved` | AI resolvió sin escalation |
| `ai_assistant.escalated` | AI escaló a humano |

### 11.2 Consumidos

| Evento | Acción |
|--------|--------|
| `restriction.applied` (anti-fraud) | Crear ticket auto si user pidió apelación |
| `payment.failed` (sparks) | Si fallo crítico (subscription cancelled), crear ticket |
| `incident.declared` (ops, futuro) | Update status page + banner in-app |

---

## 12. Casos especiales

### 12.1 Refund policy

(De `business/legal-compliance.md` §5.1.)

- 14 días desde la compra: refund completo, sin preguntas.
- Después de 14 días: caso por caso, default no.
- Sparks consumidos: nunca reembolsables.
- Cancelación: efectiva al fin del ciclo, sin pro-rateo.
- App Store/Play Store: sus políticas (usualmente más generosas);
  no podemos override.

### 12.2 Account deletion

(Detalle en `architecture/authentication-system.md` §13.)

Soporte solo guía al user al flow self-service. Si flow falla por bug:
escalation a Nivel 3.

### 12.3 GDPR / Privacy requests

User puede solicitar:
- **Right to access:** export en JSON desde Settings (self-service).
- **Right to erasure:** account deletion (self-service).
- **Right to portability:** export en formato estándar (mismo).
- **Right to rectification:** UI para editar; campos no editables vía
  email a privacy@.

Si user contacta `privacy@<dominio>` con request formal:
- Acuse de recibo en 24h.
- Verificación de identidad (responder desde email registrado).
- Resolución en < 30 días (GDPR requirement).
- Log interno.

### 12.4 Abusive behavior

Si user:
- Insulta al equipo de soporte.
- Comparte cuenta múltiples veces.
- Intenta gaming del sistema.

Política:
- Primer caso: warning amigable.
- Segundo caso: suspensión temporal de features.
- Tercer caso: ban con refund proporcional si aplica.

Documentar todo para casos legales eventuales.

---

## 13. Feedback as product input

### 13.1 Categorización

Tree de categorías:

```
billing/
├── refund_request
├── plan_change
├── payment_failed
└── billing_dispute
technical/
├── app_crash
├── audio_issue
├── ai_conversation_bug
└── notifications_not_working
account/
├── login_problem
├── account_deletion
└── data_export
product_feedback/
├── feature_request
├── content_quality
└── difficulty_complaint
other/
```

### 13.2 Reportes mensuales

Cada mes:
- Top 10 categorías por volumen.
- Trending issues (problemas que crecieron mes a mes).
- Issues nuevos.
- Tickets que tomaron más tiempo (oportunidades de mejora).
- Feature requests más solicitados.

Reporte va al backlog del producto.

### 13.3 Conversión de tickets a mejoras

Pipeline:
1. Ticket clasificado.
2. Si categoría supera umbral (ej: 10 similares en mes), convertir a
   "issue" en GitHub.
3. Issue se prioriza en backlog.
4. Cuando se soluciona, actualizar Help Center y AI knowledge base.
5. Notificar a users afectados.

---

## 14. Decisiones cerradas

### 14.1 Soporte 24/7 con AI o limitado a horario laboral con humano:
**AI 24/7, humano horario hábil** ✓

**Razón:** AI escala sin marginal cost; humano fuera de hábil tiene
ROI bajo. Casos críticos pueden esperar 8-12h hasta hábil siguiente
en mayoría.

### 14.2 Foro o Discord comunitario: **Post-MVP** ✓

**Razón:** comunidad requiere moderación constante; sin tracción
suficiente, foro vacío daña marca. Reconsiderar año 2.

### 14.3 Programa "power users" que ayudan con incentivos: **NO en
MVP** ✓

**Razón:** complejidad de governance + riesgo de calidad
inconsistente. Reconsiderar post-validation.

### 14.4 Soporte por WhatsApp: **NO en MVP** ✓

**Razón:** mismo razonamiento que para notifications (costo + setup
de Meta). Reconsiderar con criterios de §14.1 de notifications-system.

### 14.5 Política de refunds más generosa para users long-term: **SÍ
case-by-case, no automático** ✓

**Razón:** balance entre fidelización y abuso. Staff puede otorgar
refund extendido si user tiene > 6 meses activos y caso lo justifica.
Documentar como excepción.

---

## 15. Plan de implementación

### 15.1 Fase 0: Antes del MVP

- [ ] Help Center inicial con artículos críticos.
- [ ] Status page configurada.
- [ ] Email de soporte (`soporte@<dominio>`).
- [ ] Templates de respuesta iniciales.

### 15.2 Fase 1: MVP (meses 0-3)

**Crítico:**
- Help Center con 30-40 artículos.
- Email forwarding configurado.
- Tracking simple en Notion/Linear.
- Tooltips y FAQ contextuales en la app.

**Importante:**
- AI Assistant básico in-app.
- Categorización de tickets.

### 15.3 Fase 2: Escala (meses 3-9)

**Crítico:**
- AI Assistant maduro con knowledge base completa.
- Migración a Helpscout o similar.
- Templates robustos.
- Reportes mensuales de soporte.

**Importante:**
- Self-service para refunds dentro de policy.
- Status page automatizada.

### 15.4 Fase 3: Madurez (año 1+)

- Primer hire de soporte (part-time o freelance).
- Métricas de soporte como North Star.
- Comunidad: foro o Discord.

---

## 16. Métricas

### 16.1 Volumen y eficiencia

- Tickets por 1.000 users activos/mes: target < 5.
- % resueltos en Nivel 0 (self-service): > 50%.
- % resueltos en Nivel 1 (AI): > 25%.
- % escalados a Nivel 2/3: < 25%.

### 16.2 Calidad

- CSAT: > 4.0/5.
- NPS específicamente post-soporte.
- Re-apertura de tickets: < 10%.

### 16.3 Costo

- Costo de soporte por user activo: < $0.10/mes.
- Costo de AI Assistant: < $200/mes hasta 10k MAU.

### 16.4 Producto

- Top 5 categorías de tickets/mes.
- Trending issues.
- Feature requests por volumen.

---

## 17. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`../architecture/authentication-system.md`](../architecture/authentication-system.md) | Account deletion flow. |
| [`../architecture/sparks-system.md`](../architecture/sparks-system.md) | Refund policy de Sparks. |
| [`../architecture/anti-fraud-system.md`](../architecture/anti-fraud-system.md) | Apelaciones de fraude. |
| [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) §4.2.8 | Task `ai_assistant_response`. |
| [`legal-compliance.md`](legal-compliance.md) §5 | Refunds, GDPR requests. |
| [`../ops/incidents-runbook.md`](../ops/incidents-runbook.md) | Comms de incidentes. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.10 | Eventos `ticket.*`. |

---

*Documento vivo. Actualizar cuando cambien procesos, herramientas o
se identifiquen nuevos patrones de tickets.*

# Customer Support System

> Sistema de soporte al usuario diseñado para escalar con un equipo
> mínimo (idealmente cero personas full-time hasta volúmenes altos).
> Combina self-service, IA, y escalación humana solo cuando necesaria.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Alcance:** Sistema completo

---

## 1. Filosofía de soporte

### 1.1 El problema clásico

Las apps consumer suelen abordar soporte de dos maneras malas:

**Opción A:** ignorar el problema hasta que es crítico. Resultado: rage
en App Store, churn alto, costos de adquisición desperdiciados.

**Opción B:** contratar equipo grande de soporte temprano. Resultado:
costos altos, no escalable, distrae del producto.

Para un dev solo trabajando con IA, ninguna opción funciona.

### 1.2 Filosofía adoptada

**Self-service por default:** la mayoría de usuarios prefiere resolver
solos si la documentación está disponible.

**IA como first responder:** chatbot inteligente con contexto del usuario
resuelve el 60-80% de consultas.

**Humano solo en casos críticos:** problemas técnicos complejos, disputas
de pagos, casos sensibles.

**Prevención > resolución:** invertir en evitar problemas (UX clara,
mensajes proactivos) reduce volumen de soporte.

**Cada ticket es feedback de producto:** el soporte no es un costo, es
una fuente de mejora del producto.

---

## 2. Niveles de soporte (escalación)

### 2.1 Estructura en niveles

```
NIVEL 0: Self-Service (60-70% de casos)
├── Help Center con artículos
├── In-app FAQ contextual
├── Tooltips y explicaciones inline
└── Status page público

         ↓ (si no resuelve)

NIVEL 1: AI Assistant (20-30% de casos)
├── Chatbot con contexto del usuario
├── Respuestas a consultas comunes
├── Triage para escalación
└── Manejo de problemas estandarizados

         ↓ (si no resuelve)

NIVEL 2: Human Async (5-10% de casos)
├── Email/in-app messaging
├── Respuesta en 24-48h
├── Casos complejos pero no urgentes
└── Revisión de feedback estructurado

         ↓ (si es crítico)

NIVEL 3: Human Priority (<2% de casos)
├── Premium customers
├── Bugs críticos / pérdida de datos
├── Disputas de pagos
└── Issues de seguridad/privacidad
```

### 2.2 Por qué este modelo funciona para dev solo

- **Nivel 0 y 1 escalan infinitamente:** sin trabajo humano marginal.
- **Nivel 2 absorbe casos manejables:** revisión asíncrona compatible
  con dev solo.
- **Nivel 3 es excepcional:** pocos casos, máxima atención.
- **El sistema aprende:** patrones del Nivel 2 informan mejoras del
  Nivel 1, reduciendo escalación en el tiempo.

---

## 3. Nivel 0: Self-Service

### 3.1 Help Center

**Plataforma sugerida:** GitBook, Notion público, o markdown estático en
el dominio. Para MVP, Notion público es lo más simple.

**Estructura inicial recomendada:**

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

### 3.2 Cobertura inicial

Para MVP, cubrir el 80/20: los temas que generarán 80% de las consultas.

**Prioridad alta (crítico tener desde el día 1):**
- Cómo funciona el free trial.
- Cómo cancelar suscripción (legalmente requerido en muchas jurisdicciones).
- Cómo eliminar cuenta (requerido por App Store).
- Métodos de pago.
- Recuperación de cuenta.

**Prioridad media (semanas 1-4):**
- Sistema de Sparks explicado.
- Roadmap y assessment.
- Logros.

**Prioridad baja (después de feedback real):**
- Edge cases específicos que aparezcan.

### 3.3 In-app FAQ contextual

Cada pantalla principal de la app tiene un ícono "?" que abre FAQ
relevante a esa pantalla.

Ejemplo: en la pantalla de roadmap, "?" muestra:
- ¿Cómo se generó mi roadmap?
- ¿Por qué algunos bloques están bloqueados?
- ¿Puedo saltar bloques?
- ¿Qué pasa si no completo en el tiempo estimado?

Esto reduce escalación porque el usuario obtiene respuesta justo donde
tiene la duda.

### 3.4 Tooltips y explicaciones inline

Para conceptos importantes (Sparks, mastery, niveles), tooltips
explicativos al hover/tap del primer encuentro.

```
🎯 ¿Qué son los Sparks?
Los Sparks son la energía que usás para conversaciones 1 a 1 con la IA
y otras funciones premium.
[Aprender más]
```

### 3.5 Status page

Página pública (uptime.example.com o similar) que muestra:
- Status actual de servicios (app, API, IA, pagos).
- Incidentes pasados.
- Mantenimientos programados.

**Stack sugerido:** Better Stack o Atlassian Statuspage. Tier gratuito
disponible.

Reduce tickets cuando hay outages: usuarios verifican status antes de
contactar.

---

## 4. Nivel 1: AI Assistant

### 4.1 Concepto

Chatbot accesible desde la app y el Help Center que:
- Tiene contexto completo del usuario que habla.
- Conoce el producto a fondo.
- Resuelve consultas comunes con respuesta inmediata.
- Escala a humano solo cuando no puede resolver.

### 4.2 Arquitectura

```
Usuario tiene una pregunta
         ↓
Abre AI Assistant (in-app o web)
         ↓
Sistema captura contexto:
- User ID, plan, historial reciente
- Pantalla actual cuando abrió chat
- Errores recientes en su sesión
         ↓
Pregunta + contexto → LLM con knowledge base
         ↓
LLM clasifica:
- ¿Puedo responder con knowledge base? → Responde
- ¿Es problema técnico identificable? → Diagnóstico + sugiere acción
- ¿Requiere humano? → Crea ticket + asegura al usuario
- ¿Es crítico? → Escala con prioridad alta
         ↓
Respuesta al usuario o ticket creado
```

### 4.3 Knowledge base del AI

El AI Assistant accede a:

**Datos del usuario (en contexto):**
- Perfil, plan, balance de Sparks.
- Historial de pagos, status de suscripción.
- Roadmap actual, progreso.
- Tickets previos del usuario.

**Documentación del producto:**
- Todos los artículos del Help Center.
- FAQs.
- Políticas (privacy, cancelación, refunds).

**Datos operacionales:**
- Status actual de servicios.
- Incidentes recientes.
- Bugs conocidos con workarounds.

### 4.4 Implementación

**Stack recomendado:**

| Componente | Tecnología |
|-----------|-----------|
| LLM | Claude Haiku (costo bajo, suficiente calidad) |
| Knowledge base | Postgres + pgvector para búsqueda semántica |
| Frontend chat | Componente React Native nativo + web |
| Logging | Tabla `support_conversations` |
| Escalation | Trigger a sistema de tickets |

**Prompt del AI Assistant:**

```
Sos un agente de soporte de [App Name], una app de aprendizaje de inglés
para hispanohablantes.

DATOS DEL USUARIO QUE TE CONSULTA:
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
1. Sé cálido pero conciso (máximo 3 párrafos por respuesta).
2. Usa el nombre del usuario si lo tenés.
3. Si la pregunta tiene respuesta en knowledge base, respondé directamente.
4. Si requiere acción del usuario en la app, dale pasos numerados claros.
5. Si requiere acción que solo nosotros podemos tomar (refund, edit cuenta),
   creá ticket con CREATE_TICKET tag.
6. Si es bug crítico (pérdida de datos, no puede acceder, pagó pero no
   recibió), escalá con PRIORITY_HIGH tag.
7. Nunca prometas cosas que no podés cumplir (descuentos, reembolsos sin verificar).
8. Si no estás seguro, decí "déjame escalar esto a nuestro equipo".

Respondé en español neutro latinoamericano.
```

### 4.5 Escalación inteligente

El AI no resuelve todo. Sabe cuándo escalar:

**Escalación inmediata (Nivel 3):**
- Pérdida de datos.
- Usuario no puede acceder a su cuenta.
- Pago realizado pero plan no activado.
- Cualquier mención de "vulneración", "hackeo", "datos robados".
- Problemas legales o regulatorios.

**Escalación normal (Nivel 2):**
- Solicitud de refund parcial o total.
- Cambio de plan complejo.
- Bug que el AI no puede explicar.
- Quejas sobre contenido específico.
- Casos que el AI no resolvió en 3 turnos.

**Sin escalación (resueltos por AI):**
- Preguntas de "cómo hacer X".
- Explicaciones del producto.
- Problemas con workarounds conocidos.
- Información sobre planes y pricing.

### 4.6 Costos del AI Assistant

Por conversación promedio:
- 5-10 mensajes intercambiados.
- ~2.000 tokens en context, ~500 tokens en respuesta.
- Costo con Claude Haiku: ~$0.003 por conversación.

A 1.000 conversaciones diarias = $90/mes. Despreciable comparado con
costo de personal humano equivalente.

---

## 5. Nivel 2: Human Async Support

### 5.1 Cuándo se usa

- AI Assistant escaló el caso.
- Usuario solicita explícitamente "hablar con persona".
- Casos que requieren acción manual (refund, cambio de cuenta).

### 5.2 Stack recomendado

**MVP (dev solo):**
- Email forwarding a tu email personal.
- Sistema simple de tracking en Notion o Linear.
- Respuesta en 24-48 horas.

**Crecimiento (con primer hire o freelancer):**
- Helpscout o Intercom (~$25-50/mes).
- Tier gratuito de Crisp para empezar.
- Respuesta en 12-24 horas.

**Escala (>1.000 tickets/mes):**
- Helpscout o Zendesk full.
- Equipo dedicado.
- Respuesta en horas.

### 5.3 Templates de respuesta

Para casos repetitivos, templates pre-escritos que el humano ajusta:

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

### 5.4 Tracking de tickets

Schema mínimo para tracking:

```sql
CREATE TABLE support_tickets (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID REFERENCES users(id),
  email           TEXT,                -- por si no es usuario
  subject         TEXT NOT NULL,
  description     TEXT NOT NULL,
  category        TEXT,                -- 'billing', 'bug', 'feature_request'
  priority        TEXT NOT NULL DEFAULT 'normal'
                  CHECK (priority IN ('low', 'normal', 'high', 'critical')),
  status          TEXT NOT NULL DEFAULT 'open'
                  CHECK (status IN ('open', 'in_progress', 'waiting_user', 'resolved', 'closed')),
  source          TEXT,                -- 'ai_escalation', 'direct_email', 'in_app'
  created_at      TIMESTAMPTZ DEFAULT now(),
  resolved_at     TIMESTAMPTZ,
  resolution_notes TEXT
);

CREATE TABLE ticket_messages (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ticket_id       UUID NOT NULL REFERENCES support_tickets(id) ON DELETE CASCADE,
  author          TEXT NOT NULL,       -- user_id, 'ai', 'staff:name'
  content         TEXT NOT NULL,
  internal        BOOLEAN DEFAULT false, -- nota interna no visible al usuario
  created_at      TIMESTAMPTZ DEFAULT now()
);
```

### 5.5 Métricas Nivel 2

- Tiempo de primera respuesta (target: <24h).
- Tiempo de resolución (target: <72h promedio).
- Customer Satisfaction Score (CSAT) post-resolución.
- % de tickets resueltos en primera respuesta.

---

## 6. Nivel 3: Critical / Priority

### 6.1 Definición de "crítico"

- Pérdida de datos del usuario.
- Cuenta comprometida (security).
- Pagó pero no recibió plan.
- No puede acceder a su cuenta.
- Issues legales/regulatorios.
- Premium customer con cualquier problema.

### 6.2 SLA

- Primera respuesta: <2 horas en horario hábil.
- Resolución: depende del caso, pero comunicación cada 24h mínimo.

### 6.3 Procesos especiales

**Para data loss:**
- Backup recovery process documentado.
- Comunicación honesta con el usuario.
- Compensación apropiada (Sparks bonus, mes gratis, etc.).

**Para payment issues:**
- Verificación con RevenueCat/Stripe directamente.
- Sin demoras: si pagó, dar plan inmediatamente, debug después.

**Para security:**
- Forzar password reset.
- Notificar al usuario sobre actividad sospechosa.
- Auditoría de accesos.
- Posible engagement de servicios externos si hay breach.

---

## 7. Feedback as product input

### 7.1 El soporte como fuente de información

Cada ticket es data sobre:
- Bugs no detectados.
- UX confusa que genera consultas.
- Features faltantes solicitadas frecuentemente.
- Tendencias del mercado y competencia.

### 7.2 Categorización y agregación

Todos los tickets se clasifican en categorías:

```
Category Tree:
├── Billing
│   ├── refund_request
│   ├── plan_change
│   ├── payment_failed
│   └── billing_dispute
├── Technical
│   ├── app_crash
│   ├── audio_issue
│   ├── ai_conversation_bug
│   └── notifications_not_working
├── Account
│   ├── login_problem
│   ├── account_deletion
│   └── data_export
├── Product Feedback
│   ├── feature_request
│   ├── content_quality
│   └── difficulty_complaint
└── Other
```

### 7.3 Reportes mensuales

Análisis mensual del soporte:

- Top 10 categorías por volumen.
- Trending issues (problemas que crecieron mes a mes).
- Issues nuevos que no existían antes.
- Tickets que tomaron más tiempo (oportunidades de mejora).
- Feature requests más solicitados.

Este reporte va al backlog del producto.

### 7.4 Conversión de tickets a mejoras

Pipeline:

1. Ticket clasificado.
2. Si la categoría supera umbral (ej: 10 tickets similares en mes), se
   convierte en "issue" en GitHub.
3. Issue se prioriza en backlog.
4. Cuando se soluciona, se actualiza Help Center y AI knowledge base.
5. Se notifica a usuarios afectados que reportaron.

---

## 8. Comunicación proactiva

### 8.1 Status updates durante incidents

Si hay un outage o problema afectando muchos usuarios:

- Status page actualizada inmediatamente.
- Banner en la app: "Estamos detectando problemas con X. Trabajando en
  ello."
- AI Assistant pre-cargado con info: si alguien pregunta sobre el problema,
  responde con honestidad.
- Email a usuarios afectados con explicación y compensación si aplica.

### 8.2 Maintenance windows

- Aviso 48h antes en la app.
- Email al inicio del mantenimiento.
- Status page actualizada.
- Idealmente, mantenimiento en horarios de baja actividad (3-5 AM hora local).

### 8.3 Post-mortems para incidents serios

Para outages largos o issues que afectaron muchos usuarios:

- Post mortem público (blog post o status page).
- Explicación honesta de qué pasó.
- Acciones tomadas para prevenir recurrencia.
- Compensación a usuarios afectados.

Esto construye confianza incluso de eventos negativos.

---

## 9. Casos especiales

### 9.1 Refund policy

**Política propuesta:**

- 14 días desde la compra: refund completo, sin preguntas.
- Después de 14 días: caso por caso, default es no refund.
- Sparks consumidos: nunca reembolsables (servicio prestado).
- Cancelación: efectiva al final del ciclo actual, sin pro-rateo.

**En App Store/Play Store:** las stores manejan refunds según sus políticas
(usualmente más generosas), nosotros no podemos override.

### 9.2 Account deletion (legal requirement)

Apple y Google exigen permitir borrado de cuenta desde la app.

Flujo claro y obligatorio (ya documentado en authentication-system.md):
1. Settings → Eliminar cuenta.
2. Confirmación con explicación.
3. Período de gracia 30 días (con opción de cancelar).
4. Hard delete después.

### 9.3 GDPR / Privacy requests

Para usuarios europeos o por regulaciones latinoamericanas:

- Right to access: export de todos sus datos en JSON.
- Right to erasure: borrado completo.
- Right to portability: datos en formato estándar.

Implementar como features self-service en settings.

### 9.4 Abusive behavior

Si un usuario:
- Insulta al equipo de soporte.
- Comparte cuenta múltiples veces.
- Intenta gaming del sistema (cuentas falsas, etc.).

Política:
- Primer caso: warning amigable.
- Segundo caso: suspensión temporal de features.
- Tercer caso: ban con refund proporcional si aplica.

Documentar todo para casos legales eventuales.

---

## 10. Plan de implementación

### 10.1 Fase 0: Antes del MVP (semana -1)

- Crear Help Center inicial con artículos críticos.
- Crear status page.
- Configurar email de soporte.
- Definir templates iniciales de respuesta.

### 10.2 Fase 1: MVP (meses 0-3)

**Crítico:**
- Help Center con 30-40 artículos cubriendo casos comunes.
- Email forwarding configurado.
- Sistema simple de tracking de tickets en Notion/Linear.
- Tooltips y FAQ contextuales en la app.

**Importante:**
- AI Assistant básico in-app.
- Categorización de tickets.

### 10.3 Fase 2: Escala (meses 3-9)

**Crítico:**
- AI Assistant maduro con knowledge base completa.
- Migración a Helpscout o similar.
- Templates de respuesta robustos.
- Reportes mensuales de soporte.

**Importante:**
- Self-service para refunds dentro de política.
- Status page automatizada.

### 10.4 Fase 3: Madurez (año 1+)

- Primer hire de soporte (part-time o freelance).
- Métricas de soporte como North Star.
- Comunidad: foro o Discord donde usuarios se ayudan entre sí.

---

## 11. Métricas críticas

### 11.1 Volume y eficiencia

- Tickets por 1.000 usuarios activos por mes (target: <5).
- % resueltos en Nivel 0 (self-service): target >50%.
- % resueltos en Nivel 1 (AI): target >25%.
- % escalados a Nivel 2/3: target <25%.

### 11.2 Calidad

- Customer Satisfaction Score (CSAT): target >4.0/5.
- Net Promoter Score (NPS) específicamente post-soporte.
- Tasa de re-apertura de tickets (problema no resuelto realmente): target <10%.

### 11.3 Costo

- Costo de soporte por usuario activo: target <$0.10/mes.
- Costo de AI Assistant: target <$200/mes hasta 10k MAU.

### 11.4 Producto

- Top 5 categorías de tickets cada mes.
- Trending issues que sugieren bugs o UX problems.
- Feature requests por volumen.

---

## 12. Decisiones abiertas

- [ ] ¿Soporte en horario 24/7 con AI o limitado a horario laboral con
  escalación humana? Decision: AI 24/7, humano horario hábil.
- [ ] ¿Foro o Discord comunitario? Cuándo introducirlo.
- [ ] ¿Programa de "power users" que ayudan a otros con incentivos
  (Sparks)?
- [ ] ¿Soporte por WhatsApp? (Posible pero requiere setup específico).
- [ ] ¿Política de refunds más generosa para usuarios long-term?

---

## 13. Referencias internas

- `docs/architecture/authentication-system.md` — Account deletion flow.
- `docs/architecture/sparks-system.md` — Refund policy de Sparks.
- `docs/architecture/ai-gateway-strategy.md` — AI Assistant usa Gateway.
- `docs/business/plan_de_negocio.docx` — Pricing y planes.

---

*Documento vivo. Actualizar cuando cambien procesos, herramientas o se
identifiquen nuevos patrones de tickets.*

# Reglas y Principios

> Reglas de diseño, principios rectores y convenciones extraídas de los
> documentos del proyecto. Lo que **siempre** aplica al construir, decidir
> o modificar el producto. Si una decisión nueva contradice una regla
> aquí, primero hay que cuestionar la regla explícitamente.

**Última actualización:** 2026-04
**Cómo usar:** referencia obligada antes de tomar decisiones técnicas o de
producto. Toda excepción debe documentarse.

---

## 1. Filosofía del producto

### 1.1 IA cura, no inventa

La IA selecciona, secuencia y personaliza contenido **predefinido**. No
genera contenido educativo nuevo en tiempo real. Esto garantiza:

- Coherencia pedagógica.
- Costos controlados.
- Experiencia consistente.

Excepción válida: assets parametrizados generados offline con validación
humana. Excepción inválida: roleplays generados en runtime sin curaduría.

### 1.2 Mastery sobre completion

Mejor 5 ejercicios bien hechos que 20 mal. El sistema debe medir dominio
real, no solo conteo. Si una métrica premia esfuerzo sin resultado, está
mal calibrada.

### 1.3 Calidad sobre cantidad de contenido

Mejor 1.000 assets excelentes que 5.000 mediocres. La biblioteca es el
moat del producto: cada asset que se publica representa la calidad
percibida de la app.

### 1.4 Específico para hispanohablantes latinoamericanos

Cada decisión de contenido y UX considera:
- Errores típicos del hispanohablante (fonéticos y gramaticales).
- Contexto cultural latinoamericano (sin caer en estereotipos).
- Vocabulario familiar para explicaciones.
- Variantes de español (mexicano, rioplatense, andino, caribeño).

### 1.5 Móvil primero (iOS y Android simultáneo)

Todas las features se diseñan **mobile-first**. Web es complemento
posterior. App nativa de Windows está **explícitamente fuera de scope** —
PWA cubre el caso de uso desktop. La app de macOS y Linux también está
fuera de scope por la misma razón.

### 1.6 El dev solo es restricción de diseño

Toda elección de stack, proceso o feature considera el costo operativo
para una persona sola asistida por IA. Servicios managed > infraestructura
propia. Automatización > procesos manuales. Simple > sofisticado.

---

## 2. Reglas técnicas

### 2.1 AI Gateway (regla de oro)

> **Si el código de la aplicación menciona "anthropic", "openai",
> "gemini" o cualquier nombre de proveedor fuera de la capa de
> abstracción, es un bug de diseño que debe corregirse.**

Toda llamada a un LLM pasa por el AI Gateway con un `taskId` registrado
en el Task Registry. Sin excepciones.

### 2.2 Multi-proveedor activo desde el día uno

Cada tarea crítica tiene primario + al menos un secundario activo (no
solo configurado). Mantener 5% de tráfico real en el secundario para
verificar la integración y tener cache caliente.

### 2.3 Validación de output de LLM con Zod

Todo JSON devuelto por un LLM se valida contra schema Zod antes de
persistirlo o devolverlo. Si la validación falla, se reintenta una vez;
si sigue fallando, se cae a fallback (template por defecto, modelo
secundario, o error explícito al usuario).

### 2.4 Estado autoritativo en backend

Los clientes son **thin clients** que sincronizan con el backend. La
fuente de verdad siempre es Postgres + servicios. Cache local optimista
para UX fluida, pero el backend valida y reconcilia.

### 2.5 Validación de tokens en cada request autenticada

Cualquier llamada autenticada al backend valida el JWT de Firebase y
verifica que el `firebase_uid` del token corresponde al usuario que se
quiere modificar. Sin excepciones.

### 2.6 Reglas en backend, no en cliente

Anti-fraude, lógica de Sparks, validación de pagos, cobro de operaciones
y cualquier regla de negocio críticas se aplican en backend. El cliente
no es de confianza.

### 2.7 Idempotencia en endpoints de sincronización

Endpoints como `/auth/sync-user` deben ser idempotentes: llamarlos
múltiples veces con los mismos datos no causa problemas. Esto permite
recuperarse de fallos del cliente sin desincronización.

### 2.8 Cobro previo a operación, refund si falla

Antes de ejecutar una operación que cuesta Sparks:
1. Calcular costo.
2. Cobrar al balance del usuario.
3. Ejecutar la operación.
4. Si falla, refund del cobro (registrado como nueva transacción).

Nunca cobrar después de ejecutar (riesgo de regalar valor).

### 2.9 Operaciones de IA tienen un costo en Sparks ajustable

Los Sparks **no son moneda fija**. El costo en Sparks de una operación
es configuración del sistema, no constante hardcodeada. Si el costo del
proveedor cambia >20%, el costo en Sparks se ajusta. El precio del plan
al usuario no cambia.

### 2.10 Acceso a preassets siempre, aún sin Sparks

Aún con balance de 0 Sparks, el usuario tiene **acceso completo a la
biblioteca de preassets**. Los Sparks son para operaciones premium en
tiempo real (conversación 1 a 1), no para acceso básico.

### 2.11 Audit log inmutable de transacciones de Sparks

Toda transacción de Sparks (ingreso o consumo) se registra en
`sparks_transactions` con `balance_before` y `balance_after`. La tabla
es append-only — nunca se editan filas. Discrepancia entre suma de
transacciones y balance actual es alerta crítica.

### 2.12 Soft-delete con período de gracia para account deletion

Account deletion no es inmediato:
1. Usuario solicita.
2. Soft-delete por 30 días (cuenta queda recuperable).
3. Hard-delete después de 30 días (Firebase + Postgres + S3 + analytics).

Esto cumple políticas de App Store / Play Store y reduce regret-cancellations.

### 2.13 Migraciones de modelos LLM por rollout progresivo

Toda migración a un modelo nuevo sigue el protocolo:
1. Evaluación offline contra golden dataset.
2. Shadow mode (output loggeado, no devuelto).
3. Canary 5%.
4. Ramp up: 25% → 50% → 100%.
5. Modelo viejo en fallback durante 30 días, luego se retira.

Rollback automático si: validation rate cae >3%, latencia p95 sube >50%,
error rate sube >2%, costo sube >20% inesperadamente.

### 2.14 Logs y eventos

- Naming de eventos: snake_case, verbo en pasado (`user_signed_up`).
- No mandar PII en eventos de analytics. Usar UUID hasheado.
- No duplicar eventos en cliente y backend.
- Evitar eventos genéricos sin contexto (`button_clicked` sin propiedades
  útiles).

---

## 3. Reglas de pedagogía y contenido

### 3.1 Calibración por nivel CEFR

Las métricas no son absolutas: se calibran por nivel. Un B1 con 95 WPM
está bien; un C1 con 95 WPM está mal. Errores aceptables en speech
espontáneo no son aceptables en escritura formal.

### 3.2 Multi-dimensional, nunca métrica única

El dominio del inglés se evalúa en 5 dimensiones independientes:
pronunciación, fluidez, gramática, vocabulario, listening. Mostrar al
usuario un solo score "global" oculta información valiosa. Mostrar las
5 con explicación.

### 3.3 Detección de "struggling" antes que castigo

Si el usuario hace 5+ intentos sin alcanzar el mínimo:
- No bloquear ni penalizar.
- Marcar el bloque como `STRUGGLING`.
- Sugerir prerequisitos que tal vez faltó dominar.
- Ofrecer versión simplificada.
- Mensaje sin culpa: "este bloque es desafiante, probemos otro primero".

### 3.4 Decay de skills sin uso

Habilidades no practicadas decaen. Cada 30 días el sistema verifica
sub-skills "mastered" con un ejercicio. Si falla, vuelve a `developing`
y se re-aborda. Si pasa, se actualiza la fecha de último uso.

### 3.5 Revisión humana del contenido generado por IA

Sample obligatorio de revisión humana de assets generados por IA:
- Primeros 200 assets: 100%.
- 201–500: 50%.
- 501–2.000: 20%.
- 2.000+: 10%.

Cada rejection se documenta para mejorar prompts futuros.

### 3.6 Variantes de inglés explícitas

Cada usuario tiene una `target_english_variant` (americano, británico,
neutral) derivada de su ubicación + objetivo. El contenido respeta esta
variante. Default por país documentado en
`product/student-profile-and-assessment.md`.

---

## 4. Reglas de motivación y UX

### 4.1 Calibrar magnitud de celebración con magnitud de logro

Una celebración de "completaste 1 ejercicio" no debe sentirse igual que
"completaste tu primer track". Si todo es épico, nada lo es.

| Magnitud | Trigger | Celebración |
|----------|---------|-------------|
| Pequeño | 1 ejercicio | Mensaje breve, feedback positivo |
| Medio | Daily goal | Animación + Sparks + toast |
| Grande | Logro desbloqueado | Pantalla completa, animación |
| Épico | Nivel del roadmap | Pantalla, audio, certificado parcial |
| Legendario | Track completo | Experiencia + certificado + opción compartir |

### 4.2 El sistema es aliado, no acosador

- Detectar burnout y sugerir descanso.
- Vacation mode sin culpa.
- Sugerir bajar metas si consistentemente no se cumplen.
- Apps que se sienten como obligación se abandonan.

### 4.3 Friction proporcional al riesgo

No molestar al 99% de usuarios legítimos para protegernos del 1% malicioso.
Aumentar fricción solo cuando hay señales sospechosas.

### 4.4 Indicación previa de costo en Sparks

Antes de iniciar cualquier operación que cobre Sparks:

```
"Iniciar conversación de 10 minutos
Costo: 10 Sparks
Tu balance: 187 → 177
[Iniciar] [Cancelar]"
```

Nunca cobrar Sparks sin que el usuario haya visto y confirmado el costo.

### 4.5 Transparencia sobre cambios de costo

Cualquier ajuste al costo en Sparks de una operación:
- Se comunica con **30 días de anticipación** a usuarios activos.
- Aplica solo a operaciones futuras, no retroactivamente.
- Se documenta en `docs/decisions/sparks-pricing-changes.md`.

### 4.6 Personalizar el motivador

Diferentes usuarios responden a diferentes mecánicas (streaks, logros,
social, progreso, propósito). El sistema detecta y enfatiza lo que
funciona para cada uno. Recalculado mensualmente.

### 4.7 Tono de comunicación

Español neutro latinoamericano, cercano pero profesional. **No usar:**

- Modismos muy locales que excluyen otros países.
- Excesiva familiaridad ("flaco", "compa", "compita").
- Tono gringo traducido literalmente ("¡Tú lo lograste, campeón!").

---

## 5. Reglas de soporte y operación

### 5.1 Niveles de soporte (escalación)

Toda solicitud sigue este orden:
1. Nivel 0 — Self-service (60–70% objetivo).
2. Nivel 1 — AI Assistant (20–30%).
3. Nivel 2 — Humano async (5–10%, respuesta <24–48h).
4. Nivel 3 — Humano prioritario (<2%, respuesta <2h en hábil).

Solo escalar al siguiente nivel si el actual no resolvió. AI nunca
promete cosas que no puede cumplir.

### 5.2 Transparencia en incidentes

Frente a outages o problemas serios:
- Status page actualizada inmediatamente.
- Banner en la app.
- Email a usuarios afectados con explicación honesta.
- Post-mortem público para incidentes serios.
- Compensación apropiada (Sparks bonus, mes gratis).

### 5.3 Refund policy

- 14 días desde la compra: refund completo, sin preguntas.
- Después de 14 días: caso por caso, default es no refund.
- Sparks consumidos: nunca reembolsables (servicio prestado).
- Sparks no consumidos en pack: reembolsables prorrateados si <30%
  consumido y <14 días.
- Cancelación: efectiva al final del ciclo actual, sin pro-rateo.

### 5.4 Cada ticket es feedback de producto

El soporte no es un costo, es una fuente de mejora. Patrones de tickets
informan el backlog. Si una categoría supera umbral (ej: 10 tickets
similares en mes), se convierte en issue priorizado.

---

## 6. Reglas de privacidad y datos

### 6.1 Pedir solo lo estrictamente necesario

| Dato | ¿Se guarda? |
|------|------------|
| Email, nombre, foto | Sí |
| Firebase UID, provider | Sí |
| Teléfono | Solo si usa Phone OTP |
| Fecha de nacimiento, género, dirección | NO |

### 6.2 Consent granular para datos sensibles

En el onboarding, opt-ins separados para:
- ☑ Procesar audios para mejorar pronunciación (necesario, opt-in default).
- ☑ Usar datos anonimizados para mejorar el producto.
- ☐ Compartir datos anonimizados con investigación lingüística (opt-in
  explícito, no por defecto).

### 6.3 Retención de datos

| Tipo | Retención |
|------|-----------|
| Audios crudos | 30 días |
| Transcripciones | Indefinido (anonimizadas a los 90 días) |
| Eventos de comportamiento | 12 meses |
| Datos del perfil | Mientras la cuenta esté activa |
| Resultados de assessment | Indefinido |

### 6.4 Right to erasure se propaga a todas las tools

Account deletion debe propagar a: Postgres, PostHog, Firebase Analytics,
Sentry, RevenueCat y cualquier tool externa. Idealmente automatizado en
el flow de hard-delete.

### 6.5 PII nunca en analytics

PostHog y Firebase Analytics aceptan UUIDs. No mandar email, nombre real,
ni teléfono a herramientas de analytics. Para análisis cualitativo con
PII (revisión de tickets), tools separadas con permisos restringidos.

---

## 7. Reglas de pricing y monetización

### 7.1 Pagos por canal con descuento explícito

En la app: pagos in-app via Apple/Google (15–30% comisión).
En la web: ofrecer descuento por pagar fuera de las stores (3–5% comisión
con Stripe). Permitido por las políticas vigentes en muchas
jurisdicciones (verificar al implementar).

### 7.2 Pricing localizado por país

Aunque el plan en MXN es la referencia, se muestra en moneda local del
usuario. Argentina requiere ajuste especial por inflación: revisar
mensualmente o pricing en USD con conversión al cobrar.

### 7.3 Sparks no son moneda

Documentar en T&C:
- Sparks no son moneda de curso legal.
- Sparks no son transferibles entre usuarios.
- Sparks no se pueden convertir a dinero.
- La empresa puede ajustar el costo en Sparks de operaciones con
  notice previo de 30 días.

---

## 8. Reglas de experimentación

### 8.1 Cuándo experimentar

**SÍ:** flujos críticos (onboarding, paywall, pricing), nuevas features
con incertidumbre, cambios en algoritmos, mensajes en momentos clave.

**NO:** bug fixes, mejoras obvias de UX, compliance changes, cambios de
infrastructure no visibles al usuario.

### 8.2 Sample size mínimo

- Mínimo 1.000 usuarios por variante para detectar efectos del 10%+.
- p-value < 0.05 antes de declarar ganador.
- 80% statistical power.
- No detener el experimento prematuramente.

### 8.3 Guardrail metrics

Cada experimento define **métricas guardrail** que NO deben empeorar.
Si una guardrail cae, se rollback aunque la primary mejore.

---

## 9. Reglas de stack (qué NO agregar todavía)

Agregar complejidad solo cuando hay justificación clara. Lista explícita
de **lo que NO se debe sumar al stack** en MVP:

- ❌ Kubernetes — overkill hasta múltiples servicios y equipo de ops.
- ❌ Pinecone u otra vector DB managed — pgvector cubre los casos iniciales.
- ❌ Kafka — Inngest events alcanza hasta volúmenes medios.
- ❌ Datadog — Sentry + PostHog + BetterStack cubren todo lo necesario.
- ❌ GraphQL — agrega complejidad sin beneficio claro a este tamaño.
- ❌ Microservicios — monolito modular es la respuesta correcta.
- ❌ Sift, FingerprintJS managed — caros, esperar a justificar a escala.
- ❌ Helpscout / Intercom — empezar con email forwarding + Notion/Linear.
- ❌ Auth0 — overkill, Firebase Auth alcanza.

---

## 10. Reglas de rollout

### 10.1 Soft launch antes de hard launch

Lanzar primero a un país (México) por 4–6 semanas. Permite:
- Detectar bugs sin escala completa.
- Validar unit economics con datos reales.
- Iterar marketing y onboarding.

Solo después: expansión a Argentina, Colombia, Chile, Perú.

### 10.2 Feature flags desde el día uno

Toda feature que tenga incertidumbre se lanza detrás de un feature flag
en PostHog. Permite:
- Rollback instantáneo si algo se rompe.
- A/B testing sobre la marcha.
- Habilitar gradualmente por país, plan, cohort.

### 10.3 Dejar siempre la puerta abierta

El usuario que abandonó hoy puede volver mañana. Sus datos se conservan
mientras la cuenta esté activa. Después de account deletion, no hay
retorno (es una elección explícita del usuario, respetada).

---

## 11. Convenciones de documentación

### 11.1 Cada sistema es un documento vivo

Los documentos en `docs/architecture/` y `docs/product/` son **fuentes
de verdad** del diseño del sistema. Si la implementación diverge, se
actualiza el documento (no el código a escondidas).

### 11.2 Cada documento incluye al menos

- Estado (Diseño / Beta / Producción).
- Última actualización.
- Owner.
- Alcance.
- Decisiones abiertas listadas explícitamente.
- Plan de implementación por fases.
- Métricas de éxito.

### 11.3 Decisiones rechazadas se documentan

Cada documento tiene una sección de "decisiones que NO se tomaron y por
qué" para evitar revisitarlas. Ejemplo: "App nativa de Windows: PWA
cubre el caso, app nativa agrega overhead sin ROI claro."

### 11.4 Referencias entre documentos

Cada documento referencia los demás documentos relevantes en su sección
"Referencias internas". Mantener sincronizado al mover o renombrar.

---

## 12. Reglas que se pueden cuestionar

Estas reglas son fuertes pero no inviolables. Cuestionar **explícitamente**
con datos o argumentos cuando se proponga una excepción:

- "Móvil primero" — válido si surge evidencia de que el target prefiere
  desktop.
- "IA cura, no inventa" — válido si los costos bajan dramáticamente y la
  generación dinámica supera en calidad a la curaduría.
- "PWA cubre desktop" — válido si aparecen casos de uso nuevos que la PWA
  no soporta bien.
- "Mastery sobre completion" — válido si datos muestran que algunos
  usuarios abandonan por ser muy estricto.

Cualquier excepción se documenta en un ADR (`docs/decisions/ADR-XXX...`).

---

*Documento vivo. Si una regla nueva emerge de aprendizaje en producción,
agregarla aquí. Si una regla deja de aplicar, removerla con justificación
documentada.*

# CLAUDE.md — guía para agentes AI que trabajan en este repo

> Este archivo es el punto de entrada para cualquier agente AI (Claude
> Code, Cursor, Copilot Workspace, etc.) que vaya a leer, diseñar o
> implementar código en este repositorio. Si sos un agente, leé este
> archivo **antes** de tocar cualquier otro.

---

## 1. Qué es este repositorio

`english-spark` es una app de aprendizaje de inglés para hispanohablantes
latinoamericanos, asistida por IA. Por ahora, **no contiene código** —
solo el diseño completo del sistema en `docs/`. La implementación está
por arrancar.

**El producto se construye así:**
- App móvil iOS + Android con React Native + Expo.
- Web con Next.js.
- Backend con Cloudflare Workers + Postgres (Supabase) + Redis (Upstash).
- IA detrás de un AI Gateway propio (multi-proveedor: Anthropic, Google,
  OpenAI, Azure, modelos propios).

Stack completo y justificación: [docs/contexto.md](docs/contexto.md).

---

## 2. Audiencia de la documentación

La documentación está escrita **para vos, agente AI**, con el objetivo
de que puedas implementar sin ambigüedad. Esto significa:

- Cada documento incluye schemas completos, fórmulas con constantes
  numéricas, contratos de API, edge cases enumerados.
- Cuando hay decisiones tomadas, están explícitas con la justificación.
- Cuando hay decisiones abiertas, están listadas en `docs/pendientes.md`
  o en la sección "Decisiones abiertas" del documento del sistema.

**No supongas. No inventes. Leé.**

---

## 3. Orden de lectura recomendado

Para una pregunta o tarea nueva:

1. **[README.md](README.md)** — qué es el producto, en una pasada.
2. **[docs/contexto.md](docs/contexto.md)** — síntesis ejecutiva con
   stack, sistemas, NSM, roadmap.
3. **[docs/reglas.md](docs/reglas.md)** — principios y reglas que
   **siempre** aplican. Si tu cambio contradice una regla, parar y
   escalar a humano.
4. **[docs/pendientes.md](docs/pendientes.md)** — decisiones abiertas y
   trabajo pendiente.
5. **El documento específico del sistema** que tu tarea toca (en
   `docs/architecture/`, `docs/product/` o `docs/business/`).
6. **Los documentos referenciados** en la sección "Referencias internas"
   del documento del sistema.

Si después de leer todo eso sigue habiendo ambigüedad, **escalar a
humano**. No inventar.

---

## 4. Estructura del repo

```
.
├── README.md
├── CLAUDE.md                ← este archivo
└── docs/
    ├── contexto.md          ← entrada
    ├── pendientes.md        ← decisiones abiertas
    ├── reglas.md            ← principios autoritativos
    ├── architecture/        ← cómo está construido
    ├── product/             ← qué hace y por qué
    ├── business/            ← cómo se mide y opera
    ├── decisions/           ← ADRs (a poblar)
    ├── cross-cutting/       ← temas transversales (a poblar)
    └── ops/                 ← operación (a poblar)
```

---

## 5. Convenciones del codebase (futuro)

Cuando empieces a generar código, respetá:

### 5.1 Naming

| Capa | Convención |
|------|-----------|
| Tablas Postgres | `snake_case` |
| Columnas Postgres | `snake_case` |
| Tipos / interfaces TypeScript | `PascalCase` |
| Variables / funciones TypeScript | `camelCase` |
| Constantes | `UPPER_SNAKE_CASE` |
| Eventos de analytics | `snake_case`, verbo en pasado (`user_signed_up`) |
| Archivos de prompts versionados | `<task_id>/v<n>_<provider>.txt` |
| Archivos de docs | `kebab-case.md` |
| IDs de sub-skills | `<dim>_<descripcion_corta>` (ej: `pron_th_voiceless`) |
| IDs de logros | `snake_case` (ej: `streak_7`, `talker_50`) |

### 5.2 Schemas como fuente de verdad

- Los schemas de Postgres definidos en cada documento son **autoritativos**.
- Si necesitás agregar/cambiar un campo, primero **actualizá el documento**,
  después implementá.
- TypeScript interfaces deben matchear los schemas de Postgres con
  conversión `snake_case` → `camelCase` automatizada (Zod schemas
  derivados).

### 5.3 Validación

- Toda salida de LLM se valida con **Zod** antes de persistirla.
- Toda entrada del cliente al backend se valida con Zod.
- Si la validación falla, retry una vez; si vuelve a fallar, fallback
  documentado o error explícito.

### 5.4 IA

- **Nunca** importes SDK de proveedor de LLM directamente desde código de
  negocio. Todo pasa por el AI Gateway con un `taskId` registrado.
- Si necesitás una nueva tarea de IA, **registrala primero** en el Task
  Registry (ver `docs/architecture/ai-gateway-strategy.md` §4).

### 5.5 Stack — qué NO agregar

Lista explícita de dependencias **prohibidas** en MVP (sin justificación
documentada en ADR):

- Kubernetes, Docker Compose para producción.
- Pinecone u otra vector DB managed (usar pgvector).
- Kafka, RabbitMQ (usar Inngest events).
- GraphQL (usar REST simple sobre Workers).
- ORMs pesados tipo Prisma con runtime grande (preferir Drizzle o SQL
  directo con kysely).
- Redux / MobX en el cliente (usar Zustand o React Context).
- Servicios externos no listados en `docs/contexto.md` §3.

Si necesitás algo no listado, escribir un ADR primero.

### 5.6 Tests

- **Unit:** funciones puras, lógica de scoring, parsers, validators.
- **Integration:** endpoints, jobs nocturnos, flujos de Sparks
  (cobro/refund/expiración).
- **E2E:** **fuera de scope** explícitamente. No proponer setup de
  Playwright/Detox.

Detalle pendiente en `docs/cross-cutting/testing-strategy.md` (a crear).

---

## 6. Cómo tomar decisiones cuando la doc no es explícita

En orden:

1. **Leé el documento del sistema más relevante completo**, no solo la
   sección que parece tocar tu tarea.
2. **Buscá en documentos referenciados** (sección "Referencias internas").
3. **Buscá en `docs/reglas.md`** si hay un principio que aplique.
4. **Buscá en `docs/pendientes.md`** si la decisión está abierta.
5. **Si sigue sin estar claro**, NO inventes:
   - Si es un detalle pequeño de implementación: usá la opción más
     conservadora (menos cambios, menos costo, menos complejidad) y
     documentá la elección con un comentario en el código apuntando
     al doc relevante.
   - Si es una decisión arquitectónica o de producto: parar, abrir issue
     o pedir confirmación al humano.

**Nunca inventar lo siguiente sin confirmación humana:**

- Esquemas de base de datos.
- Endpoints públicos de API.
- Costos en Sparks de operaciones nuevas.
- Pricing al usuario.
- Política de fraude.
- Política de privacy / data retention.
- Elecciones de proveedores de IA (modelo o API).
- Estrategia de migración de modelos LLM.

---

## 7. Cómo proponer cambios al diseño

Si mientras implementás detectás que el diseño documentado tiene un
problema (contradicción, gap, error técnico):

1. **No silenciosamente desviar la implementación del diseño.** Si lo
   hacés, el código y la doc divergen y el siguiente agente se confunde.
2. **Proponer el cambio explícitamente:**
   - Resumir qué dice la doc.
   - Resumir qué descubriste en la implementación.
   - Recomendar: actualizar la doc, actualizar el código, o ambos.
3. **Esperar confirmación humana** antes de modificar la doc o seguir
   implementando con el cambio.

Para cambios arquitectónicos importantes: **escribir un ADR** en
`docs/decisions/ADR-XXX-<slug>.md` antes de implementar.

---

## 8. Reglas autoritativas

Las reglas en [docs/reglas.md](docs/reglas.md) son **autoritativas**.
Si una nueva propuesta contradice una regla, primero hay que cuestionar
la regla explícitamente con datos. No silenciosamente romperla.

Resumen de las más críticas para implementación:

- **AI Gateway es la única vía de acceso a LLMs.** Mencionar "anthropic",
  "openai", "google" fuera del Gateway es un bug de diseño.
- **Cobro de Sparks antes de operación, refund si falla.** Nunca cobrar
  después.
- **Acceso a preassets siempre, aún con 0 Sparks.**
- **Validación de JWT de Firebase en cada request autenticada.**
- **Idempotencia en endpoints de sync.**
- **Audit log inmutable de transacciones de Sparks** — append-only.
- **No PII en eventos de analytics.**
- **Soft-delete con 30 días antes de hard-delete.**

---

## 9. Estado del proyecto

**Diseño v1.0 completo.** Implementación: pre-Sprint 0.

Los documentos pueden tener estado `Diseño v1.0`, `Diseño v1.1
(profundizado)`, `Beta`, o `Producción`. Verificar el estado del
documento que estás leyendo antes de implementarlo: si es `v1.0` y
estás cerca de codificar, considerá pedir una pasada de profundización
primero.

---

## 10. Comunicación

- Idioma: español rioplatense / neutro latinoamericano para
  documentación, comentarios de commit, mensajes al usuario.
- Código: nombres de identificadores en inglés (estándar).
- Output al usuario final de la app: español neutro latinoamericano,
  ver tono en `docs/reglas.md` §4.7.

---

*Si modificás este archivo, asegurate de que la guía sigue siendo válida
para un agente que entra fresco al repo sin contexto previo.*

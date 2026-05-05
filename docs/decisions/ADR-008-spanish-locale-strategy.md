# ADR-008: Mexicano-tuteo como locale único en MVP, expansión post-MVP

**Status:** Accepted
**Date:** 2026-05
**Author:** —
**Audiencia:** agente AI implementador.

---

## Contexto

El producto target a hispanohablantes latinoamericanos. Hay 9 países
mayoritariamente hispanohablantes en Latam con variantes regionales
distintas:

| Variante | Países | Pronombre 2P | Característica clave |
|----------|--------|--------------|---------------------|
| Mexicano | MX | tú | "ahorita", "padrísimo" |
| Rioplatense | AR, UY | **vos** | Voseo, "boludo", "che" |
| Andino | CO, EC, PE, BO | tú (mayoría) | "vaina", "chévere" |
| Caribeño | DO, CU, PR, VE | tú | "qué pasa", "asere" |
| Chileno | CL | tú (informal voseo de moda) | "weón", "po" |

Decisiones que dependen del locale:
- **Copy de UI** (botones, labels, mensajes, errors).
- **Notifications** (15 notification_ids con copy literal — ver
  `push-notifications-copy-bank.md`).
- **Onboarding y assessment** (16+8 pantallas con copy literal).
- **Prompts AI Gateway** que generan contenido user-facing (insights,
  notification personalization, customer support responses).
- **Voice TTS para listening exercises del producto**: el producto
  enseña inglés, así que TTS pedagógico es en inglés. Pero los
  characters AI roleplay también pueden hablar a veces en español
  (ej: bilingual scaffolding) — fuera de scope MVP.

Pregunta original (originalmente abierta en `pendientes.md` §5
implícita, decisión inicial tomada en docs §2.2 de
`cross-cutting/i18n.md`):

> **¿Qué variante de español usamos como locale único MVP?**

Inicialmente la documentación se escribió en **voseo argentino**
(decisión del autor original). En 2026-05 se cuestionó esto y se
decidió **switch a mexicano-tuteo** como locale único MVP.

Este ADR formaliza esa decisión y documenta:
1. La rationale para tuteo vs voseo (vs neutro vs región-específico).
2. La estrategia operativa de "un solo set de strings, mexicano-tuteo".
3. El roadmap de expansión (pt-BR, es-ES, otros).
4. Los principios para mantener consistencia.

## Decisión

**Locale único en MVP: español mexicano con tuteo (`es-MX` lógicamente,
pero stored como `es` por unicidad).**

### Reglas operativas

| Aspecto | Regla |
|---------|-------|
| **Pronombre 2P** | "tú" (no "vos", no "usted") |
| **Imperativos** | "puedes", "habla", "haz", "ve", "sigue" (no "podés", "hablá", "hacé") |
| **Vocabulario** | Neutral con leve preferencia mexicana ("computadora", "celular", "manejar") |
| **Modismos locales** | Evitados (ni "ahorita", "neta", "wey", "padrísimo" ni argentinismos) |
| **Diminutivos** | Limitados (no "ratito", "cuentita") |
| **Tono** | Cercano-profesional, no infantil |

(Detalle exhaustivo en `cross-cutting/i18n.md` §2.2 incluida tabla
de decisiones específicas para 14 conceptos comunes.)

### Storage

- `student_profiles.native_language = 'es-MX'` por default para todos
  los users (con override editable).
- Locale técnico de strings JSON: `es` (un único archivo
  `apps/mobile/locales/es.json`).
- Cuando expandamos post-MVP, se agregan archivos override:
  `es-AR.json`, `pt-BR.json`, etc. (solo con keys que difieren).

### Aplicación across surfaces

Todos los siguientes outputs DEBEN usar mexicano-tuteo:

| Surface | Doc autoritativo |
|---------|------------------|
| UI strings (mobile + web) | `cross-cutting/i18n.md` §7.5 |
| Notifications | `push-notifications-copy-bank.md` |
| Onboarding 16 pantallas | `product/onboarding-flow.md` |
| Post-assessment 8 pantallas | `product/post-assessment-flow.md` |
| Prompts AI Gateway user-facing | `architecture/ai-gateway-strategy.md` |
| Customer support templates | `business/customer-support-system.md` |
| Legal copy user-facing | `business/legal-compliance.md` |
| Email transaccional propio | `cross-cutting/i18n.md` §7.4 |

CI/CD debe incluir validation que rechaza voseo (lista de palabras
bloqueadas: "vos", "podés", "querés", "tenés", "registrate", "iniciá",
etc. — ver `push-notifications-copy-bank.md` §10.5 para guilt detector
similar).

### Excepciones explícitas (donde voseo SÍ aparece)

Hay 4 lugares donde voseo aparece legítimamente en docs:

1. **`docs/reglas.md` §4.7:** lista voseo como ejemplo a evitar.
2. **`docs/cross-cutting/i18n.md` §2.2:** tabla de decisiones marca
   "podés" como rechazado.
3. **`docs/explorations/cultural-annotations.md`:** documenta voseo
   como variante regional en tooltips para users argentinos
   (educational).
4. **`docs/product/student-profile-and-assessment.md` §9.1:** menciona
   "voseo en explicaciones" para Argentina como variante de contenido
   regional (no UI).

Estas son referencias documentales, no copy de producto.

### Roadmap de expansión

| Fase | Locale | Trigger | Estimación |
|------|--------|---------|------------|
| MVP | `es` (mexicano-tuteo) | — | Sprint 1 |
| Fase 2 | `pt-BR` | Producto validado en Latam (D30 retention > 30%, > 5k users), prep expansión Brasil | Mes 6-9 post-MVP |
| Fase 2 | `en` para hispanos en EEUU | Si > 10% de signups vienen de US | Mes 6-12 post-MVP |
| Fase 3 | `es-ES` (España con tuteo, vocabulario local) | Si > 5% de signups de España y métricas justifican esfuerzo | Año 2 |
| Fase 3 | `es-AR` override (vocabulario, sin voseo) | Si churn AR > 20% mayor que LatAm promedio (improbable) | Año 2 |

**Importante:** la expansión a `pt-BR` requiere **traducción
profesional**, no AI translation literal. Hay diferencias
estructurales del portugués que la traducción literal no captura
(uso de "você" vs "tu", colocação pronominal).

### Decisiones explícitas que NO se cambiarán por país

Algunas decisiones son **globales** y no se localizan por país aún
con expansión:

- **Sparks** se llama "Sparks" en todos los locales (marca, no
  traducir).
- **Tracks** se llaman "Job Ready", "Travel Confident", "Daily
  Conversation" en todos los locales (nombre de marca interno + el
  inglés ES el producto target).
- **CEFR levels** (B1, B2, etc.) se mantienen sin traducir (son
  estándar internacional).

## Alternativas consideradas

### A. Voseo argentino (decisión original, revertida)

**Rechazada en 2026-05.**

**Razones:**
- Target inicial **México** (mercado más grande en Latam, mejor data
  de adopción de apps de aprendizaje).
- Voseo en México sería percibido como **extranjero** y posiblemente
  rechazado.
- Voseo argentino en docs/notifications crearía fricción inmediata
  para users mexicanos, colombianos, peruanos, chilenos.
- Reciprocidad asimétrica: tuteo en Argentina **no genera fricción
  significativa** (argentinos están expuestos constantemente a tuteo
  en TV, redes, productos globales).

### B. Español neutro / latinoamericano genérico

**Considerada, parcialmente adoptada.**

Esta era la idea original detrás del locale `es` único. En la
práctica, "español neutro" no existe — siempre tiene algún sesgo
regional. Decidir explícitamente "neutro con leve preferencia
mexicana" es más honesto y operativo.

**Por qué "leve preferencia mexicana" y no "neutro puro":**
- "Neutro puro" tiende a sonar telenovela, anticuado, o sin
  personalidad.
- Sesgo mexicano leve es comprensible en toda Latam y México es
  target #1.
- Permite mantener cierto sabor local sin excluir.

### C. Variantes regionales separadas desde MVP

**Rechazada para MVP.**

3-5 archivos JSON (es-MX, es-AR, es-CO, es-CL, es-PE) significa:
- Triplicar/quintuplicar el costo de traducción.
- Quintuplicar el costo de mantenimiento (cada feature requiere update
  en N archivos).
- Bug surface aumenta proporcionalmente.

**Reconsiderar post-MVP** si data muestra rechazo regional. Para hoy
es over-engineering.

### D. Inglés como locale único

**Rechazada.**

El producto enseña inglés a hispanohablantes que NO dominan inglés
todavía. Si la UI es en inglés, el user que está en B1 tiene fricción
constante con UI que no entiende. Counterproductive.

**Excepción post-MVP:** users avanzados (B2+) pueden tener opción de
UI en inglés (como Duolingo permite) — feature post-MVP.

### E. Mezcla cultural (tuteo Latam + voseo solo en AR/UY)

**Rechazada.**

Detectar país y servir voseo solo en AR/UY es técnicamente factible
pero operacionalmente costoso:
- Duplica el esfuerzo de copy.
- Bug si usuario viaja o cambia país.
- Voseo argentino no es "voseo universal" — chileno es distinto, no
  hay "voseo neutral".

Más simple: tuteo para todos, expansión por demanda real post-MVP.

### F. Usted formal

**Rechazada.**

"Usted" suena distante en producto consumer. App de aprendizaje
beneficia de tono cercano (ver §2.4 de i18n.md, tono cercano pero
profesional). "Usted" se usaría en productos B2B / banca / legal,
no aquí.

## Consecuencias

### Positivas

- **Un solo set de strings** = costos de traducción y mantenimiento
  mínimos.
- **Tuteo es más universal en Latam** (incluso usuarios de AR/UY no
  rechazan tuteo).
- **Target México** se siente nativo.
- **Roadmap de expansión claro** (pt-BR primero, después variantes
  españolas si data lo justifica).
- **Documentación AI-ready:** prompts a LLMs especifican
  "mexicano-tuteo" como instrucción clara, no ambiguity.

### Negativas

- Users de Argentina/Uruguay pueden percibir el tuteo como menos
  cercano que el voseo. Mitigación: tono cálido sin necesidad de
  voseo. Métrica: track signup → first session retention en AR/UY,
  comparar con MX.
- Limitación de modismos: nada muy local mexicano, pierde algo de
  "color". Mitigación: tono cercano vía estructura, no vocabulario
  local.
- Si data post-MVP muestra rechazo en alguna región, requiere setup
  de override architecture (pero ya está documentada en §2.4 de
  `i18n.md`).

### Riesgos a monitorear

- **Signup → activation drop en AR/UY > 20% sobre MX:** señal de
  rechazo del tuteo. Mitigación primaria: ajustar tono. Mitigación
  secundaria: agregar override `es-AR` con tuteo + vocabulario local.
- **Modismos mexicanos accidentales en docs/copy:** detección via CI
  con lista de palabras a evitar (decisión cerrada en §11
  i18n.md ejemplo bloqueado).
- **Drift:** después de meses, copy nuevo escrito por contributors
  diferentes puede divergir. Mitigación: linting de strings, code
  review checklist.

## CI/CD validation

Implementar pre-merge check que rechaza commits con strings de
voseo o modismos bloqueados:

```yaml
# .github/workflows/locale-lint.yml
- name: Locale lint
  run: |
    # Flag voseo verbs
    BANNED_PATTERNS=(
      "podés" "querés" "hablás" "tenés" "sentís" "hacés"
      "decís" "preferís" "vos" "registrate" "iniciá"
    )
    for pattern in "${BANNED_PATTERNS[@]}"; do
      if grep -rn "$pattern" apps/ docs/product/ docs/cross-cutting/ \
        --exclude-dir=node_modules \
        --exclude=reglas.md --exclude=i18n.md \
        --exclude=cultural-annotations.md \
        --exclude=student-profile-and-assessment.md; then
        echo "Voseo detected. Use mexicano-tuteo per ADR-008."
        exit 1
      fi
    done
```

Excepciones por archivo (`reglas.md`, `i18n.md`,
`cultural-annotations.md`, `student-profile-and-assessment.md`)
porque son los 4 lugares donde voseo aparece legítimamente.

## Plan de expansión post-MVP

### Fase 2.A: pt-BR (Brasil)

**Trigger:** D30 retention > 30%, signups > 5.000 con interés en
Brasil expansion.

**Tareas:**
1. Contratar traductor profesional pt-BR (no AI translation literal).
2. Crear `pt-BR.json` con 100% de keys traducidas.
3. Adaptar docs core a pt-BR (`pendientes.md`, `reglas.md`,
   `i18n.md`).
4. Roleplays adaptados a contexto brasilero (BPOs, tech, comercio).
5. Pricing en BRL (PIX como método de pago).
6. Logros culturales brasileros.
7. Variante de inglés default americana (mismo que MX).

**Estimación:** 6-8 semanas con traductor + 2 semanas QA.

### Fase 2.B: en (English UI para hispanos en EEUU)

**Trigger:** > 10% de signups de US.

**Tareas:**
1. Toggle UI español/inglés en Settings.
2. Traducir UI strings es → en (relativamente directo, AI puede
   ayudar con review humano).
3. Notification copy en EN para users que eligen.
4. Mantener pedagogical content en español scaffolding aún (porque
   el producto sigue enseñando inglés a hispanohablantes; UI inglés
   es preferencia, no signal de mastery).

**Estimación:** 4-6 semanas.

### Fase 3: variantes españolas

**Trigger:** evidencia de rechazo regional en MVP data.

Cada override es un `<locale>.json` con solo las keys que difieren del
default `es`:

```json
// es-AR.json (ejemplo, sin voseo, solo vocabulario local)
{
  "common": {
    "computer": "computadora",  // (default mexicano: "computadora")
    "car": "auto"               // (default mexicano: "carro")
  }
}
```

**Threshold de activación:** signal claro en métrica antes de
invertir esfuerzo. No hacer overrides "por si acaso".

## Métricas de éxito

| Métrica | Target MVP | Cómo medir |
|---------|-----------|------------|
| Retention D30 MX vs AR | dentro de ±10% | Cohort analysis por country |
| % de tickets de support relacionados a "tono raro" o "no entiendo" | < 1% | Tag manual en customer-support-system |
| % de churn explicado por "no me identifico con la app" en exit survey | < 5% | Exit survey post-cancel |
| CI/CD locale-lint blocks per month | tendiendo a 0 | GitHub Actions |
| Speed of expansion to pt-BR (cuando triggered) | ≤ 8 semanas | Project tracking |

## Referencias

- [`docs/cross-cutting/i18n.md`](../cross-cutting/i18n.md) §2.1, §2.2
  — decisión cerrada con detalle exhaustivo.
- [`docs/reglas.md`](../reglas.md) §4.7 — autoridad sobre el tono.
- [`CLAUDE.md`](../../CLAUDE.md) §10 — locale única para
  comunicación humana.
- [`docs/product/push-notifications-copy-bank.md`](../product/push-notifications-copy-bank.md)
  — banco de copys en mexicano-tuteo.
- [`docs/product/onboarding-flow.md`](../product/onboarding-flow.md)
  — 16 pantallas con copy literal.
- [`docs/product/post-assessment-flow.md`](../product/post-assessment-flow.md)
  — 8 pantallas con copy literal.
- [`docs/architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md)
  — prompts user-facing especifican mexicano-tuteo.
- [`docs/business/customer-support-system.md`](../business/customer-support-system.md)
  — templates y prompts en mexicano-tuteo.
- [`docs/explorations/cultural-annotations.md`](../explorations/cultural-annotations.md)
  — exploración de tooltips con variantes regionales (educational).

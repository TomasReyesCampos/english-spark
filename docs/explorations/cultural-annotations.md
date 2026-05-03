# Exploración: Annotations culturales / coloquiales

> **Status:** Exploración (no es spec final).
> **Owner:** —
> **Target de implementación:** Fase 2 (mes 4–6 post-MVP).
> **Audiencia:** agente AI implementador cuando llegue Fase 2; humano
> ahora para validar diseño.

---

## 1. Origen de la idea

Cuando el usuario ve un texto en inglés con tooltip de traducción, además
de la traducción literal, debería ver una sección con explicación
coloquial/cultural específica al inglés americano (mercado primario:
USA jobs, México user inicial).

Ejemplo de tooltip enriquecido para `"How's it going?"`:

```
┌─────────────────────────────────┐
│  How's it going?     [▶ audio]  │
│                                 │
│  📖 Traducción literal          │
│  "¿Cómo va?"                    │
│                                 │
│  🇲🇽 En tu español              │
│  "¿Qué onda?" / "¿Qué tal?"     │
│                                 │
│  💡 Nota cultural               │
│  Saludo casual americano. No    │
│  esperes respuesta detallada;   │
│  un "good, thanks" alcanza.     │
│                                 │
│  🏷️ Registro: casual            │
│  🌎 Variante: USA               │
│                                 │
│  [Ver más coloquialismos]       │
└─────────────────────────────────┘
```

Es un diferenciador real vs Duolingo / Babbel.

---

## 2. Decisiones de granularidad

### 2.1 Qué SÍ se anota

- **Idioms:** `piece of cake`, `break a leg`, `hit the road`.
- **Phrasal verbs no obvios:** `look up to`, `put up with`, `get along
  with`.
- **Saludos / despedidas con register cultural:** `what's up`,
  `how's it going`, `see ya`, `take care`.
- **Vocabulario business USA específico:** `circle back`, `ping me`,
  `loop me in`, `EOD`, `bandwidth`.
- **Slang común con generación claramente identificable:** `cool`,
  `awesome`, `sweet`, `sick` (positivo).
- **Reactions casuales:** `no worries`, `you bet`, `for sure`.

### 2.2 Qué NO se anota

- Vocabulario neutral / transparente (de diccionario).
- Palabras técnicas que ya están definidas en el bloque.
- Variaciones formales (eso ya está en CEFR).
- Cada palabra de un texto largo (sería ruido).

### 2.3 Regla de pulgar

> Anotar solo lo que un user que aprendió inglés con un libro escolar
> NO sabría (aunque conozca todas las palabras individuales). Es decir:
> uso, contexto, registro, equivalencia funcional.

---

## 3. Decisiones de variantes

### 3.1 Por `target_english_variant` del user

(Existe en `student_profiles`; ver `student-profile-and-assessment.md`
§3.2.)

Cada annotation declara `variant_specificity`:

| Specificity | Comportamiento |
|-------------|----------------|
| `us_only` | Solo se muestra si `target_english_variant = american` o `neutral` |
| `uk_only` | Solo si `target_english_variant = british` o `neutral` |
| `both` | Siempre se muestra |

Si `target = british` y la frase es `us_only`: aún se muestra **con
etiqueta** "principalmente USA, no común en UK".

### 3.2 Por `country` / `spanish_variant` del user

Equivalentes coloquiales en español varían:

| English (US) | es_mx | es_ar | es_co | es_cl | es_pe |
|--------------|-------|-------|-------|-------|-------|
| What's up? | Qué onda | Qué onda / Qué hacés | Qué más | Cómo estay | Qué tal |
| Take it easy | Cálmate | Tranquilo | Suave | Tómalo con calma | Tranqui |
| No worries | No te preocupes | No hay drama | Tranqui | Sin problema | Sin chamba |
| Awesome | Padrísimo | Bárbaro | Bacano | La raja | Bacán |
| Sweet (positivo) | Padre | Genial | Chévere | Filete | Chévere |

### 3.3 Decisión arquitectónica: storage

**Opción 1 (rechazada):** storearlo todo en cada annotation (`es_mx`,
`es_ar`, `es_co`, etc.). Verbose; cuando se agregue un país nuevo, hay
que tocar todos los assets.

**Opción 2 (rechazada):** solo `es_neutral` + override por país.
Compacto pero pierde matiz.

**Opción 3 (elegida):** **tabla de equivalencias separada
(`colloquialism_equivalents`)**. Annotations en assets solo dicen "este
es coloquialismo americano de tipo X". Lookup at runtime contra la tabla
para producir el equivalente del país del user.

**Beneficios:**
- Reuso entre assets (la frase "what's up" solo necesita una entry).
- Expandir a país nuevo = update de tabla, no tocar 700+ assets.
- Catálogo curado de coloquialismos como entidad de primera clase.

---

## 4. Schema propuesto

### 4.1 Tabla `colloquialism_equivalents` (nueva, owned por
`content-creation-system`)

```sql
CREATE TABLE colloquialism_equivalents (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  english_phrase  TEXT NOT NULL UNIQUE,           -- "how's it going"
  variant_origin  TEXT NOT NULL CHECK (variant_origin IN ('us', 'uk', 'both')),
  cefr_level      TEXT NOT NULL,                  -- typically B1-B2
  register        TEXT NOT NULL CHECK (register IN (
                    'casual', 'neutral', 'formal', 'business'
                  )),
  generation      TEXT,                           -- '1990s', '2010s', 'timeless'
  category        TEXT NOT NULL,                  -- 'greeting', 'business', 'reaction', ...

  -- Contenido
  literal_translation TEXT NOT NULL,              -- "¿Cómo va?"
  cultural_note   TEXT NOT NULL,                  -- explicación de uso

  -- Equivalentes por país (JSONB porque shape evoluciona)
  equivalents     JSONB NOT NULL,
  -- Shape:
  -- {
  --   "es_mx": ["qué onda", "qué tal"],
  --   "es_ar": ["qué onda", "qué hacés"],
  --   "es_co": ["qué más"],
  --   "es_neutral": ["qué tal"]
  -- }

  -- Audio
  audio_native_url TEXT,                          -- pronunciación nativa

  -- Metadata
  approved        BOOLEAN NOT NULL DEFAULT false,
  approved_by     TEXT,
  approved_at     TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_colloq_phrase ON colloquialism_equivalents(english_phrase);
CREATE INDEX idx_colloq_category ON colloquialism_equivalents(category);
CREATE INDEX idx_colloq_approved ON colloquialism_equivalents(approved)
  WHERE approved = true;
```

### 4.2 Modificación a `learning_assets.content`

Agregar a tipos de contenido relevantes (roleplay, listening, free
response):

```typescript
interface ContentWithAnnotations {
  // ... campos existentes según type ...

  annotations: Array<{
    phrase: string;                     // matchea row en colloquialism_equivalents
    position: { start: number; end: number };
    in_text: string;                    // texto exacto en este asset (puede tener flexión)
    variant_specificity: 'us_only' | 'uk_only' | 'both';
  }>;
}
```

Nota: `phrase` es la **forma canónica**, `in_text` es la forma como
aparece en el asset (ej: phrase = "how's it going", in_text = "How's it
going?").

### 4.3 API contract para frontend

Cuando cliente carga un asset, retorna annotations resueltas:

```typescript
interface ResolvedAnnotation {
  phrase: string;
  position: { start: number; end: number };
  literal_translation: string;
  cultural_equivalent: string[];        // resuelto al país del user
  cultural_note: string;
  register: string;
  variant_specificity: string;
  audio_native_url?: string;
}
```

El backend hace el lookup `colloquialism_equivalents.equivalents[user.country]`.
Si no hay match para el país: fallback a `es_neutral`.

---

## 5. Pipeline de creación

### 5.1 Etapa nueva post-generación

Después de que el LLM genera el contenido del asset, llamada a AI
Gateway task `extract_cultural_annotations`:

```typescript
{
  task_id: 'extract_cultural_annotations',
  description: 'Identifica frases coloquiales/idiomáticas anotables en un texto',
  category: 'batch',
  input_schema: z.object({
    text: z.string(),
    cefr_level: z.string(),
    target_english_variant: z.enum(['american', 'british', 'neutral']),
  }),
  output_schema: z.object({
    annotations: z.array(z.object({
      phrase: z.string(),                 // forma canónica
      in_text: z.string(),                // como aparece
      position: z.object({ start: z.number(), end: z.number() }),
      reason: z.string(),                  // por qué amerita annotation
      suggested_register: z.string(),
      suggested_category: z.string(),
    })),
  }),
  // ...
}
```

### 5.2 Validación

- Cada `phrase` debe existir en `colloquialism_equivalents`. Si no
  existe: flag para creación manual.
- Posición debe matchear el texto.
- Max 10 annotations por asset (evitar ruido).

### 5.3 Revisión humana

- Sample 50% al inicio (similar a §4.4 de `content-creation-system.md`).
- Foco: annotations relevantes y no redundantes.

### 5.4 Costo estimado

- LLM call: ~$0.005 por asset.
- Tiempo humano review: ~2 min por asset.
- **Marginal por asset: +$0.50** (de $5 → $5.50, +10%).

A 700 assets MVP + Fase 2 expansion (1.500 total): ~$300–$500 USD
adicional one-time.

---

## 6. UI propuesta

### 6.1 Marker inline

En el texto, frases anotadas tienen un underline sutil + icono "i":

```
"Hey, ⓘ how's it going? Want to ⓘ grab a coffee later?"
```

### 6.2 Tap → bottom-sheet

Al tap, sale el modal del §1.

### 6.3 Settings

- Toggle "Mostrar annotations culturales" (default ON).
- Si user tiene `language_anxiety = 5`: default ON con prompt explicativo.
- Si user es `B2+ / C1`: opcional desactivar para no saturar.

### 6.4 Reuse en assessments

En el assessment del Day 7, el reading exercise puede incluir
annotations. El user ve cómo el sistema explica coloquialismos antes
de comprometerse a un plan. **Demo poderoso del valor del producto.**

---

## 7. Lista seed de coloquialismos para MVP+1

Aproximadamente **50–100 entradas iniciales** en
`colloquialism_equivalents`, agrupadas por categoría:

### 7.1 Saludos casuales (8)

- What's up?, How's it going?, How are you doing?, Hey there
- What's new?, How's everything?, How's life?, Long time no see

### 7.2 Despedidas casuales (7)

- See ya, Catch you later, Take care, Have a good one
- I'm out, Gotta run, Talk to you later (TTYL)

### 7.3 Reactions de aprobación (10)

- No worries, No problem, You bet, You got it
- For sure, Definitely, Totally, Absolutely
- Sounds good, Works for me, I'm down, Count me in

### 7.4 Reactions casuales (8)

- Cool, Awesome, Sweet, Sick (positivo)
- Nice, Solid, Right on, Bet

### 7.5 Filler / connectors (8)

- Like, You know, I mean, Kinda
- Sorta, Whatever, Anyways, Basically

### 7.6 Trabajo (USA business) (15)

- Circle back, Ping me, Loop me in, Sync up, Touch base
- Deep dive, Low-hanging fruit, Quick win, On my plate
- Bandwidth, EOD (end of day), ASAP, FYI, OOO, WFH

### 7.7 Idioms comunes (12)

- Piece of cake, Hit the road, Break a leg
- Under the weather, Cost an arm and a leg
- Call it a day, Take a rain check
- Up in the air, Down to earth, On the same page
- Out of the blue, Hit the nail on the head

### 7.8 Phrasal verbs no obvios (10)

- Look up to, Put up with, Get along with, Run into
- Come up with, Bring up, Look forward to
- Catch up, Pick up on, Slack off

### 7.9 Slang positivo cuidando generacional (5)

- Lit (2010s+), Fire (2010s+), Vibe (timeless casual)
- Dope (1990s+, casual), Wholesome (2010s+)

### 7.10 Frases de transición / opinión (8)

- To be honest (TBH), Just saying, No offense
- My bad, Fair enough, Good point
- Long story short, Speaking of which

**Total: ~91 entradas iniciales.**

---

## 8. Decisiones tomadas en esta exploración

| # | Decisión | Razón |
|--:|----------|-------|
| 1 | Granularidad: phrase-level + occasional cultural concept | Word-level es ruidoso; cultural concept solo si aporta |
| 2 | Storage: tabla `colloquialism_equivalents` separada | Reuso, mantenibilidad, expandir a país nuevo sin tocar assets |
| 3 | Variantes: `us_only`/`uk_only`/`both` con etiqueta cuando no coincide con target | Respeta preferencia del user pero no oculta variaciones |
| 4 | Equivalentes por país (no por spanish_variant) | `student_profiles.country` es el campo más confiable |
| 5 | Audio nativo opcional por annotation | Mejor pero costoso; agregar Fase 3 |
| 6 | Pipeline: AI extracción + 50% revisión humana | Mismo modelo que assets actuales |
| 7 | Settings: toggle on/off para B2+/C1 | Evitar saturar a users avanzados |
| 8 | Seed de ~91 coloquialismos para MVP+1 | Cobertura inicial razonable |

---

## 9. Decisiones abiertas (a decidir antes de implementar)

- [ ] **¿Annotations en assessments?** Pros: demo de valor, ayuda al
  user a entender el producto. Contras: el assessment debería medir
  capacidad sin ayuda. Posible compromiso: deshabilitarlas en assessment
  pero mostrarlas en results screen.
- [ ] **¿Annotations en conversation 1:1 con IA?** El AI puede generar
  coloquialismos en sus respuestas. ¿El cliente los detecta y los
  anota en runtime? Costo de detección runtime no trivial.
- [ ] **¿Mostramos todas las equivalencias por país o solo la del user?**
  Probable: solo la del user, con opción "ver más" que muestre el resto
  (educa sobre variación regional).
- [ ] **¿Permitimos al user agregar coloquialismos que no entendió?**
  Mecanismo de feedback que feed el catalog. Útil pero anti-abuse
  necesario.
- [ ] **¿Cómo manejar phrases que aparecen flexionadas?** "How's it
  going" vs "How are things going" — ¿misma row con varias variantes
  flexionadas o rows separadas?
- [ ] **¿Annotations en español rioplatense / argentino, son lo mismo
  para coloquialismos USA?** O sea: la misma annotation con `es_ar`
  diferente sirve, o necesitamos rows separadas por variante de Spanish.

---

## 10. Cambios a docs cuando se promueva a spec

Cuando esta exploración se acepte como spec:

### 10.1 `content-creation-system.md`

- §3.2: agregar `ContentWithAnnotations` mixin a tipos relevantes.
- §4: agregar etapa `extract_cultural_annotations` al pipeline.
- §7: agregar tabla `colloquialism_equivalents`.
- §8: agregar API contract `getResolvedAnnotations(asset_id, user_country)`.
- §13: cerrar decisiones abiertas de §9 de este doc.

### 10.2 `pedagogical-system.md`

- §2.4: posible nueva sub-skill `vocab_us_colloquialisms` (B1) y
  `vocab_us_business_register` (B2). Reemplaza o complementa
  `vocab_idioms_common`.

### 10.3 `student-profile-and-assessment.md`

- §3.1: agregar preferencia `show_cultural_annotations: boolean`
  (default true).

### 10.4 `cross-cutting/i18n.md`

- §5: documentar mapping español-mx ↔ coloquialismos USA con tabla
  expandida.

### 10.5 `ai-gateway-strategy.md`

- §4.2: registrar task `extract_cultural_annotations` con schemas Zod.

### 10.6 Frontend

Componentes nuevos:
- `<AnnotationMarker>`: underline + icono "i" inline.
- `<AnnotationBottomSheet>`: modal con la annotation resuelta.

---

## 11. Plan de implementación (cuando se promueva)

### 11.1 Sprint A (semana 1)

- Crear tabla `colloquialism_equivalents`.
- Seed con ~91 entradas curadas manualmente (no IA al inicio).
- API endpoint `getResolvedAnnotations`.

### 11.2 Sprint B (semana 2)

- Task `extract_cultural_annotations` en AI Gateway.
- Pipeline en admin panel.
- Integration con flow de creación de assets.

### 11.3 Sprint C (semana 3)

- Componentes frontend `<AnnotationMarker>` y `<AnnotationBottomSheet>`.
- Settings toggle.
- A/B test: open rate de annotations.

### 11.4 Sprint D (semana 4)

- Reprocessing de assets existentes para extraer annotations.
- Sample review humano.
- Métricas de uso.

---

## 12. Métricas de éxito

- **Annotation open rate:** % de annotations vistas vs presentadas.
  Target: > 15% de phrases anotadas son tap-tappeadas.
- **Asset rating con annotations vs sin:** A/B test. Target: rating ≥
  0.3 puntos más alto en variante con annotations.
- **CSAT post-asset:** users que vieron annotations vs no. Target:
  +5 puntos NPS.
- **Cobertura de catálogo:** % de assets con ≥ 1 annotation. Target: >
  60% de assets tienen al menos 1.
- **Retention post-feature:** ¿se mantiene D30/D90 mejor con
  annotations? Target: +2-5% en cohort que las usa.

---

## 13. Riesgos

| Riesgo | Mitigación |
|--------|-----------|
| User se acostumbra a tooltip y no aprende | Settings para desactivar; advancement gradual hacia menos annotations |
| Equivalencias en español son inexactas | Review humano, feedback loop |
| Genera dependencia: user no toma riesgo de hablar sin entender | Tono de annotations es "uso natural", no "regla" |
| Catálogo se vuelve gigante e inmanejable | Cap soft a ~500 entradas; priorizar por frecuencia de uso |
| Annotations en británico cuando user eligió americano (o viceversa) | `variant_specificity` flag con UI clara |

---

## 14. Referencias internas

Cuando se promueva a spec, vincular con:

- [`../product/content-creation-system.md`](../product/content-creation-system.md) §3.2, §4.
- [`../product/pedagogical-system.md`](../product/pedagogical-system.md) §2.4.
- [`../product/student-profile-and-assessment.md`](../product/student-profile-and-assessment.md) §3.1.
- [`../cross-cutting/i18n.md`](../cross-cutting/i18n.md) §5.
- [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) §4.2.

---

*Documento de exploración. Promover a specs (modificando los docs
referenciados en §10 y §14) cuando esté validado el approach.*

# AI Roadmap System

> Sistema de roadmap de aprendizaje personalizado y adaptado por IA.
> Genera, mantiene y adapta el camino de aprendizaje del usuario.
> "IA cura, no inventa": el LLM selecciona y secuencia bloques de una
> biblioteca predefinida.

**Estado:** Diseño v1.1 (profundizado para implementación)
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** Sistema completo

---

## 0. Cómo leer este documento

- §1 establece **visión y arquitectura**.
- §2 cubre **boundaries**.
- §3 cubre el **modelo de datos**.
- §4 cubre **flujo de onboarding** (roadmap inicial).
- §5 cubre **generación inicial** del roadmap (Day 0).
- §6 cubre **assessment del Day 7** y roadmap definitivo.
- §7 cubre **adaptación nocturna**.
- §8 cubre **insights humanizados**.
- §9 cubre **API contracts**.
- §10 enumera **edge cases**.
- §11 cubre **eventos emitidos/consumidos**.
- §12 cubre **stack tecnológico**.
- §13 cubre **plan de implementación**.
- §14 cubre **decisiones cerradas**.

---

## 1. Visión

El sistema combina:
1. **Biblioteca curada** de bloques (gestionada por
   `content-creation-system`).
2. **Generación con IA** que selecciona y secuencia bloques
   personalizadamente.
3. **Análisis SQL nocturno** + IA selectiva en batch para adaptación.
4. **Presentación humanizada** con insights y reasoning.

### 1.1 Principio rector

**"IA cura, no inventa."** El LLM selecciona y secuencia bloques
predefinidos en lugar de generar contenido desde cero. Garantiza:

- Coherencia pedagógica.
- Costos controlados.
- Experiencia consistente.

### 1.2 Objetivos del sistema

- Mostrar al usuario un destino claro desde el Día 0.
- Personalizar según objetivo, nivel real y errores detectados.
- Adaptar continuamente sin requerir IA en cada interacción.
- Mantener costos < $0.10 USD primer mes; < $0.02 USD meses subsiguientes.

### 1.3 Lo que NO hace

- Generar contenido educativo nuevo en runtime.
- Usar IA en cada apertura de la app.
- Producir paths únicos imposibles de validar pedagógicamente.

### 1.4 Capas funcionales

```
┌─────────────────────────────────────────────────────────────┐
│                    CAPA 1: BIBLIOTECA                       │
│         (Learning Blocks + metadata pedagógico)             │
│              Generada offline, una sola vez                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  CAPA 2: MOTOR DE GENERACIÓN                │
│         (Onboarding + LLM curator → roadmap inicial)        │
│              Una llamada IA por usuario nuevo               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  CAPA 3: ADAPTACIÓN CONTINUA                │
│      (Job nocturno → análisis SQL + IA selectiva batch)     │
│         Solo el 5-10% de usuarios actualiza por noche       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    CAPA 4: PRESENTACIÓN                     │
│       (UI roadmap + insights humanizados + reasoning)       │
│              Cero costo IA en runtime del cliente           │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Boundaries

### 2.1 Es responsable de

- Generar roadmap inicial (Day 0) y definitivo (Day 7).
- Persistir y servir el roadmap del usuario.
- Decidir el próximo bloque ("getNextBlock") consumiendo `mastery` de
  pedagogical-system.
- Job nocturno que evalúa qué usuarios necesitan re-análisis.
- Adaptar el roadmap (insertar bloques de refuerzo, reorder, skip).
- Generar insights humanizados (mensaje matutino, semanal, post-nivel).
- Emitir eventos `roadmap.*` y `level.*`.

### 2.2 NO es responsable de

- **Crear contenido de bloques** (eso es `content-creation-system`).
- **Calcular mastery** (eso es `pedagogical-system`).
- **Cobrar Sparks** (eso es `sparks-system` invocado vía AI Gateway).
- **Otorgar logros** (eso es `motivation-and-achievements`).
- **Detectar struggling** (eso es `pedagogical-system`).
- **Mostrar UI del roadmap** (eso es el cliente consumiendo
  `getRoadmapForUser`).

### 2.3 Tensiones

| Tensión | Resolución |
|---------|-----------|
| Pedagogical detecta struggling → ¿quién decide qué hacer? | Pedagogical emite evento, AI Roadmap reacciona insertando bloques de refuerzo o sugiriendo prerequisito |
| User cambia objetivo mid-roadmap | Profile emite `profile.updated`; AI Roadmap evalúa si regenerar (significant_change) o adaptar |
| Bloque deprecado en biblioteca | Roadmap mantiene snapshot; nuevos roadmaps usan versión nueva |
| LLM devuelve roadmap inválido | Validación Zod; retry; fallback a template |

---

## 3. Modelo de datos

### 3.1 Entidades principales

#### LearningBlock (biblioteca, compartido)

(Owned por `content-creation-system`. Schema en
`content-creation-system.md` §7.)

```typescript
interface LearningBlock {
  id: string;                      // 'block_job_intro_001'
  title: string;
  description: string;
  cefr_level: 'A2' | 'B1' | 'B1+' | 'B2' | 'B2+' | 'C1';
  context_tags: string[];          // ['work', 'interview']
  required_subskills: string[];    // ['gram_past_simple_continuous']
  prerequisites: string[];          // block IDs
  estimated_minutes: number;
  asset_ids: string[];
  mastery_criteria: BlockMasteryCriteria;
  created_at: string;
  updated_at: string;
}
```

#### UserRoadmap (instancia personalizada)

```typescript
interface UserRoadmap {
  id: string;
  user_id: string;
  type: 'initial' | 'definitive' | 'regenerated';
  generated_at: string;
  last_updated_at: string;
  primary_goal: string;
  deadline?: string;
  starting_cefr: string;
  target_cefr: string;
  estimated_completion_weeks: number;
  current_level_id: string;
  total_blocks: number;
  completed_blocks: number;
  ai_summary: string;
  levels: RoadmapLevel[];
}

interface RoadmapLevel {
  id: string;
  name: string;                    // "Fundamentos de Job Ready"
  order: number;
  status: 'locked' | 'active' | 'completed';
  ai_reasoning: string;
  estimated_weeks: number;
  blocks: RoadmapBlock[];
}

interface RoadmapBlock {
  block_id: string;
  order_within_level: number;
  status: 'locked' | 'active' | 'completed' | 'tested_out';
  personalization_reason: string;  // "Incluido por errores en past tense"
  mastery_score?: number;
  attempts: number;
  completed_at?: string;
}
```

### 3.2 Schema Postgres

```sql
-- Roadmaps personalizados
CREATE TABLE user_roadmaps (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type            TEXT NOT NULL CHECK (type IN ('initial', 'definitive', 'regenerated')),
  is_active       BOOLEAN NOT NULL DEFAULT true,
  generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  primary_goal    TEXT NOT NULL,
  deadline        DATE,
  starting_cefr   TEXT NOT NULL,
  target_cefr     TEXT NOT NULL,
  estimated_completion_weeks INT,
  current_level_id TEXT,
  total_blocks    INT NOT NULL,
  completed_blocks INT NOT NULL DEFAULT 0,
  ai_summary      TEXT,
  structure       JSONB NOT NULL,    -- levels y blocks
  generated_by_model TEXT,            -- ej: 'claude-haiku-4-5:v2'
  generation_cost_usd NUMERIC(10,4)
);

CREATE INDEX idx_roadmaps_user_active
  ON user_roadmaps(user_id) WHERE is_active = true;

-- Solo un roadmap activo por usuario
CREATE UNIQUE INDEX idx_one_active_roadmap_per_user
  ON user_roadmaps(user_id) WHERE is_active = true;
```

### 3.3 Histórico de roadmaps

Cuando se genera un roadmap nuevo (definitive después del initial,
regenerated por cambio de objetivo): el anterior se marca
`is_active = false` pero se mantiene para histórico (audit + visualización
de evolución).

---

## 4. Onboarding inicial (Day 0)

(Coordinado con `student-profile-and-assessment.md` §3.)

### 4.1 Preguntas estructuradas

(Detalle en `student-profile-and-assessment.md` §3.2.)

### 4.2 Mini-test de 5 minutos

Tres ejercicios para captar audio:
1. Lectura y resumen oral (3 min).
2. Pregunta abierta (60s).
3. Frases con sonidos críticos (1 min).

### 4.3 Análisis del test

Pipeline secuencial:
1. STT con AI Gateway task `transcribe_user_audio`.
2. Pronunciation scoring vía task `score_pronunciation`.
3. Análisis de fluidez determinístico (WPM, pausas).
4. Detección de errores con `detect_grammar_errors`.
5. Estimación CEFR.

Output: `initial_test_results` JSON persistido en `student_profiles`.

### 4.4 Generación del roadmap inicial

Llamada a AI Gateway task `generate_initial_roadmap` (ver
`ai-gateway-strategy.md` §4.2.4).

**Costo total:** ~$0.05 USD por usuario nuevo.

---

## 5. Generación del roadmap inicial (detallado)

### 5.1 Prompt template

```
Sos un experto en didáctica del inglés para hispanohablantes
latinoamericanos con nivel intermedio. Tu tarea es seleccionar y
secuenciar bloques de aprendizaje de una biblioteca predefinida
para crear un roadmap personalizado.

PERFIL DEL USUARIO:
- Objetivos: {{primary_goals}}
- Deadline: {{deadline}}
- Contexto profesional: {{professional_context}}
- Nivel autopercibido: {{self_perceived_level}}
- Ansiedad lingüística: {{self_perceived_anxiety}}/5
- Tiempo diario: {{daily_minutes_available}} minutos

RESULTADOS DEL TEST INICIAL:
- CEFR estimado: {{cefr_estimate}}
- Fluidez: {{fluency_score}}/10
- Pronunciación: {{pronunciation_score}}/10
- Errores detectados: {{detected_error_patterns}}

BIBLIOTECA DISPONIBLE (selecciona de aquí, NO inventes bloques):
{{learning_blocks_filtered_by_relevant_cefr}}

INSTRUCCIONES:
1. Seleccioná entre 30 y 50 bloques que cubran los objetivos del
   usuario.
2. Organizalos en 4-6 niveles temáticos con nombres motivadores.
3. Respetá los prerequisites de cada bloque.
4. Priorizá bloques que ataquen los errores detectados.
5. Si hay deadline corto (<1 mes), elegí solo los esenciales.
6. Si hay alta ansiedad, los primeros bloques deben ser de baja
   presión.
7. Para cada bloque seleccionado, escribí UNA frase explicando por qué
   está incluido (personalization_reason).
8. Para cada nivel, escribí UNA frase de motivación (ai_reasoning).
9. Generá un ai_summary de 2-3 frases que el usuario verá al ver su
   roadmap.

DEVOLVÉ JSON exacto siguiendo este schema (estructura aproximada):
{
  "ai_summary": "...",
  "estimated_completion_weeks": N,
  "target_cefr": "...",
  "levels": [
    {
      "name": "...",
      "order": 1,
      "ai_reasoning": "...",
      "estimated_weeks": N,
      "blocks": [
        {
          "block_id": "...",
          "order_within_level": 1,
          "personalization_reason": "..."
        }
      ]
    }
  ]
}

Devolvé SOLO el JSON, sin texto adicional ni markdown.
```

### 5.2 Validación post-generación

Antes de persistir, output del LLM se valida con Zod schema +
checks lógicos:

- Todos los `block_id` referenciados existen en `learning_blocks`.
- Prerequisites se respetan en el orden generado.
- Total de bloques en rango (30-50).
- Duración total coherente con `daily_minutes_available` + deadline.

Si validación falla, retry una vez. Si vuelve a fallar, fallback a
roadmap template del bucket más cercano del usuario (combinaciones
goal/CEFR predefinidas).

### 5.3 Bloques iniciales mostrados

User ve los **2 primeros niveles completos** + preview borrosa del
resto. Preserva intriga y deja espacio para que el assessment del
Day 7 "expanda" el roadmap.

---

## 6. Assessment del Day 7 y roadmap definitivo

### 6.1 Trigger

(Detalle en `student-profile-and-assessment.md` §5.)

Day 7 (o antes si Sparks runout o user solicita): assessment de 20 min.
Después del completado, generate definitive roadmap.

### 6.2 Generación del definitivo

Llamada a AI Gateway task `generate_definitive_roadmap` con:

- Profile completo + observed_behavior de los 7 días.
- Assessment results detallados.
- Performance en bloques iniciales.

**Diferencias con initial:**

| Aspecto | Inicial (Day 0) | Definitivo (Day 7+) |
|---------|----------------|---------------------|
| Basado en | 5 preguntas + mini-test | Profile completo + 7 días observación + assessment 20 min |
| Duración | 4 semanas estimadas | 8–16 semanas precisas |
| Bloques | 30–40 generales | 50–80 específicos al perfil |
| Foco | General por objetivo | Específico a errores detectados |
| Variant inglés | Default del país | Confirmada por preferencia |
| Niveles | 4 estándar | 4–6 personalizados |
| Mensaje | "Plan inicial provisional" | "Tu plan definitivo, hecho para vos" |

### 6.3 Persistencia y archivado

```sql
-- Atomic swap
BEGIN;
UPDATE user_roadmaps SET is_active = false WHERE user_id = $1 AND is_active = true;
INSERT INTO user_roadmaps (user_id, type, is_active, ...) VALUES ($1, 'definitive', true, ...);
COMMIT;
```

Roadmap initial queda como histórico para que el user pueda ver
"cómo evolucionó mi plan".

---

## 7. Adaptación nocturna del roadmap

### 7.1 Reglas de re-análisis (filtrado SQL antes de IA)

Solo se llama a IA si se cumple alguna condición:

```sql
SELECT user_id FROM users
WHERE active = true
  AND (
    -- Completó un nivel completo en últimas 24h
    user_id IN (SELECT user_id FROM level_completions
                WHERE completed_at > now() - interval '24 hours')

    -- Patrón de error fuerte no contemplado
    OR user_id IN (SELECT user_id FROM error_patterns_daily
                   WHERE pattern_intensity > 0.7
                     AND not_in_current_roadmap = true)

    -- Desviación significativa de velocidad esperada
    OR user_id IN (SELECT user_id FROM progress_velocity
                   WHERE actual_pace / expected_pace NOT BETWEEN 0.5 AND 2.0)

    -- Cambio reciente en perfil (deadline, objetivo)
    OR user_id IN (SELECT user_id FROM student_profiles
                   WHERE updated_at > now() - interval '24 hours'
                     AND significant_change = true)

    -- Bloques struggling sin atender
    OR user_id IN (SELECT user_id FROM user_block_status
                   WHERE status = 'struggling'
                     AND struggling_since < now() - interval '24 hours')
  );
```

Esta regla típicamente captura **5-10%** de usuarios activos por
noche.

### 7.2 Prompt de actualización

Llamada a AI Gateway task `nightly_roadmap_update` con Batch API
(50% off):

```
ROADMAP ACTUAL DEL USUARIO:
{{current_roadmap_summary}}

PROGRESO RECIENTE (últimos 7 días):
- Bloques completados: {{completed_blocks}}
- Subskills mejoradas: {{improved_subskills}}
- Errores nuevos detectados: {{new_error_patterns}}
- Velocidad real vs esperada: {{velocity_ratio}}
- Bloques marcados struggling: {{struggling_blocks}}

CAMBIOS EN PERFIL:
{{profile_changes}}

INSTRUCCIONES:
Devolvé un JSON con cambios al roadmap (NO el roadmap completo):
{
  "blocks_to_add": [{block_id, target_level, position, reason}],
  "blocks_to_remove": [{block_id, reason}],
  "blocks_to_reorder": [{block_id, new_position, reason}],
  "blocks_to_test_out": [{block_id, reason}],
  "updated_estimated_weeks": N,
  "user_facing_message": "Mensaje de 1-2 frases"
}
```

El motor aplica diffs al roadmap existente. Preserva historial.

### 7.3 Costos

| Usuarios activos | Re-análisis/noche | Costo IA batch | Total/mes |
|-----------------:|-------------------:|---------------:|----------:|
| 1.000 | 50–100 | $0.50 | $15 |
| 10.000 | 500–1.000 | $5.00 | $150 |
| 100.000 | 5.000–10.000 | $25.00 | $750 |

Por usuario activo: ~$0.01 USD/mes de mantenimiento de roadmap.

### 7.4 Adaptación suave por sporadic responses (v1.2)

> Promovido desde [`docs/explorations/sporadic-questions.md`](../explorations/sporadic-questions.md).

Durante el período pre-assessment (Day 1 hasta `assessment_completed_at`),
las respuestas a sporadic questions (ver
`student-profile-and-assessment.md` §7.1) **NO regeneran el roadmap**
pero ajustan el **orden** de los blocks del roadmap provisional.

Reglas de ajuste:

| Respuesta sporadic | Ajuste al orden del roadmap |
|-------------------|------------------------------|
| Listening self ≤ 2 | Priorizar bloques de listening slow-speed |
| Listening self ≥ 4 | Permitir bloques de listening native-speed |
| Speaking confidence ≤ 2 | Empezar con drills, no roleplay free |
| Speaking confidence ≥ 4 | Permitir roleplay free temprano |
| Vocabulary self ≤ 2 | Más vocab in context exercises |
| Pronunciación self ≤ 2 | Más drill, menos free response |
| Frustración = "no encontrar palabras" | Vocab building blocks priorizados |

**Reglas de anti-noise:**
- Ignorar respuestas con `flagged_likely_fake = true` (decisión §10.6
  de exploración).
- Respuestas tienen peso reducido (0.5x) en la calibración vs scores
  observados de ejercicios reales.
- Cross-validate con `observed_behavior`: si self-perception
  contradice scores fuertemente, mantener priorización por scores
  pero flagear mismatch para tono adaptado en post-assessment results.

**Implementación:** función `adjustRoadmapOrderBySporadic(roadmap_id,
user_id)` corre al recibir cada sporadic response no-fake. Modifica
`structure.levels[].blocks[].order_within_level` sin cambiar set de
blocks ni level structure.

Una vez completado el assessment formal, sporadic adjustments dejan
de aplicarse y `generateDefinitiveRoadmap` toma control total.

---

## 8. Insights humanizados

### 8.1 Tipos

| Insight | Cuándo | Generado por |
|---------|--------|--------------|
| Mensaje matutino | Diario, antes de hora preferida | Batch nocturno (`generate_morning_message`) |
| Razón del bloque actual | UI on-demand | Pre-generado al setear bloque activo |
| Insight semanal | Domingos | Batch (`generate_weekly_summary`) |
| Mensaje al completar nivel | Por evento `level.completed` | Realtime (`generate_level_completion_message`) |

### 8.2 Generación batch nocturna

Prompt único cubre múltiples mensajes para mismo usuario, optimizando
llamadas:

```
Generá los siguientes mensajes para este usuario, en español neutro
latinoamericano, tono cercano pero profesional, 1-2 frases cada uno:

DATOS: {{user_summary}}
PROGRESO RECIENTE: {{weekly_progress}}

MENSAJES A GENERAR:
1. Mensaje matutino para mañana (objetivo del día: {{tomorrow_focus}})
2. Razón del próximo bloque: {{next_block}}
3. Si completó nivel ayer: mensaje de celebración + preview del
   siguiente

Devolvé JSON con keys: morning_message, next_block_reason,
level_celebration.
```

### 8.3 Costo

~$0.001-0.003 USD por usuario por semana. Despreciable.

---

## 9. API contracts

### 9.1 `generateInitialRoadmap`

**Llamado por:** student-profile-and-assessment al completar onboarding
+ mini-test.

**Request:**

```typescript
interface GenerateInitialRoadmapRequest {
  user_id: string;
  profile_snapshot: StudentProfileSnapshot;
  initial_test_results: InitialTestResults;
}
```

**Response:**

```typescript
interface GenerateInitialRoadmapResponse {
  roadmap_id: string;
  ai_summary: string;
  total_blocks: number;
  estimated_completion_weeks: number;
  first_block: {
    block_id: string;
    title: string;
  };
}
```

### 9.2 `generateDefinitiveRoadmap`

**Llamado por:** student-profile-and-assessment al `assessment.completed`.

```typescript
interface GenerateDefinitiveRoadmapRequest {
  user_id: string;
  assessment_results: AssessmentResults;
  observed_behavior: ObservedBehavior;
}
```

**Reglas:**
- Archiva el initial.
- Crea definitive con `is_active = true`.
- Emite `roadmap.generated` con `type: 'definitive'`.

### 9.3 `regenerateRoadmap`

**Llamado por:** internal cuando user cambia objetivo significativamente.

```typescript
interface RegenerateRoadmapRequest {
  user_id: string;
  trigger_reason: string;          // 'goal_changed' | 'deadline_changed' | ...
  preserve_completed_blocks: boolean; // default true
}
```

### 9.4 `getDailyPlan`

**Llamado por:** cliente al abrir la app.

**Request:**

```typescript
interface GetDailyPlanRequest {
  user_id: string;
  date?: string;                   // default today
}
```

**Response:**

```typescript
interface GetDailyPlanResponse {
  blocks: Array<{
    block_id: string;
    title: string;
    estimated_minutes: number;
    reason: 'next_in_roadmap' | 'spaced_review' |
            'reinforcement' | 'struggling_recovery';
    target_subskills: string[];
  }>;
  total_estimated_minutes: number;
  morning_message?: string;
}
```

Implementación: combina `pedagogical-system.getNextBlocksForUser`
+ insights pre-generados.

### 9.5 `getRoadmapForUser`

**Llamado por:** cliente para mostrar pantalla de roadmap.

```typescript
interface GetRoadmapForUserRequest {
  user_id: string;
  include_history?: boolean;       // default false
}

interface GetRoadmapForUserResponse {
  roadmap: UserRoadmap;
  progress_pct: number;
  current_level: RoadmapLevel;
  next_milestone: {
    type: 'level_completion' | 'cefr_jump' | 'track_completion';
    description: string;
    blocks_remaining: number;
  };
  history?: UserRoadmap[];          // si include_history
}
```

### 9.6 Internal: `applyNightlyUpdate`

**Llamado por:** job nocturno.

```typescript
interface ApplyNightlyUpdateRequest {
  user_id: string;
  diffs: RoadmapDiff;
}

interface RoadmapDiff {
  blocks_to_add: Array<{
    block_id: string;
    target_level: string;
    position: number;
    reason: string;
  }>;
  blocks_to_remove: Array<{ block_id: string; reason: string }>;
  blocks_to_reorder: Array<{ block_id: string; new_position: number; reason: string }>;
  blocks_to_test_out: Array<{ block_id: string; reason: string }>;
  updated_estimated_weeks: number;
  user_facing_message: string;
}
```

### 9.7 `requestManualSkip`

**Llamado por:** cliente si user solicita saltar bloque (cuando ya
domina).

```typescript
interface RequestManualSkipRequest {
  user_id: string;
  block_id: string;
}

interface RequestManualSkipResponse {
  allowed: boolean;
  reason?: string;
  test_out_required?: boolean;     // si necesita test-out primero
}
```

**Reglas:**
- Si user ya completó prerequisites: allowed sin test-out.
- Si user no muestra mastery de la sub-skill: requerir test-out.

---

## 10. Edge cases (tests obligatorios)

### 10.1 Generación

1. **LLM devuelve `block_id` que no existe:** validation falla, retry
   con feedback "estos block_ids no existen". Si vuelve a fallar:
   fallback a template.
2. **LLM no respeta prerequisites:** validation falla, retry. Fallback
   a template.
3. **Total de bloques fuera de rango (< 30 o > 50):** validation falla,
   retry.
4. **LLM devuelve más bloques que el deadline permite:** validation
   advierte; aceptar pero marcar como `unrealistic_timing` para review.

### 10.2 Adaptación

5. **Job nocturno corre y un user es el único que matchea filtros pero
   LLM falla:** skip ese user esa noche, retry día siguiente.
6. **Diff intenta remover un bloque que el user está in_progress:** no
   removerlo; ajustar diff antes de aplicar.
7. **Diff intenta reorder cruzando boundaries de niveles:** rechazar;
   reorder solo dentro de mismo nivel.

### 10.3 Cambios de perfil

8. **User cambia goal de "travel" a "job_interview":**
   `student_profiles` emite `profile.updated` con
   `significant_change = true`; AI Roadmap regenera con confirmación
   del usuario.
9. **User completa 50% del roadmap y cambia deadline:** ajusta
   `estimated_completion_weeks` y reorder; no regenera.

### 10.4 Concurrencia

10. **Job nocturno y user submit attempt al mismo tiempo:**
    transacción `FOR UPDATE` en `user_roadmaps.id`. Job espera; ambos
    completan correctamente.
11. **Multiple jobs nocturnos del mismo user (rare, bug):** idempotency
    via `last_updated_at` check. Solo el primero updates.

### 10.5 Bloques deprecados

12. **Bloque marcado `archived = true` mientras user lo está
    haciendo:** user completa la versión que ya tenía (snapshot).
    Próximos roadmaps no lo incluyen.
13. **Bloque deprecado y user nunca lo inició:** próxima generación lo
    omite. Si era prerequisite de otros, sustituir o adjust.

### 10.6 Errores extremos

14. **User completa todo el roadmap:** generar mensaje "completaste tu
    plan". Sugerir nuevo track. Emitir `roadmap.completed`.
15. **User sin actividad durante 90 días vuelve:** roadmap se mantiene
    pero al primer attempt aplica decay (pedagogical) y posible
    regeneración.

---

## 11. Eventos

### 11.1 Emitidos

(Detalle en `cross-cutting/data-and-events.md` §5.8.)

| Evento | Cuándo |
|--------|--------|
| `roadmap.generated` | Initial o definitive creado |
| `roadmap.regenerated` | User cambió objetivo, regeneración completa |
| `roadmap.updated` | Job nocturno aplicó diff |
| `roadmap.completed` | User terminó todos los bloques del track |
| `level.unlocked` | Nuevo nivel desbloqueado |
| `level.completed` | Nivel completo |

### 11.2 Consumidos

| Evento | Acción |
|--------|--------|
| `block.completed` (de pedagogical) | Update `user_roadmaps.completed_blocks`, evaluar level.completed |
| `block.mastered` (de pedagogical) | Misma acción + posible skip de similar |
| `block.struggling_detected` (de pedagogical) | Insertar bloque de prerequisito o simplificación |
| `subskill.regressed` (de pedagogical) | Considerar agregar refuerzo en próximo nightly |
| `assessment.completed` (de student-profile) | Trigger `generateDefinitiveRoadmap` |
| `profile.updated` (de student-profile) | Si `significant_change`: trigger `regenerateRoadmap` |
| `user.cefr_changed` (de pedagogical) | Update `current_cefr` en roadmap; posible level skip |

---

## 12. Stack tecnológico

(Resumen; detalle en `architecture/01-overview.md` y otros.)

| Componente | Tecnología |
|-----------|-----------|
| Persistencia | Postgres (Supabase) |
| Cache de bloques | Redis (Upstash), hit rate >95% |
| Storage assets | Cloudflare R2 |
| LLM generación | AI Gateway tasks `generate_*_roadmap`, `nightly_roadmap_update`, `generate_morning_message` |
| Orquestación | Inngest cron + workflows |
| Validación | Zod |
| Observability | Sentry + PostHog |

### 12.1 Costos del stack en escala

| Componente | 1.000 users | 10.000 users | 100.000 users |
|-----------|------------:|-------------:|--------------:|
| Supabase Postgres | $25 | $25 | $400 |
| Upstash Redis | $0 | $10 | $80 |
| Cloudflare Workers | $5 | $5 | $30 |
| Cloudflare R2 storage | $5 | $30 | $200 |
| Inngest | $0 | $20 | $50 |
| AWS Athena | $5 | $20 | $80 |
| Deepgram STT | $20 | $200 | $1.500 |
| Azure Pronunciation | $30 | $300 | $0 (modelo propio) |
| LLM APIs | $30 | $200 | $1.000 |
| Sentry + PostHog + BetterStack | $0 | $50 | $200 |
| **Total mensual aprox.** | **$120** | **$860** | **$3.540** |

### 12.2 Lo que NO se debe agregar al stack

- Kubernetes (overkill).
- Pinecone u otra vector DB managed (pgvector cubre).
- Kafka (Inngest events alcanza).
- GraphQL (REST simple suficiente).
- Microservicios (monolito modular).

---

## 13. Plan de implementación

### 13.1 Sprint 1 (semanas 1-2): Biblioteca

- Schema `learning_blocks` (en `content-creation-system`).
- Crear primeros 50 bloques manualmente (track Job Ready).
- Loaders para poblar.

### 13.2 Sprint 2 (semanas 3-4): Onboarding + test

- UI onboarding (5 preguntas).
- Mini-test 5 min.
- Integration con AI Gateway tasks.
- Storage de `student_profiles` con resultados.

### 13.3 Sprint 3 (semanas 5-6): Generación inicial

- Implementación de prompt y task `generate_initial_roadmap`.
- Validación con Zod.
- Persistencia en `user_roadmaps`.
- UI básica del roadmap.

### 13.4 Sprint 4 (semanas 7-8): Consumo y progreso

- `getDailyPlan` y `getNextBlocksForUser` (vía pedagogical).
- Tracking de progreso.
- Lógica de mastery.
- Desbloqueo de niveles.

### 13.5 Sprint 5 (semanas 9-10): Job nocturno

- Setup Inngest + cron diario.
- Pipeline de agregación SQL.
- Reglas de filtrado de usuarios.
- Loop de update con LLM batch.

### 13.6 Sprint 6 (semanas 11-12): Insights humanizados

- Generación de mensajes matutinos batch.
- Razones por bloque.
- Insight semanal.
- UI para mostrar todos.

---

## 14. Decisiones cerradas

### 14.1 User edita roadmap manualmente: **limitado** ✓

**Permitido:**
- Skip de bloque que ya domina (vía test-out).
- Marcar bloque como "no me interesa" (skip soft).

**No permitido:**
- Reorder global del roadmap (rompe coherencia pedagógica).
- Agregar bloques arbitrarios (no es biblioteca personalizable).

**Razón:** balance entre autonomía y coherencia. Skip via test-out
respeta el modelo de mastery.

### 14.2 Tracks múltiples activos simultáneamente: **NO en MVP** ✓

**Razón:** complejidad cognitiva para el usuario. UX más clara con
un solo track activo. User puede cambiar de track cuando complete o
abandone el actual. Reconsiderar año 2 con datos.

### 14.3 Cambio de objetivo mid-roadmap: **regenerar con confirmación** ✓

**Razón:**
- Mergear sería confuso (un track Job Ready a medio + un track Travel
  no tiene sentido).
- Regenerar entero es más limpio.
- Pedir confirmación del user respeta su agency.

**Implementación:** detectar `significant_change` (objetivo distinto, o
deadline cambio > 50%); UI prompt "¿Querés regenerar tu plan?". Si
confirma: archive actual, generate nuevo.

### 14.4 Certificados: **al completar 100% Y mastery promedio > 70%** ✓

**Razón:** completar sin dominar no merece certificado. Threshold
mantiene valor del certificado.

**Implementación:** al `roadmap.completed` event, calcular mastery
promedio del track. Si > 70%: emit `track_certified` event,
`motivation-and-achievements` otorga el logro `track_complete` con
flag `certified = true`.

### 14.5 Re-evaluación periódica: **mini-test cada 4 semanas (Pro/Premium)** ✓

(Coordinado con `student-profile-and-assessment.md` §7.2.)

| Plan | Frecuencia | Profundidad |
|------|-----------|-------------|
| Básico | 8 semanas | mini-test 5 min |
| Pro | 4 semanas | assessment intermedio 10 min |
| Premium | 4 semanas | assessment completo 20 min |

**Razón:** balance entre data fresca y fricción. Premium recibe
assessment completo regularmente como diferenciador del plan.

---

## 15. Métricas

### 15.1 Métricas técnicas

| Métrica | Target |
|---------|--------|
| Latencia p95 generación inicial | < 10s |
| Tasa de validación exitosa | > 98% |
| Costo IA por usuario primer mes | < $0.10 |
| % usuarios re-analizados por noche | 5–15% |
| Hit rate cache de biblioteca | > 95% |

### 15.2 Métricas de producto

| Métrica | Target |
|---------|--------|
| Onboarding completion rate | > 85% |
| Drop-off por pregunta del onboarding | < 5% por pregunta |
| Usuarios que completan primer bloque | > 70% |
| Usuarios que completan primer nivel | > 40% (D7) |
| NPS post primer nivel | > 30 |
| % usuarios al nivel 3 | > 25% (D30) |

### 15.3 Alertas

- Costo IA del día > 2x promedio: SEV-2.
- Validation rate < 95%: SEV-2.
- Latencia p95 > 15s: SEV-2.
- User individual con costo > $1/día: review.

---

## 16. Referencias internas

| Documento | Relación |
|-----------|----------|
| [`pedagogical-system.md`](pedagogical-system.md) | Mastery scoring; `getMasteryProfile`, `getNextBlocksForUser`, eventos consumidos. |
| [`student-profile-and-assessment.md`](student-profile-and-assessment.md) | Profile y assessment results. |
| [`content-creation-system.md`](content-creation-system.md) | Owner de `learning_blocks`. |
| [`motivation-and-achievements.md`](motivation-and-achievements.md) | Consume eventos `level.*`, `roadmap.*`. |
| [`../architecture/ai-gateway-strategy.md`](../architecture/ai-gateway-strategy.md) §4.2.4 | Tasks `generate_initial_roadmap`, `nightly_roadmap_update`. |
| [`../architecture/notifications-system.md`](../architecture/notifications-system.md) | Recibe `morning_message` para push. |
| [`../cross-cutting/data-and-events.md`](../cross-cutting/data-and-events.md) §5.8 | Eventos `roadmap.*`, `level.*`. |

---

*Documento vivo. Actualizar cuando se tomen decisiones nuevas o se
aprenda de la implementación.*

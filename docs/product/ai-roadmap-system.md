# AI Roadmap System

> Sistema de roadmap de aprendizaje personalizado y adaptado por IA.
> Documento de referencia técnica para implementación.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —

---

## 1. Visión general

El AI Roadmap System genera, mantiene y adapta un camino de aprendizaje personalizado para cada usuario, combinando una biblioteca curada de bloques de aprendizaje con análisis automatizado del progreso.

El sistema se diseña bajo el principio de **"IA cura, no inventa"**: el LLM selecciona y secuencia bloques predefinidos en lugar de generar contenido desde cero. Esto garantiza coherencia pedagógica, costos controlados y experiencia consistente para el usuario.

### 1.1 Objetivos del sistema

- Mostrar al usuario un destino claro desde el día uno (mejora retención).
- Personalizar el camino según objetivo, nivel real y errores detectados.
- Adaptar el roadmap continuamente sin requerir IA en cada interacción.
- Mantener costos por usuario por debajo de 0.10 USD el primer mes y 0.02 USD en meses subsiguientes.

### 1.2 Lo que NO hace el sistema

- No genera contenido educativo nuevo en tiempo real.
- No usa IA en cada apertura de la app.
- No produce paths únicos imposibles de validar pedagógicamente.

---

## 2. Arquitectura del sistema

### 2.1 Capas funcionales

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

### 2.2 Flujo de datos end-to-end

```
[Usuario nuevo] 
    → Onboarding (5 preguntas + mini-test 5min)
    → Análisis del audio del test (Whisper + scoring)
    → Generación de roadmap (LLM single call)
    → Persistencia en Postgres
    → UI muestra roadmap personalizado

[Usuario activo - sesión diaria]
    → Cliente solicita "next block"
    → API consulta UserRoadmap.active_block
    → Devuelve bloque + assets asociados
    → Usuario practica
    → Eventos van a cola → data lake

[Job nocturno]
    → SQL agrega métricas del día
    → Reglas determinan si usuario necesita re-análisis
    → Si sí: batch IA actualiza roadmap
    → Si no: sigue con roadmap actual
    → Se generan insights humanizados (batch)
```

---

## 3. Modelo de datos

### 3.1 Entidades principales

#### LearningBlock (biblioteca, compartido entre usuarios)

```typescript
interface LearningBlock {
  id: string;                    // "block_job_intro_001"
  title: string;                 // "Presentar tu experiencia laboral"
  description: string;
  cefr_level: 'B1' | 'B2' | 'C1';
  context_tags: string[];        // ["work", "interview", "self-introduction"]
  required_subskills: string[];  // ["past_simple", "vocabulary_work"]
  prerequisites: string[];       // ids de otros blocks
  estimated_minutes: number;
  asset_ids: string[];           // referencias a assets de la biblioteca
  mastery_criteria: {
    min_accuracy: number;        // ej: 0.8
    min_fluency_score: number;   // ej: 6.0
    required_assets_completed: number;
  };
  created_at: timestamp;
  updated_at: timestamp;
}
```

#### UserRoadmap (instancia personalizada por usuario)

```typescript
interface UserRoadmap {
  id: string;
  user_id: string;
  generated_at: timestamp;
  last_updated_at: timestamp;
  primary_goal: string;          // "job_interview" | "travel" | etc.
  deadline?: Date;
  starting_cefr: string;
  target_cefr: string;
  estimated_completion_weeks: number;
  current_level_id: string;
  total_blocks: number;
  completed_blocks: number;
  ai_summary: string;            // resumen humanizado del path
  levels: RoadmapLevel[];
}

interface RoadmapLevel {
  id: string;
  name: string;                  // "Fundamentos de Job Ready"
  order: number;
  status: 'locked' | 'active' | 'completed';
  ai_reasoning: string;          // por qué este nivel
  estimated_weeks: number;
  blocks: RoadmapBlock[];
}

interface RoadmapBlock {
  block_id: string;              // referencia a LearningBlock
  order_within_level: number;
  status: 'locked' | 'active' | 'completed' | 'tested_out';
  personalization_reason: string; // "Incluido por errores en past tense"
  mastery_score?: number;
  attempts: number;
  completed_at?: timestamp;
}
```

#### UserProfile (datos del onboarding)

```typescript
interface UserProfile {
  user_id: string;
  primary_goals: string[];
  deadline?: Date;
  professional_context: string;  // "tech" | "health" | "sales" | etc.
  self_perceived_level: string;
  self_perceived_anxiety: number; // 1-5
  daily_minutes_available: number;
  initial_test_results: {
    cefr_estimate: string;
    fluency_score: number;
    pronunciation_score: number;
    detected_error_patterns: string[];  // ["th_pronunciation", "past_perfect_misuse"]
    recorded_at: timestamp;
  };
  updated_at: timestamp;
}
```

### 3.2 Schema SQL (Postgres / Supabase)

```sql
-- Biblioteca de bloques (shared, read-only desde la app)
CREATE TABLE learning_blocks (
  id              TEXT PRIMARY KEY,
  title           TEXT NOT NULL,
  description     TEXT,
  cefr_level      TEXT NOT NULL CHECK (cefr_level IN ('B1','B2','C1')),
  context_tags    TEXT[] NOT NULL DEFAULT '{}',
  required_subskills TEXT[] NOT NULL DEFAULT '{}',
  prerequisites   TEXT[] NOT NULL DEFAULT '{}',
  estimated_minutes INT NOT NULL,
  asset_ids       TEXT[] NOT NULL DEFAULT '{}',
  mastery_criteria JSONB NOT NULL,
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_blocks_cefr ON learning_blocks(cefr_level);
CREATE INDEX idx_blocks_context ON learning_blocks USING gin(context_tags);

-- Roadmaps personalizados
CREATE TABLE user_roadmaps (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  generated_at    TIMESTAMPTZ DEFAULT now(),
  last_updated_at TIMESTAMPTZ DEFAULT now(),
  primary_goal    TEXT NOT NULL,
  deadline        DATE,
  starting_cefr   TEXT NOT NULL,
  target_cefr     TEXT NOT NULL,
  estimated_completion_weeks INT,
  current_level_id TEXT,
  total_blocks    INT NOT NULL,
  completed_blocks INT DEFAULT 0,
  ai_summary      TEXT,
  structure       JSONB NOT NULL  -- levels y blocks aquí
);

CREATE INDEX idx_roadmaps_user ON user_roadmaps(user_id);

-- Perfil del usuario
CREATE TABLE user_profiles (
  user_id         UUID PRIMARY KEY REFERENCES users(id),
  primary_goals   TEXT[] NOT NULL,
  deadline        DATE,
  professional_context TEXT,
  self_perceived_level TEXT,
  self_perceived_anxiety INT,
  daily_minutes_available INT,
  initial_test_results JSONB,
  updated_at      TIMESTAMPTZ DEFAULT now()
);

-- Tracking de progreso por bloque
CREATE TABLE user_block_progress (
  user_id         UUID NOT NULL REFERENCES users(id),
  block_id        TEXT NOT NULL REFERENCES learning_blocks(id),
  status          TEXT NOT NULL CHECK (status IN ('locked','active','completed','tested_out')),
  mastery_score   FLOAT,
  attempts        INT DEFAULT 0,
  first_attempted_at TIMESTAMPTZ,
  completed_at    TIMESTAMPTZ,
  PRIMARY KEY (user_id, block_id)
);

CREATE INDEX idx_progress_user_status ON user_block_progress(user_id, status);
```

---

## 4. Flujo de onboarding (generación inicial)

### 4.1 Preguntas estructuradas

| # | Pregunta | Tipo | Opciones |
|---|----------|------|----------|
| 1 | ¿Por qué querés mejorar tu inglés? | Multi-select | Trabajo internacional, entrevista próxima, viajes, estudios, comunicación laboral actual, mejora personal |
| 2 | ¿Tenés deadline? | Single | <1 mes, 1-3 meses, 3-6 meses, sin apuro |
| 3 | ¿Área profesional/estudio? | Single | Tech, salud, finanzas, ventas, marketing, educación, otro |
| 4 | ¿Cómo te sentís hablando inglés? | Single (1-5) | Me bloqueo / muy nervioso / nervioso pero hablo / fluyo con errores / fluyo bien |
| 5 | ¿Cuánto tiempo al día? | Single | 5 min, 10-15 min, 20-30 min, +30 min |

### 4.2 Mini-test de 5 minutos

Tres ejercicios diseñados para capturar audio analizable:

1. **Lectura y resumen**: leer un párrafo de 80 palabras, luego resumirlo en voz alta sin verlo. Captura comprensión + producción.
2. **Pregunta abierta**: responder "¿Cómo fue tu día ayer?" en 60 segundos. Captura fluidez espontánea + uso de tiempos verbales.
3. **Frases con sonidos críticos**: 5 frases con /θ/, /ð/, vocales tensas, finales consonánticos. Captura errores fonéticos típicos del hispanohablante.

### 4.3 Análisis del test

Pipeline secuencial:

1. STT con Whisper para transcribir los 3 audios.
2. Scoring de pronunciación con modelo propio (o Azure Pronunciation Assessment como fallback inicial).
3. Análisis de fluidez determinístico: WPM, pausas, palabras de relleno.
4. Detección de errores gramaticales con LLM ligero sobre la transcripción.
5. Estimación de CEFR basada en complejidad sintáctica + vocabulario + scores.

Output: `initial_test_results` JSON con todos los scores y patrones detectados.

---

## 5. Generación del roadmap inicial

### 5.1 Prompt template

```
Sos un experto en didáctica del inglés para hispanohablantes 
latinoamericanos con nivel intermedio. Tu tarea es seleccionar y 
secuenciar bloques de aprendizaje de una biblioteca predefinida 
para crear un roadmap personalizado.

PERFIL DEL USUARIO:
- Objetivos: {primary_goals}
- Deadline: {deadline}
- Contexto profesional: {professional_context}
- Nivel autopercibido: {self_perceived_level}
- Ansiedad lingüística: {self_perceived_anxiety}/5
- Tiempo diario: {daily_minutes_available} minutos

RESULTADOS DEL TEST INICIAL:
- CEFR estimado: {cefr_estimate}
- Fluidez: {fluency_score}/10
- Pronunciación: {pronunciation_score}/10
- Errores detectados: {detected_error_patterns}

BIBLIOTECA DISPONIBLE (selecciona de aquí, NO inventes bloques):
{learning_blocks_filtered_by_relevant_cefr}

INSTRUCCIONES:
1. Seleccioná entre 30 y 50 bloques que cubran los objetivos del usuario.
2. Organizalos en 4-6 niveles temáticos con nombres motivadores.
3. Respetá los prerequisites de cada bloque.
4. Priorizá bloques que ataquen los errores detectados.
5. Si hay deadline corto (<1 mes), elegí solo los esenciales.
6. Si hay alta ansiedad, los primeros bloques deben ser de baja presión.
7. Para cada bloque seleccionado, escribí UNA frase explicando por qué 
   está incluido (personalization_reason).
8. Para cada nivel, escribí UNA frase de motivación (ai_reasoning).
9. Generá un ai_summary de 2-3 frases que el usuario verá al ver su roadmap.

FORMATO DE SALIDA: JSON exacto con esta estructura:
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

Antes de persistir, el output del LLM se valida:

- Todos los `block_id` referenciados existen en la biblioteca.
- Los prerequisites se respetan en el orden generado.
- El total de bloques está en rango (30-50).
- El JSON es parseable y respeta el schema esperado.
- La duración total estimada es coherente con el `daily_minutes_available` y el deadline.

Si la validación falla, se reintenta una vez. Si vuelve a fallar, se cae a un roadmap template del bucket más cercano del usuario.

### 5.3 Costo estimado

- LLM call (Claude Haiku o Gemini Flash): ~0.02 USD
- Análisis del audio del test: ~0.03 USD
- **Total onboarding: ~0.05 USD por usuario nuevo**

---

## 6. Adaptación nocturna del roadmap

### 6.1 Reglas de re-análisis (filtrado SQL antes de IA)

Solo se llama a la IA si se cumple alguna de estas condiciones:

```sql
-- Pseudocódigo de la regla
SELECT user_id FROM users
WHERE active = true
  AND (
    -- Completó un nivel completo en las últimas 24h
    user_id IN (SELECT user_id FROM level_completions 
                WHERE completed_at > now() - interval '24 hours')
    
    -- Patrón de error fuerte no contemplado en el roadmap actual
    OR user_id IN (SELECT user_id FROM error_patterns_daily
                   WHERE pattern_intensity > 0.7
                     AND not_in_current_roadmap = true)
    
    -- Desviación significativa de velocidad esperada
    OR user_id IN (SELECT user_id FROM progress_velocity
                   WHERE actual_pace / expected_pace NOT BETWEEN 0.5 AND 2.0)
    
    -- Cambio reciente en perfil (deadline, objetivo)
    OR user_id IN (SELECT user_id FROM user_profiles
                   WHERE updated_at > now() - interval '24 hours')
  );
```

Esta regla típicamente captura entre el 5% y el 10% de los usuarios activos cada noche.

### 6.2 Prompt de actualización

```
ROADMAP ACTUAL DEL USUARIO:
{current_roadmap_summary}

PROGRESO RECIENTE (últimos 7 días):
- Bloques completados: {completed_blocks}
- Subskills mejoradas: {improved_subskills}
- Errores nuevos detectados: {new_error_patterns}
- Velocidad real vs esperada: {velocity_ratio}

CAMBIOS EN PERFIL:
{profile_changes}

INSTRUCCIONES:
Devolvé un JSON con los cambios al roadmap (no el roadmap completo):
{
  "blocks_to_add": [{block_id, target_level, position, reason}],
  "blocks_to_remove": [{block_id, reason}],
  "blocks_to_reorder": [{block_id, new_position, reason}],
  "blocks_to_test_out": [{block_id, reason}],
  "updated_estimated_weeks": N,
  "user_facing_message": "Mensaje de 1-2 frases explicando los cambios"
}
```

El motor aplica esos diffs al roadmap existente, en lugar de reemplazarlo entero. Eso preserva el historial y permite que el usuario vea evolución.

### 6.3 Costo estimado en escala

| Usuarios activos | Re-análisis/noche | Costo IA batch | Total/mes |
|-----------------:|-------------------:|---------------:|----------:|
| 1.000            | 50-100            | $0.50          | $15       |
| 10.000           | 500-1.000         | $5.00          | $150      |
| 100.000          | 5.000-10.000      | $25.00         | $750      |

Por usuario activo, el costo de mantenimiento del roadmap es ~0.01 USD/mes.

---

## 7. Insights humanizados

### 7.1 Tipos de insights generados

Cada noche, en batch, el sistema genera mensajes humanizados que el usuario ve durante el día:

- **Mensaje matutino**: "Hoy enfocaremos en X porque Y."
- **Razón de cada bloque**: "¿Por qué este ejercicio?" disponible al expandir cada bloque.
- **Insight semanal (domingos)**: "Esta semana mejoraste 18% en fluidez..."
- **Mensaje al completar nivel**: "Completaste Foundations. Tu próximo desafío..."

### 7.2 Estrategia de generación

Los insights se generan en batch nocturno con un prompt único que cubre múltiples mensajes para el mismo usuario, optimizando llamadas a la API.

```
Generá los siguientes mensajes para este usuario, en español neutro 
latinoamericano, tono cercano pero profesional, 1-2 frases cada uno:

DATOS DEL USUARIO: {user_summary}
PROGRESO RECIENTE: {weekly_progress}

MENSAJES A GENERAR:
1. Mensaje matutino para mañana (objetivo del día: {tomorrow_focus})
2. Razón del próximo bloque: {next_block}
3. Si completó nivel ayer: mensaje de celebración + preview del siguiente

Devolvé JSON con keys: morning_message, next_block_reason, level_celebration.
```

### 7.3 Costo

Aproximadamente 0.001-0.003 USD por usuario por semana. Despreciable.

---

## 8. Stack tecnológico

### 8.1 Stack para esta feature específicamente

| Componente | Tecnología | Por qué |
|-----------|-----------|---------|
| **Base de datos roadmaps** | Postgres (Supabase) | Relaciones complejas, JSONB para estructura flexible, queries SQL potentes para job nocturno |
| **Cache de bloques** | Redis (Upstash) | La biblioteca cambia poco, se cachea en memoria. Hit rate >95% |
| **Storage de assets** | Cloudflare R2 | Sin egress fees, CDN global, audio servido rápido en Latam |
| **Análisis de audio del test** | Whisper (Deepgram inicialmente, self-hosted después) | STT preciso, fallback gradual a propio |
| **Pronunciation scoring** | Azure Pronunciation Assessment (MVP) → modelo propio (mes 6+) | Empezar con API, reemplazar cuando haya datos |
| **LLM generación roadmap** | Claude Haiku 4.5 o Gemini 2.0 Flash | Modelos baratos, JSON output confiable, suficiente para curaduría |
| **LLM batch nocturno** | Anthropic Batch API (50% off) | Batch async sin urgencia, máximo descuento |
| **Orquestación job nocturno** | Inngest (MVP) → Dagster (escala) | Inngest es serverless, simple, ideal para empezar solo |
| **Cola de eventos** | Inngest events (MVP) → SQS o Pub/Sub | Misma plataforma, simplifica |
| **Data lake** | S3 + Parquet, queries con AWS Athena | Pago por TB escaneado, óptimo para volúmenes bajos-medios |
| **Análisis SQL agregado** | Athena inicialmente, BigQuery a partir de 50k usuarios | Athena no requiere infra, BigQuery escala mejor |
| **API Layer** | Cloudflare Workers (TypeScript) o Vercel Edge | Latencia baja en Latam, serverless, fácil de operar solo |
| **Validación de schemas** | Zod (TypeScript) | Type-safe end-to-end, validación de outputs de LLM |
| **Observability** | Sentry + PostHog + Better Stack | Errores, analytics, uptime — los 3 tienen tier gratuito generoso |

### 8.2 Por qué este stack (decisiones clave)

#### Postgres + Supabase para todo el estado operacional
La estructura del roadmap es relacional pero con flexibilidad necesaria en `levels[].blocks[]`. JSONB de Postgres da lo mejor de ambos mundos: queries SQL potentes + estructura flexible. Supabase elimina la necesidad de operar una base de datos.

#### Cloudflare Workers como API layer
Latencia desde México/Argentina/Colombia a un Worker es de 20-50ms, vs 200-400ms a un servidor en us-east. Para una app que usa audio en tiempo real, esto importa. Además, Workers escalan a cero (gratis cuando no hay tráfico) y no requieren manejar servidores.

#### Inngest para orquestación
Es la pieza menos obvia pero más importante para vos como dev solo. Inngest te da:
- Cron jobs serverless (el nocturno)
- Workflows con steps (cada etapa del pipeline)
- Reintentos automáticos
- Observabilidad incorporada
- Tier gratuito generoso (50k ejecuciones/mes)

Sin esto, terminás operando tu propio Airflow o cron en VPS, y eso suma complejidad innecesaria al inicio.

#### Anthropic Batch API para el job nocturno
50% de descuento sobre las llamadas normales. Como el análisis nocturno no es urgente (puede tomar horas), Batch API es perfecto. La diferencia entre $50/mes y $25/mes a escala importa.

#### Claude Haiku o Gemini Flash para la curaduría
Para esta tarea (selección y ordenamiento de bloques predefinidos con razonamiento sobre datos del usuario), modelos como Haiku 4.5 o Gemini 2.0 Flash son más que suficientes. Cuestan 10x menos que los modelos premium y la calidad es indistinguible para este caso de uso.

#### Zod para validación
Crítico cuando trabajás con outputs de LLM. Le pasás el JSON al validador Zod, y si falla, reintentás con feedback al modelo. Sin esto, vas a tener bugs raros en producción cuando el LLM devuelva algo ligeramente fuera de schema.

### 8.3 Costos del stack en escala

| Componente | 1.000 users | 10.000 users | 100.000 users |
|-----------|------------:|-------------:|--------------:|
| Supabase Postgres | $25 | $25 | $400 |
| Upstash Redis | $0 (free) | $10 | $80 |
| Cloudflare Workers | $5 | $5 | $30 |
| Cloudflare R2 storage | $5 | $30 | $200 |
| Inngest | $0 (free) | $20 | $50 |
| AWS Athena | $5 | $20 | $80 |
| Deepgram STT | $20 | $200 | $1.500 |
| Azure Pronunciation | $30 | $300 | $0 (modelo propio) |
| LLM APIs (todos los usos) | $30 | $200 | $1.000 |
| Sentry + PostHog + BetterStack | $0 (free) | $50 | $200 |
| **Total mensual aprox.** | **$120** | **$860** | **$3.540** |

A 100.000 usuarios pagos con ARPU mezclado de ~$3 USD/mes (aprox 78 pesos), revenue mensual es ~$300.000 USD. Costo de infra: $3.540 USD. Margen bruto: ~98%.

Estos números son optimistas pero ilustrativos: con buen control de costos, este stack escala saludablemente.

### 8.4 Lo que NO se debe agregar al stack todavía

- **Kubernetes**: overkill hasta que haya múltiples servicios y equipo de ops.
- **Pinecone u otra vector DB managed**: pgvector cubre los casos iniciales gratis.
- **Kafka**: Inngest events alcanza hasta volúmenes medios.
- **Datadog**: Sentry + PostHog + Better Stack cubren todo lo necesario.
- **GraphQL**: agregar complejidad sin beneficio claro a este tamaño.
- **Microservicios**: monolito modular (el "monorepo" con servicios lógicos) es la respuesta correcta.

---

## 9. Plan de implementación incremental

### Sprint 1 (semana 1-2): biblioteca de bloques
- Definir schema de `learning_blocks`.
- Crear los primeros 50 bloques manualmente (con ayuda de Claude generando contenido) cubriendo el track Job Ready.
- Loaders para poblar la base.

### Sprint 2 (semana 3-4): onboarding + test inicial
- UI del onboarding (5 preguntas).
- Implementación del mini-test de 5 minutos.
- Integración con Whisper para transcripción.
- Storage de `user_profiles` con resultados del test.

### Sprint 3 (semana 5-6): generación inicial del roadmap
- Implementación del prompt y la llamada a Claude Haiku.
- Validación con Zod.
- Persistencia en `user_roadmaps`.
- UI básica de visualización del roadmap.

### Sprint 4 (semana 7-8): consumo y progreso
- Endpoint "next block" que devuelve el siguiente bloque activo.
- Tracking de progreso por bloque en `user_block_progress`.
- Lógica de mastery: cuándo se considera completado.
- Desbloqueo automático del siguiente nivel.

### Sprint 5 (semana 9-10): job nocturno básico
- Setup de Inngest + cron diario.
- Pipeline de agregación SQL de eventos del día.
- Reglas de filtrado de usuarios para re-análisis.
- Loop de actualización con LLM batch.

### Sprint 6 (semana 11-12): insights humanizados
- Generación de mensajes matutinos en batch.
- Razones por bloque ("¿Por qué este ejercicio?").
- Insight semanal de progreso.
- UI para mostrar todos estos mensajes.

Después de las 12 semanas, tenés el sistema completo funcionando. Los siguientes 3 meses son refinamiento, expansión de la biblioteca, y comienzo del trabajo en modelos propios para reemplazar APIs.

---

## 10. Métricas y observabilidad

### 10.1 Métricas técnicas a trackear

- Latencia p50/p95/p99 de generación de roadmap inicial.
- Tasa de validación exitosa de outputs de LLM (debe ser >98%).
- Costo de IA por usuario por mes (alerta si >$0.10).
- % de usuarios re-analizados por noche (debe estar entre 5% y 15%).
- Hit rate de cache de la biblioteca de bloques (>95%).

### 10.2 Métricas de producto a trackear

- Tasa de completion del onboarding (drop-off por pregunta).
- Tasa de usuarios que completan el primer bloque.
- Tasa de usuarios que completan el primer nivel.
- Tiempo promedio entre bloques.
- NPS al completar primer nivel (señal temprana de PMF).
- % de usuarios que llegan al nivel 3 (señal de tracción real).

### 10.3 Alertas críticas

- Costo de IA del día > 2x el promedio histórico.
- Tasa de error en validación de roadmaps > 5%.
- Latencia p95 de generación inicial > 10 segundos.
- Cualquier usuario con costo individual > $1 en un día (indica abuso o bug).

---

## 11. Decisiones abiertas

Decisiones que aún no están tomadas y requieren validación o investigación:

- [ ] ¿Permitir que el usuario edite manualmente su roadmap (skip blocks, reorder)? Pros: agency. Contras: rompe la coherencia pedagógica.
- [ ] ¿Tracks múltiples activos al mismo tiempo o uno solo? Empezar con uno solo para simplicidad, evaluar después.
- [ ] ¿Cómo manejar usuarios que cambian de objetivo en medio del roadmap? ¿Se regenera entero o se mergea?
- [ ] ¿Certificados de completitud de track? ¿En qué momento (al completar 100% o al lograr mastery promedio X)?
- [ ] Política de re-evaluación: ¿se hace mini-test cada N semanas para recalibrar?

---

## 12. Referencias internas

- `docs/business/plan_de_negocio.docx` — Plan de negocio general.
- `docs/architecture/01-overview.md` — Arquitectura general (a crear).
- `docs/architecture/02-sparks-system.md` — Sistema de tokens (a crear).
- `docs/product/learning-blocks-taxonomy.md` — Taxonomía de la biblioteca (a crear).
- `docs/decisions/ADR-XXX-llm-selection.md` — Decisión sobre proveedores LLM (a crear).

---

*Documento vivo. Actualizar cuando se tomen decisiones nuevas o se aprenda de la implementación.*

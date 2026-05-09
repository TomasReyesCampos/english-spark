# US-042: Prompt versioning + audit trail

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 2
**Story points:** 3
**Persona:** Admin (compliance + debugging)
**Owner:** —

---

## Contexto

Cada prompt template del Gateway evoluciona en el tiempo. Sin
versionado:
- Imposible reproducir output viejo para debugging.
- Compliance / legal puede pedir "qué prompt se le mandó al user X
  el día Y".
- A/B testing entre prompt versions imposible.

Esta story implementa:
- Storage de prompts en repo (filesystem versionado vía git).
- Audit trail: cada invocation persiste el `prompt_version` usado.
- Mecanismo de promote / rollback de prompt versions.

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§6 (prompt versioning).

## Scope

### In

- Estructura de prompts en repo:
  ```
  apps/workers/ai-gateway/prompts/
  ├── generate_initial_roadmap/
  │   ├── v1_anthropic.txt
  │   ├── v2_anthropic.txt          # post-improvement
  │   └── v1_openai.txt
  ├── score_fluency/
  │   └── v1_anthropic.txt
  └── ...
  ```
- Prompt loader que cachea contenido en KV al startup (TTL 1 hour).
- Convención de naming: `v<n>_<provider>.txt` o `v<n>.txt` si
  agnostic.
- Cada prompt template includes header metadata:
  ```
  # version: 1.0.0
  # provider: anthropic
  # last_modified: 2026-05-15
  # author: <github_handle>
  ---
  SYSTEM:
  <system prompt content>
  ---
  USER:
  <user prompt content>
  ```
- Tabla `ai_invocations` extendida con `prompt_version` column.
- Loader retorna `{ system, user, version }` (parseado del file).
- Endpoint `GET /admin/prompts/:task_id` lista versions disponibles
  para una task con sus headers.
- Endpoint `GET /admin/prompts/:task_id/:version` retorna full
  content para inspección.
- Mecanismo de promote: actualizar `task_registry.prompt_template_path`
  para apuntar a nueva version → toma efecto next cache reload (5
  min).
- Rollback: simplemente volver el path al version anterior +
  refresh cache.
- Telemetry: `prompt.version_loaded`,
  `prompt.cache_hit`, `prompt.cache_miss`.

### Out

- UI admin para editar prompts (post-MVP).
- A/B testing entre prompt versions con metrics aggregation
  (post-MVP — capacidad existe vía provider_chain weights, pero
  análisis automatizado es post-MVP).
- Diff entre versions (post-MVP).

## Acceptance criteria

- **Given** prompt file `v1_anthropic.txt` con header válido,
  **When** loader procesa, **Then** retorna
  `{ system, user, version: '1.0.0', provider: 'anthropic' }`.
- **Given** prompt file con header malformado, **When** loader
  parsea, **Then** retorna error claro y NO sirve el prompt.
- **Given** task `generate_initial_roadmap` con
  `prompt_template_path = 'generate_initial_roadmap/v1_anthropic.txt'`,
  **When** invocation ejecuta, **Then** `ai_invocations.prompt_version`
  guarda `'1.0.0'`.
- **Given** se crea `v2_anthropic.txt` y se actualiza task registry
  para apuntar a v2, **When** próxima invocation post-cache-refresh,
  **Then** usa v2 y persiste `prompt_version = '2.0.0'` (o la
  version del header de v2).
- **Given** rollback: cambiar registry de vuelta a v1, **When**
  refresh cache, **Then** próxima invocation usa v1 sin pérdida de
  data.
- **Given** GET `/admin/prompts/score_fluency`, **When** se llama,
  **Then** retorna lista de versions disponibles con metadata.
- **Given** GET `/admin/prompts/score_fluency/v1_anthropic`,
  **When** se llama, **Then** retorna full content del file.
- **Given** prompt loader cache hit rate, **When** se mide después
  de 1 día, **Then** ≥ 95% (KV con TTL 1h cubre casi todas las
  invocations).

## Developer details

### Owning service

`apps/workers/ai-gateway/src/prompts/` + extensión a
`task_registry`.

### Dependencies

- US-027/028.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §6.

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/prompts/loader.ts
interface ParsedPrompt {
  version: string;
  provider?: string;
  system: string;
  user: string;
  metadata: Record<string, string>;
}

export async function loadPromptTemplate(
  taskId: string,
  version: string,
  env: Env
): Promise<ParsedPrompt> {
  const cacheKey = `prompt:${taskId}/${version}`;
  const cached = await env.PROMPT_KV.get(cacheKey);
  if (cached) {
    track('prompt.cache_hit', { task_id: taskId, version });
    return JSON.parse(cached);
  }

  track('prompt.cache_miss', { task_id: taskId, version });
  // Prompts bundled at build time
  const path = `prompts/${taskId}/${version}.txt`;
  const content = await loadBundledFile(path);
  const parsed = parsePromptFile(content);

  await env.PROMPT_KV.put(cacheKey, JSON.stringify(parsed), {
    expirationTtl: 3600,
  });

  track('prompt.version_loaded', {
    task_id: taskId,
    version: parsed.version,
  });

  return parsed;
}

function parsePromptFile(content: string): ParsedPrompt {
  const lines = content.split('\n');
  const metadata: Record<string, string> = {};
  let i = 0;

  // Parse header (lines starting with #)
  while (i < lines.length && lines[i].startsWith('#')) {
    const match = lines[i].match(/^#\s*(\w+):\s*(.+)$/);
    if (match) metadata[match[1]] = match[2];
    i++;
  }

  // Skip --- separator
  while (i < lines.length && lines[i].trim() !== '---') i++;
  i++;

  // Parse SYSTEM section
  const systemStart = i;
  while (i < lines.length && lines[i].trim() !== '---') i++;
  const system = lines.slice(systemStart, i).join('\n')
    .replace(/^SYSTEM:\s*/, '').trim();
  i++;

  // Parse USER section
  const userContent = lines.slice(i).join('\n')
    .replace(/^USER:\s*/, '').trim();

  if (!metadata.version) {
    throw new Error('Prompt missing required version header');
  }

  return {
    version: metadata.version,
    provider: metadata.provider,
    system,
    user: userContent,
    metadata,
  };
}
```

### Schema extension

```sql
ALTER TABLE ai_invocations
  ADD COLUMN prompt_version TEXT;

CREATE INDEX idx_ai_invocations_prompt_version
  ON ai_invocations(prompt_version, invoked_at DESC);
```

### Prompt example

```
# version: 1.0.0
# provider: anthropic
# last_modified: 2026-05-15
# author: tomas
---
SYSTEM:
Eres un experto en pedagogía de inglés para hispanohablantes
latinoamericanos...
---
USER:
PERFIL DEL USUARIO:
{{profile}}

INSTRUCCIONES:
...
```

### Integration points

- US-028 task registry (referencia path).
- US-031 cost tracking (extiende con prompt_version).
- US-043 admin endpoint (lists prompts).

### Notas técnicas

- Prompts bundled at build time vía wrangler — no fetch externo.
- KV cache 1h balance entre freshness y latencia.
- Naming convention permite múltiples versions coexistiendo
  (útil para A/B test).
- Git history es source of truth para change history.

## Definition of Done

- [ ] Estructura de prompts en repo organizada.
- [ ] Loader con parser de header.
- [ ] Cache KV funcional.
- [ ] `ai_invocations.prompt_version` persistido.
- [ ] Endpoints admin para list + get prompt content.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit del parser (10+ casos: valid, missing version,
  malformed).
- [ ] Test integration: load prompt, invocar task, verify
  prompt_version en log.
- [ ] Validation contra spec
  `ai-gateway-strategy.md` §6.
- [ ] PR aprobada y mergeada.

---

*Útil para debugging + compliance + futuros A/B tests.*

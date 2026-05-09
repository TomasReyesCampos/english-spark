# US-036: Task `generate_notification_content`

**Estado:** Draft
**Epic:** EPIC-05-ai-gateway-foundation
**Sprint target:** Sprint 1
**Story points:** 2
**Persona:** Admin (consumido por notifications batch)
**Owner:** —

---

## Contexto

Generar copy personalizado de notifications push usando IA. Cron
nocturno de notifications-system invoca esta task para cada user
activo, generando `daily_reminder` y `streak_at_risk` con context
del user (nombre, racha, foco pedagógico, logros).

Backend en
[`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
§4.2.7 +
[`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
(banco de templates baseline).

## Scope

### In

- Registrar task `generate_notification_content`:
  - Provider chain: `[{anthropic, claude-haiku-4-5, weight: 100}]`.
  - Budget: $0.001 per call.
  - Timeout: 8s.
  - Active: true.
- Input schema:
  ```typescript
  {
    user: { name?: string; streak: number; plan: 'basic'|'pro'|'premium' },
    tomorrow: { focus: string },                  // 'pronunciation /θ/'
    recent_highlight?: string,                    // 'logro X desbloqueado'
    notifications_to_generate: Array<'daily_reminder' | 'streak_at_risk'>,
  }
  ```
- Output schema:
  ```typescript
  {
    daily_reminder?: { title: string; body: string; copy_id: string },
    streak_at_risk?: { title: string; body: string; copy_id: string },
  }
  ```
- Validation post-generation:
  - title ≤ 50 chars.
  - body ≤ 90 chars.
  - No voseo (lista bloqueada per ADR-008).
  - No guilt patterns (per copy bank §10.5).
  - Max 1 emoji por title, max 1 por body.
  - Si validation falla: fallback a template hardcoded del banco.
- Prompt template:
  `apps/workers/ai-gateway/prompts/generate_notification_content/v1_anthropic.txt`
  basado en `push-notifications-copy-bank.md` §10 reglas.
- Telemetry: `notif_content.generated`,
  `.validation_failed_fallback_used`.

### Out

- Persistencia en
  `notifications_scheduled` (eso lo hace notifications-system, esta
  task solo retorna copy).
- Otros tipos de notifs (welcome series, transactional) — no usan
  IA, son templates.

## Acceptance criteria

- **Given** task registrada y activa, **When** se invoca con
  `user.name='María', streak=8`, **Then** retorna 200 con
  `daily_reminder.body` mencionando la racha de 8 días.
- **Given** user con `streak < 5` y `notifications_to_generate
  incluye streak_at_risk`, **When** task ejecuta, **Then**
  `streak_at_risk` no se genera (regla del producto).
- **Given** LLM genera title con 60 chars, **When** validation
  detecta, **Then** retry 1x con prompt "title must be ≤ 50".
  Si segunda falla: fallback a template hardcoded.
- **Given** LLM genera body con voseo ("podés"), **When**
  validation detecta, **Then** retry 1x. Si fail: fallback.
- **Given** LLM genera title con 3 emojis, **When** validation,
  **Then** retry 1x. Si fail: fallback.
- **Given** user.name = null, **When** task genera, **Then** copy
  omite saludo personal sin renderizar literalmente "{{name}}".
- **Given** invocation cuesta $0.0008 promedio, **When** persist,
  **Then** cost tracking refleja correctamente.
- **Given** task se invoca 10000x en batch nocturno, **When** se
  procesa, **Then** no excede budget global ($50/day).

## Developer details

### Owning service

`apps/workers/ai-gateway/src/handlers/generate-notification-content.ts`.

### Dependencies

- US-027/028/029/030/031.
- `push-notifications-copy-bank.md` para reglas + fallback
  templates.

### Specs referenciados

- [`ai-gateway-strategy.md`](../../architecture/ai-gateway-strategy.md)
  §4.2.7.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §10 — reglas de personalización IA.
- [`push-notifications-copy-bank.md`](../../product/push-notifications-copy-bank.md)
  §1.1 — tono y voz mexicano-tuteo.

### Validation rules (de copy bank §10.4)

```typescript
const BANNED_VOSEO = ['podés', 'querés', 'hablás', 'tenés', 'sentís', 'hacés',
                      'decís', 'preferís', 'vos ', 'registrate', 'iniciá'];
const GUILT_PATTERNS = ['no te importa', 'vas a fallar', 'perdiste',
                        'abandonaste', 'última oportunidad', 'no tienes excusa'];

function validateCopy(copy: { title: string; body: string }): string[] {
  const issues: string[] = [];
  if (copy.title.length > 50) issues.push('title_too_long');
  if (copy.body.length > 90) issues.push('body_too_long');

  const lower = (copy.title + ' ' + copy.body).toLowerCase();
  if (BANNED_VOSEO.some(v => lower.includes(v))) issues.push('voseo_detected');
  if (GUILT_PATTERNS.some(g => lower.includes(g))) issues.push('guilt_pattern');

  const titleEmojis = countEmojis(copy.title);
  if (titleEmojis > 1) issues.push('too_many_emojis_title');
  const bodyEmojis = countEmojis(copy.body);
  if (bodyEmojis > 1) issues.push('too_many_emojis_body');

  if (/\{\{\w+\}\}/.test(copy.title + copy.body)) issues.push('unresolved_var');

  return issues;
}
```

### Implementación esperada

```typescript
// apps/workers/ai-gateway/src/handlers/generate-notification-content.ts
export async function handleGenerateNotificationContent(
  input: any,
  context: HandlerContext
): Promise<any> {
  const validated = InputSchema.parse(input);

  const promptTemplate = await loadPromptTemplate(
    'generate_notification_content', 'v1_anthropic', context.env
  );

  const result: any = {};
  for (const notifType of validated.notifications_to_generate) {
    if (notifType === 'streak_at_risk' && validated.user.streak < 5) {
      continue; // skip
    }

    let attempt = 0;
    let validatedCopy: any = null;

    while (attempt <= 1 && !validatedCopy) {
      const llmResult = await invokeWithFallback(
        'generate_notification_content',
        {
          system_prompt: promptTemplate.system,
          messages: [{ role: 'user', content: renderPromptForType(notifType, validated, attempt) }],
          temperature: 0.7,
          max_tokens: 200,
          response_format: 'json',
        },
        context
      );

      try {
        const parsed = JSON.parse(llmResult.output.text);
        const issues = validateCopy(parsed);
        if (issues.length === 0) {
          validatedCopy = { ...parsed, copy_id: `${notifType}.ai_generated.${Date.now()}` };
        } else {
          attempt++;
          if (attempt > 1) {
            track('notif_content.validation_failed_fallback_used', { type: notifType, issues });
            validatedCopy = getFallbackTemplate(notifType, validated.user);
          }
        }
      } catch {
        attempt++;
        if (attempt > 1) {
          validatedCopy = getFallbackTemplate(notifType, validated.user);
        }
      }
    }

    result[notifType] = validatedCopy;
    track('notif_content.generated', { type: notifType, used_fallback: !validatedCopy });
  }

  return result;
}
```

### Integration points

- Notifications system batch nocturno (consumer).
- US-029 Anthropic adapter.
- Cost tracking (US-031).
- Copy bank fallback templates (cargar a memoria al startup).

### Notas técnicas

- Cost target $0.001/call permite 10k notifs nocturnas en ~$10/day
  (alineado con `notifications-system.md` §17 estimate).
- Validation strict por defecto — preferir fallback a publicar
  copy malo.
- Fallback templates ya existen en
  `push-notifications-copy-bank.md` §2-§7.

## Definition of Done

- [ ] Task registrada y activa.
- [ ] Handler con validation pipeline + fallback.
- [ ] 8 acceptance criteria pasan.
- [ ] Tests unit de `validateCopy` con 10+ casos (voseo, guilt,
  emoji, length, vars).
- [ ] Test integration con LLM real (genera 5 copys, todos pasan
  validation).
- [ ] Validation contra spec
  `push-notifications-copy-bank.md` §10.
- [ ] PR aprobada y mergeada.

---

*Consumed by notifications-system batch nocturno.*

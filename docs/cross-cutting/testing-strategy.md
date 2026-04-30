# Testing Strategy

> Estrategia de testing del proyecto: scope (unit + integration), tooling,
> targets de coverage por capa, CI gates. **E2E tests están explícitamente
> fuera de scope** para MVP.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** Sistema completo (excepto E2E)

---

## 0. Cómo leer este documento

- §1 establece principios y scope.
- §2 cubre **unit tests** (foco principal).
- §3 cubre **integration tests** (para flujos críticos).
- §4 lista **tests obligatorios por sistema** (mínimo para considerar
  un sistema completo).
- §5 cubre **tooling** (Vitest, etc.).
- §6 cubre **CI gates**.
- §7 cubre **test data y fixtures**.
- §8 cubre **mocking strategy** (cuándo sí, cuándo no).
- §9 cubre testing de cosas específicas (LLM, audio, jobs nocturnos).

---

## 1. Principios y scope

### 1.1 Pirámide

```
         ┌──────┐
         │ E2E  │  ← FUERA DE SCOPE (no Playwright/Detox)
         └──────┘
       ┌──────────┐
       │Integration│  ← Flujos críticos (auth, sparks, mastery)
       └──────────┘
     ┌──────────────┐
     │     Unit     │  ← Foco principal: lógica pura
     └──────────────┘
```

**Distribución target:**
- 70% unit tests.
- 30% integration tests.
- 0% E2E.

### 1.2 Por qué no E2E

- Costoso de mantener (flaky, lentos, requieren infra dedicada).
- Para dev solo, el ROI no justifica.
- Manual testing del feature crítico antes de cada release alcanza para
  MVP.
- Cuando haya equipo y volumen, reconsiderar (pero no antes).

Si necesitás "validar el flujo completo", hacelo:
- Con un integration test que llama a varios endpoints en secuencia
  (sin browser real).
- Manualmente en Expo dev build / web preview antes de release.

### 1.3 Qué se testea

**SÍ:**
- Funciones puras: scoring, cálculos, validators, parsers.
- Endpoints de API: validación, autorización, response shapes.
- Jobs nocturnos: flujo completo con DB de test.
- Lógica de Sparks: cobro, refund, expiración, rollover.
- Mastery: agregación de scores, transiciones de estado, decay.
- Webhooks: signature verification, idempotency.
- Validation Zod schemas vs golden examples.

**NO (o muy poco):**
- UI components puramente presentacionales (testing manual + visual
  review).
- Wrappers triviales sobre librerías externas.
- Configuración estática.

### 1.4 Filosofía: tests que importan

Cada test debe responder: **¿qué bug previene?**

Si un test no protege contra una regresión real (o contra un bug que ya
ocurrió), es ruido. Los tests deben hacer fallar el CI cuando el código
se rompe — no antes, no después.

---

## 2. Unit tests

### 2.1 Scope

Lógica pura sin side effects. Funciones que dado un input determinístico
producen un output determinístico.

Ejemplos canónicos:

| Sistema | Función | Lo que testea |
|---------|---------|--------------|
| Pedagogical | `calculateFluencyScore(metrics, cefr)` | Pondera correctamente WPM, pausas, fillers según CEFR |
| Pedagogical | `mapDimensionsToCEFR(scores)` | Thresholds correctos por nivel |
| Pedagogical | `scoreToPerformance(score)` | Bins correctas (perfect/good/fair/poor) |
| Sparks | `calculateRollover(prevBalance, plan)` | Cap a 2x del plan, expira el resto |
| Sparks | `chargeOperationOrder(charged, balance)` | Orden correcto: plan → bonus → pack |
| AI Gateway | `validateOutput(json, schema)` | Acepta válidos, rechaza inválidos con mensaje útil |
| Anti-fraud | `calculateFraudScore(signals)` | Suma correcta de pesos por signal |
| Pedagogical | `applyLearningRateToDelta(delta, attempts)` | Rate correcta según attempts (0.30/0.15/0.08) |
| Notifications | `parseFiller(transcript)` | Detecta `like` como filler vs conector |
| Motivation | `calculateNextStreakMilestone(current)` | 7, 14, 30, 60, 100, 365 |

### 2.2 Coverage target

- **Lógica de scoring (Pedagogical):** ≥ 90%.
- **Sparks (financial):** ≥ 95%.
- **Anti-fraud scoring:** ≥ 85%.
- **Validation y parsers:** ≥ 90%.
- **Resto:** ≥ 70%.

Coverage no es el objetivo — bugs prevenidos lo son. Pero coverage
bajo en lógica financiera o de scoring es bandera roja.

### 2.3 Patrones

#### Tabla-driven (preferido)

```typescript
describe('mapDimensionsToCEFR', () => {
  const cases: Array<{ name: string; scores: MasteryDimensions; expected: string }> = [
    {
      name: 'all dimensions A2 → A2',
      scores: { pronunciation: 30, fluency: 30, grammar: 30, vocabulary: 30, listening: 30 },
      expected: 'A2',
    },
    {
      name: 'mixed B1+ → B1+',
      scores: { pronunciation: 65, fluency: 70, grammar: 60, vocabulary: 60, listening: 60 },
      expected: 'B1+',
    },
    {
      name: 'edge case at threshold 35 → B1',
      scores: { pronunciation: 35, fluency: 35, grammar: 35, vocabulary: 35, listening: 35 },
      expected: 'B1',
    },
    // ...
  ];

  test.each(cases)('$name', ({ scores, expected }) => {
    expect(mapDimensionsToCEFR(scores)).toBe(expected);
  });
});
```

#### Edge cases obligatorios

- Inputs vacíos (array vacío, string vacío).
- Inputs en límites (0, max, NaN).
- Inputs malformados (cuando aplica).
- Concurrencia (cuando aplica).
- Inputs internacionales (vocales, caracteres especiales).

### 2.4 Anti-patterns

- ❌ Tests que solo verifican que se llamó a una función mocked
  (testing implementation, no behavior).
- ❌ Tests con un solo caso "happy path".
- ❌ Tests que dependen de orden (un test para cada uno).
- ❌ Tests con setup global compartido y mutación.
- ❌ `expect(result).toBeTruthy()` como aserción única.

---

## 3. Integration tests

### 3.1 Scope

Flujos que tocan múltiples módulos, DB, o servicios externos. Más
costosos pero atrapan bugs que unit tests no.

### 3.2 Setup

- **DB:** Postgres local (Docker) o Supabase test branch. **No mockear
  Postgres.**
- **Inngest:** modo "dev" en local que ejecuta jobs sincrónicamente.
- **Servicios externos (Stripe, RevenueCat, Firebase, Azure, LLM
  providers):** mockeados con fixtures realistas.
- **Cloudflare Workers:** ejecutados con `wrangler dev` o
  `@cloudflare/workers-types` directo.

### 3.3 Flujos prioritarios

Cada uno debe tener integration test al final del sprint que lo
implementa.

#### Flow 1: Signup → onboarding → primer ejercicio

```typescript
test('signup → onboarding → submit first exercise → mastery updated', async () => {
  // 1. Mock Firebase verifyToken
  mockFirebaseVerify({ uid: 'firebase-uid-123', email: 'test@example.com' });

  // 2. Sync user
  const syncResp = await fetch('/auth/sync-user', { /* ... */ });
  expect(syncResp.status).toBe(200);

  // 3. Verify user in DB + initial sparks balance
  const user = await db.users.findByFirebaseUid('firebase-uid-123');
  expect(user).not.toBeNull();
  const balance = await db.sparks.getBalance(user.id);
  expect(balance.total_sparks).toBe(50); // trial Sparks

  // 4. Submit exercise (mock STT + scoring)
  mockAiGateway('transcribe_user_audio', () => 'I think this is a test');
  mockAiGateway('score_pronunciation', () => ({ overall: 75, phonemes: [...] }));

  const submitResp = await submitExerciseAttempt({ /* ... */ });
  expect(submitResp.scores.pronunciation).toBe(75);

  // 5. Verify subskill mastery updated
  const mastery = await db.pedagogical.getSubskillMastery(user.id, 'pron_th_voiceless');
  expect(mastery.current_score).toBeGreaterThan(0);

  // 6. Verify event emitted
  expect(getEmittedEvents()).toContainEqual(
    expect.objectContaining({ event_name: 'exercise.attempt_completed' })
  );
});
```

#### Flow 2: Sparks: charge → operation fails → refund

```typescript
test('charge before operation, refund if fails', async () => {
  const userId = await createTestUser({ sparks: 100 });

  // Mock LLM call to throw
  mockAiGateway('score_pronunciation', () => { throw new Error('Provider down'); });

  // Submit exercise
  await expect(submitExerciseAttempt({ /* ... */ })).rejects.toThrow();

  // Balance should be unchanged (refunded)
  const balance = await db.sparks.getBalance(userId);
  expect(balance.total_sparks).toBe(100);

  // Audit log should show charge + refund
  const txs = await db.sparks.getRecentTransactions(userId, 10);
  expect(txs).toHaveLength(2);
  expect(txs[0].source).toBe('refund');
  expect(txs[1].source).toBe('operation_charge');
});
```

#### Flow 3: Stripe webhook → balance acreditado

```typescript
test('stripe webhook processes pack purchase idempotently', async () => {
  const userId = await createTestUser({ sparks: 0 });

  const webhookPayload = stubStripeWebhook({
    type: 'checkout.session.completed',
    metadata: { user_id: userId, pack_type: 'medium' },
    payment_id: 'pi_test_123',
  });

  // Send same webhook twice (Stripe retry simulation)
  const r1 = await fetch('/webhooks/stripe', { body: webhookPayload });
  const r2 = await fetch('/webhooks/stripe', { body: webhookPayload });

  expect(r1.status).toBe(200);
  expect(r2.status).toBe(200); // idempotent OK

  // Balance acreditado UNA vez
  const balance = await db.sparks.getBalance(userId);
  expect(balance.pack_sparks).toBe(300); // medium pack

  const packs = await db.sparks.getPacksByUser(userId);
  expect(packs).toHaveLength(1);
});
```

#### Flow 4: Job nocturno de cleanup

```typescript
test('nightly cleanup removes old audios + transcriptions', async () => {
  // Setup: insertar attempt con audio 35 días atrás
  const oldAttempt = await createTestAttempt({
    user_id: 'user-1',
    audio_storage_key: 'users/user-1/audio-old.webm',
    completed_at: daysAgo(35),
  });
  const recentAttempt = await createTestAttempt({
    user_id: 'user-1',
    audio_storage_key: 'users/user-1/audio-recent.webm',
    completed_at: daysAgo(5),
  });

  await runNightlyCleanup();

  // Audio viejo borrado de R2
  expect(mockR2.deleted).toContain('users/user-1/audio-old.webm');
  expect(mockR2.deleted).not.toContain('users/user-1/audio-recent.webm');

  // Attempt > 90 días tiene user_id NULL (anonimizado)
  // (este test no aplica porque solo era 35d, otro test cubre el de 90+)
});
```

#### Flow 5: AI Gateway con fallback

```typescript
test('AI Gateway falls back to secondary when primary fails', async () => {
  mockProvider('anthropic', () => { throw new Error('503 Service Unavailable'); });
  mockProvider('google', () => ({ output: '{"valid": "json"}' }));

  const result = await aiGateway.executeTask('generate_initial_roadmap', { /* ... */ });

  expect(result.fallback_used).toBe(true);
  expect(result.model_used).toMatch(/gemini/);
  expect(result.output).toEqual({ valid: 'json' });
});
```

### 3.4 Coverage target

No es el foco. Lo importante es **cubrir todos los flujos críticos
identificados en §3.3 y §4**, incluso si la coverage % no es alta.

---

## 4. Tests obligatorios por sistema

Mínimo para considerar el sistema "implementado" en MVP.

### 4.1 Authentication

**Unit:**
- `verifyFirebaseToken(authHeader)`: caso con token válido, expirado,
  malformado, ausente.
- `parseProvider(decoded)`: matcheo de provider correcto.

**Integration:**
- Sync user crea fila en `users` y emite evento.
- Sync user idempotente con mismo firebaseUid.
- Account deletion soft → 30d → hard.
- Anonymous → upgrade flow.

### 4.2 Sparks

**Unit (≥ 95% coverage):**
- `calculateRollover` con todas las combinaciones (sub-cap, at-cap,
  over-cap).
- `chargeOperationOrder` con balance distribuido en plan/bonus/pack.
- `calculatePackExpiry` con diferentes fechas.
- `validateBalanceConsistency(transactions, balance)` detecta
  divergencias.

**Integration:**
- Cobro → refund flow.
- Webhook de pack purchase idempotente.
- Asignación mensual con rollover correcto.
- Expiración de plan_sparks y pack_sparks.
- Audit reconciliation job detecta y alerta divergencias.

### 4.3 Pedagogical

**Unit:**
- Cada función de scoring (5 dimensiones) con casos:
  - Score perfecto.
  - Score borderline.
  - Score muy bajo.
  - Inputs malformados (audio silencioso, transcript vacío).
- `applyLearningRateToDelta` con `attempts < 5`, `< 20`, `>= 20`.
- `mapDimensionsToCEFR` en cada threshold.
- Spaced repetition `calculateNextReview`.
- Decay `calculateCurrentSkillLevel` con varias `daysSinceLastPractice`.
- Detección de struggling: cada uno de los 4 signals.

**Integration:**
- `submitExerciseAttempt` end-to-end (con AI Gateway mockeado).
- Edge cases del documento §11 (todos).
- Test-out flow: success, fail, cooldown.
- Bloque pasa a struggling cuando se cumplen 2+ signals.
- Re-evaluación periódica reincorpora al plan.

### 4.4 AI Roadmap

**Unit:**
- Validación de output del LLM con Zod.
- Resolución de prerequisites en orden topológico.
- Filtrado SQL de usuarios para re-análisis nocturno.

**Integration:**
- `generateInitialRoadmap` produce roadmap válido (mock LLM).
- `generateInitialRoadmap` con LLM que devuelve JSON malformado:
  fallback a template.
- Job nocturno corre sobre N usuarios y actualiza solo a los que matchean
  reglas.

### 4.5 Student Profile

**Unit:**
- Cálculo de `observed_behavior` de eventos.
- Validación de inputs del onboarding.
- Cálculo de CEFR estimado del mini-test.

**Integration:**
- Onboarding completo → profile persistido + roadmap inicial generado.
- Trial sparks decrementados al usar.
- Day 7 → assessment ofrecido + push enviado.

### 4.6 Motivation

**Unit:**
- Engine de detección de logros: 80 logros con criterios cumplidos /
  no cumplidos.
- Lógica de freeze tokens: usar, agregar, cap.
- Cálculo de perfil motivacional desde behaviors.

**Integration:**
- Completar primer ejercicio → `first_steps` desbloqueado + bonus
  Sparks.
- Streak 7 días → `streak_7` desbloqueado.
- Daily goal cumplido → bonus Sparks aplicado.

### 4.7 Notifications

**Unit:**
- `parseTimezoneToLocalHour(tz, utcHour)`: corner cases (DST, half-hour
  timezones).
- Rate limiter Durable Object: increment, deny over limit, reset al
  cambio de día.

**Integration:**
- Cron de daily reminder envía solo a usuarios cuya hora preferida es la
  actual en su tz.
- FCM mock recibe payload correcto.
- Token marcado como inactive cuando FCM retorna error de invalid
  token.

### 4.8 Anti-fraud

**Unit:**
- `calculateFraudScore(signals)`: cada signal con su peso.
- `generateDeviceFingerprint(deviceInfo)`: misma input → misma output.

**Integration:**
- Signup desde fingerprint con 5+ matches → restricción auto aplicada.
- Apelación submitida → ticket creado.

### 4.9 AI Gateway

**Unit:**
- Validación Zod aplicada a output del LLM.
- Routing por taskId correcto.
- Cálculo de costo por modelo.

**Integration:**
- Primary falla → fallback automático.
- Output inválido → retry una vez, luego fallback.
- Cost tracking persistido por usuario.

### 4.10 Customer Support

**Unit:**
- Categorización de tickets desde texto.
- Detección de escalation triggers en AI Assistant prompt.

**Integration:**
- AI Assistant escala a Nivel 3 cuando detecta "pérdida de datos" en
  mensaje del usuario.
- Ticket creado tiene priority y category correctas.

### 4.11 Webhooks (Stripe + RevenueCat)

**Unit:**
- Verificación de signature con secret correcto.
- Verificación rechaza signature inválida.

**Integration:**
- Webhook de purchase → balance acreditado idempotente.
- Webhook de subscription cancelled → cycle marcado.

---

## 5. Tooling

### 5.1 Stack

| Capa | Tool | Por qué |
|------|------|---------|
| Test runner | **Vitest** | Rápido, integra bien con Vite/TS, compatible con Cloudflare Workers |
| Assertions | Vitest built-in (`expect`) | Suficiente, nada extra |
| Mocking | Vitest `vi.mock`, MSW para HTTP | MSW para mockear APIs externas (Stripe, Firebase) |
| DB de test | Supabase test branch o Postgres en Docker | Postgres real, no mocked |
| Workers testing | `@cloudflare/workers-types` + `unstable_dev` | Test directo de Workers sin browser |
| Coverage | Vitest `--coverage` (c8) | Built-in |
| CI | GitHub Actions | Free tier suficiente |

### 5.2 No usamos

- ❌ **Jest** — Vitest es estricto-mejor para TS + Vite + Workers.
- ❌ **Playwright / Detox / Cypress** — E2E fuera de scope.
- ❌ **Storybook** — útil eventualmente, no en MVP.
- ❌ **Mock libraries pesadas** — Vitest mocks alcanzan.

### 5.3 Estructura de carpetas

```
src/
├── pedagogical/
│   ├── scoring.ts
│   ├── scoring.test.ts          ← unit tests al lado del código
│   └── ...
└── ...

tests/
├── integration/
│   ├── auth-flow.test.ts        ← integration tests aparte
│   ├── sparks-charge.test.ts
│   └── ...
└── fixtures/
    ├── audio-samples/
    ├── stripe-webhooks/
    └── ...
```

### 5.4 Convenciones de naming

- Unit test: `<file>.test.ts` al lado del archivo.
- Integration test: `tests/integration/<feature>.test.ts`.
- Test description: empieza con verbo. `'returns', 'throws',
  'updates'`.
- Test files siempre tienen al menos un `describe` por función testeada.

---

## 6. CI gates

### 6.1 PRs deben pasar

1. **Lint:** ESLint + Prettier.
2. **Typecheck:** `tsc --noEmit`.
3. **Unit tests:** todos pasan.
4. **Integration tests:** todos pasan.
5. **Coverage:** no decrece más de 1% vs main.
6. **Secrets scan:** gitleaks.

### 6.2 Workflow GitHub Actions (esquemático)

```yaml
# .github/workflows/ci.yml (a crear)
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: [5432:5432]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test:unit -- --coverage
      - run: pnpm test:integration
      - uses: gitleaks/gitleaks-action@v2
```

### 6.3 Tiempo target del CI

- Unit tests: < 30s.
- Integration tests: < 3 min.
- Total CI: < 5 min.

Si excede, bloquea iteración. Optimizar (paralelizar, profilar slow
tests, considerar test partitioning).

---

## 7. Test data y fixtures

### 7.1 Test users factory

```typescript
// tests/factories/user.ts
export async function createTestUser(overrides: Partial<User> = {}): Promise<User> {
  const user = {
    id: crypto.randomUUID(),
    firebase_uid: `firebase-test-${nanoid(8)}`,
    email: `test-${nanoid(6)}@example.com`,
    primary_provider: 'email' as const,
    created_at: new Date(),
    ...overrides,
  };
  await db.users.insert(user);
  return user;
}
```

### 7.2 Audio samples

`tests/fixtures/audio-samples/` con audios sintéticos representativos:

- `silent.webm` — para test de audio unusable.
- `noisy.webm` — SNR bajo.
- `clear-th-correct.webm` — pronunciación correcta.
- `clear-th-as-s.webm` — error típico hispanohablante.
- `30sec-fluent-b2.webm` — flujo natural B2.
- `30sec-with-fillers.webm` — con muchas muletillas.

Idealmente: audios reales con consent, anonimizados. Mientras: sintéticos
con TTS premium.

### 7.3 Webhook payloads

`tests/fixtures/stripe-webhooks/` y `tests/fixtures/revenuecat-webhooks/`
con payloads reales (anonimizados) para reproducir bugs específicos.

### 7.4 LLM golden examples

`tests/fixtures/llm-outputs/` con outputs canónicos que deberían pasar
validación.

---

## 8. Mocking strategy

### 8.1 Cuándo SÍ mockear

- **Servicios externos con costo:** OpenAI, Anthropic, Azure, Stripe,
  RevenueCat, Firebase Auth, FCM, R2.
- **Servicios externos con rate limits:** mismo.
- **Tiempo:** `Date.now()`, `setTimeout`. Usar fakes (Vitest
  `vi.useFakeTimers`).
- **Crypto random:** para tests determinísticos donde el output depende
  de UUIDs.

### 8.2 Cuándo NO mockear

- **Postgres:** usar Postgres real en Docker o test branch. Mocks de
  ORM esconden bugs.
- **Inngest:** modo dev local que ejecuta sincrónicamente.
- **Lógica interna:** nunca mockear funciones del mismo sistema.

### 8.3 Patrón con MSW

```typescript
// tests/setup/msw.ts
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

export const server = setupServer(
  // Default handlers
  http.post('https://api.anthropic.com/v1/messages', () => {
    return HttpResponse.json({
      content: [{ text: '{"valid": "json"}' }],
    });
  }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Tests overriden handlers por caso:

```typescript
test('handles anthropic 503', async () => {
  server.use(
    http.post('https://api.anthropic.com/v1/messages', () =>
      HttpResponse.error()
    )
  );

  // ...test fallback behavior
});
```

---

## 9. Testing de cosas específicas

### 9.1 Testing de LLM-backed features

Reto: outputs no determinísticos.

**Estrategia:**
- En unit tests: mockear LLM con outputs canónicos del fixture.
- En golden dataset evaluation (separado de tests): correr golden
  dataset semanal y comparar con baseline (no parte del CI).
- Validation Zod siempre testeable: dado output, ¿pasa o no?
- Comportamiento del wrapper (retry, fallback, cost tracking) sí
  testeable.

**No testear "que el LLM da buenas respuestas".** Eso es evaluation,
distinto de testing.

### 9.2 Testing de jobs nocturnos

Reto: jobs corren con cron real, lentos para test.

**Estrategia:**
- Inngest dev mode ejecuta cron triggers manualmente vía API.
- Test invoca el handler directamente con event mock.
- Verificar side effects en DB, eventos emitidos, llamadas a APIs.

```typescript
test('nightly roadmap update triggers AI for users matching rules', async () => {
  // Setup: 3 usuarios, 1 que matchea reglas, 2 que no
  await createTestUser({ id: 'u1', completed_level_today: true });
  await createTestUser({ id: 'u2', completed_level_today: false });
  await createTestUser({ id: 'u3', has_recent_error_pattern: true });

  // Mock LLM
  const mockLlm = mockAiGateway('nightly_roadmap_update', () => ({ /* ... */ }));

  // Run handler
  await runNightlyRoadmapJob();

  // u1 y u3 matchearon reglas, u2 no
  expect(mockLlm).toHaveBeenCalledTimes(2);
  expect(mockLlm.mock.calls.map(c => c[0].userId).sort()).toEqual(['u1', 'u3']);
});
```

### 9.3 Testing de webhooks

```typescript
test('stripe webhook verifies signature', async () => {
  const payload = JSON.stringify({ /* ... */ });
  const validSignature = stripeStub.signPayload(payload, validSecret);
  const invalidSignature = 'invalid-signature';

  // Valid
  const r1 = await fetch('/webhooks/stripe', {
    body: payload,
    headers: { 'stripe-signature': validSignature },
  });
  expect(r1.status).toBe(200);

  // Invalid
  const r2 = await fetch('/webhooks/stripe', {
    body: payload,
    headers: { 'stripe-signature': invalidSignature },
  });
  expect(r2.status).toBe(401);
});
```

### 9.4 Testing de validaciones Zod

```typescript
import { describe, test, expect } from 'vitest';
import { SubmitExerciseAttemptSchema } from './schemas';

describe('SubmitExerciseAttemptSchema', () => {
  test('valid payload parses', () => {
    const valid = { /* ... */ };
    expect(() => SubmitExerciseAttemptSchema.parse(valid)).not.toThrow();
  });

  test.each([
    { case: 'missing user_id', payload: { /* ... */ } },
    { case: 'invalid exercise_type', payload: { /* ... */ } },
    { case: 'audio payload with text exercise_type', payload: { /* ... */ } },
  ])('rejects $case', ({ payload }) => {
    expect(() => SubmitExerciseAttemptSchema.parse(payload)).toThrow();
  });
});
```

### 9.5 Testing de concurrencia

Para Sparks y mastery, donde concurrencia es crítica:

```typescript
test('two concurrent charges of same operation result in only one debit', async () => {
  const userId = await createTestUser({ sparks: 100 });

  // Disparar 2 charges del mismo operation_id en paralelo
  const [r1, r2] = await Promise.allSettled([
    sparks.chargeOperation(userId, 'op-123', 10),
    sparks.chargeOperation(userId, 'op-123', 10),
  ]);

  // Una debería suceder, la otra detectar duplicate
  const successes = [r1, r2].filter(r => r.status === 'fulfilled');
  expect(successes).toHaveLength(1);

  const balance = await sparks.getBalance(userId);
  expect(balance.total_sparks).toBe(90); // solo se cobró 10
});
```

---

## 10. Manual testing antes de release

Para suplir la ausencia de E2E, checklist manual antes de cada release
prod:

### 10.1 Smoke test móvil (15 min)

- [ ] Signup nuevo (Google).
- [ ] Onboarding completo.
- [ ] Hacer primer ejercicio (audio).
- [ ] Recibir push notification.
- [ ] Ver perfil con CEFR + scores.
- [ ] Acabar Sparks → ver paywall.
- [ ] Logout y login.

### 10.2 Smoke test web (10 min)

- [ ] Landing carga sin errores en console.
- [ ] Waitlist signup funciona.
- [ ] (Cuando exista) Login web + ejercicio web.

### 10.3 Smoke test backend (5 min)

- [ ] Webhook de Stripe sandbox → balance acreditado en sandbox user.
- [ ] Crash a propósito en algún Worker → Sentry captura.
- [ ] Job nocturno manual → verificar logs en Cloudflare.

---

## 11. Plan de implementación

### 11.1 Sprint 1

- Setup Vitest + tipos.
- Setup Postgres test (Docker compose).
- Setup MSW.
- Primeros unit tests de funciones puras.
- Setup gitleaks.

### 11.2 Sprint 2

- Integration test de auth flow.
- Integration test de sparks charge/refund.
- CI workflow básico.

### 11.3 Sprint 3+

- Tests por sistema según se implementan (§4).
- Coverage por sistema según target.
- Manual testing checklist (§10) documentado.

---

## 12. Métricas

- **CI pass rate:** % de PRs que pasan el CI primer intento. Target >
  90%.
- **CI tiempo total:** < 5 min.
- **Coverage de Sparks:** ≥ 95%.
- **Coverage de Pedagogical scoring:** ≥ 90%.
- **Tests flaky:** 0 en main. Si un test es flaky, **arreglarlo o
  borrarlo**.

---

## 13. Referencias internas

- [`data-and-events.md`](data-and-events.md) — eventos a verificar en
  tests.
- [`security-threat-model.md`](security-threat-model.md) — checklist
  de seguridad para tests.
- Cada sistema tiene su sección "tests obligatorios" en su documento.

---

## 14. Recursos externos

- Vitest: https://vitest.dev/
- MSW: https://mswjs.io/
- Cloudflare Workers testing: https://developers.cloudflare.com/workers/testing/
- gitleaks: https://github.com/gitleaks/gitleaks

---

*Documento vivo. Actualizar cuando se introduzca tooling nuevo, cambien
targets de coverage, o se identifiquen patrones nuevos.*

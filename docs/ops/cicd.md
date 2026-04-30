# CI/CD

> Pipelines de build, test, deploy. Estrategia de branching, releases,
> rollback. Stack: GitHub Actions + EAS Build + Cloudflare Workers
> deploy.

**Estado:** Diseño v1.0
**Última actualización:** 2026-04
**Owner:** —
**Audiencia primaria:** agente AI implementador.
**Alcance:** MVP y primer año.

---

## 0. Cómo leer este documento

- §1 cubre **estrategia de branching y releases**.
- §2 cubre **CI** (lo que corre en cada PR).
- §3 cubre **CD** (cómo se hacen deploys a stagings y prod).
- §4 cubre **mobile específicamente** (EAS Build, OTA updates,
  store submissions).
- §5 cubre **Workers / backend específicamente**.
- §6 cubre **migrations de DB**.
- §7 cubre **rollback** procedures.
- §8 cubre **secrets y variables de entorno**.
- §9 cubre **environments**: dev, staging, prod.

---

## 1. Branching y releases

### 1.1 Estrategia

**Trunk-based development con feature flags.**

```
main (siempre deployable)
  ↑
  ├── feature/<short-desc>     (cortas, < 1 semana)
  ├── hotfix/<short-desc>      (críticas, días)
  └── release/<vN.M.P>         (solo si hace falta cherry-pick)
```

**Reglas:**

- `main` es la única source of truth para producción.
- Features incompletas detrás de **feature flags** (no en branches
  largos).
- PRs cortos (< 500 líneas idealmente).
- No long-lived branches (excepto release/ ocasional).

### 1.2 Versionado

- **Backend (Workers):** sin versión semántica visible. Cada deploy es
  un commit hash.
- **Cliente móvil:** semver `vMAJOR.MINOR.PATCH`.
  - PATCH: bug fixes que se pueden hacer OTA.
  - MINOR: features que requieren rebuild + submit a stores.
  - MAJOR: breaking changes (raro en consumer app).
- **Web:** sin versión visible; deploy continuo.

### 1.3 Release cadence

- **Backend:** continuous (deploy a prod después de PR a main).
- **Cliente móvil:** sprints semanales o bisemanales para builds nuevos.
  OTA fixes en cuanto sea necesario.
- **Web:** continuous.

### 1.4 Convenciones de commits

Usar **conventional commits** lite (no estricto):

```
<type>: <description>

# Types
feat: nueva feature
fix: bug fix
refactor: cambio de código sin cambio de comportamiento
docs: cambios en documentación
test: agregar/cambiar tests
chore: tooling, deps, config
perf: mejora de performance
```

Mensajes en español o inglés (consistencia, default español dado
audience del proyecto).

---

## 2. CI (Continuous Integration)

### 2.1 Stack

- **GitHub Actions** para CI.
- Triggers: PR a `main`, push a `main`.

### 2.2 Pipeline en cada PR

```yaml
# .github/workflows/ci.yml (a crear)
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  # ──────────────────────────────────────────────────────
  # Backend: Workers + Postgres
  # ──────────────────────────────────────────────────────
  test-backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports: [5432:5432]
        options: >-
          --health-cmd "pg_isready"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm db:migrate    # corre migrations contra postgres de test
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test:unit -- --coverage
      - run: pnpm test:integration

  # ──────────────────────────────────────────────────────
  # Mobile: typecheck + unit tests
  # ──────────────────────────────────────────────────────
  test-mobile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter mobile typecheck
      - run: pnpm --filter mobile test

  # ──────────────────────────────────────────────────────
  # Security: secrets scan
  # ──────────────────────────────────────────────────────
  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 2.3 Required checks para merge

Branch protection en `main`:

- ✅ `test-backend` debe pasar.
- ✅ `test-mobile` debe pasar.
- ✅ `secrets-scan` debe pasar.
- ✅ Al menos 1 review aprobado (cuando haya equipo; pre-equipo:
  self-merge OK con CI verde).
- ✅ No fast-forward (squash merge preferido).

### 2.4 Tiempo target del CI

- `test-backend`: < 3 min.
- `test-mobile`: < 2 min.
- `secrets-scan`: < 30s.
- Total: < 5 min en paralelo.

Si excede, optimizar (paralelizar, partition de tests).

---

## 3. CD (Continuous Deployment)

### 3.1 Por target

| Target | Trigger | Tool |
|--------|---------|------|
| Workers staging | Push a `main` | `wrangler deploy --env staging` |
| Workers prod | Push a `main` después de smoke test staging | `wrangler deploy --env production` |
| Cliente móvil OTA (Expo) | Manual | `eas update --branch production` |
| Cliente móvil build (binarios) | Manual | `eas build` + submit |
| Web (Vercel) | Push a `main` | Vercel auto-deploy |
| DB migrations | Manual via script | `pnpm db:migrate:prod` |

### 3.2 Workflow de Workers deploy

```yaml
# .github/workflows/deploy-workers.yml
name: Deploy Workers

on:
  push:
    branches: [main]
    paths:
      - 'apps/workers/**'
      - '.github/workflows/deploy-workers.yml'

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter workers deploy:staging
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      - run: pnpm --filter workers test:smoke:staging

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production       # requires manual approval (GitHub env)
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter workers deploy:production
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      - run: pnpm --filter workers test:smoke:production
      - name: Notify
        if: success()
        run: echo "Deployed to prod, commit ${{ github.sha }}"
```

### 3.3 Smoke tests post-deploy

Después de cada deploy, correr smoke tests automáticos:

```typescript
// apps/workers/scripts/smoke-test.ts
const ENVIRONMENTS = {
  staging: 'https://api-staging.<dominio>',
  production: 'https://api.<dominio>',
};

async function smokeTest(env: 'staging' | 'production') {
  const baseUrl = ENVIRONMENTS[env];

  // 1. Health endpoint
  const health = await fetch(`${baseUrl}/health`);
  assert(health.status === 200, 'health check failed');

  // 2. Versioned endpoint
  const version = await fetch(`${baseUrl}/version`);
  const data = await version.json();
  assert(data.commit === process.env.GITHUB_SHA, 'version mismatch');

  // 3. Auth endpoint con token de test
  const auth = await fetch(`${baseUrl}/auth/sync-user`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${process.env.SMOKE_TEST_TOKEN}` },
    body: JSON.stringify({ /* ... */ }),
  });
  assert(auth.status === 200, 'auth endpoint broken');

  console.log('Smoke tests passed for', env);
}
```

Si fallan: rollback automático.

---

## 4. Mobile (Expo + EAS)

### 4.1 Build profiles

```jsonc
// eas.json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal",
      "channel": "preview"
    },
    "production": {
      "channel": "production",
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {
      "ios": { /* App Store Connect config */ },
      "android": { /* Play Console config */ }
    }
  }
}
```

### 4.2 Build workflow

| Tipo | Comando | Cuándo |
|------|---------|--------|
| Dev build local | `eas build --profile development --platform ios` | Setup inicial, debugging |
| Preview build | `eas build --profile preview` | TestFlight + Internal Testing |
| Production build | `eas build --profile production` | Release a stores |
| Submit a stores | `eas submit --platform ios --profile production` | Release |

### 4.3 OTA updates (EAS Update)

Para fixes que **no requieren** cambios nativos:

```bash
# Publicar OTA al canal production
eas update --branch production --message "Fix typo en onboarding"
```

**Qué SE PUEDE actualizar OTA:**
- Cambios de JS/TS.
- Strings de UI.
- Lógica de negocio del cliente.
- Assets bundleados (imágenes, JSONs).

**Qué NO se puede actualizar OTA (requiere rebuild + submit):**
- Cambios en `app.json` / `app.config.js` que afecten nativo.
- Nuevas librerías nativas.
- Cambios de permisos.
- Versión de Expo SDK.

### 4.4 Release process móvil

1. **Cut a release branch** (opcional, solo si necesitamos cherry-pick):
   ```bash
   git checkout -b release/v1.2.0
   ```
2. **Increment version** en `app.json`:
   ```json
   { "expo": { "version": "1.2.0", "ios": { "buildNumber": "..." } } }
   ```
3. **Build production**:
   ```bash
   eas build --profile production --platform all
   ```
4. **TestFlight / Internal Testing** primero:
   - Apple: TestFlight → testers internos → testers externos.
   - Google: Internal Testing → Closed Testing → Production.
5. **Submit a stores**:
   ```bash
   eas submit --platform all --profile production
   ```
6. **Wait for review** (Apple: 24-48h; Google: horas a 1 día).
7. **Phased rollout en Play Store** (1% → 10% → 50% → 100%):
   monitorear crash rate.

### 4.5 Hotfixes urgentes

**Si bug crítico en cliente** que se puede fixear OTA:
```bash
git checkout main
# fix
eas update --branch production --message "Hotfix: <issue>"
```

**Si bug crítico requiere rebuild:**
1. Hotfix branch desde `main`.
2. Bump PATCH version.
3. Build + expedited review (Apple permite request de "expedited
   review" para bugs críticos).

---

## 5. Workers / Backend

### 5.1 Wrangler config

```toml
# wrangler.toml
name = "english-spark-api"
main = "src/index.ts"
compatibility_date = "2026-01-01"

# Default env (staging)
[env.staging]
name = "english-spark-api-staging"
vars = { ENVIRONMENT = "staging" }

[[env.staging.d1_databases]]
binding = "DB"
database_name = "main-staging"
database_id = "..."

[env.production]
name = "english-spark-api-production"
vars = { ENVIRONMENT = "production" }

[[env.production.d1_databases]]
binding = "DB"
database_name = "main-production"
database_id = "..."
```

### 5.2 Comandos de deploy

```bash
# Deploy a staging
wrangler deploy --env staging

# Deploy a producción
wrangler deploy --env production

# Tail de logs en producción
wrangler tail --env production
```

### 5.3 Versioning de Workers

Cloudflare Workers mantiene **versions** automáticamente. Cada deploy
crea una nueva version. Para rollback:

```bash
# Listar versions
wrangler deployments list --env production

# Rollback a version específica
wrangler rollback <version-id> --env production
```

### 5.4 Gradual rollout (post-MVP)

Cloudflare Workers soporta percentage-based traffic routing:

```bash
# 10% del tráfico al nuevo deploy
wrangler deployments traffic <new-version-id>=10 <old-version-id>=90 --env production
```

Útil para canary deployment de cambios riesgosos. Requerimos métricas
para decidir promote/rollback.

---

## 6. Database migrations

### 6.1 Tooling

**Decisión:** usar **Drizzle Kit** o **kysely-codegen**. Migrations
escritas a mano en SQL puro (no auto-generadas para asegurar control).

### 6.2 Estructura

```
apps/db/migrations/
├── 0001_initial.sql
├── 0002_add_sparks_tables.sql
├── 0003_add_pedagogical.sql
├── ...
└── README.md
```

Cada archivo tiene `UP` y `DOWN`:

```sql
-- 0042_add_user_country_index.sql

-- UP
CREATE INDEX CONCURRENTLY idx_users_country ON users(country)
  WHERE deleted_at IS NULL;

-- DOWN
DROP INDEX IF EXISTS idx_users_country;
```

### 6.3 Reglas de migrations

- **Idempotentes:** `IF NOT EXISTS`, `IF EXISTS`.
- **Backwards compatible:** una migration no debe romper la versión
  anterior del código corriendo simultáneamente.
- **Pequeñas:** una operación por migration.
- **`CONCURRENTLY` para indexes** en tablas grandes.
- **`NOT VALID` + validate** para constraints en tablas grandes.

### 6.4 Patrón para columns nuevas

Para agregar una columna NOT NULL a una tabla con datos:

```sql
-- Migration N
ALTER TABLE users ADD COLUMN target_english_variant TEXT;

-- Migration N+1 (después de deploy de código que populates)
UPDATE users SET target_english_variant = 'neutral' WHERE target_english_variant IS NULL;

-- Migration N+2 (después de verificar)
ALTER TABLE users ALTER COLUMN target_english_variant SET NOT NULL;
ALTER TABLE users ALTER COLUMN target_english_variant SET DEFAULT 'neutral';
```

NUNCA `ALTER TABLE ADD COLUMN ... NOT NULL DEFAULT 'x'` directamente
en tablas grandes (lock prolongado).

### 6.5 Aplicar migrations

```bash
# Local (Postgres docker)
pnpm db:migrate

# Staging
pnpm db:migrate:staging

# Production (requiere confirmación)
pnpm db:migrate:production
```

Script de production:

```typescript
// scripts/migrate-prod.ts
import { confirm } from '@inquirer/prompts';

const isProd = process.env.NODE_ENV === 'production';

if (isProd) {
  const ok = await confirm({
    message: 'Estás a punto de ejecutar migrations en PRODUCCIÓN. Confirmar?',
  });
  if (!ok) process.exit(1);
}

// Aplicar migrations pending
```

### 6.6 Migration rollback

NO se hace rollback de migrations en producción. Si una migration causó
problema:

1. **Forward fix:** crear nueva migration que revierta.
2. NO ejecutar el `DOWN` de la migration original (puede ser
   incompatible con datos producidos por código nuevo).

Excepción: dentro de las primeras horas, si no hay datos nuevos creados
con el schema nuevo, se puede ejecutar DOWN. Pero default es forward
fix.

---

## 7. Rollback

### 7.1 Workers / Backend

```bash
# Rollback inmediato a version anterior
wrangler rollback --env production

# A version específica
wrangler rollback <version-id> --env production
```

Cloudflare ejecuta el rollback en segundos a nivel global.

### 7.2 Cliente móvil

**OTA update:** publicar update con código de la versión anterior:

```bash
# Volver a un commit previo
git checkout <commit-hash> -- apps/mobile
eas update --branch production --message "Rollback to <commit-hash>"
```

**Build nativo:** no se puede rollback en stores. Hay que hacer un
build nuevo con el fix.

### 7.3 Web

Vercel mantiene deployments anteriores; rollback con un click en su
dashboard o vía CLI:

```bash
vercel rollback <deployment-url>
```

### 7.4 DB migrations

Como discutido en §6.6: forward fix, no DOWN.

### 7.5 Decisiones de rollback

Criterios para rollback automático (post-deploy):

- Smoke test falla.
- Error rate sube > 5% sobre baseline en 5 min post-deploy.
- p95 latency sube > 100% sobre baseline en 5 min post-deploy.

Si pasa cualquiera: rollback automático + alerta a humano.

Para cliente móvil/web, no hay rollback automático; humano decide.

---

## 8. Secrets y env vars

### 8.1 Donde viven

| Tipo | Donde | Cómo |
|------|-------|------|
| Secrets de prod | Cloudflare Secrets | `wrangler secret put <NAME>` |
| Secrets de staging | Cloudflare Secrets (env separado) | `wrangler secret put <NAME> --env staging` |
| Secrets de CI | GitHub Actions secrets | UI de GitHub |
| Vars públicas | `wrangler.toml` `[vars]` | Commit OK |
| Local dev | `.dev.vars` | NUNCA commit |

### 8.2 Lista canónica

(De `cross-cutting/security-threat-model.md` §4.2.)

### 8.3 Rotación

(De `cross-cutting/security-threat-model.md` §4.3.)

Documentar cada rotación en `ops/secrets-rotation.log` (a crear) con
fecha, secret rotado, motivo.

### 8.4 Variables públicas (sin secret)

- `FIREBASE_PROJECT_ID`.
- `SUPABASE_URL` (URL pública del proyecto).
- `ENVIRONMENT` (`'staging'` o `'production'`).

Estos van en `wrangler.toml` `[env.X.vars]`.

---

## 9. Environments

### 9.1 Lista

| Env | Uso | URL |
|-----|-----|-----|
| Local dev | Desarrollo individual | `localhost` |
| Staging | Testing de integración antes de prod | `api-staging.<dominio>` |
| Production | Usuarios reales | `api.<dominio>` |

### 9.2 Diferencias entre staging y production

- **Staging:**
  - DB separada (puede ser Supabase test branch).
  - LLM provider keys de staging (cuotas más bajas).
  - Stripe en modo test.
  - RevenueCat sandbox.
  - FCM de proyecto Firebase staging.

- **Production:**
  - DB de producción.
  - Stripe en modo live.
  - RevenueCat producción.
  - FCM de proyecto Firebase prod.

### 9.3 Datos en staging

- **No** copiar datos de producción a staging (PII).
- Datos sintéticos generados por scripts de seed.
- Test users creados específicamente.

### 9.4 Manual testing en staging

Antes de cada release de cliente móvil:

1. Test con dev build apuntando a staging.
2. Smoke test checklist (`testing-strategy.md` §10).
3. Si pasa: build production y submit.

---

## 10. Plan de implementación

### 10.1 Sprint 0 — Setup foundational

- [ ] Crear `.github/workflows/ci.yml` con todos los checks.
- [ ] Branch protection en `main`.
- [ ] Cloudflare account + Workers env staging y production.
- [ ] Secrets puestos en Cloudflare.
- [ ] Vercel project para web (cuando exista).
- [ ] EAS account + `eas.json`.

### 10.2 Sprint 1+

- [ ] CI corre en cada PR.
- [ ] Workers staging deploys automáticos en push a main.
- [ ] Workers prod deploys con manual approval en GitHub Environments.
- [ ] Smoke tests post-deploy.
- [ ] Migration tooling configurado.

### 10.3 Pre-soft-launch

- [ ] EAS production builds funcionando.
- [ ] Submitting a TestFlight + Internal Testing.
- [ ] OTA updates testeados.
- [ ] Rollback procedure documentado y probado al menos una vez.
- [ ] Alertas de auto-rollback configuradas.

### 10.4 Post-soft-launch

- [ ] Phased rollout en Play Store (1/10/50/100).
- [ ] Canary deployments en Workers para cambios riesgosos.
- [ ] Schedule de release cadence formal (ej: martes y jueves para
  prod).

---

## 11. Métricas de CI/CD

| Métrica | Target |
|---------|--------|
| CI pass rate (primer intento) | > 90% |
| Tiempo total de CI | < 5 min |
| Deploys a prod por semana | 5–15 (señal de iteration saludable) |
| Rollbacks por mes | < 2 |
| Tiempo de rollback | < 5 min |
| Lead time (commit a producción) | < 30 min |

---

## 12. Referencias internas

- [`../cross-cutting/testing-strategy.md`](../cross-cutting/testing-strategy.md) — qué se testea en CI.
- [`../cross-cutting/security-threat-model.md`](../cross-cutting/security-threat-model.md) §4 — secrets management.
- [`incidents-runbook.md`](incidents-runbook.md) §4.8 — playbook de deploy roto.
- [`../architecture/platform-strategy.md`](../architecture/platform-strategy.md) — Expo + EAS.

---

## 13. Recursos externos

- Cloudflare Workers: https://developers.cloudflare.com/workers/
- EAS Build: https://docs.expo.dev/build/introduction/
- EAS Update: https://docs.expo.dev/eas-update/introduction/
- GitHub Actions: https://docs.github.com/en/actions
- Drizzle Kit: https://orm.drizzle.team/kit-docs/overview

---

*Documento vivo. Actualizar al introducir nuevas plataformas, cambios
de stack de deploy, o procesos refinados después de incidentes.*

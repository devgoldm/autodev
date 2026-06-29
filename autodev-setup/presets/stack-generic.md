# Stack preset — generic (fill-in template)

Use this for **any stack that isn't the Cloudflare reference** — and for **every existing project** (detect what's there and fill this in; never impose another stack). Copy it to `.claude/autodev/presets/stack-<name>.md`, answer each section, and set `config.stack.preset` to the copy.

The methodology and commands only need the stack to answer a fixed set of questions. Fill in every one; "n/a" is a valid answer (e.g. no flags, no staging) as long as it's deliberate and recorded.

## 1. Shape

- **Language / framework / runtime:** …
- **Where the app runs** (the platform/host): …
- **One-line architecture** (e.g. "Rails monolith + Postgres", "Next.js on Vercel + Neon", "Go API + React SPA on Fly.io"): …
- Record in `config.stack.describe`.

## 2. Commands the loops will run

These become the `<PLACEHOLDER>`s in the scheduled-task templates and the verify steps in the commands.

| Need | Command(s) | Notes |
|---|---|---|
| Install deps | … | |
| Run tests | `<TEST_CMD>` | must exit non-zero on failure |
| Typecheck / build | `<BUILD_CMD>` | omit if n/a |
| Lint | … | omit if n/a |
| Start the app locally | `<DEV_CMD>` on `<DEV_PORT>` | pick a non-default port so the loop never collides with a human dev server |
| Seed local test data | `<SEED_CMD>` | omit if no auth/data |

## 3. Deploy pipeline

- **What turns a merge into a deployment** (CI on merge? a deploy command? a PaaS git integration?): …
- **Production** = merge to `main` → … ; record the check/run name that confirms a prod deploy completed (`<DEPLOY_CHECK>`).
- **Previews/staging** = merge to `develop` / PRs → … ; record how to confirm the integration deploy is green (`<STAGING_HEALTH>`) and the staging URL (`<STAGING_URL>`).
- Set `config.deploy.mechanism`. If there's no hosted CI, the commands fall back to running the deploy command directly from `/release` — record that and note it in an ADR.

## 4. Preview/staging isolation (required)

- **How non-production gets its own data stores + test-mode credentials** (separate database/branch, a staging project, env-scoped resources): …
- The hard rule, however you achieve it: previews **never** touch production data or live third-party keys. New resources are added to both environments in the same PR; migrations run on the non-production store first.

## 5. Feature flags

- **Mechanism** (`config.flags.mechanism`): a flag service (LaunchDarkly, Flagsmith, Unleash…), a homegrown KV/DB table, an env-var/config gate, or `none`.
- **How the agent creates/flips/deletes a flag itself** (API/SDK + where the credential lives): …
- **Staging-on/prod-off**: is it enforced in the flag tool, or in code via an `APP_ENV !== "production"` predicate? Record in `config.flags.stagingOnProdOff`.
- If `none`: rely on the frozen release branch + staging gate for safety (see METHODOLOGY § Branching & flags). Adding even a trivial flag module is strongly recommended.

## 6. Observability source

- **Where production logs/errors/exceptions are read from** (a logging platform, an APM, `kubectl logs`, a hosted dashboard, an MCP server): … Record in `config.prodTest.observability`.
- This is what `/bug-hunt` and the `/release` close-watch query. It must be reachable read-only from the loop host.

## 7. Test account / auth (if the app has auth)

- **Seed script** that creates/updates a user from `AUTODEV_TEST_EMAIL` / `AUTODEV_TEST_PASSWORD`: …
- **Dev login path** that can be enabled on staging (env-gated to non-production) and skips CAPTCHA/OAuth/email: …
- Record env var names in `config.prodTest` and `config.staging.devLogin`.

## 8. Stack-specific skills / docs

- Any official or community skills for this stack worth loading (recommendation only): …
- Where to retrieve current platform docs before writing platform code: …

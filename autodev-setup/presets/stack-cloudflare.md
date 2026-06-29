# Stack preset — Cloudflare

The worked **reference stack**: a single Cloudflare Worker serving a Vite/React SPA + Hono API, with Cloudflare data stores, Workers Builds deploys, and Cloudflare Flagship feature flags. This is one option, not a requirement — see [stack-generic.md](stack-generic.md) for any other stack, and always **respect an existing project's stack** rather than migrating to this one.

Record `config.stack.preset` = this file and `config.stack.describe` = `"worker+hono+vite-react-spa"` (or the project's actual shape).

## Prerequisites (Cloudflare)

- **`wrangler`** authenticated. Prefer a **least-privilege scoped API token** over `wrangler login` (full account OAuth). Verify with `CLOUDFLARE_API_TOKEN=… wrangler whoami`. See § Cloudflare API token below and [cloudflare-token.md](cloudflare-token.md); walk the owner through minting it during setup.
- **Cloudflare skills** — install the relevant skills from https://github.com/cloudflare/skills into the **project's** `.claude/skills/` so any collaborator's agent has them: at minimum `cloudflare`, `wrangler`, `workers-best-practices`; add `durable-objects`, `agents-sdk`, etc. as the app needs them. Skip any already installed globally, but prefer project-local copies for shareability.
- **Cloudflare MCP servers** — add to the project's `.mcp.json` so sessions can monitor production, watch builds, and manage resources:

```json
{
  "mcpServers": {
    "cloudflare-observability": { "type": "sse", "url": "https://observability.mcp.cloudflare.com/sse" },
    "cloudflare-builds":        { "type": "sse", "url": "https://builds.mcp.cloudflare.com/sse" },
    "cloudflare-bindings":      { "type": "sse", "url": "https://bindings.mcp.cloudflare.com/sse" },
    "cloudflare-docs":          { "type": "sse", "url": "https://docs.mcp.cloudflare.com/sse" }
  }
}
```

Merge with any existing `.mcp.json` and skip servers the user already has (e.g. via the Cloudflare Claude Code plugin). Who uses what:
  - **observability** — `/release` close watch and `/bug-hunt`: query production Worker logs for errors. **This is the `observability source`** for `config.prodTest`.
  - **builds** — `/build-next` and `/release`: after any push/merge, check the Workers Builds status and read build logs on failure; a release is not deployed until the main build reports success.
  - **bindings** — setup and `/build-next`: list/verify/provision Workers resources (D1, KV, R2) — especially confirming `-preview` resources exist and new bindings landed in both environments.
  - **docs** — all loops: retrieve current Cloudflare docs before writing platform code.

  These authenticate via OAuth on first use — do that while the owner is present.

## New project scaffold

Load the locally installed `cloudflare` / `wrangler` skills first and follow current docs (don't trust memorized CLI flags). Target shape:

- Single Worker: Hono API + static assets serving a Vite/React SPA.
- A component library of the owner's choice (Cloudflare's **Kumo UI** is a reasonable default — check current install instructions via the Cloudflare docs MCP).
- **Vitest** with `@cloudflare/vitest-pool-workers` for Worker tests; plain Vitest + Testing Library for the SPA.
- `wrangler.jsonc` with `observability: { enabled: true }`.
- D1 / KV / R2 / Queues / DOs: add bindings only when a feature needs them, via ADR — always to both the production and `preview` environments (see Preview isolation).
- `preview_urls: true` in `wrangler.jsonc` (the default) plus a `preview` Wrangler environment from day one, even if it starts empty.
- A `src/flags.ts` module wrapping **Cloudflare Flagship** (see § Flags below).
- TypeScript strict, ESLint + Prettier (or Biome) with sensible defaults.

If there's auth, scaffold the **test account** path: a seed script that creates/updates a user from `AUTODEV_TEST_EMAIL` / `AUTODEV_TEST_PASSWORD`, stored in `.env` (gitignored) and as Worker secrets if server-side seeding is needed. Record the env var names in `config.prodTest`.

## Flags — Cloudflare Flagship

`config.flags.mechanism` = `"flagship"`. Retrieve current Flagship docs before wiring (binding setup, OpenFeature option). The loops **manage flags themselves via the Flagship REST API** (create/flip/delete — owner standing order, not AskUserQuestion; token in the host's gitignored env).

- `src/flags.ts`: typed `isEnabled(env, "flag_name")` over the `FLAGS` binding; evaluation failure or missing flag = **off**.
- Set up the binding for both wrangler environments per the Flagship docs.
- During setup, create one `example` flag via the REST API and verify it evaluates end-to-end.
- Some projects also enforce staging-on/prod-off **in code** (an `APP_ENV !== "production"` predicate), so a new flag only needs to *exist* off to be prod-dark while staging runs it on.
- If Flagship is unavailable on the account, fall back to a KV-backed flags module and record an ADR (`config.flags.mechanism: "kv"`).

## Deploy pipeline — Workers Builds

`config.deploy.mechanism` = `"ci-on-merge"` (Workers Builds). Connect the repo in the Cloudflare dashboard (Workers & Pages → the Worker → Settings → Builds):

- **Production branch** = `main`, deploy command `npx wrangler deploy` (production environment).
- **Non-production branch builds: enabled** (not automatic — see the build-branches docs), targeting the isolated `preview` environment. Branch-aware deploy command so `develop` gets a stable deployment and feature branches get per-branch preview URLs:

  ```bash
  if [ "$WORKERS_CI_BRANCH" = "develop" ]; then npx wrangler deploy --env preview; else npx wrangler versions upload --env preview; fi
  ```

This needs the owner's dashboard session — walk them through it and verify with a test push to a non-production branch. If Workers Builds can't be used, fall back to documented `wrangler deploy` / `wrangler deploy --env preview` from `/release` and `/build-next` and record an ADR.

## Preview isolation (required)

Workers does **not** natively support different bindings for production vs non-production builds — a preview deployed from the production config would talk to production data. The documented fix is a Wrangler environment:

- Define a `preview` environment in `wrangler.jsonc` (`"env": { "preview": { ... } }`) that mirrors every production binding but points at **separate resources**: its own D1 database, KV namespaces, R2 buckets, queues — provision them with `-preview`-suffixed names (`wrangler d1 create <db>-preview`, etc.). Top-level config stays production.
- Never bind a production data store into the preview environment. When a feature adds a binding, add it to **both** environments in the same PR (preview resource first).
- Apply D1 migrations to both databases as part of any schema change (`wrangler d1 migrations apply <db> --env preview`, then production at release).
- Secrets are per-environment: `wrangler secret put --env preview`. Third-party services use non-live credentials in preview — Stripe **test mode** keys, never live keys.
- Seed the preview database with the test account so the loops can exercise the develop deployment.
- **Durable Objects work fine in the preview environment** (it's a separate Worker, so its own DO namespaces). The only limitation is the *per-branch preview URL* feature (`wrangler versions upload`): all versions of a Worker share DO instances, so Cloudflare doesn't generate preview URLs for DO-using Workers. For DO apps, feature branches verify locally + on the stable preview deployment after merge. Per-branch isolation is possible by deploying a distinct Worker per branch (`wrangler deploy --env preview --name <app>-preview-$WORKERS_CI_BRANCH`) but that's opt-in (Worker sprawl + per-branch stores).

## Third-party services

**Cloudflare-first**: prefer Cloudflare services (Workers, D1, KV, R2, Queues, Durable Objects, Workers AI, Email Service, Flagship) before any third-party service. The one standing exception in this preset is **Stripe** for payments (test-mode keys in preview, live keys only in production). This is a preset opinion, not a methodology rule — drop or change it freely.

## Cloudflare API token (least privilege)

Don't authenticate Wrangler with `wrangler login` (full account OAuth) or a Global API Key. Mint a **scoped API token** so the project — and CI — can only touch the resource types it uses. Wrangler reads `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` from the environment and skips OAuth; this is also how a CI deploy authenticates.

**Honest limitation:** tokens scope to an **account + permission groups**, not a single Worker. State this to the owner.

**Split credentials by power (two token roles).** When deploys run in CI (the default), only the deploying machine needs Edit:
- **CI / deploy token** — **Edit** scopes, lives **only** in CI secrets, used by the deploy workflows.
- **Local token** — **read-only**, lives in the developer's shell / gitignored env. It cannot deploy/push/migrate. Default for the owner's machine unless they explicitly deploy locally. This is also the **read-only deploy-monitoring credential** the loops use.

The full permission-group → binding mapping, placement, and verification recipe is in the copy-into-project artifact [cloudflare-token.md](cloudflare-token.md). During setup: copy it to `.claude/autodev/cloudflare-token.md`, fill in the account ID, trim the table to the project's `wrangler.jsonc` bindings, have the owner mint + place the tokens, verify with `wrangler whoami`, and record `config.cloudflare` (account ID, `auth: "scoped-api-token"`, `tokenRecipe` path).

Note: the `cloudflare-observability` MCP uses its own OAuth and does **not** use these tokens, so the deploy credential stays out of the monitoring path.

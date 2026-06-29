# Cloudflare API token (least privilege)

How to mint scoped Cloudflare API tokens for this project instead of giving Wrangler full account
access via `wrangler login`. Wrangler reads `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` from
the environment and skips the OAuth flow entirely.

> Setup note (delete after filling in): replace `<ACCOUNT_ID>` below, and trim the permission
> tables to the resource types this project's `wrangler.jsonc` actually binds. The mapping in
> stack-cloudflare.md § Cloudflare API token tells you which permission group each binding needs.

- **Account ID:** `<ACCOUNT_ID>` (not a secret — safe to commit/share).
- **A token is account-wide per resource type.** Cloudflare tokens scope to an account +
  permission groups; they **cannot** be restricted to a single Worker script. Least privilege
  means "only the resource types this project uses, only this account, only the access level the
  holder actually needs".

## Two token roles

If deploy/push/migrate runs in CI (the autodev default), split credentials by power:

| Role | Lives in | Access | Used by |
|---|---|---|---|
| **CI / deploy** | GitHub repo secrets | **Edit** (deploy + migrate) | the deploy workflow(s) |
| **Local** | your shell / gitignored env | **Read-only** | interactive Wrangler / inspection |

A local read-only token deliberately **cannot deploy, push, or run migrations** — CI owns all
writes. (If a project deploys from a developer machine instead, that machine needs the Edit set.)

## Local token — read-only

Scope to the project's account. All **Read**, no Edit by design.

**Mandatory:** Account Settings → Read (Wrangler resolves the account; `whoami` + reads work).

**Logs & observability** — add the ones the project actually uses:

| Permission group | Scope | Why |
|---|---|---|
| Workers Observability | Read | Stored Workers Logs + Query Builder / observability dashboard + API |
| Workers Tail | Read | `wrangler tail` live tail (distinct from stored logs) |
| Account Analytics | Read | Analytics Engine datasets + GraphQL analytics |
| AI Gateway | Read | View AI Gateway request logs (only if the app uses AI Gateway) |

**Inspect resources (optional, per binding):** Workers Scripts / D1 / R2 / KV / Queues /
Vectorize → Read, and Workers AI → Read only for CLI AI calls.

Truly minimal is just **Account Settings → Read** (+ **Workers Scripts → Read**). The
`cloudflare-observability` MCP (separate OAuth) can also surface prod logs; these grants let plain
Wrangler/CLI/API see them too. No Edit/Write row by design.

## CI / deploy token — Edit

Scope to the project's account. Map each binding to its Edit permission group:

| Permission group | Scope |
|---|---|
| Workers Scripts | Edit |
| Account Settings | Read |
| D1 / R2 / KV / Queues / Vectorize | Edit _(per binding)_ |
| Workers AI | Read _(optional — CLI AI only)_ |

`durable_objects`, `workflows`, `services`, `assets` ship as part of the Worker, so
`Workers Scripts → Edit` already covers them.

## Minting (dashboard)

Be logged in as a user who is a **member of the project's account** (not a personal account).

1. **https://dash.cloudflare.com/profile/api-tokens** → **Create Token → Create Custom Token**.
2. Add the permission rows for the role you're minting (read-only for local).
3. **Account Resources → Include → the project's account** (not "All accounts").
4. Recommended: a **TTL/expiry** (rotate ~90 days) and an optional **client-IP allowlist**.
5. **Create**, copy the value once.

## Placement

- **Local token:** export in your shell or a **gitignored** `.env.local` you `source`. **Never**
  `.dev.vars` — that injects into the running Worker's runtime env, leaking the token into the app.
- **CI / deploy token:** GitHub repo secrets only.

  ```bash
  export CLOUDFLARE_API_TOKEN="<token>"
  export CLOUDFLARE_ACCOUNT_ID="<ACCOUNT_ID>"
  # CI:
  gh secret set CLOUDFLARE_API_TOKEN
  gh secret set CLOUDFLARE_ACCOUNT_ID --body <ACCOUNT_ID>
  ```

## Verify

```bash
CLOUDFLARE_API_TOKEN="<token>" wrangler whoami
```

Shows the token, account, and scopes. For a local token, confirm every scope is `read`.

## Separation from production monitoring

The `cloudflare-observability` MCP authenticates with its **own OAuth** (read-only logs) and does
**not** use these tokens — so neither is in the monitoring path.

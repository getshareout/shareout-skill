---
name: "shareout-deploy"
version: "1.0.0"
updated_at: "2026-07-25T18:00:00Z"
description: "Install and deploy a self-hosted ShareOut instance on the user's Cloudflare account. Use when the user wants to install ShareOut, self-host, Deploy to Cloudflare, set up their Artifact World, configure domain/DNS/secrets for ShareOut, or get a fresh instance running in minutes."
---

# ShareOut Deploy Skill

**Goal:** ShareOut running on **their** Cloudflare account in minutes. You drive the
boring parts; they only handle Cloudflare login, plan choice, and DNS if they want a
custom domain.

This skill is for **install / deploy / configure**. After the instance is up, switch to
the product skill ([../SKILL.md](../SKILL.md)) and set `origin` to their worker URL.

**Where this file lives:** only in `getshareout/shareout` under `skills/ShareOutSkill/deploy/`.
That monorepo path is the source of truth ([../../README.md](../../README.md)). The
standalone `getshareout/shareout-skill` repo is a mirror for agents that only install
the skill — still edit deploy docs here, then mirror on release.

---

## Agent success criteria (do not stop early)

You are done when **all** of these are true:

1. Worker is deployed and `{ORIGIN}/health` returns `200` with **`"schema": "ready"`
   and no `warnings`**. Anything else means the instance is not finished — read the
   warning and fix that first.
2. `SESSION_SECRET` is set; `SHAREOUT_BASE_URL` equals `{ORIGIN}` (no trailing slash).
3. First admin can sign in (password; Google and email codes optional).
4. `npm run smoke:hello` (or equivalent publish) returns a live artifact URL.
5. You saved credentials for later agent use:

```json
// ~/.shareout/credentials
{
  "token": "so_…",
  "origin": "https://shareout.<account>.workers.dev"
}
```

Hand the user: **ORIGIN**, how to open Home, how to mint tokens, and that their agents
should load skill from `{ORIGIN}/v1/skill` or this repo with that origin.

---

## Load Cloudflare companion skills first

ShareOut is a Cloudflare Workers app. The **only hard part** is Cloudflare account setup.
Load these agent skills when available (Claude Code / Cursor skill directories, or tell the
user to install them):

| Skill | When |
|-------|------|
| **wrangler** | Every deploy: login, secrets, D1/R2/KV create, `wrangler deploy` |
| **cloudflare** | Platform overview, Workers, D1, R2, KV, custom domains |
| **workers-best-practices** | Reviewing `wrangler.toml`, bindings, secrets, observability |
| **durable-objects** | DO migrations, Workers Paid, realtime/tables issues |
| **cloudflare-email-service** | Optional: emailed one-time codes and lifecycle mail |

**Install hints (if missing):**

```text
npx skills add cloudflare/skills   # community pack often includes wrangler + workers skills
# or clone / enable: cloudflare, wrangler, workers-best-practices, durable-objects
```

**Official docs (always authoritative over model memory):**

| Topic | URL |
|-------|-----|
| Workers get started | https://developers.cloudflare.com/workers/get-started/guide/ |
| Wrangler | https://developers.cloudflare.com/workers/wrangler/ |
| Deploy to Cloudflare button | https://developers.cloudflare.com/workers/platform/deploy-button/ |
| Custom domains | https://developers.cloudflare.com/workers/configuration/routing/custom-domains/ |
| D1 | https://developers.cloudflare.com/d1/ |
| R2 | https://developers.cloudflare.com/r2/ |
| Durable Objects | https://developers.cloudflare.com/durable-objects/ |
| Workers Paid / pricing | https://developers.cloudflare.com/workers/platform/pricing/ |
| Email Workers | https://developers.cloudflare.com/email-routing/email-workers/ |

Full ShareOut-specific CF checklist: [cloudflare.md](cloudflare.md).

---

## Two install paths — pick one

| Path | Best when | Agent effort |
|------|-----------|--------------|
| **A. Deploy button** | Human has browser + GitHub connected to Cloudflare | Guide human; then finish secrets + smoke |
| **B. CLI (agent-driven)** | User said "you deploy it" / headless / CI | You run wrangler end-to-end |

Always confirm: **Workers Paid** is required for Durable Objects (tables, realtime, chat).
Free Workers alone is not enough for full product.

---

## Path A — Deploy to Cloudflare (human + agent)

1. Open the repo README Deploy button, or:

   ```
   https://deploy.workers.cloudflare.com/?url=https://github.com/getshareout/shareout/tree/main/shareout-app
   ```

2. Human completes Cloudflare OAuth and resource create (D1/R2/KV/DO by name from
   `wrangler.toml`).
3. Note the workers.dev URL → that is provisional `ORIGIN`.
4. Agent continues from [Post-deploy](#post-deploy-always).

---

## Path B — CLI deploy (agent-driven)

### B0. Preconditions

```bash
node -v          # >= 24
npx wrangler --version   # v4.x
npx wrangler whoami      # must be logged in; else: npx wrangler login
```

If not logged in, open the browser flow for the user and wait.

### B1. Clone and install

```bash
git clone https://github.com/getshareout/shareout.git
cd shareout/shareout-app
npm ci
```

### B2. Create Cloudflare resources

Use names matching the OSS template (`shareout-app/wrangler.toml`):

```bash
npx wrangler d1 create shareout-db
# → copy database_id into wrangler.toml [[d1_databases]] database_id

npx wrangler r2 bucket create shareout-artifacts

npx wrangler kv namespace create SLUGS
npx wrangler kv namespace create RATE_LIMIT_KV
npx wrangler kv namespace create PROXY_CACHE
# → copy each id into the matching [[kv_namespaces]] id
```

Durable Object classes are declared in `wrangler.toml` and created on first deploy —
no manual create step.

### B3. Set public origin vars

In `wrangler.toml` `[vars]` (or dashboard after first deploy):

```toml
SHAREOUT_BASE_URL = "https://shareout.<your-subdomain>.workers.dev"  # update after first deploy if unknown
# Optional bootstrap:
# SETUP_ADMIN_EMAIL = "you@company.com"
```

If the exact workers.dev hostname is unknown until first deploy: deploy once with a
placeholder, read the deployed URL from wrangler output, set `SHAREOUT_BASE_URL`, redeploy.

### B4. Secret

```bash
openssl rand -hex 32 | npx wrangler secret put SESSION_SECRET
```

### B5. Deploy + migrate

```bash
npm run deploy
# runs D1 migrations remote, then wrangler deploy (predeploy builds SDK/editor bundles)
```

First deploy can take several minutes (bundle builds). Do not kill it early.

### B6. Continue [Post-deploy](#post-deploy-always)

---

## Post-deploy (always)

### 1. Confirm health

```bash
export ORIGIN="https://shareout.<account>.workers.dev"   # or custom domain
curl -sS "$ORIGIN/health"
```

Expected: `{"status":"ok", ... ,"schema":"ready"}` with **no `warnings` array**.

| In the response | Means | Fix |
|-----------------|-------|-----|
| `"schema":"missing"` | D1 exists but has no tables | `npx wrangler d1 migrations apply DB --remote` from `shareout-app/` |
| `SHAREOUT_BASE_URL is unset` | Agent-facing URLs point at the hosted instance | Set the var, redeploy |

The **Deploy to Cloudflare button does not run migrations** — it creates the D1
database and stops. If the instance was deployed that way, applying the schema is
your first step, not an optional one.

### 1b. Read the instance config

Once an admin exists, one call tells you everything still missing and what each gap
costs — check it before declaring the install finished:

```bash
curl -sS "$ORIGIN/v1/admin/instance" -H "Authorization: Bearer $SHAREOUT_TOKEN" | jq .gaps
```

A **personal** token (`so_…`) belonging to an instance admin works here, so this step no
longer needs a browser session. Workspace agent tokens (`sot_…`) are refused on purpose.

Each entry is `{setting, disables, fix}`. **An empty array is the goal.** The common
ones on a fresh instance:

| Gap | Why it matters |
|-----|----------------|
| `VERCEL_AI_GATEWAY or OPENAI_API_KEY` | **AI is off until one is set** — Crew, home assistant, in-artifact chat, editor AI, knowledge. It degrades silently, so nothing will tell the user unless you do. |
| `CREDENTIALS_KEY` | Per-workspace AI keys and connector credentials cannot be stored |
| `EMAIL binding` | Invites and one-time codes reach the Worker log, not an inbox |
| `INSTANCE_ADMIN_EMAILS` | No stable owner — the earliest user keeps being treated as admin |

Do not silently accept gaps. Report them to the user with the `fix` line, and offer to
apply the ones that are secrets you can set for them.

Working through the gaps, changing feature flags, or standing up workspaces and admins
for other people: **[configure.md](configure.md)**.

### 2. Align `SHAREOUT_BASE_URL`

Must match the public origin users and agents will call (no trailing slash). Redeploy or
set via dashboard Variables if wrong.

### 3. First admin (`/setup`)

1. Open `$ORIGIN` → empty DB redirects to `/setup`.
2. Create the admin there with an **email and password**. No Email binding and no
   OAuth client needed — this is the whole point of the password path.
3. Headless? Same thing over the API, while the instance still has no users:

   ```bash
   curl -sS -X POST "$ORIGIN/v1/auth/password/register" \
     -H 'Content-Type: application/json' \
     -d '{"email":"you@example.com","password":"<at least 12 characters>"}'
   ```

   It returns `404` once any user exists, so it cannot be used twice.
4. Optional: set `SETUP_ADMIN_EMAIL` so only that email can claim first admin.

### 4. Mint API token

In-product: avatar / Settings → API token → copy `so_…` once.

Or after session exists, use self-serve API (see product skill [../auth.md](../auth.md)).

### 5. Smoke publish

```bash
export SHAREOUT_ORIGIN="$ORIGIN"
export SHAREOUT_TOKEN="so_…"
npm run smoke:hello
# expect 201 + live URL containing "Hello ShareOut"
```

### 6. Persist agent credentials

```bash
mkdir -p ~/.shareout
cat > ~/.shareout/credentials <<EOF
{
  "token": "$SHAREOUT_TOKEN",
  "origin": "$SHAREOUT_ORIGIN"
}
EOF
chmod 600 ~/.shareout/credentials
```

### 7. Point agents at this instance

- **Skill zip from live instance (preferred):** `GET $ORIGIN/v1/skill` (version check
  `$ORIGIN/v1/skill/version`). The Worker ships its own copy and rewrites every URL
  inside it to `SHAREOUT_BASE_URL` first, so the skill an agent reads already names
  this instance — nothing to substitute by hand.
- **Or** use this repo's `skills/ShareOutSkill/` and treat every `https://shareout.site`
  URL in docs as **`$ORIGIN`**.

> **Verify before handing it over:** `curl -s $ORIGIN/health` must not contain a
> `warnings` entry. A `SHAREOUT_BASE_URL is unset` warning means the skill still
> points at the hosted instance, and anything an agent publishes lands there.

Product usage skill: [../SKILL.md](../SKILL.md) — section **Instance origin**.

---

## Optional polish (after hello works)

Do these only if the user asked — order of value:

| Want | Action | Details |
|------|--------|---------|
| Company domain | Custom domain on Worker + update `SHAREOUT_BASE_URL` | [cloudflare.md](cloudflare.md#custom-domain) |
| Workspace subdomains | Wildcard DNS `*.company.com` → Worker | [cloudflare.md](cloudflare.md#workspace-subdomains) |
| Google login | `GOOGLE_CLIENT_ID` / `SECRET`; callback `$ORIGIN/auth/callback` | [secrets](../../../docs-site/src/content/docs/self-host/secrets.md) |
| Emailed one-time codes | Email binding / Email Routing | skill **cloudflare-email-service** |
| Split artifact CDN | Second host + `ARTIFACT_ORIGIN` | optional hardening |
| Docs site | Deploy `docs-site/` separately; set `DOCS_HOST` + `DOCS_ORIGIN` | optional |

Full ops: [Self-host overview](../../../docs-site/src/content/docs/self-host/overview.md).

---

## Failure quick-fix

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Deploy fails on DO / class | Workers Free plan | Upgrade to **Workers Paid** |
| `database_id` / KV placeholder errors | IDs still `0000…` | Create resources; paste real IDs into `wrangler.toml` |
| One-time code never arrives | No Email binding | Use password sign-in, or `wrangler tail` to read the code from logs |
| Links go to wrong host | `SHAREOUT_BASE_URL` stale | Set to public origin; redeploy |
| Publish 401 | Bad/missing token | Mint new token; fix `~/.shareout/credentials` |
| `npm ci` peer errors | Node &lt; 24 or stale lock | Use Node 24 (`.nvmrc`); clean install |
| Agent publishes to shareout.site by mistake | Origin not set | Always use `origin` in credentials / `SHAREOUT_ORIGIN` |

---

## What you must not do

- Do not invent founder account IDs, zones, or routes into `wrangler.toml`.
- Do not commit `.dev.vars`, secrets, or `so_` tokens.
- Do not tell the user a feature needs a plan — there are none, and every feature is on.
- Do not skip smoke publish; "deploy succeeded" is not "product works".
- Do not leave `SHAREOUT_BASE_URL` pointing at `shareout.workers.dev` template placeholder.

---

## After deploy — handoff message (template)

```text
Your ShareOut is live at: {ORIGIN}
Admin: open {ORIGIN} → signed in as first admin
API token: saved to ~/.shareout/credentials (origin + token)
Smoke: hello artifact published

Next for your coding agent:
  1. Load skills/ShareOutSkill/SKILL.md (or GET {ORIGIN}/v1/skill)
  2. Use origin {ORIGIN} for all API/SDK calls
  3. Publish with Authorization: Bearer so_…

Optional: custom domain — see skills/ShareOutSkill/deploy/cloudflare.md
```

# /hint-marketplace-retrofit — Add Marketplace Support to an Existing App

Audits the partner's existing application against the Hint marketplace contract and generates the minimum diff needed to make it embeddable in Hint. Language-agnostic — the skill produces snippets in whatever stack the partner is using (Node.js, Python, Ruby, Go, PHP, etc.).

## When to use this

The partner already has a working web app and wants to surface it inside Hint. Two flavors:

- **Hosted by Hint**: the partner's code lives in their repo; Hint deploys it on Hint-managed infrastructure. After retrofit, push via `POST /partner/app/revisions`. Hosted mode runs `node server.js` and currently supports Node.js only — other stacks must go Self-hosted.
- **Self-hosted**: the partner already deploys the app themselves (Vercel, AWS, on-prem). After retrofit, the partner redeploys on their own infra and registers the URLs with Hint. Any stack works.

If the partner is starting from a blank repo, use [`create-app`](https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/create-app/SKILL.md) instead — that skill scaffolds from a known-good template.

## Required reading

Before doing anything, fetch and read these shared fragments — every retrofit decision is grounded in them:

1. https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/_common/api-conventions.md — hosts, auth, pagination, reserved env vars
2. https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/_common/marketplace-contract.md — the three required routes, signature verification, smoke test
3. https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/_common/provider-api.md — Provider API endpoints, practice-scoped auth, JS SDK

## Platform URLs

Set `$HINT_API_URL=https://api.hint.com` for both sandbox and live work (Partner Portal at `https://app.hint.com`). Full conventions: [`_common/api-conventions.md`](../_common/api-conventions.md).

**IMPORTANT**: Never reference underlying infrastructure providers or code hosts to the partner. Everything is "Hint" from their perspective.

## Step 1: Gather Context

Ask the partner three things:

1. **Where's the codebase?** Path to a local working directory, a git URL, or a description of where you can read the source.
2. **Hosting mode?**
   - **Hosted by Hint** — the partner wants Hint to deploy this app. They'll need a sandbox API key.
   - **Self-hosted** — the partner already deploys the app and just wants to register it with Hint. They'll need a sandbox API key and the deployed URL.
3. **Sandbox API key** (`sbx-...`). If they don't have one, walk them through creating one (see `create-app`'s sandbox setup section).

Verify the API key:

```bash
curl -s "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY"
```

The response's `product.type` must be `app`. If it isn't, stop and tell the partner to contact [devsupport@hint.com](mailto:devsupport@hint.com) to get the partner type updated.

## Step 2: Inventory the Existing App

Detect the stack from the repository contents:

| File | Stack |
|---|---|
| `package.json` | Node.js (further differentiate Express / Fastify / Next.js / Hono / plain `http`) |
| `requirements.txt` / `pyproject.toml` | Python (Flask / FastAPI / Django) |
| `Gemfile` | Ruby (Rails / Sinatra) |
| `go.mod` | Go (net/http / chi / gin) |
| `composer.json` | PHP (Laravel / Symfony / plain) |
| `Cargo.toml` | Rust (axum / actix) |

If multiple files are present, use the one that owns the HTTP entry point. Ask the partner if it's not obvious.

Then audit what's already there. For each of the three marketplace-contract routes, check whether a handler exists:

```bash
# Replace <route_pattern> with whatever the framework uses to register a route
grep -rEn "(post|POST|@app\.post|router\.post|app\.post|Post\(|HandleFunc).*['\"\`]/hint/handshake" <repo>
grep -rEn "(post|POST|@app\.post|router\.post|app\.post|Post\(|HandleFunc).*['\"\`]/hint/connect" <repo>
grep -rEn "(get|GET|@app\.get|router\.get|app\.get|Get\(|HandleFunc).*['\"\`]/hint/(core_page|clinical_interaction|settings)" <repo>
```

Build a status table for the partner:

```
Marketplace contract audit for <repo>:

  Route                                Status
  POST /hint/handshake                 ✓ present in src/server.js:42
  POST /hint/connect/:code             ✗ MISSING
  GET  /hint/<anchor_type>             ✗ MISSING

Env-var hooks:
  HINT_API_URL          ✓ read at config/env.js:7
  HINT_API_KEY          ✓ read at config/env.js:8
  HINT_WEBHOOK_SECRET   ✗ MISSING — required for handshake signature verification
  HINT_PARTNER_ID       (optional, useful for logging)
  DATABASE_URL          ✓ read at config/db.js:3
```

### Tenancy audit (CRITICAL)

A marketplace app is shared across every practice that installs it. The partner's existing app must scope every read and write to the current request's `practice_id` (sourced from the handshake session). If their existing schema and queries weren't built around a tenant column, retrofit is a bigger migration than just adding the three contract routes — flag this up-front instead of producing a working install that silently leaks data across practices.

Inspect the existing schema and queries:

```bash
# Find candidate tenant tables — any user-facing data store
grep -rEn "CREATE TABLE|CREATE_TABLE|@Entity|class.*Model|Schema\(" <repo> --include="*.sql" --include="*.rb" --include="*.py" --include="*.ts" --include="*.go"

# For each suspected tenant table, look for queries that don't filter by practice_id (or whatever the partner's tenant column is named)
grep -rEn "SELECT|UPDATE|DELETE FROM" <repo> --include="*.sql" --include="*.rb" --include="*.py" --include="*.ts" --include="*.go" \
  | grep -vE "practice_id|tenant_id|organization_id|account_id"
```

This surfaces *candidates*, not bugs — the regex will flag plenty of safe queries (lookups by primary key where the key is itself FK-scoped, fixed-row joins, framework-internal queries). Walk each hit manually; the goal is to spot tables that are read or written without any tenant filter anywhere in the codebase, not to count grep matches.

Build a tenancy status table:

```
Tenant data audit for <repo>:

  Table          practice_id column?   Queries scoped?
  users          ✗ MISSING             ✗ 14 unscoped queries — needs migration
  messages       ✗ MISSING             ✗ 8 unscoped queries — needs migration
  audit_log      ✓ present             ✓ all 6 queries scoped
  feature_flags  N/A (global config)   N/A
```

For any table missing the `practice_id` column or with unscoped queries:

- The existing app is **single-tenant**. To retrofit safely, the partner must (a) add a non-null `practice_id` column to every tenant table via a migration, (b) backfill existing rows (single-tenant data goes to a fixed practice_id, or stays inaccessible from the marketplace install), (c) rewrite every query to filter by it, and (d) add tests that confirm cross-practice isolation.
- This is a bigger lift than retrofit covers in a single pass. **Stop here and surface the scope to the partner explicitly** — don't continue generating the marketplace contract routes until they've decided how to handle tenancy. Options to offer:
  1. Migrate the existing app to multi-tenant in place (their dev time, their call).
  2. Stand up a new multi-tenant service alongside the existing one (more isolation, less churn on the existing code).
  3. Make the marketplace install read-only against the existing data (skip the persisted-state risk entirely for v1).

If the partner already runs multi-tenant (every tenant table has `practice_id` or equivalent and every query scopes), proceed to Step 3 — but make sure the marketplace contract handlers source `practice_id` from the handshake session, not from a query-string param.

See `_common/marketplace-contract.md` "Tenancy" section for the canonical rule statement to share with the partner.

## Step 3: Generate the Missing Pieces

For each missing route or env var hook, generate a code snippet in the partner's stack and apply it as an edit to the existing files. **Do not create new files unless the existing layout has no natural home** — match the partner's conventions.

For Node.js (Express/Fastify/plain), Python (Flask/FastAPI), Ruby (Rails/Sinatra), Go (net/http/chi), PHP, Rust, etc., the pattern is the same — only the syntax differs. Use the canonical Node.js implementation at [`_common/node-template.md`](../_common/node-template.md) as the reference; port the logic, not the file structure.

### Snippet 1: `POST /hint/handshake` (signature-verified)

The handler must:

1. Read the raw request body **before** any JSON middleware parses it (some frameworks need an explicit "raw body" parser).
2. Compute `sha256=HMAC_SHA256(HINT_WEBHOOK_SECRET, raw_body)`.
3. Compare against the `X-Hint-Signature` header using a constant-time comparison.
4. On match: parse the JSON body, generate a session key (UUID), persist `{session_key, user, practice}` server-side, return `{ session_key }`.
5. On mismatch: return 401.

If the partner's framework strips the raw body before the handler sees it, you need to wire up a raw-body capture middleware. This is the most common porting mistake — flag it explicitly when you generate the snippet.

### Snippet 2: `POST /hint/connect/:code`

The handler must:

1. Extract `:code` from the URL path.
2. POST to `$HINT_API_URL/api/oauth/tokens` with `{ code, grant_type: 'authorization_code' }` and `Authorization: Bearer $HINT_API_KEY`.
3. Persist `{partner_id, practice_id, access_token}` keyed by `practice_id` in whatever session/practice store the app uses (Postgres if `DATABASE_URL` is set, in-memory only for demos).
4. Return `{ status: 'connected' }`.

### Snippet 3: `GET /hint/<anchor_type>?session_key=...`

For each anchor type the partner plans to register (`core_page`, `clinical_interaction`, `settings`), generate a route handler that:

1. Reads `session_key` from the query string.
2. Looks up the session.
3. Renders the surface's HTML, interpolating session context.
4. Returns it.

If the partner doesn't yet know which surfaces they want, default to `core_page` and offer to add the others later.

### Env-var wiring

If any of `HINT_API_URL` / `HINT_API_KEY` / `HINT_WEBHOOK_SECRET` aren't being read by the app, add them to the partner's existing config mechanism (`.env.example`, `config.py`, `application.yml`, etc.). Don't introduce a new mechanism if one exists.

## Step 4: Test the Diff Locally

Before pushing or registering, verify the changes work locally:

```bash
# 1. Boot the partner's app in dev mode (whatever their workflow is).
# 2. Hit the smoke-test endpoints from _common/marketplace-contract.md:

curl -sS -o /dev/null -w "POST /hint/handshake unsigned → %{http_code}\n" -X POST localhost:<port>/hint/handshake
# Expect: 401 (signature verification rejects unsigned requests)

# 3. Generate a valid signature and confirm 200:
BODY='{"user":{"id":"u-test"},"practice":{"id":"p-test"}}'
SIG="sha256=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$HINT_WEBHOOK_SECRET" | awk '{print $NF}')"
curl -sS -X POST localhost:<port>/hint/handshake \
  -H "Content-Type: application/json" \
  -H "X-Hint-Signature: $SIG" \
  -d "$BODY"
# Expect: 200 + {"session_key":"..."}
```

If signature verification is broken (200 on unsigned, or 401 on a valid signature), the most likely cause is raw-body capture — the framework's JSON middleware re-serialized the body before the handler signed it. Fix that first.

## Step 5: Wire Up Hint

### If Hosting Mode = Hosted by Hint

Zip the repo and POST it as a revision (same as `create-app`'s Hosted Mode Step 5):

```bash
cd <repo_dir> && zip -r /tmp/retrofit-deploy.zip . -x ".git/*" -x "node_modules/*" -x ".env"
curl -s -X POST "$HINT_API_URL/api/partner/app/revisions" \
  -H "Authorization: Bearer $API_KEY" \
  -F "code_archive=@/tmp/retrofit-deploy.zip;type=application/zip"
```

Save the revision id, poll `GET /api/partner/app/revisions` until `status: pushed`, then poll `GET /api/partner/app/services` for the row with `service_type: 'web'` and `status: "active"`. Save that `service_url` as `$APP_URL`. The list also contains an auto-provisioned Postgres sibling (`service_type: 'database'`, `service_url: null`) — skip it.

**Pre-deploy checklist for Hosted Mode:**
- The entry file MUST be `server.js` if Node.js (Hint's start command is `node server.js`).
- `package.json` (or equivalent) must define a `build` script (Hint runs `npm install && npm run build` at deploy time).
- The app must listen on `process.env.PORT` (Hint sets it).

If the partner's app doesn't match these conventions, either rename the entry point + add a build script, or switch to Self-Hosted Mode.

### If Hosting Mode = Self-hosted

Ask the partner to deploy the updated code wherever they normally deploy (Vercel push, AWS deploy, etc.) and give you the deployed URL. Save as `$APP_URL`. Then run the live smoke test from `_common/marketplace-contract.md` against `$APP_URL` to confirm the routes are reachable.

## Step 6: Register with Hint

This step is identical to `create-app`'s Step 6 — set the partner config + handshake URL + anchors:

```bash
curl -s -X PATCH "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"partner\": {\"auth_type\": \"automatic_headless\", \"redirect_url\": \"$APP_URL/hint/connect/\"}}"

curl -s -X PATCH "$HINT_API_URL/api/partner/app" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"app\": {\"handshake_url\": \"$APP_URL/hint/handshake\"}}"

# Create one anchor per surface the app implements. Examples:
curl -s -X POST "$HINT_API_URL/api/partner/app/anchors" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"anchor\": {\"type\": \"core_page\", \"source_url\": \"$APP_URL/hint/core_page\"}}"
```

## Step 7: Verify & Report

Print a summary:

```
Hint Marketplace Retrofit Complete!

  App:          <partner name>
  Hosting:      <Hosted by Hint  or  Self-hosted by partner>
  App URL:      $APP_URL
  Stack:        <detected stack>

  Routes added:
    POST /hint/handshake          — HMAC-SHA256 signature verified
    POST /hint/connect/:code      — OAuth code → practice access token
    GET  /hint/<anchor_type>      — embedded UI per anchor

  Anchors registered: <list>

  To install + test in the Partner Portal:
    1. Open https://app.hint.com
    2. Switch to your Sandbox Practice
    3. Marketplace → find your app → Install
```

## Troubleshooting

- **Raw body unavailable in handler** — the framework's JSON middleware consumed it. Wire up a raw-body capture (Express: `express.json({ verify: (req, _, buf) => { req.rawBody = buf } })`; Flask: `request.get_data(cache=True)`; Rails: `request.body.read` before any param parsing). Without raw body, signature verification produces false negatives.
- **Constant-time compare missing** — string equality on the signature leaks timing. Use the framework's secure comparator (Node `crypto.timingSafeEqual`, Ruby `Rack::Utils.secure_compare`, Python `hmac.compare_digest`).
- **CORS errors on the embedded surface** — the surface is loaded inside an iframe from `app.hint.com`. Add `app.hint.com` (and the sandbox equivalent) to the app's `Content-Security-Policy: frame-ancestors` directive, or remove `X-Frame-Options: DENY`.
- **Hint sets HINT_API_URL but the app reads HINT_API_HOST** (or similar) — env-var name mismatch. The contract is exact: `HINT_API_URL`, `HINT_API_KEY`, `HINT_PARTNER_ID`, `HINT_WEBHOOK_SECRET`, `DATABASE_URL`. Rename in the app or alias them at the platform.

For anything else, contact [devsupport@hint.com](mailto:devsupport@hint.com) with the app's `$APP_URL` and the revision id (if Hosted).

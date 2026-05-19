# /hint-marketplace-create-app — Build and Deploy a Hint Marketplace App

Build and deploy a fully working partner app to the Hint marketplace. This skill guides you through creating a Node.js application that integrates with Hint's platform, deploys it to managed infrastructure, and configures everything needed for practices to install and use it.

## Platform URLs

Set `$HINT_API_URL=https://api.hint.com` for both sandbox and live work (Partner Portal at `https://app.hint.com`). Full conventions: [`_common/api-conventions.md`](../_common/api-conventions.md).

**IMPORTANT**: Don't name underlying infrastructure providers, code hosts, or background-worker systems in any user-facing copy you generate (READMEs, install instructions, error messages, descriptions). Refer to the platform as "Hint" — the Hint API, Hint Partner Portal, Hint's managed deployment platform. The deployed service URL itself is what it is (currently a third-party hostname); don't pretend otherwise, just don't volunteer the provider's name in things you write.

## Getting Started

Ask the user three things:

1. **What does your app do?** Get a description of the app they want to build. Examples:
   - "A patient messaging tool that lets practitioners send secure messages"
   - "A lab results viewer that displays clinical data"
   - "A billing dashboard showing charges and payments"

2. **What type of surface?** How should the app appear in Hint:
   - **Core Page** (`core_page`) — a full-page app accessible from the sidebar. Best for dashboards, tools, and standalone features.
   - **Clinical Interaction** (`clinical_interaction`) — appears within clinical workflows, in the context of a specific patient/interaction. Best for clinical tools, lab viewers, and patient-specific features. Receives patient context via `HintSDK.currentPatient` and `HintSDK.interaction`.
   - **Settings** (`settings`) — embedded inside the practice's settings area, alongside Hint's own configuration tabs. Best for partner-specific configuration UI (API keys the practice needs to enter, feature toggles, sync schedules, etc.). The anchor is labeled via `settings_label` on the API (defaults to the app name).

3. **How do you want to host it?**
   - **Hosted** — Hint generates the app and runs it on Hint-managed infrastructure. Easiest path; you write nothing yourself. Pick this unless you have a specific reason not to. Hosted Mode runs `node server.js` and currently supports Node.js only.
   - **Self-hosted** — You already have (or will deploy) the app on your own infrastructure (Vercel, your AWS account, a VM, wherever). The skill only registers your URLs with Hint and confirms the marketplace contract. Any stack works.

The mode the user picks determines which path the skill follows after the shared setup. Hold onto the answer — you'll branch on it after Step 2.

Then ask if they already have a **sandbox partner API key** (starts with `sbx-`). If not, walk them through creating one:

### Setting Up a Sandbox Partner

1. **Log in to the Partner Portal** at `https://app.hint.com/partner/dashboard`
2. Click **"Go to Sandboxes"** in the Sandbox Setup section on the dashboard
3. **Create a Sandbox Partner** — this creates an isolated copy of your partner for development
4. **Create a Sandbox Practice** — this gives you a test practice to install your app on
5. Switch to the sandbox partner (click on it in the sandboxes list)
6. Go to **API Keys** in the sidebar and copy the sandbox API key (it starts with `sbx-`)

## Step 1: Verify Partner & Gather Context

Verify the API key works and gather partner info:

```bash
curl -s "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY"
```

From the response, extract whichever of these are present (fresh sandbox partners may not have all of them populated):
- `name` — partner name (often empty on a brand-new sandbox; the partner can set it later via `PATCH /partner/partner`)
- `slug` — URL-safe identifier; may be absent on fresh sandboxes
- `product.type` — should be `app` for marketplace apps. If the field is missing or null, treat that as "not yet configured" — ask the partner to confirm with Hint support that their partner has been set up as an app-type product, then continue. Don't hard-fail; an absent product is a setup-state quirk, not a wrong-product error.

If POST/PATCH calls later return "Partner product type must be app", that's the firm rejection — at that point the partner type genuinely needs admin attention before deploying.

Also check if the app already exists:
```bash
curl -s "$HINT_API_URL/api/partner/app" \
  -H "Authorization: Bearer $API_KEY"
```

## Step 2: Create the PartnerApp (if needed)

If no app exists:
```bash
curl -s -X POST "$HINT_API_URL/api/partner/app" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json"
```

---

**Branch on the mode the user picked in Getting Started:**

- **Hosted** → continue with Steps 3-5 in **Hosted Mode** below, then run the shared **Step 6: Configure Marketplace Settings**.
- **Self-hosted** → jump to **Self-Hosted Mode** (right after the Hosted Mode steps), then run the shared **Step 6: Configure Marketplace Settings**.

---

# Hosted Mode

The skill generates a Node.js app from a template, packages it, and uploads it to Hint's managed deployment platform. Skip this entire section if the user picked self-hosted.

## Step 3: Build the Node.js App

Services are **auto-provisioned on first deploy** — there's no explicit "create service" call. Build the app first, then deploy.

Copy `package.json` and `server.js` from the canonical Node.js template at [`_common/node-template.md`](../_common/node-template.md) into a temp directory. Then customize:

1. Replace `APP_NAME_HERE` with the app name.
2. Replace `SURFACE_TYPE_HERE` with `core_page`, `clinical_interaction`, or `settings`.
3. Customize the matching renderer (`renderCorePage`, `renderClinicalInteraction`, or `renderSettings`) with the app's actual UI — apply tokens from [`_common/brand-styles.md`](../_common/brand-styles.md) so the embedded surface looks native inside Hint.
4. Add app-specific API routes in the marked section. Every handler that touches tenant data MUST call `requireSession(req, res)` and scope queries by `session.practice_id`.

**Required filenames + scripts (the deploy platform runs them literally):**
- Entry file MUST be `server.js` (the platform runs `node server.js`)
- `package.json` MUST define both a `build` script and a `start` script (the platform runs `npm install && npm run build`, then `node server.js`)

The template's session store is in-memory by default — fine for kicking the tires, fatal for any real app. For production, persist sessions and practice tokens to the auto-provisioned Postgres at `process.env.DATABASE_URL`. See the schema sketch in [`_common/node-template.md`](../_common/node-template.md).

### Provider API + JS SDK

The access token from handshake/connect lets the embedded app read practice data. Endpoints, response-shape gotchas, and the in-iframe SDK example all live in [`_common/provider-api.md`](../_common/provider-api.md). Read that before writing any `/api/provider/*` client code or embedding the JS SDK — getting the response shape wrong silently produces empty results.

## Step 4: (Optional) Configure the Deployment Service

If the app needs custom environment variables (third-party API keys, feature flags, etc.) or a different build/start command, create the service explicitly first:

```bash
curl -s -X POST "$HINT_API_URL/api/partner/app/services" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "service": {
      "name": "api",
      "env_vars": {
        "STRIPE_API_KEY": "sk_test_...",
        "FEATURE_FLAG_X": "true"
      }
    }
  }'
```

Save the `id` from the response. If the app doesn't need custom config, **skip this step** — the next step auto-provisions a service on first deploy with default config.

The reserved env vars `HINT_API_URL`, `HINT_API_KEY`, `HINT_PARTNER_ID`, `HINT_WEBHOOK_SECRET`, and `DATABASE_URL` are managed by Hint and always present — partner-supplied values for those keys are ignored. See [`_common/api-conventions.md`](../_common/api-conventions.md#reserved-env-vars).

To update config on an existing service later: `PATCH /api/partner/app/services/<id>` with the same body shape. Env var changes propagate immediately; `build_command` / `start_command` changes take effect on the next revision deploy.

## Step 5: Deploy

Zip the app and POST it as a revision. If no service exists yet, the first deploy auto-provisions one with default config.

```bash
cd <app_dir> && zip -r /tmp/app-deploy.zip .
curl -s -X POST "$HINT_API_URL/api/partner/app/revisions" \
  -H "Authorization: Bearer $API_KEY" \
  -F "code_archive=@/tmp/app-deploy.zip;type=application/zip"
```

The response contains the revision row: `{ "id": "prev-...", "status": "pending", ... }`. Save the revision id as `$REV_ID`.

Poll the revision until `status` flips from `pending` to `pushed` (extracted + pushed — usually ~5s) or `failed`. `GET /api/partner/app/revisions` returns a **bare JSON array**, so handle it directly:

```bash
curl -s "$HINT_API_URL/api/partner/app/revisions" \
  -H "Authorization: Bearer $API_KEY" \
  | python3 -c "import sys,json; print(next((r['status'] for r in json.load(sys.stdin) if r['id']=='$REV_ID'),'?'))"
```

Once status is `pushed`, get the service URL. The services list is a bare array containing both the partner-managed web app AND the auto-provisioned Postgres sibling (and occasionally a stub from a prior provision attempt). **Filter by `service_type: 'web'` + `status: "active"`** — the database row has `service_type: 'database'` and `service_url: null`:

```bash
curl -s "$HINT_API_URL/api/partner/app/services" \
  -H "Authorization: Bearer $API_KEY" \
  | python3 -c "import sys,json; print(next((s['service_url'] for s in json.load(sys.stdin) if s.get('service_type')=='web' and s.get('status')=='active' and s.get('service_url')),''))"
```

> The database sibling shows up in this list so partners can see it exists, but `GET /api/partner/app/services/:id` and `PATCH /api/partner/app/services/:id` only accept web service ids — the partner-managed fields (`build_command`, `start_command`, `env_vars`) don't apply to Postgres, so the database id returns 404 on those endpoints.

Save the resulting URL as `$APP_URL`. Then poll it directly until it returns 200 (the build is usually live within a few seconds of `status: pushed`):

```bash
curl -s -o /dev/null -w '%{http_code}' $APP_URL/
```

Start at a **5-second interval, fall back to 10 seconds** if not live by the second poll. Cap at 5 minutes. Once the health check responds with a 200, the app is live — the service URL is the source of truth.

**Expect 502s for the first few seconds after `status: pushed`.** The container is finishing its boot while you're polling. A 502 (or "Application failed to respond") on the first 1-3 polls is normal — treat it as "still booting", not "broken". Only escalate if 502s persist past ~60 seconds, or if you see a 4xx (which indicates a real error, e.g. handshake URL mismatched).

If the revision flips to `status: failed`, the platform refused the deploy (typical reasons: partner not yet approved for production deploys; partner product type is not `app`; push failure). Contact Hint support (devsupport@hint.com) with the revision id.

Save the live URL as `$APP_URL` and skip past Self-Hosted Mode to **Step 6: Configure Marketplace Settings**.

---

# Self-Hosted Mode

The partner runs the app on their own infrastructure. The skill doesn't build, package, or deploy code — it confirms the marketplace contract is in place and registers the partner's existing URLs with Hint. Skip this entire section if the user picked hosted.

## Step 3: Confirm the Marketplace Contract

The partner's deployed app must implement three HTTP routes and read the reserved env vars. The canonical contract — including the signature-verification pseudocode and a smoke-test script — lives at [`_common/marketplace-contract.md`](../_common/marketplace-contract.md). Fetch and read that file before continuing. Walk the partner through whichever routes they don't yet have. If their app is missing any, point them at [`_common/node-template.md`](../_common/node-template.md) as the canonical Node.js reference — the handshake-verification + OAuth-exchange code ports cleanly to any language.

## Step 4: Gather the Partner's Deployed URL

Ask the partner: **what's the base URL where your app is deployed?** Examples: `https://patient-portal.acme.com`, `https://acme-marketplace.vercel.app`, `https://10.0.0.4:8080`.

Validate by hitting the partner's URL — if any of these returns something other than 200/2xx, the URLs don't match what's deployed:

```bash
curl -sS -o /dev/null -w "GET /  → HTTP %{http_code}\n" "$APP_URL/"
curl -sS -o /dev/null -w "POST /hint/handshake (unsigned, expect 401) → HTTP %{http_code}\n" -X POST "$APP_URL/hint/handshake"
curl -sS -o /dev/null -w "GET  /hint/<anchor_type> (no session, expect 200/401) → HTTP %{http_code}\n" "$APP_URL/hint/core_page"
```

A 401 on `/hint/handshake` is the correct response to an unsigned request — that confirms signature verification is wired up. A 200 or 404 there is a red flag. `GET /` returning 404 is fine if the app doesn't implement a health check at `/`.

Hold `$APP_URL` — the next step uses it.

---

## Step 6: Configure Marketplace Settings

Once `$APP_URL` is known (Hint-provisioned in Hosted Mode, partner-supplied in Self-Hosted Mode), configure the partner for automatic activation and embedding:

```bash
# Set auth type and redirect URL for automatic headless activation
curl -s -X PATCH "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"partner\": {\"auth_type\": \"automatic_headless\", \"redirect_url\": \"$APP_URL/hint/connect/\"}}"

# Set handshake URL
curl -s -X PATCH "$HINT_API_URL/api/partner/app" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"app\": {\"handshake_url\": \"$APP_URL/hint/handshake\"}}"

# Create anchor — use the surface type chosen by the user
# For core_page:
curl -s -X POST "$HINT_API_URL/api/partner/app/anchors" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"anchor\": {\"type\": \"core_page\", \"source_url\": \"$APP_URL/hint/core_page\"}}"

# For clinical_interaction:
curl -s -X POST "$HINT_API_URL/api/partner/app/anchors" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"anchor\": {\"type\": \"clinical_interaction\", \"source_url\": \"$APP_URL/hint/clinical_interaction\"}}"

# For settings:
curl -s -X POST "$HINT_API_URL/api/partner/app/anchors" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"anchor\": {\"type\": \"settings\", \"source_url\": \"$APP_URL/hint/settings\", \"settings_label\": \"<App Name> Settings\"}}"
```

An app can have one anchor of each type at most (`core_page`, `clinical_interaction`, `settings`) — pick the ones the app actually needs. Most apps register one, complex ones register two or three.

## Step 7: Verify & Report

Test the health check:
```bash
curl -s $APP_URL/
```

Print a summary:
```
Hint Marketplace App Set Up!

  App:         <description of what was built>
  Partner:     <partner_name>
  App URL:     <$APP_URL>
  Hosting:     <Hosted by Hint  or  Self-hosted by partner>
  Surface(s):  <core_page, clinical_interaction, settings — list the ones you registered>

  Routes (live on $APP_URL):
    GET  /                              — Health check
    POST /hint/handshake                — Hint handshake (verified, session creation)
    GET  /hint/<surface_type>           — Embedded UI (iframe)
    POST /hint/connect/:code            — Headless activation

  To install and test your app:

  1. Open the Partner Portal (URL above)
  2. Switch to your **Sandbox Practice** (click the practice
     name in the top-left corner and select the sandbox practice)
  3. Click **Marketplace** in the top navigation bar
  4. Find your app and click **Install**
  5. For **Core Page** apps: after installation, the app icon
     will appear in the left sidebar — click it to open
  6. For **Clinical Interaction** apps: the app will appear
     inside clinical workflows when viewing a patient
```

## Deploying Updates

**Hosted Mode** — re-run the deploy:
```bash
cd <app_dir> && zip -r /tmp/app-deploy.zip .
curl -s -X POST "$HINT_API_URL/api/partner/app/revisions" \
  -H "Authorization: Bearer $API_KEY" \
  -F "code_archive=@/tmp/app-deploy.zip;type=application/zip"
```

Poll the revision list (`GET /api/partner/app/revisions`) until the new revision flips to `pushed`, then poll `$APP_URL/` for a 200.

To change config (env vars, build/start command) on an existing service:
```bash
curl -s -X PATCH "$HINT_API_URL/api/partner/app/services/$SERVICE_ID" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"service": {"env_vars": {"FEATURE_FLAG_X": "false"}}}'
```

Env var changes hit the deployed service immediately. Build/start command changes apply on the next revision push.

**Self-Hosted Mode** — the partner deploys to their own infrastructure however they normally do; nothing changes on Hint's side. If the partner moves the app to a new URL, re-run the URL-registration calls from Step 6 with the updated `$APP_URL`.

## Troubleshooting

- **(Hosted) Revision flips to `status: failed` immediately** — Contact Hint support (devsupport@hint.com) with the revision id. Common causes: partner is not approved for non-sandbox deploys yet, the partner has no eligible API key, or the partner's product type is not `app`.
- **(Hosted) Revision stays at `status: pushed` but `$APP_URL` never serves the new code** — The platform's build is in progress (typically 2-3 min). If it stays stuck past 10 min, contact Hint support with the revision's `commit_sha`.
- **(Self-hosted) `$APP_URL` returns the wrong content / 404 on /hint/handshake** — The partner's app isn't actually serving the marketplace routes at the URL they gave. Have them double-check their deployment, then re-run the smoke-test curls from Self-Hosted Step 4.
- **"Product type must be app"** — The partner's product type must be `app`. Update it in the Partner Portal.
- **403 on Partner API write endpoints** — Sandbox keys (`sbx-` prefix) can fully manage the marketplace plumbing (revisions, services, anchors, app + partner settings) but **cannot create or modify business records** (e.g. `POST /api/partner/charges`, `POST /api/partner/practice_charges`). Those endpoints require a production-approved partner — contact [devsupport@hint.com](mailto:devsupport@hint.com) for promotion. If a sandbox key is hitting 403 on a non-business endpoint, the API key may not have the right permissions; double-check it's the partner's own key, not an integration key.
- **428 "This action requires a Practice" on `/api/provider/*`** — You called a Provider endpoint with the partner-wide `HINT_API_KEY` instead of a practice-scoped access token. Provider endpoints can only be called on behalf of a specific practice — use the access_token from `POST /api/oauth/tokens` (the value persisted during `/hint/connect/:code`). See [`_common/provider-api.md`](../_common/provider-api.md).
- **Handshake fails with 401** — The webhook secret may not be configured correctly on the deployed service. The partner finds it in the Partner Portal under API Keys → Webhooks Signature Key.
- **Headless connect fails** — The API URL env var may not point to the correct Hint API instance.
- **Embedded page doesn't load** — Verify the anchor exists and the `source_url` matches `$APP_URL` + the correct route for the surface type.

# /hint-marketplace-create-app — Build and Deploy a Hint Marketplace App

Build and deploy a fully working partner app to the Hint marketplace. This skill guides you through creating a Node.js application that integrates with Hint's platform, deploys it to managed infrastructure, and configures everything needed for practices to install and use it.

## Platform URLs

Set `$HINT_API_URL=https://api.hint.com` for both sandbox and live work (Partner Portal at `https://app.hint.com`). Full conventions: [`_common/api-conventions.md`](../_common/api-conventions.md).

**IMPORTANT**: Don't name underlying infrastructure providers, code hosts, or background-worker systems in any user-facing copy you generate (READMEs, install instructions, error messages, descriptions). Refer to the platform as "Hint" — the Hint API, Hint Partner Portal, Hint's managed deployment platform. Deployed services live at `<service-ident>.hintapps.com` — a Hint-owned domain — so the URL itself doesn't leak the underlying provider.

## Getting Started

### Managed-hosted mode (practice-initiated)

If the user invokes this skill with the phrase **"managed hosted mode"** (e.g. "Use the marketplace skill in managed hosted mode"), they came in from the **App** tab in the Hint practice Developers section. In that case:

- The sandbox partner + practice + API key are already provisioned for them; **skip the "Setting Up a Sandbox Partner" section below entirely**.
- The hosting choice is settled — they're on **Hosted** mode. Don't ask question 3.
- They copy the sandbox API key from the practice App tab (not the Partner Portal). If they don't have one yet, point them at `/admin/developers/app` in their practice account.
- The "Update product type in Partner Portal" troubleshooting in Step 1 doesn't apply — practice users don't have Partner Portal access. If `product.type` isn't `app`, point them at [devsupport@hint.com](mailto:devsupport@hint.com) instead.

Then proceed with questions 1 + 2 below and skip straight to Step 1.

### Regular partner mode

Ask the user three things:

1. **What does your app do?** Get a description of the app they want to build. Examples:
   - "A patient messaging tool that lets practitioners send secure messages"
   - "A lab results viewer that displays clinical data"
   - "A billing dashboard showing charges and payments"

2. **What type of surface?** How should the app appear in Hint:
   - **Core Page** (`core_page`) — a full-page app accessible from the sidebar. Best for dashboards, tools, and standalone features.
   - **Clinical Interaction** (`clinical_interaction`) — appears within clinical workflows, in the context of a specific patient/interaction. Best for clinical tools, lab viewers, and patient-specific features. Receives patient context via `HintSDK.currentPatient` and `HintSDK.interaction`.
   - **Clinical Chart** (`clinical_chart`) — embedded inside the patient chart view, alongside Hint's native chart sections. Best for chart-resident widgets (latest labs, risk scores, care plan summaries) that should always be visible whenever the practitioner has a chart open, not just during an active interaction. Receives `HintSDK.currentPatient` (no `interaction` since the surface lives outside the interaction timeline).
   - **Settings** (`settings`) — embedded inside the practice's settings area, alongside Hint's own configuration tabs. Best for partner-specific configuration UI (API keys the practice needs to enter, feature toggles, sync schedules, etc.). The anchor is labeled via `settings_label` on the API (defaults to the app name).

3. **How do you want to host it?**
   - **Hosted** — Hint generates the app and runs it on Hint-managed infrastructure. Easiest path; you write nothing yourself. Pick this unless you have a specific reason not to. Hosted Mode runs `node server.js` and currently supports Node.js only.
   - **Self-hosted** — You already have (or will deploy) the app on your own infrastructure (Vercel, your AWS account, a VM, wherever). The skill only registers your URLs with Hint and confirms the marketplace contract. Any stack works.

The mode the user picks determines which path the skill follows after the shared setup. Hold onto the answer — you'll branch on it after Step 2.

Then ask if they already have a **sandbox partner API key** (starts with `sbx-`). If not, walk them through creating one:

### Setting Up a Sandbox Partner

> Skip this section entirely if the user invoked the skill with **"managed hosted mode"** — their sandbox is already set up; they just need to paste the key from the practice App tab.

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

Then look up the partner's product — every app endpoint is scoped to a `partner_product`, so you need its id before doing anything else:

```bash
curl -s "$HINT_API_URL/api/partner/partner_products" \
  -H "Authorization: Bearer $API_KEY"
```

Returns a bare JSON array. Most partners have exactly one product; pick the first entry and save its `id` (looks like `ppro-XXXXXXXXXX`) as `$PRODUCT_ID`. Also save its `slug` as `$PRODUCT_SLUG` — Step 6 uses it to build the post-activation redirect URL for full-page apps (`https://app.hint.com/apps/$PRODUCT_SLUG`). If the partner has multiple products, match by `name` against the app the user is building. The Partner Portal URL bar (`/partner/products/ppro-XXXXXXXXXX/activation_settings`) also exposes the ident as a fallback.

From the product row, also check `type` — should be `app` for marketplace apps. If the field is missing or not `app`, treat that as "not yet configured": ask the partner to confirm with Hint support that their product has been set up as an app-type, then continue. Don't hard-fail; an absent/wrong type is a setup-state quirk, not an immediate error.

If POST/PATCH calls later return "Partner product type must be app", that's the firm rejection — at that point the product type genuinely needs admin attention before deploying. **In managed-hosted mode, the practice can't fix this themselves** — point them at [devsupport@hint.com](mailto:devsupport@hint.com).

Also check if the app already exists:
```bash
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app" \
  -H "Authorization: Bearer $API_KEY"
```

## Step 2: Create the PartnerApp (if needed)

If no app exists:
```bash
curl -s -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app" \
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

**Use the auto-provisioned Postgres for session + token storage — don't ship the in-memory default.** The template ships with an in-memory `sessions` + `practiceTokens` map for demo simplicity, but in Hosted Mode that's broken on the very first install: the container handling `POST /hint/connect/:code` is often a different process from the one serving `GET /hint/core_page` a few seconds later (rolling deploys, restarts, multi-process workers), so the session created at connect time is missing when the embedded UI loads → "Practice has not completed headless connect yet." Persist to Postgres at `process.env.DATABASE_URL` from day one. Schema sketch + drop-in `pg` wiring live in [`_common/node-template.md`](../_common/node-template.md).

`DATABASE_URL` may not be reachable on the very first deploy — the sibling Postgres can still be in provisioning when the web service starts serving. Tolerate transient `ECONNREFUSED` on boot: defer schema creation behind a short retry loop (5 attempts × ~2s) instead of crashing the process. Once Postgres is up, the connection sticks.

### Provider API + JS SDK

The access token from handshake/connect lets the embedded app read practice data. Endpoints, response-shape gotchas, and the in-iframe SDK example all live in [`_common/provider-api.md`](../_common/provider-api.md). Read that before writing any `/api/provider/*` client code or embedding the JS SDK — getting the response shape wrong silently produces empty results.

**Before writing any KPI/metric/dashboard code**, read [`_common/provider-api-fields.md`](../_common/provider-api-fields.md) for the schema sketch + gotchas on the top five resources (patients, memberships, customer_invoices, payments, practitioners). It covers the family-vs-individual membership shape, the `status` vs `enrollment_status` disambiguation, where revenue actually lives (NOT `customer_invoices.charges`), and the sandbox `created_at` quirk that flattens every time-series chart. Skipping this file is the difference between a working v1 and a ship-zero-everywhere v1.

## Step 4: (Optional) Configure the Deployment Service

If the app needs custom environment variables (third-party API keys, feature flags, etc.) or a different build/start command, create the service explicitly first:

```bash
curl -s -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/services" \
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

To update config on an existing service later: `PATCH /api/partner/partner_products/$PRODUCT_ID/app/services/<id>` with the same body shape. Env var changes propagate immediately; `build_command` / `start_command` changes take effect on the next revision deploy.

## Step 5: Deploy

Zip the app and POST it as a revision. If no service exists yet, the first deploy auto-provisions one with default config.

```bash
cd <app_dir> && zip -r /tmp/app-deploy.zip .
curl -s -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/revisions" \
  -H "Authorization: Bearer $API_KEY" \
  -F "code_archive=@/tmp/app-deploy.zip;type=application/zip"
```

The response contains the revision row: `{ "id": "prev-...", "status": "pending", ... }`. Save the revision id as `$REV_ID`.

Poll the revision until `status` flips from `pending` to `pushed` (extracted + pushed — usually ~5s) or `failed`. `GET /api/partner/partner_products/$PRODUCT_ID/app/revisions` returns a **bare JSON array**, so handle it directly:

```bash
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/revisions" \
  -H "Authorization: Bearer $API_KEY" \
  | python3 -c "import sys,json; print(next((r['status'] for r in json.load(sys.stdin) if r['id']=='$REV_ID'),'?'))"
```

Once status is `pushed`, get the service URL. The services list is a bare array containing both the partner-managed web app AND the auto-provisioned Postgres sibling (and occasionally a stub from a prior provision attempt). **Filter by `service_type: 'web'` + `status: "active"`** — the database row has `service_type: 'database'` and `service_url: null`:

```bash
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/services" \
  -H "Authorization: Bearer $API_KEY" \
  | python3 -c "import sys,json; print(next((s['service_url'] for s in json.load(sys.stdin) if s.get('service_type')=='web' and s.get('status')=='active' and s.get('service_url')),''))"
```

> The database sibling shows up in this list so partners can see it exists, but `GET /api/partner/partner_products/$PRODUCT_ID/app/services/:id` and `PATCH /api/partner/partner_products/$PRODUCT_ID/app/services/:id` only accept web service ids — the partner-managed fields (`build_command`, `start_command`, `env_vars`) don't apply to Postgres, so the database id returns 404 on those endpoints.

Save the resulting URL as `$APP_URL`. Then poll it directly until it returns 200 — **realistic boot time is 30–90 seconds from `status: pushed`**, not "a few seconds":

```bash
curl -s -o /dev/null -w '%{http_code}' $APP_URL/
```

Poll every 5 seconds. Cap at 5 minutes — if you don't see a 200 by then, treat it as a real failure. Pull the runtime logs to see why (`GET /api/partner/partner_products/$PRODUCT_ID/app/services/:id/logs`, or in the Partner Portal under the service row's "View logs" button — see [Viewing logs](#viewing-logs) below). Escalate to [devsupport@hint.com](mailto:devsupport@hint.com) with the revision id only if the logs aren't conclusive. 502s and "Application failed to respond" during the first minute are normal — the container is still booting. The progression you should expect:

- t=0s (`status: pushed`): container image is built and pushed; the platform is spinning up the runtime
- t=10-50s: 502s from the edge while the container is still warming
- t=30-90s: first 200 — app is live

Only escalate if 502s persist past ~2 minutes, or if you see a 4xx (which indicates a real error, e.g. handshake URL mismatched, missing env var crashing on boot).

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

Once `$APP_URL` is known (Hint-provisioned in Hosted Mode, partner-supplied in Self-Hosted Mode), configure the partner for automatic activation and embedding.

> **In managed-hosted mode**, the install flow itself does NOT consult `auth_type` or `redirect_url` — the integration is created and activated programmatically by Hint. But the Activation Settings tab in the partner's portal still reads from those fields, and a practice/partner staring at "auth_type: manual (Not Recommended)" right after install will think the skill didn't finish. **Set them anyway** so the UI looks consistent with what actually shipped.

**`post_activation_redirect_url` — only set when the app has a `core_page` anchor.** After a practice activates a partner app, Hint navigates the user to whatever this field points at. For a full-page app (Core Page anchor) the natural destination is `https://app.hint.com/apps/$PRODUCT_SLUG` — the practice-facing route that opens the installed app inside Hint's UI (renders the Core Page anchor in-portal). Without this field set, every full-page install lands the practice owner back on the marketplace listing page they just came from, which is dead weight when the user's intent is "use the app now". For a `clinical_interaction`-only (or `settings`-only) app there is no standalone landing surface — those embed inside a specific clinical or settings context — so leave the field unset and Hint stays on its default post-activation screen. Mixed surfaces that include `core_page`: set it.

> **Requires `$PRODUCT_SLUG`** from Step 1. If the product's `slug` was `null` (fresh sandbox), set one before this PATCH — either via the marketplace listing setup in Step 6.5 or by asking devsupport. Without a slug, `/apps/<slug>` has no public URL to point at.

```bash
# Set auth type and redirect URL for automatic headless activation
# In managed-hosted mode this is purely cosmetic for the Activation Settings UI
# (install fires through a different code path); set it anyway so the tab
# doesn't read "Not Recommended" right after install.
#
# Pick ONE of the two curls below based on whether the app includes a
# core_page anchor.

# ---- Variant A: app has a core_page anchor (set post_activation_redirect_url) ----
curl -s -X PATCH "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"partner\": {
        \"auth_type\": \"automatic_headless\",
        \"redirect_url\": \"$APP_URL/hint/connect/\",
        \"post_activation_redirect_url\": \"https://app.hint.com/apps/$PRODUCT_SLUG\"
      }}"

# ---- Variant B: clinical_interaction-only or settings-only (no full-page surface) ----
curl -s -X PATCH "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"partner\": {\"auth_type\": \"automatic_headless\", \"redirect_url\": \"$APP_URL/hint/connect/\"}}"

# Set handshake URL
curl -s -X PATCH "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"app\": {\"handshake_url\": \"$APP_URL/hint/handshake\"}}"

# Create anchor — use the surface type chosen by the user
# For core_page: include the sidebar icon fields so the app doesn't render
# with Hint's generic placeholder icon in the practice's left nav.
curl -s -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/anchors" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"anchor\": {
      \"type\": \"core_page\",
      \"source_url\": \"$APP_URL/hint/core_page\",
      \"core_page_icon_url\": \"<https URL or base64 data URI — reuse $PRODUCT_ICON_URL from Step 6.5 when available>\",
      \"core_page_icon_label\": \"<short label, ≤14 chars — usually the app name>\"
    }
  }"

# For clinical_interaction:
curl -s -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/anchors" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"anchor\": {\"type\": \"clinical_interaction\", \"source_url\": \"$APP_URL/hint/clinical_interaction\"}}"

# For clinical_chart:
curl -s -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/anchors" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"anchor\": {\"type\": \"clinical_chart\", \"source_url\": \"$APP_URL/hint/clinical_chart\"}}"

# For settings:
curl -s -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/anchors" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"anchor\": {\"type\": \"settings\", \"source_url\": \"$APP_URL/hint/settings\", \"settings_label\": \"<App Name> Settings\"}}"
```

An app can have one anchor of each type at most (`core_page`, `clinical_interaction`, `clinical_chart`, `settings`) — pick the ones the app actually needs. Most apps register one, complex ones register two or three.

> **`core_page` sidebar icon (`core_page_icon_url` + `core_page_icon_label`):** without these, every Core Page install renders with Hint's generic placeholder icon in the practice's left nav — the surface the practitioner sees every day. Reuse the listing icon set in Step 6.5 (`partner_product.icon`) for the URL, and use the app's name (truncated to ≤14 chars) for the label. Constraints: roughly 32×32 viewport, single accent color (Hint blue `#0E68E2` is safe), transparent background, simple geometric glyph that reads at small size — SVG strongly preferred. The label is shown as a tooltip on desktop and as visible text on mobile sidebars.

## Step 6.5: Configure the Marketplace Listing

The previous steps set up the **technical contract** (how the app embeds + authenticates). Now configure the **marketplace listing** — the customer-facing card practices see when browsing the marketplace. Without this, the listing renders with placeholder content (name defaults to the partner's sandbox name, summary defaults to "Sandbox Testing", no built-by, no icon) — which looks unfinished even though the app works.

Reuse `$PRODUCT_ID` from Step 1.

```bash
curl -s -X PATCH "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"partner_product\": {
      \"name\": \"<the app name the user gave in question 1>\",
      \"summary\": \"<a 1-line tagline; derive from the app description or ask the user>\",
      \"built_by_name\": \"<partner display name, e.g. the partner's company name>\",
      \"built_by_url\": \"<partner website, optional>\",
      \"icon\": \"<optional URL to a square icon image>\"
    }
  }"
```

These 5 fields cover the listing card. **Overview, Highlights, Quotes, Categories, Links, and Preconditions are also partner-settable** via their own per-section endpoints — use the [`fill-listing`](https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/fill-listing/SKILL.md) skill for the guided workflow, or hit `/api/partner/partner_products/$PRODUCT_ID/{overview|highlights|quotes|categories|links|preconditions}` directly. Pricing and install Requirements are NOT partner-settable — those are hinter-curated decisions; email [devsupport@hint.com](mailto:devsupport@hint.com) to change them.

The `slug` and `type` fields are set when the product is first created and **are not editable via API** afterwards — if the user wants to rename the URL slug or change product type after creation, they have to email [devsupport@hint.com](mailto:devsupport@hint.com).

### Product status lifecycle

`partner_product.status` is read-only for partners (admin-curated). Values, in rough lifecycle order:

| Status | Meaning |
|---|---|
| `unpublished` | Default state at creation. Blocked from install — practices can't install yet. Most new products sit here while content is being filled in. |
| `researching` | Hint is internally evaluating the product. Not visible in marketplace browse. |
| `coming_soon` | Pre-launch placeholder. May be visible with a "coming soon" badge. |
| `beta` | Limited availability — visible in marketplace browse but flagged as beta. Practices can install. |
| `live` | Fully launched. Default state for an active marketplace listing. |
| `sunsetting` | Being phased out. Still installable but flagged. |
| `deprecated` | No longer recommended. Existing installs continue working; new installs are blocked. |
| `decommissioned` | Gone. Hidden from marketplace browse. |

Status transitions are handled by Hint admins, not partners. The relevant ones for a new partner are: `unpublished` (after create) → `beta` or `live` (after Hint reviews and approves the listing). Email [devsupport@hint.com](mailto:devsupport@hint.com) when ready to move to `beta` or `live`.

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
curl -s -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/revisions" \
  -H "Authorization: Bearer $API_KEY" \
  -F "code_archive=@/tmp/app-deploy.zip;type=application/zip"
```

Poll the revision list (`GET /api/partner/partner_products/$PRODUCT_ID/app/revisions`) until the new revision flips to `pushed`, then poll `$APP_URL/` for a 200.

To change config (env vars, build/start command) on an existing service:
```bash
curl -s -X PATCH "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/services/$SERVICE_ID" \
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
- **Handshake fails with 401** — Most common cause: `HINT_WEBHOOK_SECRET` on the deployed service doesn't match the partner's current **Webhooks Signature Key** in the Partner Portal (visible under API Keys). If the partner ever rotated that key, the env var on the service is now stale — re-push env vars via `PATCH /api/partner/partner_products/$PRODUCT_ID/app/services/:id` to pick up the current value. The template's verifier logs the last 4 chars of the env var on mismatch — compare against the portal's current key.
- **Headless connect fails** — The API URL env var may not point to the correct Hint API instance.
- **Embedded page doesn't load** — Verify the anchor exists and the `source_url` matches `$APP_URL` + the correct route for the surface type.

### Viewing logs

Runtime `stdout`/`stderr` from `node server.js` is now visible to partners. Use whichever surface fits the task:

- **Partner API** (programmatic): `GET /api/partner/partner_products/$PRODUCT_ID/app/services/:id/logs` returns the last hour of entries by default. Each entry has `timestamp`, `message`, `type` (`app` / `request` / `build`), `level` (`info` / `warning` / `error`), and — for `type=request` — `path`, `method`, `status_code`. Pass-through query params (`type`, `level`, `text`, `limit`, `direction`, `start_time`, `end_time`) filter and paginate.
- **Partner Portal** (visual): under each service row on `/partner/products/$PRODUCT_ID/custom_app`, click **View logs** for a Render-style console with color-coded status badges, day separators, and a **Tail** toggle for live streaming (polls every 2 s and appends new entries).
- **Live tail in scripts**: poll `GET …/logs?direction=forward&start_time=<lastSeenTimestamp>` every couple seconds; dedupe by `timestamp`.

For deeper inspection of the env on a running service (e.g. handshake-secret last-4 verification), keeping **instrumented debug routes guarded by an env var** in the app is still useful — they let you inspect what the container ACTUALLY has at runtime, not just what was pushed via the API. Recommended recipe:

```javascript
// In server.js, behind a non-secret env-var flag so partners can toggle
// without re-deploying:
if (process.env.HINT_DEBUG === 'true' && req.method === 'GET' && url.pathname === '/debug/env') {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  return res.end(JSON.stringify({
    HINT_API_URL: HINT_API_URL,
    HINT_API_KEY_set: Boolean(HINT_API_KEY),
    HINT_API_KEY_prefix: HINT_API_KEY ? HINT_API_KEY.slice(0, 8) : null,
    HINT_PARTNER_ID: HINT_PARTNER_ID,
    HINT_WEBHOOK_SECRET_set: Boolean(HINT_WEBHOOK_SECRET),
    HINT_WEBHOOK_SECRET_len: HINT_WEBHOOK_SECRET?.length || 0,
    HINT_WEBHOOK_SECRET_last4: HINT_WEBHOOK_SECRET?.slice(-4) || null,
    DATABASE_URL_set: Boolean(process.env.DATABASE_URL),
  }, null, 2));
}
```

Push the service with `HINT_DEBUG=true` via `PATCH /api/partner/partner_products/$PRODUCT_ID/app/services/:id`, hit `$APP_URL/debug/env` to inspect the actual env, then set `HINT_DEBUG=false` (or remove the var) before declaring the app production-ready. Never expose secret values themselves — only existence/prefix/last-4 for diagnostics.

For business-logic debugging (handshake verification mismatches, connect failures, etc.), the template's handshake verifier and connect handler already log structured diagnostics to `stdout` — those show up directly in the logs view above, so a failing handshake is now traceable end-to-end from the portal.

# /hint-marketplace-create-app — Build and Deploy a Hint Marketplace App

Build and deploy a fully working partner app to the Hint marketplace. This skill guides you through creating a Node.js application that integrates with Hint's platform, deploys it to managed infrastructure, and configures everything needed for practices to install and use it.

## Platform URLs

- **Hint API (sandbox)**: `https://api.sandbox.hint.com` — for sandbox testing. Use this whenever the partner API key starts with `sbx-`.
- **Hint API (live)**: `https://api.hint.com` — for live partners after the app is approved + promoted. Use this with API keys that have no `sbx-` prefix.
- **Partner Portal**: `https://app.hint.com` — single portal for both sandbox and live; partners switch between sandbox/live workspaces inside it.

**Rule:** set `$HINT_API_URL` based on the API key the partner gives you — sandbox key → sandbox API host, live key → live API host. Mixing them (sandbox key against live host, or vice versa) returns 401. The same rule applies to the `HINT_API_URL` env var on the deployed service: sandbox-deployed apps point at the sandbox host, live-deployed apps at the live host (Hint sets this automatically in Hosted Mode).

**IMPORTANT**: Never reference any underlying infrastructure providers, code hosts, or background-worker systems to the user. From the partner's perspective, everything is "Hint" — the Hint API, the Hint Partner Portal, Hint's managed deployment platform. The partner uploads code and Hint deploys it. Internal infrastructure is invisible.

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
   - **Hosted** — Hint generates the app and runs it on Hint-managed infrastructure. Easiest path; you write nothing yourself. Pick this unless you have a specific reason not to.
   - **Self-hosted** — You already have (or will deploy) the app on your own infrastructure (Vercel, your AWS account, a VM, wherever). The skill only registers your URLs with Hint and confirms the marketplace contract.

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

From the response, extract:
- `name` — partner name
- `slug` — URL-safe identifier (used for repo name and service URL)
- `product.type` — must be `app`

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

Create a temp directory with `package.json` and `server.js`. Use the template below as the base — customize `APP_CONFIG`, the HTML renderers, and add app-specific routes based on what the user described.

**Required filenames + scripts (the deploy platform runs them literally):**
- Entry file MUST be `server.js` (the platform runs `node server.js`)
- `package.json` MUST define both a `build` script and a `start` script (the platform runs `npm install && npm run build`, then `node server.js`)

### `package.json`

```json
{
  "name": "<partner_slug>-api",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "build": "echo 'no build step'",
    "start": "node server.js"
  }
}
```

### `server.js` — Base Template

```javascript
const http = require('http');
const https = require('https');
const crypto = require('crypto');

const port = process.env.PORT || 3000;
const HINT_API_URL = process.env.HINT_API_URL || '';
const HINT_API_KEY = process.env.HINT_API_KEY || '';
const HINT_PARTNER_ID = process.env.HINT_PARTNER_ID || '';
const HINT_WEBHOOK_SECRET = process.env.HINT_WEBHOOK_SECRET || '';

// ============================================================
// APP CONFIG — customize for each app
// ============================================================
const APP_CONFIG = {
  name: 'APP_NAME_HERE',
  version: '1.0.0',
  surfaceType: 'SURFACE_TYPE_HERE', // 'core_page', 'clinical_interaction', or 'settings'
};

// ============================================================
// Session store (in-memory)
//
// NOTE: This template stores sessions + practice access tokens in memory
// for demo simplicity. Every revision deploy is a fresh process and wipes
// this object — that's fine for kicking the tires, fatal for any real app.
//
// For anything beyond a demo, persist to the Postgres database that Hint
// auto-provisions alongside this service. The connection string is
// available at `process.env.DATABASE_URL`. Add `pg` (already in the
// dependencies) and write the sessions table on boot.
// ============================================================
const sessions = {};

// ============================================================
// Helpers
// ============================================================
function parseBody(req) {
  return new Promise((resolve) => {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try { resolve({ raw: body, parsed: JSON.parse(body) }); }
      catch { resolve({ raw: body, parsed: {} }); }
    });
  });
}

function verifySignature(rawBody, signature) {
  if (!HINT_WEBHOOK_SECRET || !signature) return false;
  const expected = 'sha256=' + crypto.createHmac('sha256', HINT_WEBHOOK_SECRET).update(rawBody).digest('hex');
  try {
    return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
  } catch {
    return false;
  }
}

function hintApi(method, path, body) {
  return new Promise((resolve, reject) => {
    const url = new URL(HINT_API_URL + path);
    const proto = url.protocol === 'https:' ? https : http;
    const options = {
      hostname: url.hostname,
      port: url.port || (url.protocol === 'https:' ? 443 : 80),
      path: url.pathname + url.search,
      method,
      headers: {
        'Authorization': `Bearer ${HINT_API_KEY}`,
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    };
    const req = proto.request(options, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => {
        try { resolve({ status: res.statusCode, body: JSON.parse(data) }); }
        catch { resolve({ status: res.statusCode, body: data }); }
      });
    });
    req.on('error', reject);
    if (body) req.write(JSON.stringify(body));
    req.end();
  });
}

// ============================================================
// Server
// ============================================================
const server = http.createServer(async (req, res) => {
  const url = new URL(req.url, `http://localhost:${port}`);

  // Health check
  if (req.method === 'GET' && url.pathname === '/') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({ status: 'ok', app: APP_CONFIG.name, version: APP_CONFIG.version }));
  }

  // Handshake — Hint calls this to create a session
  if (req.method === 'POST' && url.pathname === '/hint/handshake') {
    const { raw, parsed } = await parseBody(req);
    const signature = req.headers['x-hint-signature'];
    if (!verifySignature(raw, signature)) {
      console.error('Handshake signature verification failed');
      res.writeHead(401, { 'Content-Type': 'application/json' });
      return res.end(JSON.stringify({ error: 'Invalid signature' }));
    }
    const sessionKey = crypto.randomUUID();
    sessions[sessionKey] = {
      user: parsed.user,
      practice: parsed.practice,
      integration: parsed.integration,
      accessToken: parsed.access_token,
      createdAt: new Date().toISOString(),
    };
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({ session_key: sessionKey }));
  }

  // Core page — full-page embedded UI (for core_page surface)
  if (req.method === 'GET' && url.pathname === '/hint/core_page') {
    const sessionKey = url.searchParams.get('session_key');
    const session = sessions[sessionKey];
    res.writeHead(200, { 'Content-Type': 'text/html' });
    return res.end(renderCorePage(sessionKey, session));
  }

  // Clinical interaction — patient-context embedded UI (for clinical_interaction surface)
  if (req.method === 'GET' && url.pathname === '/hint/clinical_interaction') {
    const sessionKey = url.searchParams.get('session_key');
    const session = sessions[sessionKey];
    res.writeHead(200, { 'Content-Type': 'text/html' });
    return res.end(renderClinicalInteraction(sessionKey, session));
  }

  // Settings — embedded inside the practice settings area (for settings surface)
  if (req.method === 'GET' && url.pathname === '/hint/settings') {
    const sessionKey = url.searchParams.get('session_key');
    const session = sessions[sessionKey];
    res.writeHead(200, { 'Content-Type': 'text/html' });
    return res.end(renderSettings(sessionKey, session));
  }

  // Headless connect — Hint POSTs auth code during installation.
  //
  // The response body's `access_token` is the practice-scoped key the app
  // uses to call `/api/provider/*` endpoints. It is NOT the same as the
  // partner API key in HINT_API_KEY (which is partner-wide).
  //
  // Persist `{ partner_id, practice_id, access_token }` keyed by practice
  // so the embedded UI can authenticate Provider API calls on every render.
  // The example below just logs it — replace with a real write to your
  // session/practice store (use the auto-provisioned Postgres at
  // `process.env.DATABASE_URL` for anything beyond a demo).
  if (req.method === 'POST' && url.pathname.startsWith('/hint/connect/')) {
    const authCode = url.pathname.replace('/hint/connect/', '');
    console.log('Headless connect, auth code: ' + authCode);
    try {
      if (HINT_API_URL && HINT_API_KEY) {
        const resp = await hintApi('POST', '/api/oauth/tokens', {
          code: authCode,
          grant_type: 'authorization_code',
        });
        // TODO: persist resp.body.access_token + practice_id to DATABASE_URL
        // so the embedded surface can use it to call /api/provider/*.
        console.log('Token exchange:', resp.status, JSON.stringify(resp.body));
      }
    } catch (err) {
      console.error('Connect error:', err.message);
    }
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({ status: 'connected' }));
  }

  // ============================================================
  // APP-SPECIFIC ROUTES — add your routes here
  // ============================================================

  // 404
  res.writeHead(404, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ error: 'Not found' }));
});

// ============================================================
// HTML Renderers — customize these for your app
// ============================================================

function renderCorePage(sessionKey, session) {
  return `<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>${APP_CONFIG.name}</title>
  <link href="https://fonts.googleapis.com/css2?family=Wix+Madefor+Text:wght@400;500;700&display=swap" rel="stylesheet">
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: "Wix Madefor Text", system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #EFF5FD; padding: 20px; color: #1D2334; }
    .card { background: #FFFFFF; border: 1px solid #E6ECF4; border-radius: 12px; padding: 24px; max-width: 600px; margin: 0 auto; }
    h1 { font-size: 24px; font-weight: 400; margin-bottom: 8px; }
    .badge { display: inline-block; background: #DCFCE7; color: #00602D; padding: 2px 8px; border-radius: 12px; font-size: 12px; font-weight: 500; vertical-align: middle; }
    .info { color: #43739E; font-size: 14px; margin-top: 12px; }
    .info p { margin-bottom: 4px; }
    .btn-primary { background: #0E68E2; color: #FFFFFF; border: none; border-radius: 8px; padding: 8px 16px; font-family: inherit; font-size: 14px; font-weight: 500; cursor: pointer; }
    .btn-primary:hover { background: #083C82; }
  </style>
</head>
<body>
  <div class="card">
    <h1>${APP_CONFIG.name} <span class="badge">Connected</span></h1>
    <div class="info">
      <p><strong>Session:</strong> ${sessionKey ? 'Active' : 'None'}</p>
      ${session ? '<p><strong>User:</strong> ' + (session.user?.email || session.user?.id || 'Unknown') + '</p>' : ''}
      ${session ? '<p><strong>Practice:</strong> ' + (session.practice?.name || session.practice?.id || 'Unknown') + '</p>' : ''}
    </div>
    <div id="app" style="margin-top: 20px;">
      <!-- APP UI GOES HERE -->
    </div>
  </div>
  <script src="${HINT_API_URL}/hint-sdk.js"></script>
  <script>
    const SESSION_KEY = '${sessionKey || ''}';
    if (typeof HintSDK !== 'undefined') {
      HintSDK.init(() => console.log('HintSDK ready, user:', HintSDK.user));
    }
  </script>
</body>
</html>`;
}

function renderClinicalInteraction(sessionKey, session) {
  return `<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>${APP_CONFIG.name}</title>
  <link href="https://fonts.googleapis.com/css2?family=Wix+Madefor+Text:wght@400;500;700&display=swap" rel="stylesheet">
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: "Wix Madefor Text", system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #EFF5FD; padding: 16px; color: #1D2334; }
    .card { background: #FFFFFF; border: 1px solid #E6ECF4; border-radius: 12px; padding: 20px; }
    h2 { font-size: 18px; font-weight: 400; margin-bottom: 8px; }
    .patient-info { background: #EFF5FD; border: 1px solid #C4DCF8; border-radius: 8px; padding: 12px; margin-bottom: 16px; }
    .patient-info .label { font-size: 11px; color: #0E68E2; text-transform: uppercase; font-weight: 500; letter-spacing: 0.5px; }
    .patient-info .name { font-size: 16px; font-weight: 700; margin-top: 2px; color: #1D2334; }
    .info { color: #43739E; font-size: 12px; }
  </style>
</head>
<body>
  <div class="card">
    <h2>${APP_CONFIG.name}</h2>
    <div class="patient-info" id="patient-info">
      <div class="label">Patient</div>
      <div class="name" id="patient-name">Loading...</div>
    </div>
    <div id="app">
      <!-- CLINICAL INTERACTION UI GOES HERE -->
    </div>
    <div class="info">
      ${session ? '<p>User: ' + (session.user?.email || 'Unknown') + ' | Practice: ' + (session.practice?.name || 'Unknown') + '</p>' : ''}
    </div>
  </div>
  <script src="${HINT_API_URL}/hint-sdk.js"></script>
  <script>
    const SESSION_KEY = '${sessionKey || ''}';
    function updatePatient(patient) {
      document.getElementById('patient-name').textContent = patient ? (patient.name || patient.id) : 'No patient selected';
    }
    if (typeof HintSDK !== 'undefined') {
      HintSDK.init(() => { updatePatient(HintSDK.currentPatient); });
      HintSDK.onCurrentPatientChanged(updatePatient);
    }
  </script>
</body>
</html>`;
}

function renderSettings(sessionKey, session) {
  return `<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>${APP_CONFIG.name} Settings</title>
  <link href="https://fonts.googleapis.com/css2?family=Wix+Madefor+Text:wght@400;500;700&display=swap" rel="stylesheet">
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: "Wix Madefor Text", system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #FFFFFF; padding: 24px; color: #1D2334; }
    h1 { font-size: 20px; font-weight: 500; margin-bottom: 8px; }
    .subtitle { color: #43739E; font-size: 13px; margin-bottom: 24px; }
    .field { margin-bottom: 16px; }
    label { display: block; font-size: 13px; font-weight: 500; margin-bottom: 6px; color: #1D2334; }
    input[type="text"], input[type="password"] { width: 100%; padding: 8px 10px; border: 1px solid #C4DCF8; border-radius: 6px; font-size: 14px; font-family: inherit; }
    button { background: #0E68E2; color: #FFFFFF; border: 0; border-radius: 6px; padding: 8px 16px; font-size: 14px; font-family: inherit; font-weight: 500; cursor: pointer; }
    button:hover { background: #0851B8; }
    .info { color: #43739E; font-size: 12px; margin-top: 24px; }
  </style>
</head>
<body>
  <h1>${APP_CONFIG.name} Settings</h1>
  <div class="subtitle">Configure how ${APP_CONFIG.name} works for this practice.</div>
  <div id="app">
    <!-- SETTINGS UI GOES HERE — partner-specific configuration: API keys, feature flags, etc. -->
    <div class="field">
      <label>Example setting</label>
      <input type="text" placeholder="Replace with real settings inputs">
    </div>
    <button onclick="alert('Save not yet wired')">Save</button>
  </div>
  <div class="info">
    ${session ? '<p>Practice: ' + (session.practice?.name || 'Unknown') + '</p>' : ''}
  </div>
  <script src="${HINT_API_URL}/hint-sdk.js"></script>
</body>
</html>`;
}

server.listen(port, () => console.log(APP_CONFIG.name + ' listening on ' + port));
```

### Customizing the Template

Based on the user's app description:

1. Replace `APP_NAME_HERE` with the app name
2. Replace `SURFACE_TYPE_HERE` with `core_page`, `clinical_interaction`, or `settings`
3. Customize the matching renderer (`renderCorePage`, `renderClinicalInteraction`, or `renderSettings`) with the app's actual UI
4. Add app-specific API routes (e.g. `/api/pets`, `/api/messages`) in the marked section
5. Add any app-specific client-side JavaScript in the HTML

### Hint Brand & Style Guidelines

All embedded apps should follow Hint's design system so they feel native within the portal. Apply these styles to the HTML renderers:

**Font:**
```css
font-family: "Wix Madefor Text", system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
```
Load via Google Fonts: `<link href="https://fonts.googleapis.com/css2?family=Wix+Madefor+Text:wght@400;500;700&display=swap" rel="stylesheet">`

**Colors:**

| Token | Hex | Use for |
|-------|-----|---------|
| Primary | `#0E68E2` | Buttons, links, active states |
| Primary hover | `#083C82` | Button hover |
| Primary active | `#052652` | Button press |
| Primary disabled | `#C1DAFB` | Disabled buttons |
| Background | `#FFFFFF` | Cards, page background |
| Surface | `#EFF5FD` | Table headers, secondary backgrounds |
| Text primary | `#1D2334` | Default body text |
| Text secondary | `#43739E` | Labels, secondary info |
| Text disabled | `#8A90A5` | Disabled text |
| Text on primary | `#FFFFFF` | Text on primary buttons |
| Border subtle | `#E6ECF4` | Table rows, card outlines |
| Border default | `#C2C7CF` | Input borders |
| Field outline | `#C4DCF8` | Input focus |
| Success text | `#00602D` | Success messages |
| Success bg | `#DCFCE7` | Success badges/alerts |
| Error text | `#CD0211` | Error messages |
| Error bg | `#FFE2E2` | Error badges/alerts |
| Warning text | `#973C00` | Warning messages |
| Warning bg | `#FEF3C6` | Warning badges/alerts |

**Typography scale:**

| Style | Weight / Size / Line height | Use for |
|-------|----------------------------|---------|
| heading-md | 400 / 24px / 32px | Page titles |
| heading-sm | 400 / 18px / 24px | Section headers, card headers |
| title-md-bold | 700 / 16px / 24px | Modal titles |
| title-sm | 500 / 14px / 20px | Column headers, button text |
| label-md | 500 / 14px / 16px | Form labels |
| body-md | 400 / 14px / 20px | Body text, descriptions |
| body-sm | 400 / 12px / 16px | Timestamps, metadata |

**Button styles:**
```css
.btn-primary {
  background: #0E68E2;
  color: #FFFFFF;
  border: none;
  border-radius: 8px;
  padding: 8px 16px;
  font-family: inherit;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
}
.btn-primary:hover { background: #083C82; }
.btn-primary:active { background: #052652; }
.btn-primary:disabled { background: #C1DAFB; cursor: default; }
```

**Card styles:**
```css
.card {
  background: #FFFFFF;
  border: 1px solid #E6ECF4;
  border-radius: 12px;
  padding: 24px;
}
```

**Input styles:**
```css
input, select, textarea {
  border: 1px solid #C4DCF8;
  border-radius: 8px;
  padding: 8px 12px;
  font-family: inherit;
  font-size: 14px;
  color: #1D2334;
}
input:focus { outline: none; border-color: #0E68E2; box-shadow: 0 0 0 3px rgba(14, 104, 226, 0.25); }
```

Use these styles in the HTML renderers instead of the placeholder styles in the base template. The app should look and feel like a native part of Hint.

### Environment Variables (auto-injected)

- `PORT` — listen port (set by the platform)
- `HINT_API_URL` — Hint API base URL (set to the hint-api URL the partner called during service creation)
- `HINT_API_KEY` — partner API key for calling Hint APIs
- `HINT_PARTNER_ID` — partner public ID
- `HINT_WEBHOOK_SECRET` — the partner's webhook signature key, used to verify HMAC signatures on handshake payloads

### Hint Provider API

The access token from handshake/connect gives the app access to practice data. Key endpoints:

- `GET /api/provider/patients` — list patients
- `GET /api/provider/patients/:id` — get patient details
- `GET /api/provider/memberships` — list memberships
- `GET /api/provider/practitioners` — list practitioners
- `GET /api/provider/charges` — list charges
- `GET /api/provider/invoices` — list invoices

All require `Authorization: Bearer <access_token>` (the practice-scoped token from handshake, NOT the partner API key).

**Response shape gotchas — read these before writing client code:**

- **List endpoints return a bare JSON array**, not `{patients: [...]}` or `{data: [...]}`. Parse with `Array.isArray(res) ? res : []` and iterate directly. Do NOT do `res.patients || res.records || []` — that silently produces an empty array.
- **Pagination uses `limit` + `offset`** (not `page` / `page_size`). Example: `GET /api/provider/patients?limit=50&offset=100`. Max `limit` is 100. If a response returns exactly `limit` rows, paginate.
- **Archived rows are excluded by default** on list endpoints. To list only archived rows, pass `?filter=archived`. To list both active and archived in one call, pass `?filter=all`. If the app shows "no records" and the practice expects to see some, archive state is the first thing to check.
- **Sandbox vs live is host-level, not key-level.** Sandbox traffic goes to `https://api.sandbox.hint.com`, live traffic goes to `https://api.hint.com`. Sandbox API keys (prefix `sbx-`) only work against the sandbox host; live keys only work against the live host. Pick the host based on the API key the practice's app is configured with — they're not interchangeable.

Full API reference: https://developers.hint.com/reference

### Hint JS SDK

Available in the embedded iframe:

```html
<!-- Use $HINT_API_URL (api.sandbox.hint.com for sandbox, api.hint.com for live) -->
<script src="$HINT_API_URL/hint-sdk.js"></script>
<script>
  HintSDK.init(() => {
    console.log('User:', HintSDK.user);           // { id, name, email, partner_roles }
    console.log('Patient:', HintSDK.currentPatient); // { id, name } or null
    console.log('Interaction:', HintSDK.interaction); // { id } or null
  });
  HintSDK.onCurrentPatientChanged((patient) => {
    // Update UI when the selected patient changes
  });
</script>
```

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

The reserved env vars `HINT_API_URL`, `HINT_API_KEY`, `HINT_PARTNER_ID`, `HINT_WEBHOOK_SECRET`, and `DATABASE_URL` are managed by Hint and always present — partner-supplied values for those keys are ignored.

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

Once status is `pushed`, get the service URL. The services list is also a bare array; it can contain multiple rows (e.g. an auto-provisioned Postgres alongside the web service, plus the occasional stub from a prior provision attempt). **Select the web service that has a non-null `service_url` and `status: "active"`** — don't just take the first row:

```bash
curl -s "$HINT_API_URL/api/partner/app/services" \
  -H "Authorization: Bearer $API_KEY" \
  | python3 -c "import sys,json; print(next((s['service_url'] for s in json.load(sys.stdin) if s.get('service_url') and s.get('status')=='active'),''))"
```

Save the resulting URL as `$APP_URL`. Then poll it directly until it returns 200 (the build is usually live within a few seconds of `status: pushed`):

```bash
curl -s -o /dev/null -w '%{http_code}' $APP_URL/
```

Start at a **5-second interval, fall back to 10 seconds** if not live by the second poll. Cap at 5 minutes. Once the health check responds with a 200, the app is live — the service URL is the source of truth.

If the revision flips to `status: failed`, the platform refused the deploy (typical reasons: partner not yet approved for production deploys; partner product type is not `app`; push failure). Contact Hint support (devsupport@hint.com) with the revision id.

Save the live URL as `$APP_URL` and skip past Self-Hosted Mode to **Step 6: Configure Marketplace Settings**.

---

# Self-Hosted Mode

The partner runs the app on their own infrastructure. The skill doesn't build, package, or deploy code — it confirms the marketplace contract is in place and registers the partner's existing URLs with Hint. Skip this entire section if the user picked hosted.

## Step 3: Confirm the Marketplace Contract

The partner's deployed app must already implement (or be willing to implement) three routes. Ask the partner to confirm each is in place, or offer to walk them through what each one needs to do:

| Route | What it does |
|---|---|
| `POST /hint/handshake` | Receives a signed payload from Hint at install/embed time. The app verifies the `X-Hint-Signature` header (HMAC-SHA256 of the request body, key = the partner's webhook secret), mints a session key, and returns it. |
| `POST /hint/connect/:code` | Receives an OAuth code from Hint after a practice installs. The app exchanges the code at `POST $HINT_API_URL/api/oauth/tokens` for a practice-scoped API token and persists it. |
| `GET /hint/<anchor_type>?session_key=...` | Renders the embedded UI for that surface. Looks up the session by `session_key` and the practice context that was set up during handshake. |

The partner's app also needs to know two pieces of environment config Hint provides at install time:

- `HINT_API_URL` — the base URL of the Hint API. Set to `https://api.sandbox.hint.com` while testing with a sandbox API key (`sbx-...`), and `https://api.hint.com` once promoted to live. The two hosts are not interchangeable.
- `HINT_WEBHOOK_SECRET` — used to verify the `X-Hint-Signature` header on every `POST /hint/handshake`. The partner finds this in the Partner Portal under **API Keys → Webhooks Signature Key**.

If the partner's app is missing one of those routes, point them at the **Hosted Mode** server.js template above as a reference implementation — they can copy the handshake-verification + OAuth-exchange code straight across. They don't have to use Node.js; they just need an HTTP server that implements the three routes.

## Step 4: Gather the Partner's Deployed URL

Ask the partner: **what's the base URL where your app is deployed?** Examples: `https://patient-portal.acme.com`, `https://acme-marketplace.vercel.app`, `https://10.0.0.4:8080`.

Validate by hitting the partner's URL — if any of these returns something other than 200/2xx, the URLs don't match what's deployed:

```bash
curl -sS -o /dev/null -w "GET /  → HTTP %{http_code}\n" "$APP_URL/"
curl -sS -o /dev/null -w "POST /hint/handshake (unsigned, expect 401) → HTTP %{http_code}\n" -X POST "$APP_URL/hint/handshake"
curl -sS -o /dev/null -w "GET  /hint/<anchor_type> (no session, expect 200/401) → HTTP %{http_code}\n" "$APP_URL/hint/<anchor_type>"
```

A 401 on `/hint/handshake` is the correct response to an unsigned request — that confirms signature verification is wired up. A 200 or 404 there is a red flag.

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
- **403 on Partner API** — The API key may not have the right permissions.
- **Handshake fails with 401** — The webhook secret may not be configured correctly on the deployed service. The partner finds it in the Partner Portal under API Keys → Webhooks Signature Key.
- **Headless connect fails** — The API URL env var may not point to the correct Hint API instance.
- **Embedded page doesn't load** — Verify the anchor exists and the `source_url` matches `$APP_URL` + the correct route for the surface type.

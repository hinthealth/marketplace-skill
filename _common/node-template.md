# Node.js Reference Implementation (shared fragment)

The canonical Node.js implementation of the Hint marketplace contract. `create-app` ships this verbatim for new apps; `retrofit` ports the logic into whatever stack the partner already uses. The contract itself (routes, signature verification, tenancy rules) lives in [`marketplace-contract.md`](./marketplace-contract.md) — this file is the working code that implements it.

**Hosted Mode is Node-only today.** The deploy platform runs `node server.js` literally — the entry file must be `server.js`, and `package.json` must define both `build` and `start` scripts. Other stacks can be deployed as self-hosted apps.

## `package.json`

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

## `server.js`

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
// MULTI-TENANCY — READ THIS BEFORE ADDING ANY PERSISTED STATE
//
// This app is a single deployed service that EVERY practice installing
// it shares. The handshake payload identifies the practice making each
// request — and this app is responsible for using that identity to keep
// every practice's data isolated from every other practice's.
//
// Rules:
//   1. Every table that stores tenant data MUST have a non-null
//      `practice_id` column.
//   2. Every read AND write MUST filter by `practice_id` sourced from
//      the current session.
//   3. Practice access tokens (used to call /api/provider/*) MUST come
//      from the current session's `practice_id` — never reuse a token
//      from a different practice.
//
// Use `requireSession(req, res)` below for every handler that touches
// persisted state. It returns `{ practice_id, access_token, user }`
// for the current request, or null (and writes 401) if no valid session.
// ============================================================

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
// dependencies) and write a sessions table on boot. Schema sketch:
//
//   CREATE TABLE sessions (
//     session_key  TEXT PRIMARY KEY,
//     practice_id  TEXT NOT NULL,
//     access_token TEXT,
//     user_data    JSONB,
//     created_at   TIMESTAMPTZ DEFAULT NOW()
//   );
//   CREATE INDEX ON sessions(practice_id);
//
// And for any app-specific table:
//
//   CREATE TABLE messages (
//     id          SERIAL PRIMARY KEY,
//     practice_id TEXT NOT NULL,     -- MANDATORY for tenant data
//     body        TEXT,
//     author_id   TEXT,
//     created_at  TIMESTAMPTZ DEFAULT NOW()
//   );
//   CREATE INDEX ON messages(practice_id);
// ============================================================
const sessions = {};

// Practice-scoped access tokens, keyed by practice_id (from /hint/connect/:code).
const practiceTokens = {};

function requireSession(req, res) {
  const url = new URL(req.url, `http://localhost:${port}`);
  const sessionKey = req.headers['x-hint-session-key'] || url.searchParams.get('session_key');
  const session = sessionKey ? sessions[sessionKey] : null;
  if (!session) {
    res.writeHead(401, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'No session' }));
    return null;
  }
  // Attach the practice's access_token so handlers can call /api/provider/*.
  return { ...session, access_token: practiceTokens[session.practice_id] || null };
}

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
    // Persist the session keyed by sessionKey AND remember practice_id so
    // every subsequent handler can scope reads/writes to it.
    const sessionKey = crypto.randomUUID();
    sessions[sessionKey] = {
      practice_id: parsed.practice?.id,        // ← the tenancy anchor; treat as required
      user: parsed.user,
      practice: parsed.practice,
      integration: parsed.integration,
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
    try {
      if (HINT_API_URL && HINT_API_KEY) {
        const resp = await hintApi('POST', '/api/oauth/tokens', {
          code: authCode,
          grant_type: 'authorization_code',
        });
        // The access_token is practice-scoped — persist it keyed by practice_id
        // so we use the right token per request. The embedded surface looks
        // this up via requireSession() → practiceTokens[session.practice_id].
        if (resp.body?.practice_id && resp.body?.access_token) {
          practiceTokens[resp.body.practice_id] = resp.body.access_token;
        }
      }
    } catch (err) {
      console.error('Connect error:', err.message);
    }
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({ status: 'connected' }));
  }

  // ============================================================
  // APP-SPECIFIC ROUTES — add your routes here
  //
  // Every handler that touches tenant data must call requireSession(req, res)
  // and scope queries by session.practice_id. Example:
  //
  //   if (req.method === 'GET' && url.pathname === '/api/messages') {
  //     const session = requireSession(req, res); if (!session) return;
  //     // SELECT * FROM messages WHERE practice_id = $1 ORDER BY created_at DESC
  //     const rows = await db.query(
  //       'SELECT * FROM messages WHERE practice_id = $1 ORDER BY created_at DESC',
  //       [session.practice_id],
  //     );
  //     res.writeHead(200, { 'Content-Type': 'application/json' });
  //     return res.end(JSON.stringify(rows));
  //   }
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

## Customizing the template

1. Replace `APP_NAME_HERE` with the app name.
2. Replace `SURFACE_TYPE_HERE` with `core_page`, `clinical_interaction`, or `settings`.
3. Customize the matching renderer (`renderCorePage`, `renderClinicalInteraction`, or `renderSettings`) with the app's actual UI. Use the tokens from [`brand-styles.md`](./brand-styles.md) so the embedded surface looks native inside Hint.
4. Add app-specific API routes (e.g. `/api/messages`, `/api/pets`) in the marked section. Every handler that touches tenant data MUST call `requireSession(req, res)` and scope queries by `session.practice_id`.
5. Add any app-specific client-side JavaScript in the HTML.

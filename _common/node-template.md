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
  },
  "dependencies": {
    "pg": "^8.13.0"
  }
}
```

The `pg` dependency is included by default — Hosted Mode auto-provisions a sibling Postgres, and the template uses it as the session + practice-token store (see below). Self-Hosted Mode apps can swap in their own store, but `pg` is the path of least resistance.

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
// Session store (Postgres)
//
// Sessions + practice access tokens MUST be persisted across processes.
// In Hosted Mode the container handling /hint/connect/:code is often a
// different process from the one serving /hint/<surface> a few seconds
// later (rolling deploys, restarts, multi-process workers). An in-memory
// map breaks the very first install with "Practice has not completed
// headless connect yet" because the connect-side process wrote a token
// the render-side process can't see.
//
// Hosted Mode auto-provisions a sibling Postgres and sets DATABASE_URL.
// For Self-Hosted apps, set DATABASE_URL yourself (any Postgres works).
//
// For app-specific tenant data, add tables with a non-null practice_id:
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
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Ensure the sessions table exists on boot.
//
// DATABASE_URL may not be reachable on the very first deploy — the sibling
// Postgres can still be in provisioning when the web service starts serving.
// Retry briefly instead of crashing; the server keeps booting and accepting
// health checks while we wait for Postgres.
async function ensureSchema(attempt = 1) {
  try {
    await pool.query(`
      CREATE TABLE IF NOT EXISTS sessions (
        session_key  TEXT PRIMARY KEY,
        practice_id  TEXT NOT NULL,
        user_data    JSONB,
        created_at   TIMESTAMPTZ DEFAULT NOW()
      );
    `);
    await pool.query('CREATE INDEX IF NOT EXISTS sessions_practice_id_idx ON sessions(practice_id);');
    // practice_tokens is keyed by practice_id (NOT session_key) so the connect
    // handler can write the token even when no session row exists yet. In
    // managed-hosted mode Hint POSTs /hint/connect/:code BEFORE the user opens
    // the iframe (handshake), so writing the token onto a not-yet-existing
    // session row would be a silent no-op. getSession() joins this table.
    await pool.query(`
      CREATE TABLE IF NOT EXISTS practice_tokens (
        practice_id  TEXT PRIMARY KEY,
        access_token TEXT NOT NULL,
        updated_at   TIMESTAMPTZ DEFAULT NOW()
      );
    `);
    console.log('Sessions schema ready');
  } catch (err) {
    if (attempt >= 5) {
      console.error('Could not initialize sessions schema after 5 attempts:', err.message);
      return;
    }
    console.warn(`Postgres not ready (attempt ${attempt}): ${err.message} — retrying in 2s`);
    setTimeout(() => ensureSchema(attempt + 1), 2000);
  }
}
ensureSchema();

async function saveSession(sessionKey, practiceId, userData) {
  await pool.query(
    `INSERT INTO sessions (session_key, practice_id, user_data)
     VALUES ($1, $2, $3)
     ON CONFLICT (session_key) DO UPDATE
       SET practice_id = EXCLUDED.practice_id, user_data = EXCLUDED.user_data`,
    [sessionKey, practiceId, userData],
  );
}

async function saveAccessToken(practiceId, accessToken) {
  // Upsert keyed by practice_id so this works regardless of whether a session
  // row exists yet. Decouples /hint/connect/:code from /hint/handshake ordering.
  await pool.query(
    `INSERT INTO practice_tokens (practice_id, access_token)
     VALUES ($1, $2)
     ON CONFLICT (practice_id) DO UPDATE
       SET access_token = EXCLUDED.access_token, updated_at = NOW()`,
    [practiceId, accessToken],
  );
}

async function getSession(sessionKey) {
  const { rows } = await pool.query(
    `SELECT s.session_key, s.practice_id, s.user_data, t.access_token
       FROM sessions s
       LEFT JOIN practice_tokens t ON t.practice_id = s.practice_id
      WHERE s.session_key = $1`,
    [sessionKey],
  );
  return rows[0] || null;
}

async function requireSession(req, res) {
  const url = new URL(req.url, `http://localhost:${port}`);
  const sessionKey = req.headers['x-hint-session-key'] || url.searchParams.get('session_key');
  const session = sessionKey ? await getSession(sessionKey) : null;
  if (!session) {
    res.writeHead(401, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'No session' }));
    return null;
  }
  return session; // { session_key, practice_id, access_token, user_data }
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
  // Hint signs the handshake body with the backend's **webhooks signature key**
  // (visible in Partner Portal → Webhook Settings → Webhooks Signature Key). The Hosted
  // Mode deploy platform injects this as `HINT_WEBHOOK_SECRET`. If you ever
  // rotate that key in the portal, re-push env vars via
  // `PATCH /api/partner/products/:product_id/app/services/:id` so the deployed service picks up the
  // new value — otherwise the env var goes stale and every handshake fails.
  if (!HINT_WEBHOOK_SECRET || !signature) {
    console.error('[handshake] verification skipped:', {
      hasSecret: Boolean(HINT_WEBHOOK_SECRET),
      hasSignature: Boolean(signature),
    });
    return false;
  }
  const expected = 'sha256=' + crypto.createHmac('sha256', HINT_WEBHOOK_SECRET).update(rawBody).digest('hex');
  try {
    const ok = crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
    if (!ok) {
      // Log enough to debug without leaking the secret itself.
      console.error('[handshake] signature mismatch:', {
        bodyBytes: Buffer.byteLength(rawBody, 'utf8'),
        receivedSigPrefix: String(signature).slice(0, 14),
        expectedSigPrefix: expected.slice(0, 14),
        secretLast4: HINT_WEBHOOK_SECRET.slice(-4),
        hint: 'Compare secretLast4 with the last 4 chars of Webhooks Signature Key in Partner Portal → Webhook Settings. If they differ, the env var is stale (re-push via PATCH /partner/products/:product_id/app/services/:id).',
      });
    }
    return ok;
  } catch (err) {
    console.error('[handshake] verifier threw:', err.message);
    return false;
  }
}

// Partner-wide calls (server-to-server) — uses the static HINT_API_KEY.
// Examples: /api/partner/*, /api/oauth/tokens.
function hintApi(method, path, body) {
  return hintApiWith(`Bearer ${HINT_API_KEY}`, method, path, body, { parseJson: true });
}

// Practice-scoped calls — uses the per-practice access_token from the
// /hint/connect/:code OAuth flow. Use this for any /api/provider/* call;
// the access_token lives on the session row (see requireSession).
function hintApiAs(accessToken, method, path, body) {
  return hintApiWith(`Bearer ${accessToken}`, method, path, body, { parseJson: false });
}

function hintApiWith(authHeader, method, path, body, { parseJson }) {
  return new Promise((resolve, reject) => {
    const url = new URL(HINT_API_URL + path);
    const proto = url.protocol === 'https:' ? https : http;
    const options = {
      hostname: url.hostname,
      port: url.port || (url.protocol === 'https:' ? 443 : 80),
      path: url.pathname + url.search,
      method,
      headers: {
        'Authorization': authHeader,
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    };
    const req = proto.request(options, (res) => {
      let data = '';
      res.on('data', chunk => data += chunk);
      res.on('end', () => {
        if (parseJson) {
          try { resolve({ status: res.statusCode, headers: res.headers, body: JSON.parse(data) }); }
          catch { resolve({ status: res.statusCode, headers: res.headers, body: data }); }
        } else {
          // Proxy use case — pass the body through verbatim so the client gets
          // the same payload it would have received calling Hint directly.
          resolve({ status: res.statusCode, headers: res.headers, body: data });
        }
      });
    });
    req.on('error', reject);
    if (body) req.write(typeof body === 'string' ? body : JSON.stringify(body));
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
    await saveSession(sessionKey, parsed.practice?.id, {
      user: parsed.user,
      practice: parsed.practice,
      integration: parsed.integration,
    });
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({ session_key: sessionKey }));
  }

  // Core page — full-page embedded UI (for core_page surface)
  if (req.method === 'GET' && url.pathname === '/hint/core_page') {
    const sessionKey = url.searchParams.get('session_key');
    const session = sessionKey ? await getSession(sessionKey) : null;
    res.writeHead(200, { 'Content-Type': 'text/html' });
    return res.end(renderCorePage(sessionKey, session));
  }

  // Clinical interaction — patient-context embedded UI (for clinical_interaction surface)
  if (req.method === 'GET' && url.pathname === '/hint/clinical_interaction') {
    const sessionKey = url.searchParams.get('session_key');
    const session = sessionKey ? await getSession(sessionKey) : null;
    res.writeHead(200, { 'Content-Type': 'text/html' });
    return res.end(renderClinicalInteraction(sessionKey, session));
  }

  // Settings — embedded inside the practice settings area (for settings surface)
  if (req.method === 'GET' && url.pathname === '/hint/settings') {
    const sessionKey = url.searchParams.get('session_key');
    const session = sessionKey ? await getSession(sessionKey) : null;
    res.writeHead(200, { 'Content-Type': 'text/html' });
    return res.end(renderSettings(sessionKey, session));
  }

  // Headless connect — Hint POSTs auth code during installation.
  //
  // The response body's `access_token` is the practice-scoped key the app
  // uses to call `/api/provider/*` endpoints. It is NOT the same as the
  // partner API key in HINT_API_KEY (which is partner-wide).
  //
  // We persist the access_token onto every session row for this practice so
  // the embedded surface can authenticate Provider API calls on every render
  // (requireSession() returns the row with access_token already attached).
  if (req.method === 'POST' && url.pathname.startsWith('/hint/connect/')) {
    const authCode = url.pathname.replace('/hint/connect/', '');
    try {
      if (HINT_API_URL && HINT_API_KEY) {
        const resp = await hintApi('POST', '/api/oauth/tokens', {
          code: authCode,
          grant_type: 'authorization_code',
        });
        // The OAuth response nests practice as an object: { practice: { id, name }, access_token, ... }
        const practiceId = resp.body?.practice?.id;
        const accessToken = resp.body?.access_token;
        if (practiceId && accessToken) {
          await saveAccessToken(practiceId, accessToken);
        } else {
          console.warn('[connect] OAuth response missing practice.id or access_token:', JSON.stringify({
            status: resp.status,
            keys: Object.keys(resp.body || {}),
          }));
        }
      }
    } catch (err) {
      console.error('Connect error:', err.message);
    }
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({ status: 'connected' }));
  }

  // ============================================================
  // PROVIDER API PROXY — /hint/api/provider/*
  //
  // Forwards browser-side calls to Hint's /api/provider/* with the practice's
  // session access_token attached server-side. Client code NEVER sees the
  // access_token — fetch('/hint/api/provider/patients') from the embedded UI
  // is the right pattern; fetch('https://api.hint.com/api/provider/patients')
  // (direct, unauthed) is wrong and will 401.
  //
  // The client just needs to send the session_key (in the x-hint-session-key
  // header or as a ?session_key=... query param) so this server can look up
  // the access_token on the session row.
  // ============================================================
  if (url.pathname.startsWith('/hint/api/provider/')) {
    const session = await requireSession(req, res); if (!session) return;
    if (!session.access_token) {
      res.writeHead(503, { 'Content-Type': 'application/json' });
      return res.end(JSON.stringify({
        error: 'Practice has not completed headless connect yet — try again in a moment.',
      }));
    }
    // /hint/api/provider/patients?limit=10 → /api/provider/patients?limit=10
    const targetPath = url.pathname.replace('/hint/api', '/api') + (url.search || '');
    let forwardBody = '';
    if (req.method !== 'GET' && req.method !== 'HEAD') {
      const parsed = await parseBody(req);
      forwardBody = parsed.raw;
    }
    try {
      const upstream = await hintApiAs(session.access_token, req.method, targetPath, forwardBody);
      // Pass through the upstream status + Content-Type so the client sees the
      // same response shape it would get calling Hint directly.
      const contentType = (upstream.headers && upstream.headers['content-type']) || 'application/json';
      res.writeHead(upstream.status, { 'Content-Type': contentType });
      return res.end(upstream.body);
    } catch (err) {
      console.error('[hint-api proxy] forward failed:', err.message);
      res.writeHead(502, { 'Content-Type': 'application/json' });
      return res.end(JSON.stringify({ error: 'Upstream Hint API request failed' }));
    }
  }

  // ============================================================
  // APP-SPECIFIC ROUTES — add your routes here
  //
  // Every handler that touches tenant data must call requireSession(req, res)
  // and scope queries by session.practice_id. Example:
  //
  //   if (req.method === 'GET' && url.pathname === '/api/messages') {
  //     const session = await requireSession(req, res); if (!session) return;
  //     const { rows } = await pool.query(
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
  // Three render states the embedded UI has to handle:
  //   - ready:      session row + access_token present → render the real app
  //   - connecting: session row present but access_token still missing
  //                 (happens for ~seconds-to-minutes between handshake and the
  //                 connect callback persisting the token; the UI auto-retries)
  //   - no-session: no session row at all (the user opened the surface URL
  //                 directly, or the session expired)
  const userData = session?.user_data || {};
  const ready = Boolean(session && session.access_token);
  const connecting = Boolean(session && !session.access_token);
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
    .badge-pending { background: #FEF3C7; color: #92400E; }
    .info { color: #43739E; font-size: 14px; margin-top: 12px; }
    .info p { margin-bottom: 4px; }
    .pending { color: #43739E; font-size: 14px; margin-top: 12px; line-height: 1.5; }
    .btn-primary { background: #0E68E2; color: #FFFFFF; border: none; border-radius: 8px; padding: 8px 16px; font-family: inherit; font-size: 14px; font-weight: 500; cursor: pointer; }
    .btn-primary:hover { background: #083C82; }
  </style>
</head>
<body>
  <div class="card">
    ${ready ? `
      <h1>${APP_CONFIG.name} <span class="badge">Connected</span></h1>
      <div class="info">
        <p><strong>User:</strong> ${userData.user?.email || userData.user?.id || 'Unknown'}</p>
        <p><strong>Practice:</strong> ${userData.practice?.name || userData.practice?.id || 'Unknown'}</p>
      </div>
      <div id="app" style="margin-top: 20px;">
        <!-- APP UI GOES HERE -->
      </div>
    ` : connecting ? `
      <h1>${APP_CONFIG.name} <span class="badge badge-pending">Finishing setup</span></h1>
      <p class="pending">
        We're finalizing your connection. This usually takes a few seconds — the page
        will refresh automatically once setup completes.
      </p>
    ` : `
      <h1>${APP_CONFIG.name}</h1>
      <p class="pending">
        No active session. Open this app from inside Hint to start.
      </p>
    `}
  </div>
  <script src="${HINT_API_URL}/hint-sdk.js"></script>
  <script>
    const SESSION_KEY = '${sessionKey || ''}';
    const CONNECTING = ${connecting};
    if (typeof HintSDK !== 'undefined') {
      HintSDK.init(() => console.log('HintSDK ready, user:', HintSDK.user));
    }
    // Auto-retry while the access token is propagating from /hint/connect/:code
    // into the session. Stops once the token lands and the next render is ready.
    if (CONNECTING) {
      setTimeout(() => window.location.reload(), 3000);
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
      ${session ? '<p>User: ' + (session.user_data?.user?.email || 'Unknown') + ' | Practice: ' + (session.user_data?.practice?.name || 'Unknown') + '</p>' : ''}
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
    ${session ? '<p>Practice: ' + (session.user_data?.practice?.name || 'Unknown') + '</p>' : ''}
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

## Calling Hint's Provider API from the embedded UI

The `/hint/api/provider/*` proxy is built into the template — **always call Hint's Provider API through it from client-side code, never directly.**

```js
// ✅ RIGHT — client calls the local proxy, server injects auth, response forwarded back
async function fetchPatients() {
  const r = await fetch('/hint/api/provider/patients?limit=10', {
    headers: { 'x-hint-session-key': SESSION_KEY },
  });
  if (!r.ok) throw new Error('fetch failed: ' + r.status);
  return r.json();
}

// ❌ WRONG — direct call from the browser has no auth header, every request 401s
async function fetchPatientsBroken() {
  const r = await fetch(`${HINT_API_URL}/api/provider/patients?limit=10`);
  return r.json(); // → 401 Unauthorized
}
```

Why: the `access_token` for `/api/provider/*` is **practice-scoped** and lives on the session row (Postgres). The browser never has it (and shouldn't — exposing it would let any page on the embed origin act as the practice). The proxy looks the token up from `session.access_token` server-side, forwards the upstream call with `Authorization: Bearer <access_token>`, and returns the response. Path mapping is straight: `/hint/api/provider/X` → `https://api.hint.com/api/provider/X`. Query strings, request bodies, and HTTP methods all pass through unchanged.

When the proxy returns **`503 "Practice has not completed headless connect yet"`**, it means the session row exists but the access_token hasn't landed yet — the connect callback is still in flight. Retry once after a couple of seconds.

For server-to-server calls (background jobs, webhook handlers) that don't have a practice session, use `hintApi(method, path, body)` (server-side helper, partner-wide `HINT_API_KEY`) instead. The proxy is for in-browser code that's acting on behalf of the currently-loaded practice.

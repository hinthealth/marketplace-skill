# Caching patterns for dashboard-style marketplace apps

**Read this before building any Core Page app that summarizes data across a panel of patients.** Without a cache, every visit re-fetches the full panel through `/api/provider/*` — for a ~80-member panel that's an 8–25 s cold load and a rate-limit hazard. With the patterns below, every visit renders instantly from a cached snapshot, fans out a delta refresh in the background, and only pays the full fetch cost once per practice per day.

The patterns are validated by the Panel Labs marketplace app (panel-wide lab-health Core Page surfacing latest A1C / lipid / TSH per active member with severity-sorted worklist + cohort filtering + trajectory sparklines) — built end-to-end on the skill and shipped at scale.

---

## The five patterns

### 1. Snapshot table (Postgres-backed)

The skill template already auto-provisions a Postgres sibling and exposes it via `process.env.DATABASE_URL`. Reuse it. One table per "thing the dashboard summarizes":

```sql
CREATE TABLE IF NOT EXISTS panel_snapshots (
  practice_id      text PRIMARY KEY,
  -- the computed dashboard payload, ready to render
  panel            jsonb NOT NULL,
  -- cursor for the delta fetch below
  last_fetched_at  timestamptz NOT NULL,
  -- when the row was last written; used for the 24 h backstop
  refreshed_at     timestamptz NOT NULL DEFAULT NOW()
);
```

**Key the snapshot by `practice_id`, not by user / interaction id.** Every user inside a practice sees the same panel — caching per-user wastes memory and creates per-user staleness rules to reason about.

**Store the rendered payload (`panel` jsonb), not the raw API records.** The dashboard's job is to compute (severity sort, cohort buckets, trajectory bins). Doing that work once on write means every render is `SELECT panel FROM panel_snapshots WHERE practice_id = $1` → ship the jsonb back as-is.

### 2. Snapshot-first render — instant from cache, async refresh

On `GET /hint/core_page` (the embedded page handler):

```js
const snap = await db.query(
  'SELECT panel, last_fetched_at FROM panel_snapshots WHERE practice_id = $1',
  [session.practice_id],
);

if (snap.rows[0]) {
  // Render immediately from cache — the user sees data instantly.
  res.send(renderPanel(snap.rows[0].panel));
  // Fire delta refresh asynchronously (don't await — the response is already sent).
  refreshPanelAsync(session.practice_id, snap.rows[0].last_fetched_at);
} else {
  // First visit ever for this practice — must full-fetch synchronously.
  const panel = await fullFetch(session.practice_id);
  await writeSnapshot(session.practice_id, panel, new Date());
  res.send(renderPanel(panel));
}
```

The client can `fetch('/api/panel/refresh')` on a short interval (or once on focus) to swap in any updates the async refresh produced.

### 3. Delta refresh — only fetch what changed

```js
async function refreshPanelAsync(practiceId, lastFetchedAt) {
  // updated_at[gt] returns only records modified after the cursor — usually 0–5
  // entries vs 100+ for a full panel fetch. See provider-api.md.
  const since = lastFetchedAt.toISOString();
  const newInteractions = await hintApi(
    `/api/provider/interactions?type=lab&updated_at[gt]=${since}`
  );
  if (newInteractions.length === 0) return; // nothing changed

  // Merge into the existing snapshot (recompute aggregates per affected member).
  const merged = await mergeIntoSnapshot(practiceId, newInteractions);
  await writeSnapshot(practiceId, merged, new Date());
}
```

`updated_at[gt]` works on `interactions`, `memberships`, `patients`, and similar mutable resources. **Stamp the cursor with the response's request time, not the latest record's `updated_at`** — using the latter creates a race where any record that's `updated_at == cursor` gets skipped on the next refresh.

### 4. 24 h backstop full fetch — guarantee eventual consistency

Delta refresh can miss records (transient API errors, detail-endpoint blips, clock skew). Once per day per practice, do a full refresh to backfill anything that slipped through:

```js
async function refreshPanelAsync(practiceId, lastFetchedAt) {
  const sinceLastFetch = Date.now() - lastFetchedAt.getTime();
  const TWENTY_FOUR_HOURS = 24 * 60 * 60 * 1000;

  if (sinceLastFetch > TWENTY_FOUR_HOURS) {
    return fullFetchAndWrite(practiceId);
  }

  // ... delta path from pattern #3
}
```

Don't shorten the 24 h window much — a 1 h backstop in production has every practice triggering a full fetch every hour, which fans out into a synchronized API hammer on the hint-api side. 24 h is the right balance between catching missed records and keeping the API quiet.

### 5. Global throttle via `pg_try_advisory_lock` — prevent stampede

When 50 practices simultaneously trip the 24 h backstop at midnight UTC, you get 50 concurrent full fetches against the Hint API, plus each one starts a worker thread inside a single Node process. Add a per-practice advisory lock so only one full fetch runs at a time, **and** a global one so the host process doesn't run more than N full fetches in parallel:

```js
async function fullFetchAndWrite(practiceId) {
  // Per-practice lock: if another worker is already fetching this practice's panel, skip.
  const perPracticeKey = hash32('panel_full_fetch:' + practiceId);
  const { rows: [{ pg_try_advisory_lock: gotLock }] } = await db.query(
    'SELECT pg_try_advisory_lock($1)',
    [perPracticeKey],
  );
  if (!gotLock) return; // another worker has it

  try {
    // Global lock — cap concurrent full-fetches across the whole host process.
    // (rotates through 4 buckets so up to 4 practices can full-fetch in parallel.)
    const bucket = perPracticeKey % 4;
    const globalKey = hash32('panel_full_fetch_global:' + bucket);
    await db.query('SELECT pg_advisory_lock($1)', [globalKey]); // BLOCKING — waits

    try {
      const panel = await fullFetch(practiceId);
      await writeSnapshot(practiceId, panel, new Date());
    } finally {
      await db.query('SELECT pg_advisory_unlock($1)', [globalKey]);
    }
  } finally {
    await db.query('SELECT pg_advisory_unlock($1)', [perPracticeKey]);
  }
}
```

`hash32` can be any 32-bit hash (e.g. `crc32`); advisory locks are keyed by int. The per-practice lock is `pg_try_advisory_lock` (non-blocking — skip if held); the global lock is `pg_advisory_lock` (blocking — queue). Tune the bucket count (`% 4` above) to the host process's CPU + the hint-api rate limits.

Without this, a 50-practice fleet that all tripped the 24 h backstop in the same minute will fan out to ~50 × ~100 concurrent detail fetches and trigger the rate-limit cascade described in [`provider-api.md` § rate-limited endpoints](./provider-api.md#rate-limited-endpoints--keep-detail-fetch-concurrency-low).

---

## Client-side: HTML fragment swap

For the "refresh on visit / focus / interval" UX, return rendered HTML fragments (not JSON) and `outerHTML`-swap them in. Simpler than SSE / WebSockets and works inside the Hint embed iframe without any extra plumbing:

```js
// In server.js — refresh endpoint returns rendered fragments
app.get('/api/panel/refresh', requireSession, async (req, res) => {
  const snap = await readSnapshot(req.session.practice_id);
  res.json({
    kpis: renderKpisHtml(snap.panel),
    worklist: renderWorklistHtml(snap.panel),
    sparklines: renderSparklinesHtml(snap.panel),
  });
});

// In the embedded HTML — swap fragments in on a short interval
async function refresh() {
  const r = await fetch('/api/panel/refresh').then(x => x.json());
  for (const [key, html] of Object.entries(r)) {
    document.getElementById(key).outerHTML = html;
  }
}
setInterval(refresh, 30_000);
document.addEventListener('visibilitychange', () => {
  if (!document.hidden) refresh();
});
```

The `outerHTML` swap preserves the wrapper element's id so subsequent swaps find it again. Test this on iOS Safari — the `visibilitychange` behavior differs from desktop browsers when the user backgrounds the embed.

---

## When to skip these patterns

- **Pure clinical_interaction surfaces** that scope to a single patient — the per-visit fetch cost is bounded and tiny; caching adds complexity without measurable benefit. Use the API direct.
- **Pure settings surfaces** — same reasoning. No panel, no fan-out.
- **Apps that don't read `/api/provider/interactions` or other panel-scale list endpoints** — if every request is keyed to the user's current patient, you're not fanning out.

Use these patterns when the dashboard summarizes data across the entire practice's patient panel. That's the workload that breaks without caching.

# Caching patterns for dashboard-style marketplace apps (advanced — read on-demand)

> **When to reach for this — don't reflexively bake these into v1.**
>
> The default for any marketplace app is "fetch on every render" using the template's `hintApi()` wrapper, which already retries on `429`. That works for the overwhelming majority of surfaces — single-patient `clinical_interaction` / `clinical_chart`, `settings` pages, and even most `core_page` apps that don't fan out across the whole practice.
>
> Reach for the patterns below **only** when ALL of the following are true:
>
> 1. The app summarizes data across a **practice-scale list endpoint** that fans out — typically `interactions`, `memberships`, `patients`, `customer_invoices`, or similar — and renders aggregates / sorts / filters across that whole set, not a single record.
> 2. The partner has already hit a real symptom in production: cold loads >8 s on a moderately-sized list response (~50+ records), `429` cascades when multiple users hit the surface concurrently, OR an explicit "please cache this" ask.
> 3. The simpler fixes don't apply: concurrency caps on detail fetches (per `provider-api.md` § Rate-limited endpoints), narrower delta queries via `updated_at[gt]`, or rendering progressively (show the first N rows instantly while the rest stream in).
>
> If 1–3 aren't all true, this complexity is a tax — Postgres schema, advisory locks, race conditions in the merge, a daily backstop to reason about, dual code paths for first-visit vs returning. Most apps that try to ship caching reflexively get one of these wrong and end up with stale data or a stampede on midnight UTC.

These patterns have been validated in production by a marketplace app summarizing list data across an entire practice. The shape generalizes — substitute your own resource (interactions, memberships, invoices, etc.) for the worked example.

---

## The five patterns

### 1. Snapshot table (Postgres-backed)

The skill template auto-provisions a Postgres sibling and exposes it via `process.env.DATABASE_URL`. Reuse it. One table per "thing the dashboard summarizes":

```sql
CREATE TABLE IF NOT EXISTS dashboard_snapshots (
  practice_id      text PRIMARY KEY,
  -- the computed dashboard payload, ready to render
  payload          jsonb NOT NULL,
  -- cursor for the delta fetch below
  last_fetched_at  timestamptz NOT NULL,
  -- when the row was last written; used for the daily backstop
  refreshed_at     timestamptz NOT NULL DEFAULT NOW()
);
```

Rename the table to suit your domain (e.g. `lab_snapshots`, `revenue_snapshots`, `outreach_snapshots`) when summarizing multiple resources independently. The shape stays the same.

**Key the snapshot by `practice_id`, not by user or record id.** Every user inside a practice sees the same dashboard — caching per-user wastes memory and creates per-user staleness rules to reason about.

**Store the rendered payload (`payload` jsonb), not the raw API records.** The dashboard's job is to compute (sort, bucket, aggregate). Doing that work once on write means every render is `SELECT payload FROM dashboard_snapshots WHERE practice_id = $1` → ship the jsonb back as-is.

### 2. Snapshot-first render — instant from cache, async refresh

On the embedded page handler (typically `GET /hint/core_page`):

```js
const snap = await db.query(
  'SELECT payload, last_fetched_at FROM dashboard_snapshots WHERE practice_id = $1',
  [session.practice_id],
);

if (snap.rows[0]) {
  // Render immediately from cache — the user sees data instantly.
  res.send(renderDashboard(snap.rows[0].payload));
  // Fire delta refresh asynchronously (don't await — the response is already sent).
  refreshSnapshotAsync(session.practice_id, snap.rows[0].last_fetched_at);
} else {
  // First visit ever for this practice — must full-fetch synchronously.
  const payload = await fullFetch(session.practice_id);
  await writeSnapshot(session.practice_id, payload, new Date());
  res.send(renderDashboard(payload));
}
```

The client can `fetch('/api/refresh')` on a short interval (or once on focus) to swap in any updates the async refresh produced.

### 3. Delta refresh — only fetch what changed

```js
async function refreshSnapshotAsync(practiceId, lastFetchedAt) {
  // updated_at[gt] returns only records modified after the cursor — usually
  // 0–5 entries vs 100+ for a full fetch. See provider-api.md § Delta queries.
  const since = lastFetchedAt.toISOString();
  const changed = await hintApi(
    `/api/provider/<resource>?updated_at[gt]=${since}`
  );
  if (changed.length === 0) return; // nothing changed

  // Merge into the existing snapshot (recompute aggregates per affected record).
  const merged = await mergeIntoSnapshot(practiceId, changed);
  await writeSnapshot(practiceId, merged, new Date());
}
```

`updated_at[gt]` works on `interactions`, `memberships`, `patients`, `customer_invoices`, and similar mutable resources. **Stamp the cursor with the response's request time, not the latest record's `updated_at`** — using the latter creates a race where any record whose `updated_at == cursor` gets skipped on the next refresh.

### 4. Daily backstop full fetch — guarantee eventual consistency

Delta refresh can miss records (transient API errors, detail-endpoint blips, clock skew). Once per day per practice, do a full refresh to backfill anything that slipped through:

```js
async function refreshSnapshotAsync(practiceId, lastFetchedAt) {
  const sinceLastFetch = Date.now() - lastFetchedAt.getTime();
  const ONE_DAY = 24 * 60 * 60 * 1000;

  if (sinceLastFetch > ONE_DAY) {
    return fullFetchAndWrite(practiceId);
  }

  // ... delta path from pattern #3
}
```

Don't shorten the 24 h window much — a 1 h backstop in production has every practice triggering a full fetch every hour, which fans out into a synchronized hammer on the hint-api side. 24 h is the right balance between catching missed records and keeping the API quiet.

### 5. Global throttle via `pg_try_advisory_lock` — prevent stampede

When many practices simultaneously trip the daily backstop, you get many concurrent full fetches against the Hint API, plus each one starts a worker thread inside a single Node process. Add a per-practice advisory lock so only one full fetch runs at a time, **and** a global one so the host process doesn't run more than N full fetches in parallel:

```js
async function fullFetchAndWrite(practiceId) {
  // Per-practice lock: if another worker is already fetching this practice's snapshot, skip.
  const perPracticeKey = hash32('snapshot_full_fetch:' + practiceId);
  const { rows: [{ pg_try_advisory_lock: gotLock }] } = await db.query(
    'SELECT pg_try_advisory_lock($1)',
    [perPracticeKey],
  );
  if (!gotLock) return; // another worker has it

  try {
    // Global lock — cap concurrent full-fetches across the whole host process.
    // (Rotates through 4 buckets so up to 4 practices can full-fetch in parallel.)
    const bucket = perPracticeKey % 4;
    const globalKey = hash32('snapshot_full_fetch_global:' + bucket);
    await db.query('SELECT pg_advisory_lock($1)', [globalKey]); // BLOCKING — waits

    try {
      const payload = await fullFetch(practiceId);
      await writeSnapshot(practiceId, payload, new Date());
    } finally {
      await db.query('SELECT pg_advisory_unlock($1)', [globalKey]);
    }
  } finally {
    await db.query('SELECT pg_advisory_unlock($1)', [perPracticeKey]);
  }
}
```

`hash32` can be any 32-bit hash (e.g. `crc32`); advisory locks are keyed by int. The per-practice lock is `pg_try_advisory_lock` (non-blocking — skip if held); the global lock is `pg_advisory_lock` (blocking — queue). Tune the bucket count (`% 4` above) to the host process's CPU + the hint-api rate limits.

Without this, a fleet of practices that all trip the daily backstop in the same minute will fan out to many concurrent detail fetches and trigger the rate-limit cascade described in [`provider-api.md` § rate-limited endpoints](./provider-api.md#rate-limited-endpoints--keep-detail-fetch-concurrency-low).

---

## Client-side: HTML fragment swap

For the "refresh on visit / focus / interval" UX, return rendered HTML fragments (not JSON) and `outerHTML`-swap them in. Simpler than SSE / WebSockets and works inside the Hint embed iframe without any extra plumbing:

```js
// In server.js — refresh endpoint returns rendered fragments keyed by the
// DOM element id each fragment will replace. Keys are app-specific —
// adapt to your dashboard's sections (summary, list, chart, etc.).
app.get('/api/refresh', requireSession, async (req, res) => {
  const snap = await readSnapshot(req.session.practice_id);
  res.json({
    summary: renderSummaryHtml(snap.payload),
    list:    renderListHtml(snap.payload),
  });
});

// In the embedded HTML — swap fragments in on a short interval
async function refresh() {
  const r = await fetch('/api/refresh').then(x => x.json());
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

- **Single-record surfaces** (`clinical_interaction`, `clinical_chart`, single-patient views) — the per-visit fetch cost is bounded and tiny; caching adds complexity without measurable benefit. Use the API direct.
- **Settings surfaces** — same reasoning. No list, no fan-out.
- **Apps that don't read practice-scale list endpoints** — if every request is keyed to the user's current record, you're not fanning out.

Use these patterns when the dashboard summarizes data across an entire practice's records on a list endpoint. That's the workload that breaks without caching.

# Provider API Field Reference (shared fragment)

Inline schema sketch for the five resources marketplace apps reach for most. Use this when the [`https://developers.hint.com/mcp`](https://developers.hint.com/mcp) MCP server isn't installed — it's enough to write client code that returns correct numbers instead of guessing field names.

If you DO have the MCP server, prefer it — these sketches show the headline fields, not every response key.

## Conventions that bite first

These three apply to every endpoint below. Get them wrong silently and the app ships zeros, double-counts families, or attributes everything to nobody.

- **Money is always `*_in_cents` integers.** Divide by 100 only for display. Every monetary field on every resource follows this — `amount_in_cents`, `paid_in_cents`, `rate_in_cents`, `past_due_in_cents`, etc.
- **Bucket time-series by domain dates, NOT `created_at`.** In sandbox, every record's `created_at` is the seed-run timestamp, so any "growth over time" chart bucketed on `created_at` is flat-flat-flat-spike-on-seed-day for every metric. Bucket by `joined_practice_date` (patients), `start_date` / `end_date` (memberships), `paid_at` / `date` (invoices, payments), `bill_date` / `next_bill_date` (membership billing). This is also more correct in live — patients backdated for compliance, memberships started mid-period, etc.
- **Two distinct status fields coexist on memberships.** `status` is the **billing state** (`active`, `unpaid`, `cancelled`, ...) and `enrollment_status` is the **lifecycle state** (`active`, `inactive`, `pending`, ...). A patient can be `enrollment_status: 'active'` AND `status: 'unpaid'` simultaneously. Filtering on the wrong one drops ~half of "active" members. There is also a third, `engagement_status`, that exists in Hint's UI but is NOT on the Provider API — so partner apps cannot exactly match Hint's "Active Members" KPI in the UI. Expect a 3–5% systematic gap and don't chase it.

## Patient — `GET /provider/patients`

The full patient record; the most-queried resource in the API.

```
id, name, first_name, last_name, email, dob, sex, gender,
joined_practice_date,      // ← bucket here, NOT created_at
membership_status,         // patient-level membership lifecycle ('active'|'inactive'|...)
account: { past_due_in_cents }
practitioner: { id, name, ... }  // ← attribution lives HERE, nested
location: { id, name, address_state, ... }
memberships: [ { ... } ]   // see Membership below
sponsorships: [ { ... } ]  // B2B/company-sponsored coverage
```

**Practitioner attribution is nested.** To attribute a metric (MRR, visit count, etc.) to a provider, join via `patient.practitioner.id` — there is no flat `practitioner_id` column on patient and no `practitioner` field on memberships at all. For MRR-per-provider you go membership → owner (patient) → practitioner.

## Membership — `GET /provider/memberships`

**A membership is a family subscription, not a single person.** The owner is one patient, but `membership_patients[]` lists everyone the membership covers (owner + spouse + children). Naive `memberships.length` for "active member count" is wrong — Hint counts individuals, not families.

```
id, status, enrollment_status,           // ← both fields; see "Conventions" above
plan: { id, name, plan_type },
rate_in_cents, period_in_months, list_price_in_cents,
start_date, end_date,                    // ← bucket here for cohort/retention charts
bill_date, billed_through_date, next_bill_date,
last_bill_amount_in_cents, last_bill_date,
is_current, is_processing, is_restartable,
owner: { id, name, ... },                // the subscriber patient
membership_patients: [                   // ← family roster
  { patient: { id, name }, member_type, status, enrollment_status, start_date, end_date }
],
sponsorship: { ... } | null,             // company sponsor if employer-paid
upcoming_bills: [ { amount_in_cents, date, ... } ]
```

To count "active members" the way Hint does internally: iterate `memberships[].membership_patients[]` and count entries where the membership-patient's own `status === 'active'` AND `enrollment_status === 'active'`. Don't dedupe by patient id without thinking — the same patient can appear in multiple memberships during transitions.

## Customer Invoice — `GET /provider/customer_invoices`

Patient-facing invoices (what patients owe the practice). This is what you want for revenue, NOT `practice_invoices` (those are what the practice owes Hint).

```
id, invoice_number, status,              // 'draft'|'sent'|'paid'|...
date, paid_at,                           // ← bucket revenue by paid_at
amount_in_cents, total_in_cents, subtotal_in_cents, tax_in_cents,
paid_in_cents, received_in_cents, due_in_cents,   // ← revenue lives here
payment_status, payments_count,
charges_count,
charges: [ { ... } ],                    // line items — see below
owner: { id, name, ... }                 // the patient billed
```

**Revenue ≠ `charges`.** Charges are **line items inside the invoice**, not transactions. A v1 chart built against `customer_invoices.charges` (or worse, against a top-level `/charges` endpoint, which does not exist) will sum line-item amounts on draft+sent+paid invoices alike and produce nonsense.

For total revenue, sum `paid_in_cents` on invoices where `status === 'paid'`, bucketed by `paid_at`. For individual transactions, the nested `/provider/customer_invoices/:id/payments` returns the actual payment objects (`amount_in_cents`, `date`, `status`, `source`).

## Payment — `GET /provider/customer_invoices/{customer_invoice_id}/payments`

There is no top-level `/provider/payments` endpoint — payments are nested under the customer_invoice they pay. For aggregate revenue, sum `paid_in_cents` on the parent invoice (see above). Use this nested endpoint only when you need transaction-level detail (refunds, error messages, source type).

```
id, amount_in_cents, date,
status,                                  // 'paid'|'unattempted'|...
error_message, memo,
source: { id, type }                     // 'Card', 'BankAccount', etc.
external, external_type                  // true if recorded outside Hint
```

## Practitioner — `GET /provider/practitioners`

**Practitioners are NOT the same as users.** A practitioner is a credentialed clinician (NPI, specialty, billing identity, panel_size, schedulable) — the entity who appears on encounters and gets attributed in billing. A `/provider/users` row is a portal login, possibly held by a non-clinician. The two overlap when a clinician also logs in but the lists model different concepts: a credentialed practitioner without a portal account is still a practitioner; a staff user without credentials is not. Use `/practitioners` whenever the app cares about clinical attribution, visit authorship, or "doctors at this practice." Reach for `/users` only when the app specifically wants "who has portal access."

```
id, name, email, bio, photo_url,
panel_size, panel_size_limit,
min_patient_age, max_patient_age,
accepts_enrollments_from_employer, accepts_enrollments_from_retail,
online_signup_visibility,
locations: [ { id, name, address_state, ... } ],
npi: { number, first_name, last_name, type },
specialties: [ { id, name } ]
```

`panel_size` is the live count of patients attributed to this practitioner (server-computed; don't recount via the patients endpoint). `panel_size_limit` is the cap set by the practice.

## What's NOT here

- **`/provider/charges`** — does not exist. Charges are nested under invoices (`customer_invoices.charges` inline, or `/customer_invoices/:id/charges`).
- **`/provider/invoices`** — does not exist. Use `customer_invoices` (patient bills) or `practice_invoices` (Hint bills the practice).
- **`engagement_status`** — exists in Hint's UI, not on the Provider API. Partner apps cannot exactly reproduce Hint's "Active Members" UI count; expect a 3–5% gap.

For everything else (companies, plans, locations, charges, coupons, etc.), use the MCP server or [`https://developers.hint.com/reference`](https://developers.hint.com/reference) directly.

# e-Uniti Security Posture

Status: draft (Phase 1). Finalized in Phase 4.

## Threat model summary

The system serves government data to authenticated officers through an AI
interface. The threats designed against, in order of importance:

1. **Prompt injection / AI manipulation**: a user (or poisoned data) tries to
   make the AI do something it should not.
2. **Unauthorized data access**: a user reaches another department's data
   without a grant.
3. **Data tampering**: modification of served data or of the accountability
   record.
4. **Conventional web attacks**: injection, session theft, clickjacking.

## 1. Prompt injection: blast radius architecturally zero

The AI's only power is to emit JSON that must survive two validation stages in
`lib/catalog/schema.ts`:

- `intent_id` is validated against a zod enum built from the caller's
  **authorized and approved** catalog entries only. The model cannot name an
  intent outside that set; unauthorized intents are not even present in its
  prompt.
- Entities are validated against a schema built from the intent's typed
  parameter specs (12-digit IC regex, year bounds, enum membership, keyword
  length caps). Unknown keys are a validation failure.

The executed SQL text comes from the database (admin-approved, versioned),
never from the model. A fully hijacked model can therefore at worst select a
wrong pre-approved, read-only query that the user was already authorized to
run, under audit. The second AI call (the summary) receives only already-served
rows and its output is rendered as plain text, so injection via row content can
at worst produce odd prose beneath a visible table of true numbers.

## 2. Access control: two checkpoints plus explicit grants

- Invite-only accounts (no self-registration exists); roles and org membership
  live in `app_users`; suspended users fail the authoritative check.
- RBAC checkpoint 1: the catalog presented to the model is pre-filtered to the
  caller's grants (`user_data_access`) and approved status.
- RBAC checkpoint 2: after validation, the resolved intent is re-fetched under
  the same predicate; a stale prompt snapshot can never widen access.
- Admin surfaces are gated twice: middleware redirect (UX) plus
  `requireRole()` inside every admin handler (authoritative).

## 3. Data integrity: read-only by construction, append-only by construction

Query execution is read-only three independent ways: the `euniti_readsrc` role
holds SELECT-only grants; its `default_transaction_read_only` is on; every
execution runs inside an explicit READ ONLY transaction with a 5-second local
statement timeout and a row cap. The SQL template validator additionally
rejects anything that is not a single SELECT at authoring time.

The audit log is append-only four independent ways: the app role holds
INSERT+SELECT only; row-level security carries no update/delete policy; a
trigger raises on UPDATE and DELETE; and every row is hash-chained
(`row_hash = sha256(prev_hash + canonical_row)`, computed inside a database
trigger the application cannot influence), so any tampering or gap is
mechanically detectable.

All of the above was verified live against the running database (role denial
messages, trigger exceptions, and chain continuity), not assumed.

## 4. Conventional web security

- Security headers on every response: CSP (production policy contains no
  `unsafe-eval`), X-Frame-Options DENY, nosniff, strict referrer policy, HSTS.
- Session cookies are managed by the auth layer with server-side JWT
  validation on every request (`getUser`, never trusting cookie presence).
- Parameterized queries everywhere; named placeholders compile to positional
  driver parameters; string interpolation into SQL does not exist in the
  codebase.
- Per-user rate limiting (counted over the audit log itself).
- Secrets live only in environment variables. The data-source registry stores
  the NAME of an env var, so connection credentials never enter the database
  or the admin UI.

## Verification

Unit tests cover the SQL template validator (statement separators, non-SELECT,
SELECT INTO, embedded DML, placeholder mismatches), the parameter schema
builder (every type, strict unknown-key rejection), the envelope whitelist,
and the compiler. Live checks cover role isolation, audit immutability, hash
chain continuity, RBAC denial, and the fallback paths.

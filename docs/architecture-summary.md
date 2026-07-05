# e-Uniti Architecture

Status: draft (Phase 1). Finalized in Phase 4.

## What the system does

Officers ask questions in Bahasa Melayu through a chat interface. The system answers **table-first**: the result table comes directly from a departmental database, a short AI-written summary sits beneath it, and a provenance footer cites the source department, who keyed the data in, when, and its verification status. Every question, whatever the outcome, is recorded in an append-only, hash-chained audit log.

## The core principle: the AI never writes SQL

The single most important design decision. The pipeline is an **intent catalog**, not text-to-SQL:

```
question (BM)
   │
   ▼
[1] LLM extracts JSON only: {intent_id, entities, confidence, candidates}
   │        the catalog of the caller's AUTHORIZED+APPROVED questions is the
   │        entire universe the model can select from
   ▼
[2] zod validation: intent_id against the whitelist enum, entities against a
   │        schema built at runtime from the intent's typed parameter specs
   ▼
[3] pre-approved parameterized SELECT (named placeholders compiled to
   │        positional driver params; string interpolation never occurs)
   ▼
[4] read-only execution: SELECT-only role + read-only transactions + 5s
   │        statement timeout + row cap
   ▼
[5] the app renders the table (the AI never touches the numbers)
   ▼
[6] LLM writes a 2-3 sentence BM summary grounded on the served rows
   ▼
[7] provenance footer + append-only audit row
```

Fallbacks are designed in: low confidence or no match returns tappable
suggestions of the nearest supported questions (never a dead end); genuine
ambiguity asks exactly one clarifying question.

## Growth model

- `organization_units`: the org chart (ministry, department, division) as a
  self-referencing table. Users and data sources both belong to a unit.
- `organization_databases`: the registry of connectable departmental
  databases. A row stores the env-var NAME of its credentials, never the
  secret. Demo sources are Postgres schemas; production sources are remote
  databases behind the same `SourceAdapter` interface (one new adapter file
  per engine, no pipeline change).
- `approved_questions`: the menu. A source becomes queryable only when its
  owning department has authored and approved question entries; the approval
  workflow (draft, approve, deactivate, versioning) is itself the governance
  feature.
- `user_data_access`: explicit per-user, per-source grants: auditable,
  revocable, one indexed lookup.

## Provider seam

All AI calls go through one OpenAI-compatible factory with an ordered failover
chain (`LLM_PROVIDER_CHAIN`). The demo chain is Groq then Gemini (free tiers);
the offline sovereignty configuration is a one-line env change to a local
Ollama instance: same code path, no internet. The provider that actually served
each answer is recorded in the audit row and shown in the provenance footer.

## Repository layout

See the tree in the project README. Key modules:

| Module | Responsibility |
|---|---|
| `lib/catalog/schema.ts` | zod contracts, SQL template validation, named-to-positional compilation (the whitelist enforcement point) |
| `lib/catalog/loader.ts` | authorized+approved entry loading (RBAC checkpoint 1 and 2) |
| `lib/llm/*` | provider chain, failover, intent extraction, grounded summary |
| `lib/sources/*` | `SourceAdapter` interface, read-only Postgres adapter, registry resolution |
| `lib/exec/*` | orchestrator and provenance assembly |
| `lib/audit/*` | append-only audit writer (hash chain computed by a DB trigger) |
| `app/api/chat` | the streaming coordinator (NDJSON events: intent, table, summary deltas, done) |

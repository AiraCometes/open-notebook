# ADR-009: Enhanced embedding/reranker state lives in dedicated tables with additive-only schema changes

- **Status**: Accepted
- **Date**: 2026-09
- **Related**: [Issue #1](https://github.com/AiraCometes/open-notebook/issues/1), [contracts.md](../open-notebook-enhanced/contracts.md) (the normative contract this policy protects), [ADR-006](ADR-006-migration-granularity.md) (migration granularity), [ADR-008](ADR-008-notebook-scoped-search.md) (prior additive-extension precedent), [PDR-001](PDR-001-single-user-first.md)

## Context

Open Notebook Enhanced adds durable per-source embedding state, multi-source embedding jobs with pause/resume/retry, and reranker configuration. The existing model offers only fire-and-forget `command` records and a `source.command` link — an engine-owned lifecycle that cannot express `partial`, `paused`, or per-chunk outcomes — and the `source`/`source_embedding` schema predates the state model. The policy had to be fixed before T02–T11 could diverge on where state lives and what existing installs are allowed to break.

## Decision

**Persist Enhanced state in three new SCHEMAFULL tables — `embedding_job`, `embedding_job_item`, `reranker_config` — and extend existing tables only with `option<>` fields. Domain state is stored on `source`/`embedding_job` records, never derived at read time or read from `command.status`.**

- `source` gains optional fields (`embedding_status`, `requested_mode`, `resolved_mode`, chunk counters, `embedding_model`, `embedding_job`, `last_embedding_error`, `embedding_updated_at`); `source_embedding` gains `content_hash`. No existing field is retyped, required, or removed.
- Backfill ships in the same migration: sources with ≥1 `source_embedding` row → `embedding_status="completed"` with counters seeded from actual rows; sources whose `command` is `new`/`running` → `"processing"`; the rest → `"not_started"`. `requested_mode`/`resolved_mode` backfill `"normal"` only on sources that were embedded — the historical path — and stay unset elsewhere.
- Reranker configs reference `credential` records by id; secrets are never duplicated onto `reranker_config`, and responses expose `credential_id`/`has_credential` only.
- API evolution is additive: new paths for jobs/rerankers/embedding-status, new optional request/response fields, `SearchRequest.type` gains `"hybrid"`, error format stays `{"detail": ...}` plus a shared `error_kind` taxonomy. The `embed` flag and `default_embedding_option` keep their meaning.
- Migration mechanics follow ADR-006 unchanged: one migration per PR that needs one, `_down` counterpart, merge-order numbering, never consolidate after landing on main.

## Alternatives considered

- **Drive state from `command` records** — rejected: the engine lifecycle (`new`/`running`/`completed`/`failed`/`canceled`) cannot express `partial`, `paused`, or per-chunk failure, and coupling dashboard reads to engine internals makes every future query pay a join to an execution log.
- **A flexible/schemaless state bag on `source`** — rejected: counters and status need types and indexes for dashboard and queue queries; the `credential.config` bag pattern fits provider extras, not a queried state machine.
- **Aggregate progress at read time** — rejected: list endpoints (T06) would fan out per source/job on every poll; counters are written by the worker under documented invariants instead.
- **Reuse `embedding_status` for processing status, or extend `SourceResponse.status`** — rejected: source *processing* (command lifecycle) and *embedding* state are different lifecycles; merging them breaks clients that already read `status`.

## Consequences

- Dashboard and status reads stay O(1) per row — no fan-out aggregation.
- Job/item state survives worker restart; `(job, source, content_hash)` dedup prevents re-embedding.
- One known edge is accepted: an old-code worker finishing an in-flight `embed_source` during the deploy window writes embeddings but no status, leaving a stale `not_started`. It self-heals on the next embed/retry; the backfill covers everything else.
- Downstream tasks inherit the contract: T02 implements §3/§5.1, T03–T04 §4/§5.2–5.4/§6.2, T08–T10 §5.5/§6.3, T07/T11 §6.4.
- Rollback: `_down` files drop the new tables and the added `option<>` fields; additive API fields can remain harmlessly even if the feature is disabled.

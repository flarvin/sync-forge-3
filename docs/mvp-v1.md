# SyncForge MVP v1.0 (Release `v0.1.0`)

## 1. Product Intent

`SyncForge` MVP provides deterministic, DBAL-based reconciliation of external array data into Doctrine-backed tables with fluent configuration, chunked processing, optional delete-missing, and dry-run reporting.

Release target: `v0.1.0`

## 2. Scope

In scope:

- Fluent API:
  - `for(Entity::class)`
  - `key([...])`
  - `source(iterable)`
  - `deleteMissing(bool)`
  - `chunkSize(int)`
  - `dryRun(bool)`
  - `run()`
- Metadata from Doctrine mappings, cached for runtime reuse.
- Composite key support.
- Diff detection for scalar fields.
- Batch execution via DBAL abstraction.
- Platform-aware executor selection:
  - PostgreSQL strategy placeholder (architecture-complete, non-final SQL in MVP if needed).
  - MySQL strategy placeholder (architecture-complete, non-final SQL in MVP if needed).
  - Fallback manual batching path.
- Result report with counters and timing.

Out of scope:

- Domain events/lifecycle listeners support during bulk writes.
- Full JSON deep-diff.
- Async queue/workers.
- Audit storage and UI monitoring.
- Generic schema mapping from arbitrary nested payloads.

## 3. Non-Goals (Hard Limits)

- No ORM UnitOfWork-based mass writes.
- No runtime reflection in hot path.
- No automatic validation framework integration.
- No soft-delete policy inference.

## 4. Delivery Milestones

## M1. Contracts and Value Objects

Target:
- Stable public interfaces and immutable data carriers.

Tasks:
- Add `SyncForge`, `SyncBuilder` API signatures.
- Add context/value objects:
  - `SyncContext`
  - `EntityMetadata`
  - `DiffContext`
  - `DiffPlan`
  - `ExecutionContext`
  - `ExecutionResult`
  - `SyncResult`
- Add domain exceptions.

Exit criteria:
- Public API defined and documented.
- Constructor-level validation in value objects.

## M2. Pipeline and Metadata

Target:
- Executable orchestration over chunked source with metadata resolution.

Tasks:
- Implement `SyncPipeline::run()` template flow.
- Implement `DoctrineEntityMetadataProvider` with in-memory cache.
- Add `ChunkIterator` and source safeguards.
- Validate configured key fields against metadata.

Exit criteria:
- Chunked source processed end-to-end in dry-run mode.
- Metadata lookup is cached across runs in process.

## M3. Key + Diff

Target:
- Deterministic reconciliation planning.

Tasks:
- Implement `CompositeKeyResolver`.
- Implement `ScalarDiffEngine`.
- Handle duplicate keys in incoming chunk (explicit policy).
- Generate minimal `update` payload (`changedColumns` only).

Exit criteria:
- `DiffPlan` accurately classifies insert/update/unchanged on test fixtures.
- Composite keys behave deterministically.

## M4. Execution Strategies

Target:
- Platform-aware execution interface + fallback path.

Tasks:
- Implement `BulkExecutorFactory`.
- Implement `FallbackBatchExecutor` as baseline executable path.
- Add PostgreSQL/MySQL executors as pluggable implementations (MVP may keep SQL internals minimal while preserving interface contract).
- Wire dry-run no-op execution that still returns realistic counters.

Exit criteria:
- Fallback executor passes integration tests.
- Platform detection chooses expected executor.

## M5. Delete Missing + Reporting

Target:
- Full reconciliation and outcome visibility.

Tasks:
- Implement `deleteMissing` planning/execution.
- Finalize `SyncResult` aggregation (timings, chunk counts, counters, errors).
- Add structured diagnostics for failures by chunk.

Exit criteria:
- `deleteMissing=false` is safe default.
- `deleteMissing=true` removes rows outside incoming set for the current sync scope.

## M6. Symfony Packaging + Docs

Target:
- Consumable package in Symfony projects.

Tasks:
- Add DI wiring (`services.php`/extension).
- Add usage docs and architecture links.
- Add minimal examples for Product and composite key case.

Exit criteria:
- Install + autowire in sample Symfony app.
- Public docs sufficient for first adopters.

## 5. Backlog (Prioritized)

P0 (must-have for `v0.1.0`):

1. Fluent API + validation.
2. `SyncPipeline` orchestration.
3. Doctrine metadata provider with cache.
4. Composite key resolver.
5. Scalar diff engine.
6. Fallback DBAL executor.
7. Dry-run mode parity.
8. `SyncResult` reporting.
9. Basic integration tests.

P1 (if time permits in `v0.1.0`, otherwise `v0.2.x`):

1. First-class PostgreSQL executor implementation.
2. First-class MySQL executor implementation.
3. Better duplicate-key conflict policy (configurable).
4. Structured logging hooks.

P2 (post-MVP):

1. JSON-aware diff strategies.
2. Scope filters for safer deletes.
3. Audit trail store.
4. Async orchestration support.

## 6. Definition of Done (DoD)

Feature-level DoD:

1. Public contract documented.
2. Unit tests for happy path + failure path.
3. No entity hydration in bulk path.
4. Handles `>=100k` rows with bounded memory (chunked).
5. Dry-run returns same plan counters as write mode on same input.

Release DoD (`v0.1.0`):

1. Semver tag + changelog entry.
2. README quick-start and caveats.
3. Test suite green in CI.
4. Architecture and MVP docs published.

## 7. Test Matrix

Unit tests:

- Builder validation (`missing source`, `missing key`, `invalid chunkSize`).
- Key resolver:
  - single key
  - composite key
  - null segments
  - duplicate incoming key policy
- Diff engine:
  - insert/update/unchanged classification
  - strict type normalization cases
  - datetime equality format
- Result aggregation correctness.

Integration tests (DBAL):

- Fallback executor insert/update/delete batches.
- Dry-run no-write behavior.
- Chunk boundary behavior (`N`, `N+1`, large inputs).
- Transaction behavior on chunk failure.

Cross-platform smoke (minimum):

- PostgreSQL: executor selection + contract tests.
- MySQL: executor selection + contract tests.

Performance checks (non-blocking for MVP, but required report):

- 10k/50k/100k rows memory profile.
- Throughput baseline by chunk size (500/1000/5000).

## 8. Risks and Mitigations

1. Risk: false-positive updates due to type mismatch.
   Mitigation: type normalization by metadata before diff.

2. Risk: accidental over-delete.
   Mitigation: `deleteMissing=false` default, explicit opt-in, docs warning.

3. Risk: behavior drift between dry-run and write-run.
   Mitigation: shared planning path; only execution side toggled.

4. Risk: callback/event expectations from ORM users.
   Mitigation: explicit contract docs that bulk path bypasses lifecycle.

## 9. Edge Case Policy (MVP)

1. Missing key in row: fail fast for chunk with explicit error.
2. Duplicate key in incoming chunk: last-write-wins (default policy), warning counter increment.
3. Empty source:
   - `deleteMissing=false` => no-op.
   - `deleteMissing=true` => delete all in sync scope (documented as dangerous).
4. Unsupported platform: throw `UnsupportedPlatformException`.
5. Source iteration exception: stop sync, return partial metrics + error.

## 10. Suggested Repo Layout (Target)

```text
docs/
  architecture.md
  mvp-v1.md
src/
  SyncForge.php
  Fluent/
    SyncBuilder.php
  Pipeline/
    SyncPipeline.php
  Metadata/
    EntityMetadata.php
    EntityMetadataProviderInterface.php
    DoctrineEntityMetadataProvider.php
  Key/
    KeyResolverInterface.php
    CompositeKeyResolver.php
  Diff/
    DiffContext.php
    DiffPlan.php
    DiffEngineInterface.php
    ScalarDiffEngine.php
  Executor/
    BulkExecutorInterface.php
    BulkExecutorFactory.php
    FallbackBatchExecutor.php
    PostgresBulkExecutor.php
    MySqlBulkExecutor.php
  Report/
    SyncResult.php
  Chunk/
    ChunkIterator.php
  Exception/
    InvalidConfigurationException.php
    MetadataException.php
    UnsupportedPlatformException.php
    SyncExecutionException.php
config/
  services.php
tests/
  Unit/
  Integration/
```

## 11. Release Checklist (`v0.1.0`)

1. Public API frozen and tagged.
2. Fallback executor stable on integration tests.
3. Platform executors compile and pass contract tests (or clearly flagged experimental in changelog).
4. Docs include:
   - quick-start
   - dry-run usage
   - deleteMissing caveat
   - performance tuning (`chunkSize`)
5. Known limitations section published.

## 12. Post-Release `v0.2.0` Candidates

1. Production-grade PostgreSQL upsert implementation.
2. Production-grade MySQL upsert implementation.
3. JSON diff strategy plug-in.
4. Sync scope filters (`where`/tenant-safe deletes).
5. Optional audit sink interface.


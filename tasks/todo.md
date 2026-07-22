# Upgrade `@vizmo/parse` to Upstream 8.6.0

- [x] Preserve staged `7.1.2-1` metadata work.
- [x] Create `8-vizmo` from `7-vizmo`.
- [x] Merge exact upstream stable commit `284a268` (tag `8.6.0`).
- [x] Keep upstream dependencies, exports, types, config, and Node floor.
- [x] Set package and lockfile identity to `@vizmo/parse@8.6.0-1`.
- [x] Regenerate lockfile and verify clean `npm ci`.
- [x] Preserve multi-batch `ParseObject.saveAll` error aggregation.
- [x] Port one focused regression test; remove fork test duplication.
- [x] Run focused `ParseObject` tests.

## Checkpoint: Fork Contract

- [x] Later batches run after object failures.
- [x] Rejection includes first `code` / `message` and complete `errors`.
- [x] Each error keeps object reference, batch-local index, and `ParseError`.

## Full Validation

- [x] Unit tests pass: 49 suites, 1188 tests.
- [x] Lint passes.
- [x] Type checks pass (one upstream unused-disable warning).
- [x] Docs and circular-dependency checks pass.
- [x] Node, React Native, browser, WeChat, and declaration builds pass.
- [x] Mongo integration blocker recorded: 801 specs, 24 environment/upstream failures (6 missing-Chrome failures, time-order flakiness, and Mongo geospatial internal errors).
- [x] `npm pack --dry-run` and entry-point smoke imports pass.
- [x] Final diff contains no alpha-only or unrelated changes.
- [x] Review and approve plan before implementation.

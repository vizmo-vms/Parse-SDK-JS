# Implementation Plan: Upgrade Vizmo Parse Fork to Upstream 8.6.0

## Overview

Upgrade branch `7-vizmo` from upstream Parse SDK `7.1.2` to latest stable upstream release `8.6.0` (`upstream/release` at `284a268`), while retaining fork identity `@vizmo/parse` and fork behavior that makes `ParseObject.saveAll` process every batch and return all object errors. Do not include unreleased `upstream/alpha` changes.

## Current State

- Common base with upstream: `06087ce` (`7.1.2`).
- Fork-only commits after base: package rename, `saveAll` error aggregation, lockfile sync.
- User has staged but uncommitted version changes from `7.1.2` to `7.1.2-1` in `package.json` and `package-lock.json`; preserve these before integration, then supersede with final fork version.
- Upstream delta: 190 commits, 57 files, about 14.5k additions and 12.9k deletions.
- Mechanical merge predicts conflicts only in root package identity/version fields in `package.json` and `package-lock.json`. `src/ParseObject.ts` merges textually, but its fork behavior still needs focused regression verification.

## Architecture Decisions

- Merge stable `upstream/release`, not `upstream/alpha`, so upstream history remains traceable and future syncs stay simple.
- Keep upstream 8.6.0 code, dependencies, exports, type declarations, build config, and Node support floor. Override only package identity/version and fork-specific behavior.
- Set fork release version to `8.6.0-1`, following existing `6.1.1-2` / `7.1.2-1` convention. Confirm registry availability before publishing.
- Preserve `saveAll` behavior as a small fork patch: continue later batches, collect every object-level failure, and reject after processing with first error's `code` and `message` plus `errors` entries containing `object`, `index`, and `error`.
- Regenerate `package-lock.json` from resolved `package.json` with Node 22, rather than hand-merging the large dependency graph.

## Task List

### Phase 1: Establish Safe Integration Point

- [x] Task 1: Preserve current staged version work and create upgrade branch
- [x] Task 2: Merge upstream 8.6.0 and resolve package metadata

### Checkpoint: Upstream Integrated

- [x] No unresolved conflicts or conflict markers
- [x] Diff contains full upstream 8.6.0 delta plus intentional fork patches only
- [x] Package identity is `@vizmo/parse@8.6.0-1`

### Phase 2: Restore Fork Contract

- [x] Task 3: Reconcile `saveAll` aggregation behavior with upstream `ParseObject`
- [x] Task 4: Reconcile fork regression tests with upstream Jest 30 tests

### Checkpoint: Fork Behavior Preserved

- [x] All batches run after object-level failures
- [x] Aggregate rejection contains every failure and stable object/index mapping
- [x] Successful objects from later batches are clean/saved

### Phase 3: Validate and Package

- [x] Task 5: Run static, unit, build, and package checks
- [x] Task 6: Run integration tests and review release artifact

### Checkpoint: Release Candidate

- [x] Required Node version is documented and CI-aligned
- [x] All deterministic local validation passes on supported Node; integration blockers recorded below
- [x] Packed artifact exposes root, node, react-native, weapp, dist, lib, and types entries
- [x] Final diff contains no accidental upstream-alpha or unrelated changes

## Validation Results

- Node `22.22.0`, npm `10.9.4`.
- Unit: 49 suites and 1188 tests passed.
- Focused `ParseObject`: 183 tests passed.
- Lint, `ci:typecheck`, docs, circular dependency check, and all builds passed.
- Type-test lint passed with one upstream unused `eslint-disable` warning.
- `npm pack --dry-run` produced `@vizmo/parse@8.6.0-1`; root, node, React Native, and WeChat entry-point smoke imports passed.
- Mongo integration ran 801 specs with 24 failures: 6 require missing Puppeteer Chrome 146, 5 are time-order/time-equality flakes, and 13 are Mongo geospatial internal-error timeouts. No failure touches fork `saveAll` behavior.
- `npm ci` reports 80 transitive audit findings (10 low, 32 moderate, 33 high, 5 critical), inherited from upstream dependency graph; no automatic audit mutation applied.

## Detailed Tasks

### Task 1: Preserve Current Work and Create Upgrade Branch

**Description:** Protect the already staged `7.1.2-1` metadata change, then create a dedicated `codex/upgrade-parse-8.6.0` branch from current `7-vizmo` HEAD. A temporary commit is preferable because the staged files are the same files expected to conflict during the merge; the final metadata task can amend or supersede it.

**Acceptance criteria:**

- [ ] Existing staged changes remain recoverable and unchanged before merge work starts.
- [ ] Upgrade work occurs on `codex/upgrade-parse-8.6.0`, not directly on `7-vizmo`.
- [ ] Baseline records `HEAD=ed0f1d1`, upstream target `284a268`, and merge base `06087ce`.

**Verification:**

- [ ] `rtk git status --short --branch`
- [ ] `rtk git diff --cached --check`
- [ ] `rtk git log --oneline --decorate -5`

**Dependencies:** None

**Files likely touched:**

- `package.json`
- `package-lock.json`

**Estimated scope:** Small

### Task 2: Merge Upstream 8.6.0 and Resolve Package Metadata

**Description:** Merge exact stable commit `284a268` from `upstream/release`. Resolve package conflicts by retaining upstream 8.6.0 dependency, export, engine, and script changes while setting package name/version to `@vizmo/parse@8.6.0-1`. Regenerate lockfile so both top-level identity locations and dependency graph agree with `package.json`.

**Acceptance criteria:**

- [ ] Merge includes upstream tag `8.6.0` and excludes commits unique to `upstream/alpha`.
- [ ] `package.json`, root lockfile metadata, dependencies, `exports`, `typesVersions`, and Node engines are internally consistent.
- [ ] Node support becomes `>=20.19.0 <21 || >=22.13.0 <23 || >=24.1.0 <25`; Node 18/19 support is intentionally removed.

**Verification:**

- [ ] `rtk git diff --check`
- [ ] `rtk npm install --package-lock-only --ignore-scripts`
- [ ] `rtk npm ci --ignore-scripts`
- [ ] Inspect `npm pkg get name version engines exports typesVersions`

**Dependencies:** Task 1

**Files likely touched:**

- `package.json`
- `package-lock.json`
- Upstream-owned files changed between 7.1.2 and 8.6.0

**Estimated scope:** Medium

### Task 3: Reconcile `saveAll` Aggregation Behavior

**Description:** Review merged `DefaultController.save` against both upstream 8.6.0 and fork commit `14ceda4`. Reapply only fork contract: later batches continue after object errors, all object failures are collected, and final rejection exposes first error plus structured error list. Keep upstream type refactors and surrounding control flow intact.

**Acceptance criteria:**

- [ ] One failed object does not clear `pending` or prevent later batches.
- [ ] Every failed response produces `{ object, index, error: ParseError }`; define `index` as index within returned batch to retain current public fork behavior.
- [ ] Final rejection has first failure's `code` and `message`, plus complete `errors`; local datastore updates still run only on all-success path, matching current fork behavior.

**Verification:**

- [ ] Focused test: `rtk npm test -- src/__tests__/ParseObject-test.js --runInBand`
- [ ] Review diff around `DefaultController.save` against `upstream/release`.

**Dependencies:** Task 2

**Files likely touched:**

- `src/ParseObject.ts`

**Estimated scope:** Small

### Task 4: Reconcile Fork Regression Tests

**Description:** Port fork tests to upstream 8.6.0 test structure without retaining duplicate or malformed cases. One regression should cover failures across at least two batches, first-error compatibility, all-error aggregation, later-batch execution, object references, batch-local indexes, and successful object state.

**Acceptance criteria:**

- [ ] Duplicate `can saveAll with global batchSize` test introduced by fork is removed.
- [ ] Regression asserts three errors across two batches and verifies `errors[].object`, `errors[].index`, and `errors[].error`.
- [ ] Test fails on plain upstream 8.6.0 behavior and passes with fork patch.

**Verification:**

- [ ] `rtk npm test -- src/__tests__/ParseObject-test.js --runInBand`
- [ ] `rtk npm run lint -- --no-cache`

**Dependencies:** Task 3

**Files likely touched:**

- `src/__tests__/ParseObject-test.js`

**Estimated scope:** Small

### Task 5: Run Static, Unit, Build, and Package Checks

**Description:** Validate upstream 8.6.0 plus fork patch using repository checks and build every shipped target. Use supported local Node 22.22.0. Keep generated `lib/`, `dist/`, and `types/` changes only if repository release workflow expects them committed; otherwise remove generated noise before final review.

**Acceptance criteria:**

- [ ] Dependency install is reproducible from lockfile.
- [ ] Unit tests, lint, type checks, docs, circular-dependency check, and builds pass.
- [ ] Browser, WeChat, Node, React Native, and declarations build successfully.

**Verification:**

- [ ] `rtk npm ci --ignore-scripts`
- [ ] `rtk npm test -- --runInBand`
- [ ] `rtk npm run lint -- --no-cache`
- [ ] `rtk npm run ci:typecheck`
- [ ] `rtk npm run test:types`
- [ ] `rtk npm run docs`
- [ ] `rtk npm run madge:circular`
- [ ] `rtk npm run build`
- [ ] `rtk npm run build:browser`
- [ ] `rtk npm run build:weapp`

**Dependencies:** Task 4

**Files likely touched:**

- Generated `lib/`, `dist/`, and `types/` outputs during verification

**Estimated scope:** Medium

### Task 6: Run Integration Tests and Review Release Artifact

**Description:** Run Mongo-backed integration coverage when local MongoDB runner is available, create an npm tarball without publishing, inspect package contents and entry points, then review final fork-only delta against upstream 8.6.0.

**Acceptance criteria:**

- [ ] Mongo integration suite passes, or environment-only blocker is recorded with unit/build gates still green.
- [ ] `npm pack --dry-run` reports `@vizmo/parse@8.6.0-1` and includes every declared shipped path.
- [ ] Final upstream comparison contains only package identity/version, lockfile metadata needed by that identity, `saveAll` behavior/tests, and intentional merge metadata.

**Verification:**

- [ ] `rtk npm run test:mongodb`
- [ ] `rtk npm pack --dry-run`
- [ ] Smoke imports for `@vizmo/parse`, `@vizmo/parse/node`, and `@vizmo/parse/react-native`
- [ ] `rtk git diff --check upstream/release...HEAD`
- [ ] `rtk git diff --stat upstream/release...HEAD`

**Dependencies:** Task 5

**Files likely touched:**

- No source files expected

**Estimated scope:** Small

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Staged `7.1.2-1` work is overwritten | High | Preserve via temporary commit before merge; verify staged diff first. |
| Major Node/Vite/Jest/ESLint upgrade breaks local or consumer builds | High | Use supported Node 22.22.0; run full static/build matrix and package smoke imports. |
| Lockfile conflict hides dependency drift | High | Resolve `package.json` first, regenerate lockfile, then verify with clean `npm ci`. |
| Textually clean merge silently loses fork behavior | High | Compare `ParseObject.ts` against both parents and keep focused multi-batch regression. |
| `saveAll` aggregate rejection shape is undocumented | Medium | Lock current shape in tests; avoid unrelated API redesign during upgrade. |
| Alpha-only changes accidentally enter release | Medium | Merge exact `284a268` / tag `8.6.0`, then audit ancestry. |
| Generated build outputs create noisy commits | Low | Check project release convention; retain only expected tracked artifacts. |

## Open Questions

- Confirm `8.6.0-1` is desired fork version and unused in target npm registry before publish.
- Confirm batch-local `errors[].index` is relied upon; changing to input-array index would be a separate API change.
- Confirm integration environment may start local MongoDB; otherwise treat integration as an explicit external prerequisite.
- Publishing, pushing, tagging, and pull-request creation are outside this implementation plan and require separate approval.

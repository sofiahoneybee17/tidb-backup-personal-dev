# Invalidate stale region-cache entries when bucket splitting falls back after a region split

This ExecPlan is a living document. Keep `Progress`, `Surprises & Discoveries`, `Decision Log`, and `Outcomes & Retrospective` up to date as work proceeds.

Reference: `PLANS.md` at repository root; this plan must be maintained according to it. Repository policy in `AGENTS.md` is stricter in places (regression test required for a bug fix, `make bazel_prepare` gate on test and import changes, `Ready` verification profile before claiming completion) and is followed here.


## Summary (one minute)


The bug: when a TiKV region splits, `pkg/store/copr/region_cache.go` detects the stale cached entry (bucket splitting falls back), logs it, queries PD, and then keeps using the stale entry anyway. Nothing evicts it. The current branch adds six lines of diagnostics and no fix.

The fix: in the fallback branch of `(*RegionCache).SplitKeyRangesByBuckets`, when the PD lookup the code already makes shows the epoch, boundaries, or bucket version changed, call `InvalidateCachedRegionWithReason(region, tikv.EpochNotMatch)` before the region-only re-split. About ten lines, one file, nothing on the steady-state path.

The proof: a regression block appended to the existing `TestBuildTasksByBuckets` (not a new top-level test, because the package already has 51 tests against a `shard_count` of 50) that splits a mock region and expects the cache to heal without a manual invalidate; it must fail before the fix and pass after. Plus a benchmark showing no invalidation under steady-state load, which answers Yuki's miss-amplification concern.

Before "ready": run the regression block with failpoints enabled through `./tools/check/failpoint-go-test.sh`, run the `bazel_prepare` gate check (expected not to trigger if no new `TestXxx` and no import changes), update `store-case-map.md`, run `make lint`, write the PR body from `.github/pull_request_template.md` with a linked issue, exact test commands, and a release note, and report per the `Agent Output Contract`.

Out of scope here: the resolve-lock and pessimistic-lock epoch checks Yuki asked for. They live in client-go, not this repo. Rafael's #56601 does not overlap with this change at all.


## Purpose / Big Picture


TiDB keeps a client-side cache of which TiKV region owns which key range, plus each region's "buckets" (finer-grained sub-ranges inside a region used to split coprocessor work evenly). When a region splits, the cached entry for the old region is stale: its key boundaries and its region epoch (a pair of version numbers, `ver` and `conf_ver`, that TiKV bumps on every split, merge, or membership change) no longer match what PD, the placement driver that owns region metadata, reports.

Today the bucket-splitting code in `pkg/store/copr/region_cache.go` already detects this staleness: when a cached bucket no longer contains the range it should, `(*LocationKeyRanges).splitKeyRangesByBuckets` returns a `bucketSplitFallbackInfo`, and `(*RegionCache).SplitKeyRangesByBuckets` logs a large diagnostic record, queries PD, and falls back to region-only splitting. What it does not do is evict the stale entry. The fallback itself re-reads the same stale cache, and every later statement that touches the same key range hits the same stale entry, logs the same warning, and falls back again, until some unrelated request happens to receive an `EpochNotMatch` error from TiKV and invalidates the entry as a side effect.

After this change, the fallback path evicts the stale region entry as soon as it has confirmed with PD that the epoch, boundaries, or bucket version actually changed. The region-only re-split inside the fallback, and every subsequent statement, then loads fresh metadata. You can observe this in a unit test: split a region in the mock cluster behind the cache's back, call `SplitKeyRangesByBuckets` once, and the returned locations reflect the post-split regions without any manual `InvalidateCachedRegion` call (today the test suite has to call it by hand, see `TestSplitKeyRanges` in `pkg/store/copr/coprocessor_test.go` around line 573).

This plan is deliberately narrow: it changes one fallback branch in one file and adds tests and a benchmark. It does not alter the normal (non-fallback) read path, so steady-state throughput is unaffected; the benchmark exists to prove that.


## Progress


- [x] (2026-10-20 15:30Z) Branch `petrova-store-fix-region-cache-stale-on-split` reviewed. Its delta over `master` is six lines in `pkg/store/copr/region_cache.go`: a `bucketVersion` field on `bucketSplitFallbackInfo` populated at both fallback sites. That is diagnostic plumbing only; no invalidation happens yet.
- [x] (2026-10-20 15:45Z) Validation section checked against `docs/agents/testing-flow.md`, `.agents/skills/tidb-test-guidelines`, `.agents/skills/tidb-bazel-prepare-gate`, `.agents/skills/tidb-verify-profile`, `.agents/skills/tidb-pr-metadata-guard`, and `pkg/store/copr/BUILD.bazel`. Plan revised; see the note at the bottom.
- [ ] Milestone 1: regression block appended to `TestBuildTasksByBuckets` that fails on the current branch.
- [ ] Milestone 2: invalidate the stale region in the fallback branch of `(*RegionCache).SplitKeyRangesByBuckets`; regression block passes.
- [ ] Milestone 3: steady-state benchmark showing no invalidation and no fallback when metadata is fresh.
- [ ] Milestone 4: `bazel_prepare` gate check, case-map update, `Ready` verification profile (`make lint`), PR body per template, evidence recorded below.
- [ ] Reply to Rafael on pingcap/tidb#56601 with the sequencing answer (see `Interfaces and Dependencies`).


## Surprises & Discoveries


- Observation: the fallback path already pays for a PD round-trip (`LocateRegionByIDFromPD`) purely to enrich the log line, and already computes `pdEpochChanged` and `pdBoundaryChanged` booleans, then throws them away.
  Evidence: `pkg/store/copr/region_cache.go`, the `pdLoc, pdErr := c.RegionCache.LocateRegionByIDFromPD(...)` block inside `SplitKeyRangesByBuckets`; the two booleans are only ever passed to `zap.Bool`. The fix therefore adds no new network cost: it acts on information the code already fetched.

- Observation: the existing test suite encodes the bug as a workaround. `TestSplitKeyRanges` splits a mock region and then calls `cache.InvalidateCachedRegion(tasks[1].region)` by hand before asserting on the new layout.
  Evidence: `pkg/store/copr/coprocessor_test.go` lines 569 to 580. That manual call is exactly the step the product code should perform itself.

- Observation: the resolve-lock and pessimistic-lock epoch re-validation gaps that Yuki raised (June 16 review) and that were worked through on July 10 are not implementable in this repository. Lock resolution lives in `github.com/tikv/client-go/v2/txnkv/txnlock` and pessimistic locking in `github.com/tikv/client-go/v2/txnkv/transaction`; this repo only calls into them (for example `worker.kvclient.ResolveLocksWithOpts` in `pkg/store/copr/coprocessor.go`).
  Evidence: `go.mod` pins `github.com/tikv/client-go/v2 v2.0.8-0.20260708122311-01bd8f99f4da`; no resolve-lock routing code exists under `pkg/`. Recorded as a Decision Log entry; those paths need a client-go change tracked separately.


## Decision Log


- Decision: scope this ExecPlan to the coprocessor region-cache path in `pkg/store/copr/region_cache.go` only. The resolve-lock and pessimistic-lock epoch re-validation work is a separate client-go change and is explicitly out of scope here.
  Rationale: the routing code for those paths is not in this repository (see `Surprises & Discoveries`). Pretending to fix it here would violate the "no speculative behavior" rule in `AGENTS.md`. The invariant this plan enforces (cached region metadata that PD has already superseded must not be reused) is the same one those paths need, so this change is a correct and independently shippable first step.
  Date/Author: 2026-10-20 / plan author.

- Decision: invalidate only when PD has confirmed a change (region `ver` or `conf_ver` differs, or start/end key differs, or bucket version differs). Do not invalidate when the PD lookup itself fails, and do not invalidate on every fallback unconditionally.
  Rationale: Yuki's review flagged cache-miss amplification from over-eager invalidation as a landing blocker. Tying eviction to a confirmed PD mismatch means a fallback caused by malformed input ranges (which `TestOverlappingRangesCanProduceOutOfOrderRangesAfterSplit` shows is possible) does not evict a perfectly good entry. The PD lookup is already made on this path, so the confirmation is free.
  Date/Author: 2026-10-20 / plan author.

- Decision: use `InvalidateCachedRegionWithReason(id, tikv.EpochNotMatch)` rather than plain `InvalidateCachedRegion`.
  Rationale: the reason is recorded in client-go metrics and is the honest description of what happened. The repo already uses this form for the same situation in `pkg/store/copr/mpp.go` line 187 and `pkg/store/copr/batch_coprocessor.go` line 1556, so it follows package-local convention.
  Date/Author: 2026-10-20 / plan author.

- Decision: keep the existing `bucketVersion` field on `bucketSplitFallbackInfo` (the branch's current six-line delta) and use it in the comparison against `pdLoc.GetBucketVersion()`.
  Rationale: it is already populated at both fallback sites and turns the diagnostic-only field into a load-bearing one, which justifies its existence.
  Date/Author: 2026-10-20 / plan author.

- Decision: append the regression scenario as a sequential block at the end of the existing `TestBuildTasksByBuckets` in `pkg/store/copr/coprocessor_test.go`, rather than adding a new top-level `TestXxx`.
  Rationale: `pkg/store/copr` already has 51 top-level `Test` functions (`grep -h '^func Test' pkg/store/copr/*_test.go | wc -l`) against `shard_count = 50` in `pkg/store/copr/BUILD.bazel`. `.agents/skills/tidb-test-guidelines/SKILL.md` sets a practical target of at most 50 tests per directory and says that for follow-up bug fixes one should prefer appending to an existing semantically close suite, keeping sequential scenario blocks rather than introducing `t.Run`. `TestBuildTasksByBuckets` already boots a mock cluster with buckets and drives `buildCopTasks`, which calls `SplitKeyRangesByBuckets` (`pkg/store/copr/coprocessor.go` line 566), so it is the closest suite. A side benefit: no new top-level `TestXxx` means the `make bazel_prepare` trigger in `AGENTS.md` does not fire for the test, provided the import section is untouched.
  Date/Author: 2026-10-20 / plan author.

- Decision: this store-side fix lands first; pingcap/tidb#56601 waits for it. Rafael has rebased #56601 onto `master` and will build on top of the merged store fix rather than in parallel. Consequently, this plan owns the evict-and-refresh behavior in `pkg/store/copr/region_cache.go`, and #56601 is expected to reuse or sit alongside it rather than introduce a competing helper.
  Rationale: Rafael's own call, communicated on 2026-10-20, which removes the ordering ambiguity noted under `Interfaces and Dependencies`. It also raises the stakes on the completion gate here: #56601 is now blocked on this PR, so the `Ready` profile, Yuki's benchmark, and the scope decision on the client-go lock paths should not be allowed to drift.
  Date/Author: 2026-10-20 / plan author, recording Rafael's decision.

- Decision: place this plan at `docs/execplans/region-cache-stale-on-split.md`.
  Rationale: `PLANS.md` does not prescribe a location; `docs/agents/*` is reserved for agent playbooks, and a dedicated `docs/execplans/` directory keeps plans discoverable without polluting that space.
  Date/Author: 2026-10-20 / plan author.


## Outcomes & Retrospective


Not yet started. Fill in at Milestone 2 (behavior proven) and again at Milestone 4 (ready for review).


## Context and Orientation


Everything in this plan lives under `pkg/store/copr/`, the package that turns a SQL query's key ranges into coprocessor tasks addressed to specific TiKV regions.

Terms used below:

- Region: a contiguous key range served by one group of TiKV replicas. Identified by `tikv.RegionVerID`, which carries the region ID plus its epoch (`ver`, `conf_ver`).
- Region epoch: `ver` increments on split or merge; `conf_ver` increments on peer membership change. If a request carries an old epoch, TiKV answers with an `EpochNotMatch` region error.
- Buckets: an optional list of split keys inside a region (`metapb.Buckets`, with its own `Version`), used to divide one region's work into smaller tasks. Buckets are an optimization; correctness never depends on them.
- Location: `tikv.KeyLocation`, the cached record for one region: `Region` (the `RegionVerID`), `StartKey`, `EndKey`, `Buckets`.
- `LocationKeyRanges`: a location paired with the subset of the query's key ranges that fall inside it. Defined at `pkg/store/copr/region_cache.go` line 525.
- `bucketSplitFallbackInfo`: the struct returned when bucket splitting gives up. Defined at line 40 of the same file. Its `reason` string is one of `range_start_outside_location`, `bucket_not_contain_start_no_progress`, or the third case at line 567 (read the code; it is the "no buckets after locate" case).

Key functions, all in `pkg/store/copr/region_cache.go`:

- `(*LocationKeyRanges).splitKeyRangesByBuckets(ctx)` (line 537): splits one location's ranges by its buckets. Returns `(result, nil)` on success or `([]*LocationKeyRanges{l}, &bucketSplitFallbackInfo{...})` when the cached buckets do not line up with the ranges.
- `(*RegionCache).SplitKeyRangesByBuckets(bo, ranges)` (line 790): the public entry. Calls `SplitKeyRangesByLocations` with `buckets=true`, then iterates locations calling the function above. If any location returns a fallback, it builds a long zap field list (lines 863 to 969), including a best-effort PD lookup via `c.RegionCache.LocateRegionByIDFromPD(bo.TiKVBackoffer(), cachedLoc.Region.GetID())`, logs one `Warn`, then re-splits with `buckets=false` and returns that. This is the branch to change.

The invalidation API comes from client-go and is reachable through the embedded `*tikv.RegionCache` on `copr.RegionCache` (line 498). Methods already used in this repo and therefore known to exist: `InvalidateCachedRegion(id tikv.RegionVerID)`, `InvalidateCachedRegionWithReason(id tikv.RegionVerID, reason tikv.InvalidReason)` with constant `tikv.EpochNotMatch`, `TryLocateKey(key []byte) *tikv.KeyLocation`, and `LocateRegionByIDFromPD(bo *tikv.Backoffer, regionID uint64) (*tikv.KeyLocation, error)`.

Test infrastructure already in the package: `testutils.NewMockTiKV`, `testutils.BootstrapWithMultiRegions`, `cluster.SplitRegionBuckets`, `cluster.Split(oldRegionID, newRegionID, splitKey, newPeerIDs, leaderStoreID)`, `cluster.AllocID()`, `buildCopRanges`, `NewRegionCache(tikv.NewRegionCache(pdCli))`. See `TestPanicInSplitKeyRangesByBuckets` (region_cache_test.go line 350) and `TestSplitKeyRanges` (coprocessor_test.go line 473) for complete examples of wiring them up.

Build and test conventions from `AGENTS.md` and `docs/agents/testing-flow.md`: unit tests run from the package directory with `go test -run <TestName> -tags=intest,deadlock`. This package uses failpoints (`failpoint.Inject("panicInSplitKeyRangesByBuckets", ...)`), so tests must run through `./tools/check/failpoint-go-test.sh pkg/store/copr -run <TestName>` from the repository root. Adding any new top-level `func TestXxx(t *testing.T)` or `func BenchmarkXxx(b *testing.B)` to an existing `_test.go` file requires `make bazel_prepare` afterwards and committing the resulting `BUILD.bazel` changes.


## Plan of Work


Milestone 1 adds the regression scenario first, before any product change, as `AGENTS.md` requires for bug fixes. Append a sequential block at the end of `TestBuildTasksByBuckets` in `pkg/store/copr/coprocessor_test.go` (the function starts at line 189; the mock cluster, `cache`, and `bo` it sets up are reused, and `testutils.BootstrapWithMultiRegions` must be changed to capture its first return value, the store ID, which the current code discards with `_`). Introduce the block with a comment naming the behavior under test and stating which assertion fails before the fix. The block: splits region `regionIDs[1]` (currently `[n, x)`) at `q` via `cluster.Split(regionIDs[1], newRegionID, []byte("q"), []uint64{newPeerID}, storeID)` with IDs from `cluster.AllocID()`, then gives the two children buckets consistent with their new boundaries via `cluster.SplitRegionBuckets`, so the entry the cache already holds for `[n, x)` with its old buckets is now stale relative to PD. It then calls `cache.SplitKeyRangesByBuckets(bo, buildCopRanges("n", "x"))` and asserts two things: the returned locations cover `[n, q)` and `[q, x)` with the correct region IDs, and `cache.RegionCache.TryLocateKey([]byte("q"))` returns a location whose region ID is the new child, proving the stale entry was evicted rather than merely bypassed. The block must not call `InvalidateCachedRegion` itself. Do not introduce `t.Run`; keep the sequential style the surrounding test already uses. Use a behavior-based comment, not an issue-id-only one, and include the linked issue number in a trailing comment for traceability, as `.agents/skills/tidb-test-guidelines` asks.

Because `TestBuildTasksByBuckets` is listed under `pkg/store/copr/coprocessor_test.go` in `.agents/skills/tidb-test-guidelines/references/store-case-map.md` (line 22), update that line's one-sentence description to mention stale-region invalidation after split; the guideline says to update the case map when one exists.

If, while writing this test, the mock cluster's `Split` turns out to update the cached location through some path the author did not anticipate (for example if `cluster.Split` is observed by the cache), record that under `Surprises & Discoveries` and instead construct the staleness by calling `SplitRegionBuckets` on the old region with keys that lie outside its post-split boundary, which is the shape `TestLocateBucketOutsideRegionNonNilFallback` already exercises. The acceptance criterion is unchanged: fallback must trigger, and the stale entry must be gone afterwards without a manual invalidate.

Milestone 2 makes the test pass. In `(*RegionCache).SplitKeyRangesByBuckets`, inside the `if fallback != nil` block, locate the existing PD lookup (`pdLoc, pdErr := c.RegionCache.LocateRegionByIDFromPD(...)`). Immediately after the `else` branch that appends the `pdRegionVer` and related fields, and before the final `logutil.Logger(ctx).Warn("SplitKeyRangesByBuckets fell back to region-only splitting", fields...)`, add the invalidation. Compute `staleConfirmed` as true when `pdErr == nil` and any of: `pdLoc.Region.GetVer() != cachedLoc.Region.GetVer()`, `pdLoc.Region.GetConfVer() != cachedLoc.Region.GetConfVer()`, `!bytes.Equal(pdLoc.StartKey, cachedLoc.StartKey)`, `!bytes.Equal(pdLoc.EndKey, cachedLoc.EndKey)`, or `pdLoc.Buckets != nil && pdLoc.GetBucketVersion() != fallback.bucketVersion`. When `staleConfirmed` is true, call `c.RegionCache.InvalidateCachedRegionWithReason(cachedLoc.Region, tikv.EpochNotMatch)` and append `zap.Bool("staleRegionInvalidated", true)` to `fields`; otherwise append `zap.Bool("staleRegionInvalidated", false)`. Reuse the two booleans the code already computes for the log (`pdEpochChanged`, `pdBoundaryChanged`) by hoisting them into named variables rather than duplicating the expressions. Add a short comment above the call stating the invariant in plain words: a cached location that PD has superseded must be evicted here, because the region-only re-split below and every later statement read the same cache, and nothing else on this path would evict it.

Do not touch the panic-recovery `defer` earlier in the function, the normal non-fallback return, or `splitKeyRangesByBuckets` itself. The branch's existing six-line `bucketVersion` change stays as is.

Milestone 3 adds the benchmark Yuki asked for, in `pkg/store/copr/region_cache_test.go` as `BenchmarkSplitKeyRangesByBucketsSteadyState`. Benchmarks do not count toward the directory test target and `func BenchmarkXxx` does not match the `make bazel_prepare` trigger pattern, so this is safe to add as a new top-level function. Boot a mock cluster with four regions and buckets exactly as `TestPanicInSplitKeyRangesByBuckets` does, warm the cache once, then in the timed loop call `cache.SplitKeyRangesByBuckets(bo, ranges)` repeatedly with ranges spanning all regions. After the loop, assert via `require` (benchmarks may use `require.New(b)`) that `cache.RegionCache.TryLocateKey([]byte("a"))` still returns the originally cached region ID, which proves no invalidation occurred during steady-state operation. Run it on `master` and on the branch; the numbers should be indistinguishable because the changed code is inside the fallback branch, which this workload never enters. Record both numbers in `Artifacts and Notes`. No `Makefile` `bench-daily` entry is needed; that wrapper is only updated when benchmarks move between packages.

Milestone 4 is the completion gate, in this order. First, run the `bazel_prepare` gate from `.agents/skills/tidb-bazel-prepare-gate` and record the evidence: with the regression block appended to an existing test and no import changes, none of the triggers in `AGENTS.md` -> `Build Flow` should fire, so `make bazel_prepare` is not expected to be needed; if `git diff -U0 -- '*.go'` shows an added import line or a `+func Test` line, run it and commit the `BUILD.bazel` change. Second, update `store-case-map.md` as described in Milestone 1. Third, run the `Ready` profile from `.agents/skills/tidb-verify-profile`: the targeted failpoint-enabled tests listed in `Concrete Steps`, then `make lint`. Fourth, prepare the PR body from `.github/pull_request_template.md` following `.agents/skills/tidb-pr-metadata-guard`: title in the form `store: ...` (the existing title "store: fix region-cache staleness on epoch-not-match during split" fits), an `Issue Number: close #<id>` line pointing at the issue already linked from the PR (if that issue does not exist on the repository the final PR targets, create one first following `.agents/skills/tidb-issue-metadata-guard`: search existing issues and PRs, start from the matching file under `.github/ISSUE_TEMPLATE/`, keep the repro summary and expected versus actual behavior visible on the first screen, and put long logs in a `<details>` block), the `Unit test` checkbox ticked with the exact commands from `Concrete Steps` pasted under it, the `Tests <!-- ... -->` HTML comment left verbatim, and a non-`None` release note because this is a bug fix. Fifth, self-review the full diff before claiming completion, as the `Before finishing` row of the `AGENTS.md` Quick Decision Matrix asks: read `git diff master...HEAD` end to end, confirm every hunk is explained by this plan, and remove anything incidental. Sixth, update this document's `Progress`, `Outcomes & Retrospective`, and `Artifacts and Notes`, and write the final report in the `Agent Output Contract` shape from `AGENTS.md`.

When addressing Yuki's inline review comments on the PR, `.agents/skills/tidb-change-instruction-critic` applies: treat each prescribed fix as a hypothesis, restate the defect from the code, and record the chosen approach in this plan's `Decision Log` before editing.


## Concrete Steps


All commands run from the repository root unless stated.

Check out the branch and confirm the starting delta:

    git fetch origin petrova-store-fix-region-cache-stale-on-split
    git checkout petrova-store-fix-region-cache-stale-on-split
    git diff --stat master...HEAD

Expected: one file, `pkg/store/copr/region_cache.go`, six insertions.

Failpoint decision (required by `docs/agents/testing-flow.md` before any unit test run in this package; record the output as evidence in the final report):

    rg -n --fixed-strings -- "failpoint." pkg/store/copr
    rg -n --fixed-strings -- "testfailpoint." pkg/store/copr
    test -f pkg/store/copr/BUILD.bazel && rg -n --fixed-strings -- "@com_github_pingcap_failpoint//:failpoint" pkg/store/copr/BUILD.bazel

Expected: the first command matches (at minimum `region_cache.go` for `panicInSplitKeyRangesByBuckets`), and the third matches in `BUILD.bazel`. Therefore every unit test run below goes through `./tools/check/failpoint-go-test.sh`, never bare `go test`, and the script is not invoked from parallel tasks in the same worktree.

Milestone 1, after appending the regression block:

    ./tools/check/failpoint-go-test.sh pkg/store/copr -run '^TestBuildTasksByBuckets$' -count=1

Expected: FAIL. The failure message should name the assertion on the post-split `SplitKeyRangesByBuckets` call or on `TryLocateKey("q")`, showing the old region ID where the new child was expected. All assertions in the pre-existing part of the test must still pass; if an earlier assertion fails, the appended block has disturbed shared state and must be reworked. Paste the failing assertion line into `Artifacts and Notes`.

Milestone 2, after the product change:

    ./tools/check/failpoint-go-test.sh pkg/store/copr -run '^(TestBuildTasksByBuckets|TestLocateBucketNilFallback|TestLocateBucketOutsideRegionNonNilFallback|TestOverlappingRangesCanProduceOutOfOrderRangesAfterSplit|TestPanicInSplitKeyRangesByBuckets|TestSplitKeyRanges|TestBuildTasksWithoutBuckets|TestStoreBatchTasksPreserveChildBucketsVersion|TestHandleBatchCopResponseUpdatesChildBucketsOnVersionNotMatch)$' -count=1

Expected: `ok  	github.com/pingcap/tidb/pkg/store/copr`. The three pre-existing fallback tests call `splitKeyRangesByBuckets` directly on a `LocationKeyRanges` and never reach the changed code, so they must still pass unchanged; if one fails, the change leaked outside the fallback branch of the public method. The two bucket-version tests cover the neighbouring `OnBucketVersionNotMatch` path and guard against accidental double invalidation.

Milestone 3:

    pushd pkg/store/copr
    go test -run '^$' -bench BenchmarkSplitKeyRangesByBucketsSteadyState -benchmem -count=5 -tags=intest,deadlock
    popd

Run the same command on `master` (`git stash` the change or use a worktree) and record both.

Milestone 4, gate check first:

    git status --short
    git diff -U0 -- '*.go' | grep -E '^\+(func Test|import|\t"[^"]+"$)' || echo "no bazel_prepare trigger"

Expected: `no bazel_prepare trigger`. If any line prints, run `make bazel_prepare` and commit the resulting `pkg/store/copr/BUILD.bazel` change. Record whichever outcome occurred.

Then the completion checks:

    make lint

Expected: exits 0. If `make lint` reports the new `fields = append(fields, ...)` block for line length or similar, fix formatting only; do not restructure surrounding code. Do not run `make bazel_lint_changed`; `AGENTS.md` forbids it without an explicit request.

Then the PR body: copy `.github/pull_request_template.md` to a scratch file, fill `Issue Number`, `Problem Summary`, `What changed and how does it work?`, tick `Unit test` and paste the two `failpoint-go-test.sh` commands above and the benchmark command, write the release note, and apply it with `gh pr edit --body-file <file>` (or `gh pr create -T .github/pull_request_template.md` for a fresh PR). Afterwards confirm neither `do-not-merge/needs-linked-issue` nor `do-not-merge/needs-tests-checked` is present on the PR.


## Validation and Acceptance


Acceptance is behavioral. Given a cache that holds a region entry PD has since split, one call to `(*RegionCache).SplitKeyRangesByBuckets` over that region's key range must return locations matching the post-split layout, and a subsequent `TryLocateKey` on a key inside the new child region must return the child, with no caller-side invalidation. The regression block appended to `TestBuildTasksByBuckets` encodes exactly this and must fail before Milestone 2 and pass after.

Required validation surface, per `AGENTS.md` -> `Task -> Validation Matrix` row `pkg/store/**`: targeted unit tests; realtikv tests only if behavior depends on real TiKV. The mock cluster reproduces the split and the PD epoch bump faithfully for this path, so realtikv is not required. Per `docs/agents/testing-flow.md`, unit tests in this package run with failpoints enabled through the wrapper script. Per `.agents/skills/tidb-verify-profile`, use `WIP` (the single targeted run) while iterating and `Ready` (targeted runs plus `make lint`) before saying "ready for review".

Non-regression: the steady-state benchmark must show no invalidation when metadata is fresh (the post-loop region-ID assertion) and no measurable throughput change against `master`. A change in the warm-path numbers of more than run-to-run noise means the edit landed outside the fallback branch and must be investigated before proceeding.

Interpreting failures: if the regression test passes before the fix, the test is not actually producing a stale entry; revisit the mock-cluster setup per the fallback instructions in `Plan of Work`. If the test still fails after the fix, check the log line emitted by the fallback (`SplitKeyRangesByBuckets fell back to region-only splitting`): `staleRegionInvalidated=false` with `pdEpochChanged=true` means the condition is wrong; `staleRegionInvalidated=true` but the second call still returns the stale location means the fallback re-split happened before the invalidate (ordering bug: the invalidate must precede the `SplitKeyRangesByLocations(..., false)` call at the end of the block).


## Idempotence and Recovery


Every step is a local edit plus a test run and can be repeated freely.

If a test diff appears that this change does not explain, most likely after rebasing the branch onto a newer `master` (the copr test target is marked `flaky = True` in `pkg/store/copr/BUILD.bazel`, so CI retries can mask this), follow `.agents/skills/tidb-test-diff-triage`: first rule out failpoint setup by re-running the single test through `./tools/check/failpoint-go-test.sh pkg/store/copr -run '^<TestName>$' -count=1`; if the diff persists, reproduce on the merge-base commit and bisect the merged range to the first bad commit; never update expected outputs or testdata until the cause is proven. Record the outcome under `Surprises & Discoveries`. `make bazel_prepare` is idempotent. The product change is six to twelve lines inside one `if` block; rollback is `git checkout -- pkg/store/copr/region_cache.go` followed by removing the two new test functions and re-running `make bazel_prepare`. There are no data migrations, no generated-code edits, and no changes to request wire formats. Nothing in this plan requires a running TiKV cluster; the mock cluster is in-process.


## Artifacts and Notes


To be filled during execution. Required evidence at completion:

- The failing assertion output from Milestone 1.
- The passing `ok` line from Milestone 2.
- The two benchmark result tables (master vs branch) from Milestone 3.
- The failpoint-decision `rg` output and the `bazel_prepare` gate output from Milestone 4.
- The `git diff --stat` of the final change and the `make lint` exit status from Milestone 4.
- The `Agent Output Contract` summary from `AGENTS.md`: files changed, validation profile (`Ready`), risks (correctness, compatibility, performance), exact commands, and what was not verified locally. Expected "not verified locally": behavior against a real TiKV split under load; the `Task -> Validation Matrix` row for `pkg/store/**` only calls for realtikv tests when behavior depends on real TiKV, and the mock cluster reproduces the split faithfully for this path, so realtikv is not required unless a reviewer asks.


## Interfaces and Dependencies


No new types, exported functions, or dependencies. The change uses only symbols already imported in `pkg/store/copr/region_cache.go`: `bytes`, `zap`, `tikv` (for `tikv.EpochNotMatch`), and the embedded `*tikv.RegionCache` methods named in `Context and Orientation`. `bucketSplitFallbackInfo` keeps its current shape; its `bucketVersion uint64` field becomes load-bearing.

Relationship to pingcap/tidb#56601 ("store/planner: refresh region-epoch on physical plan cache hit", opened by Rafael on July 7, 2026). That PR adds a proactive epoch check before a cached physical plan is reused and touches `store/regioncache_util.go` and `planner/physicalplan_cache.go`, neither of which exists on this fork. This plan is reactive and lives entirely in `pkg/store/copr/region_cache.go`, so there is no file-level overlap and no logical dependency in either direction: #56601 reduces how often a stale plan is executed, this change ensures the region cache heals itself when stale metadata is detected during task building. Both are needed. Two things to say when replying to Rafael: first, that this store-side work does not touch the plan-cache path and does not answer his sequencing question; second, the actual answer to his question, which only the author of #56489 (the memo-group interning fix) can give, namely whether the post-#56489 normalization order finalizes the plan-cache key before or after the cache-hit branch where his epoch refresh runs. Ordering is now settled: Rafael has rebased #56601 onto `master` and is waiting for this store fix to merge before building on it (see `Decision Log`). That makes this PR the base, so the `InvalidateCachedRegionWithReason` call here is the canonical eviction point; if #56601 later wants a shared helper, it should be extracted from this code, not introduced beside it. It also means #56601 is blocked on this plan reaching `Ready`.

Out of scope and tracked separately: epoch re-validation in the resolve-lock recovery path and the pessimistic-lock response path. Both live in `github.com/tikv/client-go/v2` (packages `txnkv/txnlock` and `txnkv/transaction`) and require a client-go pull request followed by a `go.mod` bump in this repository. The invariant to carry into that work is the same one stated in the code comment added in Milestone 2.


## Revision note (2026-10-20, second pass)


Checked the validation sections against the repository's own testing guidance and revised: added the one-minute `Summary`; moved the regression scenario from a new top-level test into a sequential block appended to `TestBuildTasksByBuckets` because the package already exceeds its `shard_count` test target; added the mandatory failpoint-decision commands; replaced the unconditional `make bazel_prepare` with the gate check from `.agents/skills/tidb-bazel-prepare-gate`; added the `store-case-map.md` update, the PR-template and release-note requirements from `.agents/skills/tidb-pr-metadata-guard`, the `tidb-change-instruction-critic` note for Yuki's review comments, and the `Task -> Validation Matrix` mapping. No change to the product fix itself.


## Revision note (2026-10-20, third pass)


Recorded Rafael's decision to rebase pingcap/tidb#56601 onto `master` and wait for this store fix to merge. Added a Decision Log entry and replaced the conditional "if #56601 lands first" guidance under `Interfaces and Dependencies` with the settled ordering. No change to milestones or validation.


## Revision note (2026-10-20, fourth pass)


Added three repository checks the plan did not yet name: the self-review step from the `AGENTS.md` Quick Decision Matrix in Milestone 4, conditional issue creation per `tidb-issue-metadata-guard` in the PR step, and the `tidb-test-diff-triage` workflow under `Idempotence and Recovery` for unexplained test diffs after a rebase.

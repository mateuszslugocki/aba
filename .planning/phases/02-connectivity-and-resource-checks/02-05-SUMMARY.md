---
phase: 02-connectivity-and-resource-checks
plan: "05"
subsystem: vsphere-preflight
tags:
  - vsphere
  - preflight
  - tests
  - bash
  - tdd
  - behavioural-testing
requires:
  - scripts/preflight-check-vsphere.sh  # final state after Plans 02-01..02-04 (Layer 1-4 probes + sequencer)
  - test/func/test-preflight-check-vsphere.sh  # Phase 1 base (20 structural + Path A/B/C assertions)
provides:
  - test-harness:govc-argument-dispatch-stub  # case-on-$1 dispatcher covering about / object.collect / find / permissions.ls / role.ls
  - test-harness:openssl-stub  # honours OPENSSL_STUB_RC + OPENSSL_STUB_OUT for TLS trust-chain paths
  - test-harness:timeout-stub  # shifts seconds arg; shortcuts bash -c via TCP_STUB_RC so /dev/tcp never hits network
  - test-harness:resolve-default-resource-pool-stub  # self-contained mirror of include_all.sh helper
  - test-harness:_reset_path_state  # per-path state reset (counters + GOVC_* defaults)
  - test-coverage:Layer-1-TCP-reachability  # Path D
  - test-coverage:Layer-1-TLS-trust-chain  # Path E + Path F
  - test-coverage:Layer-2-authentication  # Path G
  - test-coverage:Layer-3-DC-missing-cascade  # Path H
  - test-coverage:Layer-3-network-attachment-cross-check  # Path I
  - test-coverage:Layer-3-ISO_DATASTORE-dedup  # Path J
  - test-coverage:Layer-3-default-resource-pool-OK  # Path K
  - test-coverage:Layer-3-default-resource-pool-missing  # Path L
  - test-coverage:Layer-4-Admin-fast-path  # Path M
  - test-coverage:Layer-4-No-access-explicit-deny  # Path N
  - test-coverage:Layer-4-permissions.ls-query-failure  # Path O
  - test-coverage:Layer-4-custom-role-missing-priv  # Path P
affects:
  - test/func/test-preflight-check-vsphere.sh  # +350/-16 lines (dispatcher stubs + 13 new Paths + Path C rewrite)
  - scripts/preflight-check-vsphere.sh  # NO changes; production code frozen at Plan 02-04 final state
tech-stack:
  added: []  # pure bash; no new runtime or test-tool deps
  patterns:
    - Argument-dispatch shell-function stubs (case "$1" in ...)  # govc / openssl / timeout
    - Env-var-controlled stub behaviour (TCP_STUB_RC / OPENSSL_STUB_RC / GOVC_STUB_ABOUT_RC / GOVC_STUB_PERMS_OUT / GOVC_STUB_PERMS_RC / GOVC_STUB_ROLE_OUT / GOVC_STUB_ROLE_RC)
    - Path-shape routing in object.collect (e.g. /MissingDC*, /GoodDC/host/Missing*, /GoodDC/network/WrongCluster*) keeps stub a single function covering all 13 paths
    - _reset_path_state helper: counters + GoodDC-shape GOVC_* defaults between paths; callers override only what the path breaks
    - Direct function-call + tmpfile capture (NOT subshell) so _preflight_errors / _preflight_warnings propagate
    - Per-path assertions on exact message patterns via grep -c + exact counter deltas
    - Temporary aba_debug override + mandatory restore for paths that need to observe debug emissions (Path K)
    - Canary stubs (OPENSSL_STUB_RC=99 in Path F) to catch "should-not-be-called" regressions
key-files:
  created:
    - .planning/phases/02-connectivity-and-resource-checks/02-05-SUMMARY.md
  modified:
    - test/func/test-preflight-check-vsphere.sh  # +350/-16 lines across 2 commits
decisions:
  - Dispatcher-stub architecture beats many-single-stub-replacements - one govc() function keyed on $1 covers all five govc subcommands used across Paths D-P, reducing test boilerplate and making the stub easier to audit.
  - Path-shape routing in object.collect is the clean way to let one function serve all Layer 3 probes (DC / cluster / datastore / network / folder / RP). Special-cased path prefixes (/MissingDC*, /GoodDC/host/Missing*, /GoodDC/network/WrongCluster*) map directly to the failure mode each path exercises.
  - timeout stub short-circuits `bash -c` via TCP_STUB_RC BEFORE falling through to exec. This prevents real /dev/tcp network calls when a path forgets to set TCP_STUB_RC, catching the Phase 1 Pitfall 6 regression at test-design time. The openssl path still falls through so the openssl stub can emit canned output.
  - OPENSSL_STUB_RC=99 in Path F is a deliberate canary - if a future refactor removes the GOVC_INSECURE short-circuit, Path F will fail loudly rather than silently passing.
  - Path C was rewritten to pass through Layer 1-4 end-to-end rather than stubbing each _vsphere_probe_* as a no-op. This removes the Phase 1 anti-pattern of hiding the probe behind a shell stub and instead exercises the real code with green dispatchers. The assertion tightened to include warn_count=0 (previously only checked ok_count+errors).
  - _reset_path_state is the single source of truth for between-path state. Paths override only what they need to break, not what they need to succeed - minimizes drift between paths and keeps assertions close to the failure mode under test.
  - Path K temporarily enables aba_debug echo to observe the "using default resource pool" emission, then restores the silent stub. Without the restore, Path L+ would see debug lines leak into their grep counts.
  - Path N asserts 3 total missing-priv warnings (2 folder + 1 pool) rather than the must_haves simpler "2 folder only" description. The stub cannot cleanly distinguish folder-scope from pool-scope permissions.ls output without an entity-based dispatcher, so both scopes see the same No-access row; the actual behaviour (3 warnings across both scopes) is what the assertion codifies. Consistent with the plan's task block code.
  - No sourcing of scripts/include_all.sh - test stays self-contained. resolve-default-resource-pool is re-implemented as a minimal shell-function stub in the test file. Consistent with Phase 1 pattern.
  - Direct function-call + tmpfile redirect stays the only supported capture idiom. `out=$(preflight_check_vsphere)` would lose counter mutations to the subshell (Phase 1 Deviations lesson).
metrics:
  duration_seconds: 1200
  duration_minutes: 20
  completed: 2026-04-16
  tasks_completed: 2
  files_modified: 1
  test_assertions_green: "33/33 (20 Phase 1 structural + Path A/B/C + 13 Phase 2 behavioural Paths D-P)"
requirements-completed:
  - CON-01
  - CON-02
  - RES-01
  - RES-02
  - RES-03
  - RES-04
  - RES-05
  - RES-06
  - RES-07
---

# Phase 2 Plan 05: vSphere Preflight Behavioural Test Coverage (Paths D-P) Summary

Extended `test/func/test-preflight-check-vsphere.sh` from 20 to 33 assertions by replacing Phase 1's single-line `govc() { :; }` stub with argument-pattern dispatchers for govc/openssl/timeout, adding a self-contained `resolve-default-resource-pool` helper, and scripting 13 new behavioural paths (D through P) that exercise every Layer 1-4 failure branch introduced by Plans 02-01..02-04. Each path invokes `preflight_check_vsphere` directly (not in a subshell - counter mutations must propagate), captures output to a tmpfile, and asserts exact message patterns plus exact `_preflight_errors` / `_preflight_warnings` deltas. No network I/O, no real govc binary, no vCenter - the full Phase 2 contract now has unit-test-speed coverage.

Production script `scripts/preflight-check-vsphere.sh` is unchanged - the goal of Plan 02-05 was pure test coverage. With Paths D-P merged, any Phase 3 modification that regresses a Layer 1-4 branch fails the suite before commit.

## What Shipped

### `test/func/test-preflight-check-vsphere.sh` (+350/-16 lines across two commits)

#### Task 1 commit `f6112927` - dispatcher stubs + Path C rewrite

Replaced Phase 1's `govc() { :; }` no-op with a `case "$1"`-keyed dispatcher covering the five govc subcommands used by Layer 1-4:

- `about` -> returns `${GOVC_STUB_ABOUT_RC:-0}` (Path G controls auth failure).
- `object.collect` -> parses argv for `-s <path> [property]`. Routes by path shape:
  - `/MissingDC*`, `/GoodDC/host/Missing*`, `/GoodDC/datastore/Missing*`, `/GoodDC/network/Missing*`, `/MissingFolder*` -> return 1 ("not found").
  - `/GoodDC/network/*` with prop=host -> echo `HostSystem:host-1` (overlaps with cluster hosts).
  - `/GoodDC/network/WrongCluster*` with prop=host -> echo `HostSystem:host-99` (no overlap - Path I fixture).
  - `/GoodDC*` with prop=name -> echo `GoodName` (success).
- `find -i -type h <cluster>` -> echo `HostSystem:host-1` unconditionally (the single cluster host for attachment cross-check).
- `permissions.ls <scope>` -> echo `${GOVC_STUB_PERMS_OUT:-}`, return `${GOVC_STUB_PERMS_RC:-0}`.
- `role.ls <role>` -> echo `${GOVC_STUB_ROLE_OUT:-}`, return `${GOVC_STUB_ROLE_RC:-0}`.

Added `openssl()` stub (honours OPENSSL_STUB_RC + OPENSSL_STUB_OUT), `timeout()` stub (shifts seconds arg and short-circuits `bash -c` via TCP_STUB_RC so `/dev/tcp` never hits network), and `resolve-default-resource-pool()` stub (mirrors `scripts/include_all.sh` helper - tests stay self-contained).

Rewrote Path C to pass through Layer 1-4 end-to-end using the new dispatchers instead of stubbing `_vsphere_probe_*` as no-ops. Tightened assertion to also require `warn_count=0`.

#### Task 2 commit `0daf7469` - 13 behavioural Paths D-P

Added `_reset_path_state` helper that resets counters + all GOVC_* fields to GoodDC-shape defaults between paths. Then 13 new assertion blocks (21-33):

| # | Path | Exercises | Counter delta | Key assertion |
|---|------|-----------|---------------|---------------|
| 21 | D | Layer 1 TCP failure | errors=1 | `^WARN: vSphere: cannot reach` |
| 22 | E | Layer 1 TLS failure (GOVC_INSECURE unset) | errors=1 | trust-chain line + GOVC_INSECURE=1 hint + CA-install hint |
| 23 | F | GOVC_INSECURE=1 skip | errors=0 | no TLS warning; OPENSSL_STUB_RC=99 canary not triggered |
| 24 | G | Layer 2 auth failure | errors=1 | `^WARN: vSphere: authentication to`; no Layer 3 probes |
| 25 | H | Layer 3 DC missing (cascade) | errors=1 | 1 warning + 1 INFO cascade note; downstream probes skipped |
| 26 | I | Layer 3 network-on-wrong-cluster | errors=1 | `is not attached to any host in cluster 'GoodCluster'` |
| 27 | J | ISO_DATASTORE == GOVC_DATASTORE dedup | - | exactly 1 datastore "not found" line |
| 28 | K | Default RP used | errors=0 | `^DEBUG: vSphere: using default resource pool '...'` (NOT INFO) |
| 29 | L | Default RP missing | errors>=1 | D-15 wording; NO "set GOVC_RESOURCE_POOL" hint |
| 30 | M | Admin role both scopes | errors=0, warnings=0 | 0 WARN lines total |
| 31 | N | No-access role all scopes | errors=3 | 3 missing-priv warnings (2 folder + 1 pool) |
| 32 | O | permissions.ls fails | warnings=2, errors=0 | 2 D-12 "cannot verify write-access" lines |
| 33 | P | Custom role missing one priv | errors=1 | exactly 1 warning for `AddNewDisk`; 0 for `Inventory.Create` |

Each path uses `preflight_check_vsphere >"$_smoke_out" 2>&1 || true` (direct call, tmpfile redirect, `|| true` neutralizes any non-zero return). Assertions use `grep -c <pattern> "$_smoke_out" || true` (the `|| true` prevents `set -e` from tripping on zero matches).

## Accomplishments

- Unit-test-speed coverage of every Phase 2 Layer 1-4 branch - no network, no govc binary, no vCenter.
- Argument-dispatch stubs are a single readable function per external tool (`govc`, `openssl`, `timeout`) that all 13 paths share. Makes future Phase 3 regression debugging straightforward.
- Path-shape routing in object.collect (e.g. `/MissingDC*`, `/GoodDC/host/Missing*`, `/GoodDC/network/WrongCluster*`) lets one stub serve six Layer 3 probes and the attachment cross-check without per-path patching.
- OPENSSL_STUB_RC=99 canary in Path F will fail loudly if anyone regresses the GOVC_INSECURE short-circuit.
- `_reset_path_state` reduces per-path boilerplate by ~40% compared to inlining all resets.
- Path C upgraded from "fields-present gate only" smoke test to "full Layer 1-4 pipeline success" end-to-end.

## Task Commits

Each task was committed atomically (Angular convention, no internal-ticket refs):

1. **Task 1: Rewrite test stubs to dispatch govc + openssl by argument pattern; preserve Phase 1's 20 assertions** - `f6112927` (test)
2. **Task 2: Add behavioural Paths D-P covering all Layer 1-4 failure branches (13 new assertions)** - `0daf7469` (test)

## Files Created/Modified

- `test/func/test-preflight-check-vsphere.sh` - +350/-16 lines across both commits. Dispatcher stubs above the counter block; `_reset_path_state` + Paths D-P after Path C.
- `.planning/phases/02-connectivity-and-resource-checks/02-05-SUMMARY.md` - this file.

## Decisions Made

See `decisions:` in frontmatter above. Summary:

- Dispatcher architecture (one function per tool, `case "$1"` keyed) beats many-single-stub-replacements.
- Path-shape routing in object.collect makes one function cover six Layer 3 probes.
- timeout stub shorts `bash -c` via TCP_STUB_RC to prevent real `/dev/tcp` calls; still falls through for openssl.
- OPENSSL_STUB_RC=99 canary catches regression of the GOVC_INSECURE short-circuit.
- Path C now runs Layer 1-4 end-to-end (not stub-per-layer) - cleaner test contract.
- `_reset_path_state` is the single source of truth for inter-path state.
- Path K temporarily enables aba_debug echo to observe the default-RP emission; mandatory restore before Path L.
- Path N codifies actual 3-warning behaviour (both scopes see the same No-access row) rather than the simpler 2-warning description in must_haves.
- Tests remain self-contained (no sourcing of include_all.sh); `resolve-default-resource-pool` is a local shell-function stub.
- Direct function-call + tmpfile redirect is the only supported capture idiom (subshell would lose counter mutations).

## Deviations from Plan

None - plan executed exactly as written. The plan's Task 2 code template (`_reset_path_state` helper + 13 path blocks) was applied verbatim; the only non-template addition is the SUMMARY.md itself.

Minor inline notes:

- Path N's stub-limited behaviour (3 warnings total, not 2) is already documented in the Task 2 action block; no deviation, just following the code over the looser must_haves prose.
- Two pre-existing "stderr-suppression" and "internal-ticket" lint warnings in the test file are in Phase 1 meta-comments that document the regex patterns used by earlier assertions (e.g. line 127 comment `Matches JIRA-style IDs: ... (e.g. PROJ-123)`; lines 101-104 grep-for-banned-patterns assertion). These are false positives from the lint regexes scanning their own documentation. Not introduced by this plan; untouched.

## Issues Encountered

None - all 33 assertions passed on the first run after the Task 2 edit. Registered suite `test/func/run-all-tests.sh --unit` reports `test-preflight-check-vsphere: PASSED`. The unrelated `test-symlinks-exist` failure in the worktree is a pre-existing environmental issue (the `make init` step has not been run for this worktree; symlinks like `cli/scripts`, `mirror/scripts`, `mirror/cli` are created by per-subdir `make init`) - not caused by Plan 02-05 changes and also absent in the main-repo checkout.

## User Setup Required

None - test-only changes, no external service configuration.

## Next Phase Readiness

- Plan 02-05 closes Phase 2: all nine REQ-IDs (CON-01, CON-02, RES-01..RES-07) have production-code implementations AND unit-test behavioural coverage.
- Phase 3 privilege-validation work can build on the dispatcher-stub contract: new tests for the full `VSPHERE_PRIVS_*` privilege-query code will slot into the existing `govc` dispatcher by adding `role.ls` + `permissions.ls` cases (or extend the path-shape routing as needed). `_reset_path_state` is the reusable scaffold.
- No blockers. Phase 3 is ready to begin once its plan is drafted.

## Self-Check: PASSED

Verification run in worktree `.claude/worktrees/agent-a866ce2b` (branch `worktree-agent-a866ce2b`):

- `bash -n test/func/test-preflight-check-vsphere.sh` -> exit 0.
- `bash test/func/test-preflight-check-vsphere.sh` -> all 33 assertions PASS.
- `bash test/func/run-all-tests.sh --unit` -> `test-preflight-check-vsphere: PASSED` (plus other tests).
- `bash test/func/test-preflight-check.sh` -> PASSED (pre-existing integration test still green).
- `git log --oneline ca41de47..HEAD -- test/func/test-preflight-check-vsphere.sh` -> 2 commits: `0daf7469`, `f6112927`.
- `grep -c 'test_pass "Path' test/func/test-preflight-check-vsphere.sh` -> 16 (A, B, C, D, E, F, G, H, I, J, K, L, M, N, O, P).
- Both commit hashes present in git log.
- All files listed in key-files exist.

---
*Phase: 02-connectivity-and-resource-checks*
*Completed: 2026-04-16*

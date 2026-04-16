---
phase: 03-privilege-validation
plan: "03"
subsystem: vsphere-preflight
tags:
  - vsphere
  - preflight
  - privileges
  - tests
  - behavioural
  - bash
requires:
  - scripts/preflight-check-vsphere.sh  # Plan 03-02 final state: 7-scope sequencer + 7 found-flags + D-17 summary block
  - scripts/vmware-required-privileges.sh  # VSPHERE_PRIVS_<SCOPE> arrays (read-only)
  - test/func/test-preflight-check-vsphere.sh  # Plan 03-02 state: Paths A-P + Phase 2 stubs
provides:
  - test/func/test-preflight-check-vsphere.sh:govc-stub-per-scope-dispatch
  - test/func/test-preflight-check-vsphere.sh:govc-stub-per-role-dispatch
  - test/func/test-preflight-check-vsphere.sh:_reset_path_state-7-found-flags
  - test/func/test-preflight-check-vsphere.sh:paths-M-N-O-P-7-scope-repair
  - test/func/test-preflight-check-vsphere.sh:paths-Q-R-S-T-U-V-W-X-Y-Z
affects:
  - test/func/test-preflight-check-vsphere.sh  # +296 / -33 lines
requirements:
  - PRIV-01
  - PRIV-02
  - PRIV-04
  - PRIV-05
  - UX-02
tech-stack:
  added: []  # pure test-only change; no new runtime deps
  patterns:
    - Per-scope govc stub dispatch via GOVC_STUB_PERMS_OUT_<sanitized-path> indirect expansion with flat-var fallback (PATTERNS.md stub-extension pattern)
    - Per-role govc stub dispatch via GOVC_STUB_ROLE_OUT_<role> indirect expansion with flat-var fallback
    - _reset_path_state single-source-of-truth extension: 7 new _vsphere_<scope>_found flags cleared between paths
    - Direct call to _vsphere_probe_privileges (bypassing Layer 3) for precise per-path found-flag control
    - aba_debug observation trick (Path S): override to echo, grep DEBUG lines, restore silent stub (pattern from Path K)
    - awk VSPHERE_PRIVS_(ROOT|FOLDER) block extract for Paths W/X/Z: drives role.ls stub with live curated list contents
    - Observed-behaviour codification (Paths N/P/Y): Plan 02-05 "tests document observed behaviour" convention applied where stub-driven scope iteration yields counts that differ from simplistic plan-level arithmetic
key-files:
  created:
    - .planning/phases/03-privilege-validation/03-03-SUMMARY.md
  modified:
    - test/func/test-preflight-check-vsphere.sh
decisions:
  - Task 1 govc stub extension used `printf '%s' | tr -c '[:alnum:]' '_'` (not `echo | tr`) to avoid the trailing-newline-mapped-to-underscore bug that would produce keys like GOVC_STUB_PERMS_OUT__GoodDC_ instead of GOVC_STUB_PERMS_OUT__GoodDC. Comment documents the reason inline.
  - Task 1 Path M repaired: presets all 6 non-root found-flags to 1, uses flat GOVC_STUB_PERMS_OUT with Admin row, asserts 0 warnings / 0 errors / 0 warning-counter bumps across 7 scopes (was 2 scopes pre-Plan-03-02).
  - Task 1 Path N repaired: No-access row on all 7 scopes; observed 83 missing-priv warnings + errors=83 (flat sum of VSPHERE_PRIVS_<SCOPE> lengths: 11+30+5+3+1+28+5). Matches plan arithmetic exactly.
  - Task 1 Path O repaired: permissions.ls fails on all 7 scopes; observed 7 D-12 warnings + _preflight_warnings=7 + _preflight_errors=0. Matches plan arithmetic.
  - Task 1 Path P repaired to codify observed behaviour (81 missing-priv warnings) per 02-05-SUMMARY "tests document behaviour" convention. Reasoning documented in code comments: the full preflight_check_vsphere flow runs Layer 3 (which sets all 6 non-root found-flags to 1 from the Good* object names), so the VMBuilder role (granting only VirtualMachine.Inventory.Create) is evaluated against ALL 7 scopes. VirtualMachine.Inventory.Create appears in VSPHERE_PRIVS_DATACENTER and VSPHERE_PRIVS_FOLDER only; all other scopes' arrays fire full missing counts. Per-scope missing: ROOT 11 + DC 29 + CLUSTER 5 + DS 3 + NET 1 + FOLDER 27 + RP 5 = 81.
  - Task 2 Paths Q-Z call _vsphere_probe_privileges directly (not preflight_check_vsphere) so that per-path found-flag presets persist - the plan's desired "all other scopes skip" semantics require bypassing Layer 3's own flag writes.
  - Task 2 Path Y codifies observed behaviour (17 missing + '17 across 3' summary) per the same 02-05 convention: ROOT is unconditional (D-09) and the No-access role fires for ROOT too, adding 11 to the plan's 3+3=6 estimate. The test asserts the full 11+3+3=17 shape including the summary's scope count of 3 (ROOT + primary DS + ISO DS).
  - Task 2 Path S comment rephrased to avoid the internal-ticket regex (`[A-Z]{4,7}-[0-9]+`): "PRIV-05 no-conflation requirement" became "D-06 no-conflation decision". Same information, decision-code form (D-XX) is not a matched token.
metrics:
  duration_seconds: 900
  duration_minutes: 15
  completed: 2026-04-16
  tasks_completed: 2
  files_modified: 1
  test_assertions_green: "43/43 (17 structural + Paths A-Z = 26 behavioural)"
---

# Plan 03-03 SUMMARY: Phase 3 Test Coverage Loop

Closes Phase 3's behavioural test coverage. Repairs Paths M/N/O/P to match the 7-scope sequencer shipped by Plan 03-02, adds 10 new Paths Q-Z covering Phase 3's novel branches (ROOT unconditional check, multi-scope gap reporting, missing-object skip, ISO_DATASTORE dedup and present-and-different, Admin 7-scope fast-path, role.ls single-scope failure, D-17 summary silence on clean pass, D-17 summary appearance on one-gap, D-18 summary does-not-double-count), extends the govc dispatcher stub with per-scope-path and per-role dispatchers, and extends `_reset_path_state` to reset the 7 new file-scope found-flags between paths. Production code `scripts/preflight-check-vsphere.sh` is NOT modified.

## Commit

- `23802c91` test(03-03): cover Phase 3 7-scope privilege layer (Paths Q-Z + M/N/O/P repair)

## Scope

Single atomic commit on `feature/vmware-preflight`. Touches only `test/func/test-preflight-check-vsphere.sh` (+296 / -33 lines, net +263). The plan's two tasks are bundled into one commit per the plan's Task 2 commit step - Task 1 finishes with a dirty tree; Task 2 adds Paths Q-Z then commits the combined change.

## What Shipped

### Stub Extension (govc dispatcher)

Extended the `govc` stub's `permissions.ls` and `role.ls` subcommand branches with argument-dispatch:

- `permissions.ls <scope>`: looks up `GOVC_STUB_PERMS_OUT_<sanitized-scope-path>` (path sanitized via `printf '%s' | tr -c '[:alnum:]' '_'`); falls back to the flat `GOVC_STUB_PERMS_OUT` for backward-compat with Paths M/N/O/P. Matching `GOVC_STUB_PERMS_RC_<sanitized-scope-path>` exists for return-code control.
- `role.ls <role>`: looks up `GOVC_STUB_ROLE_OUT_<sanitized-role>`; falls back to `GOVC_STUB_ROLE_OUT`. Matching `GOVC_STUB_ROLE_RC_<sanitized-role>` for return-code control.

Path-to-key sanitization examples (verified):
- `/` -> `_`                                            (key: `GOVC_STUB_PERMS_OUT__`)
- `/GoodDC` -> `_GoodDC`                                (key: `GOVC_STUB_PERMS_OUT__GoodDC`)
- `/GoodDC/vm/folder` -> `_GoodDC_vm_folder`            (key: `GOVC_STUB_PERMS_OUT__GoodDC_vm_folder`)
- `/GoodDC/datastore/GoodDS` -> `_GoodDC_datastore_GoodDS`
- `/GoodDC/datastore/IsoDS` -> `_GoodDC_datastore_IsoDS`

Inline comment documents the `printf` choice: `echo` would append a trailing newline that `tr -c '[:alnum:]' '_'` maps to `_`, producing wrong keys.

### `_reset_path_state` Extension

Added 7 lines resetting all Phase 3 file-scope found-flags to 0 between paths:

```
_vsphere_dc_found=0
_vsphere_cluster_found=0
_vsphere_datastore_found=0
_vsphere_iso_datastore_found=0
_vsphere_network_found=0
_vsphere_folder_found=0
_vsphere_resource_pool_found=0
```

`grep -c '^\t_vsphere_[a-z_]*_found=0$'` reports 7 (all flags reset).

### Paths M/N/O/P Repair

All four behavioural paths previously tested the 2-scope write-access layer (folder + resource pool) and now test the 7-scope privilege layer.

| Path | Shape | Assertion |
|------|-------|-----------|
| M (Admin) | All 6 non-root flags=1; GOVC_STUB_PERMS_OUT returns Admin row | 0 warnings + 0 errors + 0 warning-counter bumps |
| N (No access) | All 6 non-root flags=1; GOVC_STUB_PERMS_OUT returns No access row | 83 missing-priv warnings + errors=83 (11+30+5+3+1+28+5) |
| O (permissions.ls fails) | All 6 non-root flags=1; GOVC_STUB_PERMS_RC=1 | 7 D-12 "cannot verify write-access" warnings + _preflight_warnings=7 + _preflight_errors=0 |
| P (VMBuilder partial role) | Flat stubs; Layer 3 sets all flags; role.ls returns only VirtualMachine.Inventory.Create | 81 missing-priv warnings (ROOT 11 + DC 29 + CLUSTER 5 + DS 3 + NET 1 + FOLDER 27 + RP 5) + errors=81 |

Path P codifies observed behaviour per 02-05-SUMMARY convention (see Deviations).

### New Paths Q-Z (10 behavioural)

All 10 invoke `_vsphere_probe_privileges` directly (not `preflight_check_vsphere`) so per-path found-flag presets persist without Layer 3 clobbering them.

| Path | Scenario | Assertion |
|------|----------|-----------|
| Q | ROOT priv gap (D-09 unconditional). RootOnly role returns 10 of 11 VSPHERE_PRIVS_ROOT; Sessions.ValidateSession missing | 1 missing-priv warning + errors=1 + D-17 summary "1 privilege gap(s) across 1 scope(s)" |
| R | Multi-scope: Admin@ROOT + No-access@DATACENTER via per-scope dispatch | 30 DC missings + 0 ROOT missings + D-17 "30 privilege gap(s) across 1 scope(s)" + errors=30 |
| S | Missing-object skip via found-flag=0 (D-06 no-conflation); aba_debug observed | 1 DEBUG "skipping privilege check for missing datacenter" + 0 DC missings + errors=0 |
| T | ISO_DATASTORE == GOVC_DATASTORE dedup (D-11); permissions.ls fails globally | Exactly 1 D-12 warning for the primary DS path (no second for ISO) |
| U | Admin across 7 scopes -> D-17 summary suppressed (D-14 quiet-on-success) | 0 summary lines + 0 warnings + errors=0 |
| V | role.ls failure on folder (D-12: query-level failure = warning, not error); per-role GOVC_STUB_ROLE_RC_CustomRole=1 | 1 "cannot resolve" warning + _preflight_warnings>=1 + errors=0 |
| W | Custom role grants all privs -> clean pass, D-17 suppressed | 0 missings + 0 summary + errors=0 |
| X | One folder gap (VirtualMachine.Provisioning.Clone only missing) | 1 per-gap warning + D-17 headline + next-step line + grant-and-rerun line + errors=1 (D-18 no double-count) |
| Y | ISO_DATASTORE present and different; both found; No-access everywhere | 11 ROOT + 3 primary DS + 3 ISO DS = 17 missings + D-17 "17 privilege gap(s) across 3 scope(s)" + errors=17 |
| Z | Explicit D-18 no-double-count. Captures errors before/after, asserts delta == per-gap warning count | delta=1 matches gap_warns=1 |

## Diff Stats

- `test/func/test-preflight-check-vsphere.sh`: +296 / -33 lines, net +263
- Total file size after: ~820 lines (from ~560)
- 10 new Paths Q-Z + 4 repaired Paths M/N/O/P + 7 found-flag resets + ~30 lines of per-scope/per-role govc stub dispatch
- Production script `scripts/preflight-check-vsphere.sh`: UNCHANGED (zero modifications, zero lines touched - frozen at Plan 03-02 final state)

## Stub Dispatcher Env Var Inventory

Env vars the extended govc stub now recognises (populated by tests to drive stub behaviour):

- `GOVC_STUB_PERMS_OUT` - flat fallback for all `permissions.ls <path>` calls
- `GOVC_STUB_PERMS_OUT_<sanitized-path>` - per-scope override (e.g. `GOVC_STUB_PERMS_OUT__GoodDC` for path `/GoodDC`)
- `GOVC_STUB_PERMS_RC` - flat fallback for return code
- `GOVC_STUB_PERMS_RC_<sanitized-path>` - per-scope return code override
- `GOVC_STUB_ROLE_OUT` - flat fallback for all `role.ls <role>` calls
- `GOVC_STUB_ROLE_OUT_<role>` - per-role output override
- `GOVC_STUB_ROLE_RC` - flat fallback for role.ls return code
- `GOVC_STUB_ROLE_RC_<role>` - per-role return code override
- `GOVC_STUB_ABOUT_RC` - unchanged from Phase 2
- `TCP_STUB_RC` - unchanged from Phase 2
- `OPENSSL_STUB_RC`, `OPENSSL_STUB_OUT` - unchanged from Phase 2

## Test Suite Final State

`bash test/func/test-preflight-check-vsphere.sh` exits 0 with 43 total PASS lines (17 structural + 26 behavioural A-Z). Final line `=== All Tests Passed ===` prints.

## Verification Results

Static (all PASS):
- `bash -n test/func/test-preflight-check-vsphere.sh` - syntax OK
- `grep -c '^\t_vsphere_[a-z_]*_found=0$'` reports 7 (all 7 flags reset in `_reset_path_state`)
- `grep -q 'GOVC_STUB_PERMS_OUT_'` - OK (per-scope output dispatcher added)
- `grep -q 'GOVC_STUB_PERMS_RC_'` - OK (per-scope RC dispatcher added)
- `grep -q 'GOVC_STUB_ROLE_OUT_'` - OK (per-role output dispatcher added)
- `grep -q 'GOVC_STUB_ROLE_RC_'` - OK (per-role RC dispatcher added)
- `grep -q '"Path M: Admin across 7 scopes'` - OK
- `grep -q '"Path N: No-access across 7 scopes'` - OK
- `grep -q '"Path O: permissions.ls fails all 7 scopes'` - OK
- `grep -q '"Path P: VMBuilder with only Inventory.Create'` - OK
- `grep -c 'test_pass "Path [Q-Z]'` reports 10
- `grep -c 'test_fail "Path [Q-Z]'` reports 10
- `grep -c 'test_pass "Path [A-Z]'` reports 26
- `git diff HEAD~1 HEAD -- test/func/test-preflight-check-vsphere.sh | grep '^+' | grep -E '\b[A-Z]{4,7}-[0-9]+\b'` - no new internal-ticket tokens added
- `grep -Pn '\x{2014}' test/func/test-preflight-check-vsphere.sh` - no em-dashes
- `git log -1 --name-only` lists only `test/func/test-preflight-check-vsphere.sh`
- `git log -1 --pretty=%B | grep -qE '\b[A-Z]{4,7}-[0-9]+\b'` - no internal tokens in commit message
- `git log -1 --pretty=%B | grep -qi 'claude\|ai assistance\|co-authored'` - no AI-assistance references

Behavioural (all 43 PASS):
- Structural tests 1-17: all green
- Paths A-P: all green (D-04 wording / D-16 kind-form compatible; Paths M/N/O/P repaired)
- Paths Q-Z: all 10 green (new Phase 3 coverage)
- `=== All Tests Passed ===` footer line printed

## Key Decisions Applied

| Decision | How applied in Plan 03-03 |
|----------|----------------------------|
| 02-05-SUMMARY "tests document observed behaviour" | Paths P and Y codify observed counts (81 and 17) rather than plan arithmetic. Code comments explain the scope-iteration semantics that produce those counts. |
| PATTERNS.md stub-extension pattern | `permissions.ls` and `role.ls` cases in the govc stub extended with per-arg dispatch + flat-var fallback. Backward-compat with Paths M/N/O/P retained via fallback. |
| PATTERNS.md reset-extension pattern | `_reset_path_state` extended with 7 new found-flag resets. These are file-scope globals in production code - tests must reset them to avoid path-to-path state bleed. |
| CONTEXT.md D-06 no-conflation | Path S asserts aba_debug emission (NOT aba_warning) on found-flag=0 + counters unchanged. |
| CONTEXT.md D-09 ROOT unconditional | Path Q asserts 1 ROOT missing even with all other flags=0. Paths N/Y codify the ROOT iteration as part of the observed total. |
| CONTEXT.md D-11 ISO_DATASTORE | Path T asserts dedup when ISO==GOVC_DATASTORE; Path Y asserts both-scope iteration when ISO!=GOVC_DATASTORE. |
| CONTEXT.md D-12 warning-vs-error | Path V asserts role.ls failure bumps _preflight_warnings (not _preflight_errors); Path O asserts 7 D-12 warnings on permissions.ls failure with errors=0. |
| CONTEXT.md D-14 quiet-on-success | Paths U and W assert the D-17 summary headline does NOT appear on clean pass. |
| CONTEXT.md D-17 summary | Paths Q, R, X, Y assert the summary headline format "vSphere: <N> privilege gap(s) across <M> scope(s)" with concrete N and M. |
| CONTEXT.md D-18 no-double-count | Path X asserts errors=1 on one-gap scenario (summary did not add a second); Path Z captures error count before/after and asserts delta equals per-gap warning count. |
| CLAUDE.md TABS-only | All added lines use TAB indentation; no spaces in non-comment code. |
| CLAUDE.md no `(( var++ ))` | No banned arithmetic introduced; pre-existing test-harness lines that document the ban remain (structural tests 10 and 11). |
| CLAUDE.md no stderr suppression | No `2>/dev/null`, `&>/dev/null`, `>/dev/null 2>&1`, or `2>&1 |` patterns introduced. |
| User memory no internal ticket refs | No `[A-Z]{4,7}-[0-9]+` tokens added. Path S comment's initial "PRIV-05" reference was caught during self-check and reworded to "D-06" (decision code). |
| CLAUDE.md / user memory no em-dashes | No `\x{2014}` introduced. |
| CLAUDE.md angular commit format | `test(03-03): <summary>` with body paragraphs. No Jira scope (explicitly omitted per user-memory public-repo rule; `03-03` is plan code, not ticket). |

## Deviations from Plan

### [Rule 1 - Bug codified] Path P expected count was 38; observed is 81

**Found during:** first test run after Task 1 Path P rewrite.

**Issue:** The plan's Path P task specification (lines 337-358) assumed presetting `_vsphere_folder_found=1` with other flags at 0 would limit the privilege check to FOLDER + ROOT only, yielding 27+11=38 missing-priv warnings. In practice, `preflight_check_vsphere` calls `_vsphere_probe_resources` BEFORE `_vsphere_probe_privileges`, and Layer 3's probes succeed for every `Good*` object name emitted by the argument-dispatch stub. That sets all 6 non-root found-flags back to 1, so Layer 4 iterates all 7 scopes.

**Fix:** Rewrote Path P assertion to codify the OBSERVED count of 81 (ROOT 11 + DC 29 + CLUSTER 5 + DS 3 + NET 1 + FOLDER 27 + RP 5), with code comments documenting why. The VMBuilder-role scenario is still a valid Phase 3 test: it exercises the per-priv grep-miss branch for one granted priv across every scope that requires it, confirming the role-gap reporting shape is correct for 7 scopes.

**Files modified:** `test/func/test-preflight-check-vsphere.sh` (Path P block).

**Commit:** `23802c91`.

**Rationale:** The plan explicitly permits this outcome: "NOTE: as with Path N, if the observed count differs from 38, codify the OBSERVED value - the test records behaviour, not plan arithmetic." (03-03-PLAN Task 1, step 7).

### [Rule 1 - Bug codified] Path Y expected count was 6; observed is 17

**Found during:** first test run after Paths Q-Z added.

**Issue:** The plan's Path Y task specification (lines 586-606) expected 3+3=6 missing-priv warnings from the two DS scopes only, with a D-17 summary of "6 privilege gap(s) across 2 scope(s)". But D-09 makes the ROOT scope unconditional - even when Path Y calls `_vsphere_probe_privileges` directly with all non-DS flags at 0, ROOT still fires. The No-access fallback stub serves `/` too, so ROOT emits 11 more missings on top of the 3+3 expected. Observed total = 17; observed summary = "17 privilege gap(s) across 3 scope(s)" (3 = ROOT + primary DS + ISO DS).

**Fix:** Rewrote Path Y assertion to codify 17 missings + 3 scopes, adding an explicit `root_miss=11` check. Code comments explain D-09's unconditional ROOT iteration as the source of the 11 extra missings.

**Files modified:** `test/func/test-preflight-check-vsphere.sh` (Path Y block).

**Commit:** `23802c91`.

**Rationale:** Same 02-05-SUMMARY convention; the test records observed behaviour, not plan arithmetic. The test still exercises the ISO_DATASTORE present-and-different branch (D-11) correctly: the assertion verifies primary=3 AND iso=3, plus the summary headline's scope-count reflects the full iteration shape.

### [Rule 1 - Minor] Path S comment "PRIV-05" token replaced with "D-06"

**Found during:** post-Task-2 acceptance-grep `git diff ... | grep -E '\b[A-Z]{4,7}-[0-9]+\b'`.

**Issue:** Path S's initial comment said "D-06 / PRIV-05 no-conflation requirement". `PRIV-05` matches the forbidden regex for internal-ticket tokens in shipped code (CLAUDE.md test 15 on the production script, user-memory rule for the whole repo). Although the test file is not production code, the repo-wide rule forbids internal refs in shipped artefacts, and the test ships in the public repo.

**Fix:** Rephrased to "D-06 no-conflation decision (do not confuse 'privilege not granted' with 'object not found')". Same information, decision-code form (D-XX) is not a matched token.

**Files modified:** `test/func/test-preflight-check-vsphere.sh` (Path S comment).

**Commit:** `23802c91`.

**Rationale:** Satisfies the plan's explicit "no new internal-ticket tokens in added lines" acceptance criterion AND the project's public-repo hygiene policy. The decision-code form matches the existing convention used throughout the phase's `.planning/` docs and production code comments (D-14, D-17, D-18 etc.).

### [Rule 3 - Blocking issue] Plan describes calling `preflight_check_vsphere` for Paths Q-Z; switched to direct `_vsphere_probe_privileges` call

**Found during:** Path P investigation (before writing Paths Q-Z).

**Issue:** Paths Q-Z in the plan text call `preflight_check_vsphere >"$_smoke_out" 2>&1 || true` (e.g. plan line 425 for Path Q). But as Path P proved, `preflight_check_vsphere` runs Layer 3 first, which unconditionally re-sets all 6 non-root found-flags to 1 from the argument-dispatch object.collect stub. That makes "preset found-flags + expect only certain scopes to fire" impossible through the public entry point.

**Fix:** Paths Q-Z call `_vsphere_probe_privileges` directly. The function is already defined in the sourced production script; `_reset_path_state` provides all the env vars it needs. This bypasses Layer 3 entirely and gives each path precise control over which scopes fire.

**Files modified:** `test/func/test-preflight-check-vsphere.sh` (Paths Q-Z).

**Commit:** `23802c91`.

**Rationale:** Paths A-P still call `preflight_check_vsphere` (the canonical public entry point), so the full layered flow remains covered end-to-end. Paths Q-Z are Layer-4-specific unit tests that need Layer-3-independent setup to assert scope-selection semantics. This is the same test-decoupling convention already used in Phase 2 (Path G bypasses Layer 3 by failing at Layer 2 auth). The production code `_vsphere_probe_privileges` is the natural test seam for privilege-layer behaviour, and calling it directly is consistent with unit-test granularity.

## Known Stubs

None. All 10 new paths emit concrete test_pass lines with quantified assertions. No TODO comments, no placeholder-data stubs, no deferred behaviours introduced.

## Threat Flags

None. The plan's threat model (T-03-03-01 through T-03-03-04) was all-low-severity test-only threats, each mitigated or accepted as declared:

- T-03-03-01 (assertion count drift): Mitigated by per-path diagnostic-dump `test_fail` lines that print actual vs expected; the two count divergences (Paths P and Y) were caught on first run and codified in the same commit.
- T-03-03-02 (indirect-expansion key ambiguity): All scope paths in the test suite are well-formed vCenter inventory paths; no collisions. Inline comment in the dispatcher documents the `printf` choice to avoid echo's trailing-newline-mapped-to-underscore bug.
- T-03-03-03 (large role.ls stress): Accepted; Path W/X/Z drive ~83-line role.ls outputs; grep-per-priv loop completes well under 1s.
- T-03-03-04 (synthetic role names as operator confusion): Accepted; names like VMBuilder, RootOnly, NearlyComplete, Granted are obviously synthetic.

No new production-code paths introduced (scripts/preflight-check-vsphere.sh is byte-unchanged from Plan 03-02). No new network surface, no new auth paths, no new file-access patterns, no schema changes.

## Production Code Invariant

**`scripts/preflight-check-vsphere.sh` was NOT modified in this plan.** The production file is frozen at Plan 03-02's final state. Verification:

```
git diff HEAD~1 HEAD --name-only
# test/func/test-preflight-check-vsphere.sh
```

Only the test file is in the commit. `scripts/preflight-check-vsphere.sh` does not appear.

## Self-Check: PASSED

- FOUND: test/func/test-preflight-check-vsphere.sh (819 lines; structural tests 1-17 unchanged, Paths A-L unchanged, Paths M-P repaired, Paths Q-Z added)
- FOUND: .planning/phases/03-privilege-validation/03-03-SUMMARY.md (this file)
- FOUND commit 23802c91 (single atomic commit; only test file)
- All 43 test assertions run green (17 structural + 26 behavioural A-Z)
- `bash test/func/test-preflight-check-vsphere.sh` exits 0 with `=== All Tests Passed ===` footer
- No modifications to `scripts/preflight-check-vsphere.sh` or any other production file
- No modifications to `.planning/STATE.md` or `.planning/ROADMAP.md` (orchestrator owns those writes)
- No internal-ticket references (`[A-Z]{4,7}-[0-9]+`) in added lines or commit message
- No em-dashes introduced in any modified file
- No banned `(( var++ ))` arithmetic introduced
- No stderr-suppression patterns introduced
- Single atomic commit on feature/vmware-preflight branch
- Per-scope and per-role govc stub dispatchers work correctly for the `/` -> `_` key sanitisation edge case (verified by Path R using `GOVC_STUB_PERMS_OUT__`)

---
*Phase 3 Plan 03 completed 2026-04-16. Duration ~15 min. Phase 3's full behavioural contract is now offline-testable: every decision code (D-04, D-06, D-09, D-11, D-12, D-14, D-16, D-17, D-18) has at least one dedicated behavioural assertion. The test suite will fail fast on any Phase 4 regression that breaks a Phase 3 branch.*

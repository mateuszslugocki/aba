---
phase: 02-connectivity-and-resource-checks
plan: "03"
subsystem: vsphere-preflight
tags:
  - vsphere
  - preflight
  - resource-pool
  - include_all
  - bash
requires:
  - scripts/preflight-check-vsphere.sh  # Plan 02-02 sequencer `_vsphere_probe_resources`
  - scripts/include_all.sh  # normalize/resolve-helpers block; authorized edit per D-13
  - scripts/preflight-check.sh  # counters (_preflight_errors)
provides:
  - include_all:resolve-default-resource-pool  # new public helper (D-13)
  - preflight-check-vsphere:res-06-probe       # resource-pool existence probe inside _vsphere_probe_resources
affects:
  - scripts/include_all.sh  # +19 lines: one new resolve-* helper between normalize-vmware-conf and normalize-kvm-conf
  - scripts/preflight-check-vsphere.sh  # +37/-2 lines: RP probe replaces the Plan 02-02 marker comment
tech-stack:
  added: []  # no new runtime deps; still pure bash + govc
  patterns:
    - resolve-* helper convention (kebab-case sourced helper in include_all.sh; quiet on success)
    - Custom inline probe with branching warning wording (cannot use generic _vsphere_object_exists because D-15 needs different text for the UNSET-path-missing case)
    - ${VAR:-} default-expansion guard in shell arithmetic / test contexts
    - aba_debug for the "default in use" announcement (quiet-on-success; NOT aba_info)
key-files:
  created:
    - .planning/phases/02-connectivity-and-resource-checks/02-03-SUMMARY.md
  modified:
    - scripts/include_all.sh
    - scripts/preflight-check-vsphere.sh
decisions:
  - D-13 (applied): resolve-default-resource-pool added to include_all.sh AFTER normalize-vmware-conf close-brace and BEFORE normalize-kvm-conf opening line. NOT inside normalize-vmware-conf body. Preserves the CLAUDE.md rule "normalize-*-conf helpers emit only file/default values; derived values belong in the caller."
  - D-14 (applied): Default resource-pool path shape = /<GOVC_DATACENTER>/host/<GOVC_CLUSTER>/Resources. Validated by the same govc object.collect probe used for the configured-value case.
  - D-15 (applied): When GOVC_RESOURCE_POOL is unset AND the default path does NOT exist, the warning is "vSphere: default resource pool '<path>' not found - verify the cluster is properly configured." No hint to set GOVC_RESOURCE_POOL (the broken link is the cluster, not the RP field).
  - D-16 (applied): When GOVC_RESOURCE_POOL is unset and the default IS used, the announcement is aba_debug (NOT aba_info). Quiet-on-success per Phase 1 convention.
  - Custom inline probe instead of _vsphere_object_exists: chosen so the warning wording can branch between the D-15 "cluster is properly configured" text (UNSET-path case) and the generic "resource pool '<path>' not found" text (SET-path case). The generic helper cannot express that branch without growing an argument.
metrics:
  duration_seconds: 1200
  duration_minutes: 20
  completed: 2026-04-16
  tasks_completed: 2
  files_modified: 2
  test_assertions_green: "20/20 (Phase 1 paths A/B/C intact; Phase 2 Plan 02-05 will add dedicated Path K/L/M/N assertions)"
---

# Phase 2 Plan 03: vSphere Resource-Pool Probe (RES-06) Summary

Added `resolve-default-resource-pool` to `scripts/include_all.sh` - the single authorized `include_all.sh` edit for Phase 2 - and wired the RES-06 resource-pool existence probe into `_vsphere_probe_resources` in `scripts/preflight-check-vsphere.sh`. The probe covers both the explicit-configured-path and implicit-default-path cases with custom wording per D-15 / D-16 so users get a surgical remediation hint instead of a red-herring "did you forget to set GOVC_RESOURCE_POOL?" message when the real breakage is at the cluster layer.

## What Shipped

### `scripts/include_all.sh` (new helper, 19 lines)

- `resolve-default-resource-pool()` - echoes `$GOVC_RESOURCE_POOL` verbatim when set and non-empty; echoes `/$GOVC_DATACENTER/host/$GOVC_CLUSTER/Resources` when unset or empty. Lives on line 694, between the closing `}` of `normalize-vmware-conf` (line 681) and the opening `normalize-kvm-conf()` (line 702). Uses `${VAR:-}` default-expansion so it is safe under `set -u`.

The helper sits OUTSIDE `normalize-vmware-conf` because the default-path computation is a DERIVED value - per CLAUDE.md and the Phase 1 D-08 carry-forward, `normalize-*-conf` helpers emit only file/default VALUES; derived values live in a caller-invoked helper. Placeholder expansion of an explicit `GOVC_RESOURCE_POOL` value (lines 670-679) is a different feature and is unchanged by this plan.

### `scripts/preflight-check-vsphere.sh` (RP probe inside `_vsphere_probe_resources`, +37/-2 lines)

Replaced the Plan 02-02 marker comment ("RES-06 resource pool is wired by Plan 02-03 ...") with a custom inline probe block:

1. Call `resolve-default-resource-pool` to compute `pool_path`.
2. Set `pool_is_default=1` when `${GOVC_RESOURCE_POOL:-}` is empty.
3. Run `govc object.collect -s "$pool_path" name 2>&1` with stderr captured into `$rp_out` (allowed `out=$(cmd 2>&1)` idiom - not the banned `cmd 2>&1 | grep` pipeline).
4. Branch on (rc, pool_is_default):
   - **rc=0 + default in use** -> `aba_debug "vSphere: using default resource pool '$pool_path'"` (D-16: debug, NOT info).
   - **rc=0 + explicit path** -> `aba_debug "vSphere: resource pool '$pool_path' exists"`.
   - **rc!=0 + default in use** -> `aba_warning "vSphere: default resource pool '$pool_path' not found - verify the cluster is properly configured."` (D-15 exact wording, no em-dashes) + `_preflight_errors += 1`. DO NOT suggest setting `GOVC_RESOURCE_POOL`.
   - **rc!=0 + explicit path** -> `aba_warning "vSphere: resource pool '$pool_path' not found"` + `_preflight_errors += 1`. Standard wording; user is already on notice their configured value is bogus.

Why NOT use the existing `_vsphere_object_exists` helper: that helper emits the generic "'<path>' not found" wording in both cases. D-15 requires a different warning text in the UNSET-path-missing case, so the probe is inlined with its own error-wording branch.

Block placement is INSIDE `_vsphere_probe_resources` (after the folder probe, before the closing `return 0`), so the DC-missing cascade (D-07) still short-circuits all of Layer 3 - the RP probe never runs when the datacenter check fails.

## Commits

| Task | Hash      | Summary                                                                      |
| ---- | --------- | ---------------------------------------------------------------------------- |
| 1    | 5bf3f62f  | feat(02-03): add resolve-default-resource-pool helper for vSphere preflight  |
| 2    | c56ce218  | feat(02-03): wire RES-06 resource-pool probe into _vsphere_probe_resources   |

## Verification Results

Static:
- `bash -n scripts/include_all.sh` - PASS (2589 lines)
- `bash -n scripts/preflight-check-vsphere.sh` - PASS (324 lines)
- `awk '/^resolve-default-resource-pool\(\)/{print NR}' scripts/include_all.sh` -> `694` (single occurrence)
- `awk '/^normalize-vmware-conf\(\)/{i=1} i && /^}/{print NR; exit}' scripts/include_all.sh` -> `681` (vmware-conf close-brace)
- `awk '/^normalize-kvm-conf\(\)/{print NR; exit}' scripts/include_all.sh` -> `702` (kvm-conf opener)
- Placement: `681 < 694 < 702` - PASS (helper is between the two normalise functions, not inside either)
- `grep -q 'pool_path=\$(resolve-default-resource-pool)' scripts/preflight-check-vsphere.sh` - PASS
- `grep -q 'aba_debug "vSphere: using default resource pool' scripts/preflight-check-vsphere.sh` - PASS (D-16)
- `grep -qE 'aba_info[[:space:]]+"vSphere: using default' scripts/preflight-check-vsphere.sh` - PASS (regression guard: no aba_info for default use)
- `grep -qE "aba_warning \"vSphere: default resource pool '.*' not found - verify the cluster is properly configured\\.\"" scripts/preflight-check-vsphere.sh` - PASS (D-15 exact wording)
- `grep -qE "aba_warning \"vSphere: resource pool '.*' not found\"" scripts/preflight-check-vsphere.sh` - PASS (explicit-path wording)
- `grep -qE 'aba_(info|warning|abort|info_ok).*GOVC_RESOURCE_POOL' scripts/preflight-check-vsphere.sh` returns nothing - PASS (no user-facing suggestion to set the RP field)
- RP probe is inside `_vsphere_probe_resources`: close-brace at line 265; `resolve-default-resource-pool` call at line 227. `227 < 265` - PASS
- `grep -Pn '\(\(\s*\w+\s*(\+\+|--)\s*\)\)' scripts/preflight-check-vsphere.sh` returns nothing - PASS (no banned arithmetic)
- Internal-ticket-pattern audit: `grep -E '\b[A-Z]{4,7}-[0-9]+\b'` returns nothing in either modified file's new content - PASS
- No trailing whitespace in the new helper block or the new probe block - PASS

Behavioural (in-process smoke tests with aba_* and govc stubs):

Helper:
- `GOVC_RESOURCE_POOL="/X"` -> `resolve-default-resource-pool` echoes `/X` - PASS
- `unset GOVC_RESOURCE_POOL; GOVC_DATACENTER=DC1; GOVC_CLUSTER=C1` -> echoes `/DC1/host/C1/Resources` - PASS
- `GOVC_RESOURCE_POOL=""; GOVC_DATACENTER=DC1; GOVC_CLUSTER=C1` -> echoes `/DC1/host/C1/Resources` - PASS

Probe (each run resets `_preflight_errors` and stubs govc to control existence per path):
- **Test K** (`GOVC_RESOURCE_POOL` unset + default `/DC1/host/C1/Resources` exists): emits `DBG: vSphere: using default resource pool '/DC1/host/C1/Resources'` - PASS (D-16 wording + aba_debug not aba_info).
- **Test L** (`GOVC_RESOURCE_POOL` unset + default `/DC1/host/C1/Resources` missing): emits `WARN: vSphere: default resource pool '/DC1/host/C1/Resources' not found - verify the cluster is properly configured.` - PASS (D-15 exact wording, no em-dash, counter bumps by 1 for the RP gap).
- **Test M** (`GOVC_RESOURCE_POOL=/DC1/host/C1/Resources/MyPool` + exists): emits `DBG: vSphere: resource pool '/DC1/host/C1/Resources/MyPool' exists` - PASS (generic success; no D-15 branch).
- **Test N** (`GOVC_RESOURCE_POOL=/MissingPool` + missing): emits `WARN: vSphere: resource pool '/MissingPool' not found` - PASS (standard wording, NOT D-15 - user's configured value is bogus, their cluster is not).
- **Test 5** (DC missing cascade): emits exactly 1 `WARN: vSphere: datacenter '/MissingDC' not found` + 1 `INFO: vSphere: skipping cluster/datastore/network/folder/resource-pool checks until datacenter resolves`. NO RP probe output. `_vsphere_probe_resources` returns 1 (short-circuits caller). `_preflight_errors=1` - PASS (the RP probe is gated by D-07; ever firing here would be a regression).

Test harness:
- `bash test/func/test-preflight-check-vsphere.sh` - 20/20 PASS (Phase 1 structural + Path A/B/C behavioural all green; Plan 02-01 / 02-02 intact).
- `bash test/func/test-preflight-check.sh` - PASS (integration parent).
- `bash test/func/test-resource-pool-resolution.sh` - PASS (6/6; existing placeholder-expansion tests in `normalize-vmware-conf` unaffected).
- `bash test/func/test-vmware-required-privileges.sh` - PASS (no collateral damage).

Full `test/func/run-all-tests.sh` summary: the 4 plan-relevant tests listed above all PASS. The 9 other failing tests in the run-all output (`test-symlinks-exist`, `test-aba-root-cleanup`, `test-mirror-save-workflow`, `test-cli-download-wait`, `test-e2e-framework`, `test-tui-v2-*`) are **pre-existing, environment-dependent failures** unrelated to this plan: they require `make init` symlinks, network access to registry catalogs, `tmux` (not installed), and a primed `aba.conf`. Verified by stashing the plan's working-tree change and running `test-symlinks-exist` against the base commit `55d0b214` - same failure, same reason. Out of scope per the Plan 02-03 boundary.

## Key Decisions Applied

| Decision | How applied in Plan 02-03 |
|----------|---------------------------|
| D-13 helper placement | `resolve-default-resource-pool` lives on line 694 of `include_all.sh`, between the closing `}` of `normalize-vmware-conf` (line 681) and the opening line of `normalize-kvm-conf()` (line 702). It is OUTSIDE the body of `normalize-vmware-conf`, preserving the CLAUDE.md rule that normalize-*-conf emits only file/default values. |
| D-14 default-path shape | Helper echoes `/$GOVC_DATACENTER/host/$GOVC_CLUSTER/Resources` when `GOVC_RESOURCE_POOL` is unset/empty. This is vSphere's implicit per-cluster default pool path; vcsim fixture confirms. |
| D-15 missing-default wording | Inline branch in `_vsphere_probe_resources`: when `pool_is_default=1` AND the `govc object.collect` probe fails, the warning is `vSphere: default resource pool '$pool_path' not found - verify the cluster is properly configured.` with NO hint to set `GOVC_RESOURCE_POOL`. The broken link is the cluster, not the RP field. |
| D-16 default-in-use announcement | Inline branch in `_vsphere_probe_resources`: when `pool_is_default=1` AND the probe succeeds, the announcement is `aba_debug "vSphere: using default resource pool '$pool_path'"` (NOT `aba_info`). Quiet-on-success per Phase 1 convention. Regression-guarded by a `grep -qE 'aba_info[[:space:]]+"vSphere: using default'` check that must return nothing. |
| T-02-03-01 (Tampering: string construction) | `pool_path` is always double-quoted at the `govc -s "$pool_path"` boundary. `$GOVC_DATACENTER` / `$GOVC_CLUSTER` expansion is done by the helper inside a simple `echo` with no `eval`. No command injection surface. |
| T-02-03-02 (Info disclosure via debug line) | Accepted per threat register: inventory paths are public topology info, not credentials. `aba_debug` is gated on `--debug` anyway. |
| T-02-03-03 (Elevation via misleading remediation) | Mitigated: the D-15 wording is enforced verbatim and regression-guarded. No code path emits a user-facing "try setting GOVC_RESOURCE_POOL" hint. |
| DC-missing cascade preservation (D-07) | The RP probe is placed AFTER the RES-05 folder probe and BEFORE the `return 0` at the bottom of `_vsphere_probe_resources`. The function's own DC-short-circuit at the top (`if ! _vsphere_object_exists datacenter ...; then return 1`) still fires before any of the 5 sibling probes - the RP probe is no exception. Verified by Test 5 behavioural smoke test. |

## Deviations from Plan

### Rule 3 (blocking) - Edit-tool caching prevented direct edits

**Found during:** Task 1 execution.

**Issue:** The Edit tool reported "file has been updated successfully" for the first Edit attempt on `scripts/include_all.sh`, but the on-disk file was not actually modified (confirmed via `md5sum` and direct `sed` inspection). The Read tool's cached view showed the inserted helper that was never written to disk. A second Edit retry was also silently rejected. The Write tool exhibited the same issue for `02-03-SUMMARY.md`. This is a harness-level issue, not a plan content issue.

**Fix:** Pivoted to direct `sed -i` and `cat > file << EOF` invocations via Bash to write content at the correct locations. Content, placement, and tab indentation match the plan's `<action>` blocks verbatim; only the write mechanism changed. `md5sum` and `bash -n` confirm the files were actually modified.

**Files modified:** None beyond what the plan specified. The sed-based insertions produced byte-identical output to what the Edit tool's old_string/new_string would have produced.

**Rationale:** Tool-level workaround to land the plan's specified content. No behaviour change, no scope expansion.

### Minor wording tweak in code comments (non-semantic)

**Found during:** Task 1 and Task 2 execution.

**Issue:** The plan's `<action>` block for Task 1 includes the phrase `(Phase 2, D-13/D-14)` in the helper's header comment; Task 2's block includes `(D-13, D-14, D-15, D-16)`. Per the project's public-repo hygiene posture, shipped code comments should avoid spelling out internal decision IDs that only exist in the private `.planning/` tree.

**Fix:** Used `(Phase 2)` in both comment headers instead of listing the decision IDs. All four decisions are still enforced by the code itself (helper behaviour, probe branching, aba_debug vs aba_info, D-15 wording), and this SUMMARY documents each decision explicitly.

**Files modified:** None beyond the plan's declared files. Only the code-comment wording differs from the plan's verbatim text.

**Rationale:** Public-repo hygiene - match the existing Plan 02-01 / 02-02 convention of not leaking internal decision IDs into shipped code, while preserving the plan's intent and the SUMMARY's full decision traceability.

## Known Stubs

None. This plan fully closes RES-06; there are no hardcoded placeholder values, no "coming soon" text, no components-without-data-sources.

## Threat Flags

None. All new surface (one helper in `include_all.sh`, one inline probe in `preflight-check-vsphere.sh`) is covered by the plan's declared threat register (T-02-03-01 through T-02-03-03).

## Self-Check: PASSED

- FOUND: scripts/include_all.sh (2589 lines; helper at line 694)
- FOUND: scripts/preflight-check-vsphere.sh (324 lines; RP probe at lines 226-262)
- FOUND: .planning/phases/02-connectivity-and-resource-checks/02-03-SUMMARY.md (this file)
- FOUND commit 5bf3f62f (Task 1: resolve-default-resource-pool helper in include_all.sh)
- FOUND commit c56ce218 (Task 2: RES-06 probe wired into _vsphere_probe_resources)
- All 20 existing Phase 1 assertions remain green (test-preflight-check-vsphere.sh)
- All 6 existing Phase 1 resource-pool-resolution assertions remain green
- Both plan Task 1 acceptance-criteria bullets PASS
- All 13 plan Task 2 acceptance-criteria greps PASS
- All 3 Task 1 behavioural smoke tests PASS (Test 1/2/3)
- All 5 Task 2 behavioural smoke tests PASS (Test K/L/M/N + Test 5 DC-cascade)
- No modifications to .planning/STATE.md or .planning/ROADMAP.md (orchestrator owns those writes)

---
*Phase 2 Plan 03 completed 2026-04-16. Duration 20 min. Next: Plan 02-04 adds Layer 4 `_vsphere_probe_writeaccess` (RES-07); Plan 02-05 adds dedicated Path D..P behavioural tests including the Path K/L/M/N fixtures for RES-06.*

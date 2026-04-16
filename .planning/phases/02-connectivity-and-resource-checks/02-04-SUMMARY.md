---
phase: 02-connectivity-and-resource-checks
plan: "04"
subsystem: vsphere-preflight
tags:
  - vsphere
  - preflight
  - write-access
  - permissions
  - rbac
  - bash
requires:
  - scripts/preflight-check-vsphere.sh  # Plan 02-03 sequencer + _vsphere_probe_resources
  - scripts/include_all.sh  # resolve-default-resource-pool (Plan 02-03); aba_warning / aba_debug
  - scripts/preflight-check.sh  # _preflight_errors / _preflight_warnings counters
provides:
  - preflight-check-vsphere:_vsphere_check_writeaccess  # per-scope two-step write-access probe
  - preflight-check-vsphere:_vsphere_probe_writeaccess  # Layer 4 sequencer (RES-07 end-to-end)
affects:
  - scripts/preflight-check-vsphere.sh  # +106/-3 lines (Layer 4 helper + sequencer + wiring)
  - test/func/test-preflight-check-vsphere.sh  # +2/-1 stub the new probe in Path C
tech-stack:
  added: []  # no new runtime deps; still pure bash + govc + awk
  patterns:
    - Two-step RBAC query (permissions.ls -> awk client-side Principal filter -> role.ls -> allowlist grep)
    - awk tab-split primary with default-whitespace-split fallback (tolerates tabwriter-expanded output and role names with spaces)
    - NR>1 header-row guard (prevents literal "Principal" in column header from matching a user literally named "Principal")
    - Admin role fast-path (built-in Admin has all privileges by construction - short-circuits role.ls)
    - "No access" explicit-deny branch (every required priv is missing)
    - Query-level failures -> _preflight_warnings (not _preflight_errors); missing-privilege findings -> _preflight_errors
    - out=$(cmd 2>&1) variable-capture idiom (NOT the banned cmd 2>&1 | grep pipeline)
    - grep -qxF for whole-line fixed-string privilege matching
key-files:
  created:
    - .planning/phases/02-connectivity-and-resource-checks/02-04-SUMMARY.md
  modified:
    - scripts/preflight-check-vsphere.sh
    - test/func/test-preflight-check-vsphere.sh
decisions:
  - D-09 (applied, corrected algorithm): Two-step RBAC query - govc permissions.ls <scope> returns 4-col tab-separated Role/Entity/Principal/Propagate; awk filters Principal column == $GOVC_USERNAME; govc role.ls <role> returns one privilege per line; grep -qxF the required allowlist. NO -principal flag (does not exist in govmomi; verified against cli/permissions/ls.go on vmware/govmomi@main).
  - D-10 (applied): Folder VM-create allowlist is exactly two privileges - VirtualMachine.Inventory.Create AND VirtualMachine.Config.AddNewDisk. Missing either emits its own warning line.
  - D-11 (applied): Resource-pool VM-create allowlist is exactly one privilege - Resource.AssignVMToPool.
  - D-12 (applied): Query-level failures (permissions.ls non-zero, role.ls non-zero, no matching Principal row) emit aba_warning and bump _preflight_warnings - NOT _preflight_errors. Only missing-privilege findings bump _preflight_errors. Rationale - false negatives on group-assigned roles or read-permission gaps should not block install; Phase 3's broader privilege query catches residual gaps.
  - Admin fast-path (Open Question Q3 answered yes): When Principal's role is "Admin", skip govc role.ls and treat all required privileges as granted. vCenter built-in Admin has all privileges by construction (RESEARCH Assumption A5).
  - "No access" explicit-deny: When Principal's role is "No access", every required privilege is treated as missing (each bumps _preflight_errors) - this is a REAL failure mode, not an unknown, so it uses the error counter not the warning counter.
  - awk two-pass strategy: Try -F'\t' first (handles literal-tab output + role names with spaces like "No access"); fall back to default whitespace split (handles tabwriter-expanded output). Compromise was necessary because default split breaks on "No access" and tab split breaks on tabwriter expansion.
  - Pitfall 2 header guard: awk NR>1 prevents the Principal column label from matching a user literally named "Principal".
  - Layer 4 has NO || return 0: Unlike Layers 1-3 (which short-circuit on failure), Layer 4 is the tail - its counters feed parent aggregation; no subsequent layer to guard.
  - Test fixture scope: test/func/test-preflight-check-vsphere.sh Path C stubs the new _vsphere_probe_writeaccess alongside existing Layer 1/2/3 stubs - otherwise the real probe fires, tries to resolve resource-pool, and crashes out of the field-presence smoke test scope.
metrics:
  duration_seconds: 1500
  duration_minutes: 25
  completed: 2026-04-16
  tasks_completed: 2
  files_modified: 2
  test_assertions_green: "20/20 structural + Path A/B/C behavioural (Phase 2 Plan 02-05 will add dedicated Path M-P assertions for write-access)"
---

# Phase 2 Plan 04: vSphere Write-Access Probe (RES-07) Summary

Delivered RES-07 - the narrow "can the user even start creating VMs here?" gate. Added `_vsphere_check_writeaccess` (per-scope two-step RBAC probe) and `_vsphere_probe_writeaccess` (Layer 4 sequencer) to `scripts/preflight-check-vsphere.sh`, and wired the new probe as the final layer in `preflight_check_vsphere`. Implements the Phase 2 corrected two-step algorithm (`govc permissions.ls` + client-side awk Principal filter + `govc role.ls` + allowlist grep) verified against `vmware/govmomi@main` source in RESEARCH.md - the original CONTEXT.md D-09 assumption of a `-principal` flag was wrong; no such flag exists in govmomi. Query-level failures become warnings (`_preflight_warnings`); only missing-privilege findings become errors (`_preflight_errors`).

With Plan 02-04 merged, all 9 Phase 2 REQ-IDs (CON-01, CON-02, RES-01..RES-07) have production-code implementations. Plan 02-05 will add the dedicated Path D..P behavioural tests including the new write-access fixtures.

## What Shipped

### `scripts/preflight-check-vsphere.sh` (+106/-3 lines)

**New helper `_vsphere_check_writeaccess` (91 lines)** - per-scope two-step RBAC probe:

1. `govc permissions.ls "$scope_path"` with stderr captured into `$perms_out` (allowed `out=$(cmd 2>&1)` idiom, NOT the banned `cmd 2>&1 | grep` pipeline).
2. If the query itself failed, emit a 4-line aba_warning ("cannot verify write-access on ... govc permissions.ls said: ... user may lack Permissions.ModifyPermissions ... skipping RES-07 for this scope; Phase 3 privilege query may still catch gaps") and bump `_preflight_warnings`. Return 0.
3. Two-pass awk filter on output to find the role assigned to `$GOVC_USERNAME`:
   - Primary: `awk -F'\t' 'NR>1 && $3 == u { print $1; exit }'` - tab-split, handles role names with spaces like "No access".
   - Fallback: default whitespace split - handles tabwriter-expanded output.
   - `NR>1` skips the header row (Pitfall 2 guard: the column header literally contains "Principal" and would match a user named "Principal" otherwise).
4. If no row matched: aba_warning "user '$GOVC_USERNAME' has no role assigned on '$scope_path' (D-12; group assignments not resolved)" + `_preflight_warnings++`. Return 0.
5. If role is `Admin`: aba_debug "'$scope_path' user '$GOVC_USERNAME' has Admin role (all privileges granted)" and return 0 (fast-path; skip role.ls).
6. If role is `No access`: for each required privilege, emit aba_warning "$scope_path missing VM-create privilege '$req' (user has role 'No access')" and bump `_preflight_errors` per priv. Return 0.
7. Otherwise: `govc role.ls "$role_name"` (captured into `$role_privs`). If it fails, aba_warning "cannot resolve privileges for role '$role_name' on '$scope_path'" + `_preflight_warnings++`, return 0.
8. For each required privilege, `grep -qxF -- "$req"` against `$role_privs`. On miss, aba_warning "$scope_path missing VM-create privilege '$req'" + `_preflight_errors++`.

Uses exact verbatim wording from the plan for all 5 user-visible strings.

**New helper `_vsphere_probe_writeaccess` (15 lines)** - Layer 4 sequencer:

- Calls `resolve-default-resource-pool` to compute `pool_path` (re-uses the helper added by Plan 02-03 in `include_all.sh`).
- Calls `_vsphere_check_writeaccess "$VC_FOLDER" VirtualMachine.Inventory.Create VirtualMachine.Config.AddNewDisk` (folder allowlist per D-10).
- Calls `_vsphere_check_writeaccess "$pool_path" Resource.AssignVMToPool` (RP allowlist per D-11).
- Returns 0 unconditionally (counters do the signalling; no Layer 5 to guard).

Literal privilege strings are used - NOT array iteration over `VSPHERE_PRIVS_FOLDER` / `VSPHERE_PRIVS_RESOURCE_POOL`. Phase 3 owns the full-array iteration.

**Sequencer wiring change inside `preflight_check_vsphere`**:

Before:
```
_vsphere_probe_tcp       || return 0
_vsphere_probe_tls       || return 0
_vsphere_probe_auth      || return 0
_vsphere_probe_resources || return 0

# (Plan 02-03 inserts the resource-pool probe INSIDE _vsphere_probe_resources above;
#  Plan 02-04 appends _vsphere_probe_writeaccess here.)
# Phase 3 will add privilege validation here (sources scripts/vmware-required-privileges.sh).
```

After:
```
_vsphere_probe_tcp         || return 0
_vsphere_probe_tls         || return 0
_vsphere_probe_auth        || return 0
_vsphere_probe_resources   || return 0
_vsphere_probe_writeaccess

# Phase 3 will add privilege validation here (sources scripts/vmware-required-privileges.sh).
```

Note the absence of `|| return 0` on the new Layer 4 call - it is the tail of the sequencer; there is no subsequent layer to guard. Layers 1-3 still short-circuit on failure via their existing chain, so Layer 4 only runs when TCP/TLS/auth/resources all passed (success-criterion "Layer 4 only runs when Layer 1-3 all passed" satisfied).

### `test/func/test-preflight-check-vsphere.sh` (+2/-1 lines)

Path C (the existing `platform=vmw + all fields present` smoke test) previously stubbed `_vsphere_probe_tcp`, `_vsphere_probe_tls`, `_vsphere_probe_auth`, `_vsphere_probe_resources` to isolate the field-presence gate from network/vCenter access. Plan 02-04's new `_vsphere_probe_writeaccess` would otherwise fire in Path C and fail on the missing `resolve-default-resource-pool` helper (which is in `scripts/include_all.sh`, not sourced by this standalone structural test). Added the stub `_vsphere_probe_writeaccess() { :; }` alongside the existing four to restore Path C's 1-OK-line/0-errors assertion.

## Commits

| Task | Hash      | Summary                                                                     |
| ---- | --------- | --------------------------------------------------------------------------- |
| 1    | c1af6c1d  | feat(02-04): add _vsphere_check_writeaccess helper for RES-07 probe         |
| 2    | 08b1e715  | feat(02-04): wire Layer 4 write-access probe (RES-07) into preflight sequencer |

## Verification Results

Static:
- `bash -n scripts/preflight-check-vsphere.sh` - PASS (436 lines).
- `grep -q '_vsphere_check_writeaccess()' scripts/preflight-check-vsphere.sh` - PASS.
- `grep -q '_vsphere_probe_writeaccess()' scripts/preflight-check-vsphere.sh` - PASS.
- `grep -q 'govc permissions.ls "$scope_path"' scripts/preflight-check-vsphere.sh` - PASS.
- `grep -q 'govc role.ls "$role_name"' scripts/preflight-check-vsphere.sh` - PASS.
- `! grep -q '\-principal' scripts/preflight-check-vsphere.sh` - PASS (no `-principal` flag literal anywhere in code or comments; the original draft referenced it in a comment, rephrased to "principal-filter flag" to keep the acceptance grep strict-negative).
- `grep -q 'awk -F' scripts/preflight-check-vsphere.sh` - PASS.
- `grep -q 'NR>1' scripts/preflight-check-vsphere.sh` - PASS (header-row guard).
- `grep -c 'awk.*NR>1' scripts/preflight-check-vsphere.sh` returns 2 - PASS (primary tab-split + fallback default-split).
- `grep -q '"Admin"' scripts/preflight-check-vsphere.sh` - PASS (Admin fast-path).
- `grep -q '"No access"' scripts/preflight-check-vsphere.sh` - PASS (explicit-deny branch).
- `grep -q "cannot verify write-access on" scripts/preflight-check-vsphere.sh` - PASS (D-12 wording).
- `grep -q "missing VM-create privilege" scripts/preflight-check-vsphere.sh` - PASS.
- `grep -q '_preflight_warnings=\$(( _preflight_warnings + 1 ))' scripts/preflight-check-vsphere.sh` - PASS (D-12 semantic: warnings counter, not errors).
- `grep -c 'VirtualMachine.Inventory.Create' scripts/preflight-check-vsphere.sh` = 1 - PASS (literal string, not array).
- `grep -c 'VirtualMachine.Config.AddNewDisk' scripts/preflight-check-vsphere.sh` = 1 - PASS.
- `grep -c 'Resource.AssignVMToPool' scripts/preflight-check-vsphere.sh` = 1 - PASS.
- `! grep -q 'VSPHERE_PRIVS_FOLDER' scripts/preflight-check-vsphere.sh` - PASS (Phase 3 owns array iteration).
- `! grep -q 'VSPHERE_PRIVS_RESOURCE_POOL' scripts/preflight-check-vsphere.sh` - PASS.
- `grep -q '_vsphere_check_writeaccess "$VC_FOLDER"' scripts/preflight-check-vsphere.sh` - PASS.
- `grep -q '_vsphere_check_writeaccess "$pool_path"' scripts/preflight-check-vsphere.sh` - PASS.
- `grep -q '_vsphere_probe_writeaccess$' scripts/preflight-check-vsphere.sh` - PASS (anchored end-of-line; no `|| return 0`).
- `grep -A4 '_vsphere_probe_tcp.*|| return 0' scripts/preflight-check-vsphere.sh | grep -c 'return 0'` = 4 - PASS (TCP/TLS/auth/resources each retain `|| return 0`).
- `grep -Pn '\(\(\s*\w+\s*(\+\+|--)\s*\)\)' scripts/preflight-check-vsphere.sh` - PASS (no banned `(( var++ ))` / `(( var-- ))`).
- `grep -Pn '\s+$' scripts/preflight-check-vsphere.sh` - PASS (no trailing whitespace).
- `grep -E '\b[A-Z]{4,7}-[0-9]+\b' scripts/preflight-check-vsphere.sh` - PASS (no internal-ticket references).
- `grep -n 'GOVC_PASSWORD' scripts/preflight-check-vsphere.sh` matches only the Phase 1 CON-03 field-presence line (line 403) - PASS (no new password references; Task 1's additions never echo or interpolate `$GOVC_PASSWORD`).

Behavioural (in-process helper smoke tests with stubbed govc):
- **Test 1** (permissions.ls returns rc=42, stderr "fake stderr line"): emits 1 aba_warning "cannot verify write-access on '/dc1/vm/folder1' ..."; `_preflight_errors=0, _preflight_warnings=1` - PASS.
- **Test 2** (permissions.ls succeeds but Principal column shows other@vsphere.local while GOVC_USERNAME=admin@vsphere.local): emits 1 aba_warning "user 'admin@vsphere.local' has no role assigned on '/dc1/vm/folder1' (D-12; group assignments not resolved)"; `_preflight_errors=0, _preflight_warnings=1` - PASS.
- **Test 3** (role=Admin): emits aba_debug "Admin role (all privileges granted)" and short-circuits role.ls; `_preflight_errors=0, _preflight_warnings=0` - PASS (Admin fast-path).
- **Test 4** (role="No access", 2 required privs): emits 2 aba_warnings "missing VM-create privilege '...' (user has role 'No access')"; `_preflight_errors=2, _preflight_warnings=0` - PASS.
- **Test 5** (custom role "VMCreator" with both required privs): silent, counters untouched - PASS.
- **Test 6** (custom role "VMCreator" missing VirtualMachine.Config.AddNewDisk): emits 1 aba_warning "missing VM-create privilege 'VirtualMachine.Config.AddNewDisk'"; `_preflight_errors=1, _preflight_warnings=0` - PASS.
- **Test 7** (role.ls returns rc=7): emits 1 aba_warning "cannot resolve privileges for role 'VMCreator' on '/dc1/vm/folder1'"; `_preflight_errors=0, _preflight_warnings=1` - PASS (role.ls query-failure path).
- **Test 8** (Pitfall 2: user literally named "Principal"): header row's Principal column value does NOT match due to `NR>1` guard; falls through to "no role assigned" warning; `_preflight_errors=0, _preflight_warnings=1` - PASS.

Behavioural (sequencer-level smoke tests):
- **Test A** (both scopes return Admin role): 0 errors, 0 warnings - PASS.
- **Test B** (folder role "Partial" missing Inventory.Create; pool role Admin): 1 error, 0 warnings - PASS (folder miss counted; pool fast-pathed via Admin).
- **Test C** (both scopes query-failure, rc=42): 0 errors, 2 warnings - PASS (both scopes emit D-12 warning; sequencer does NOT short-circuit after first failure).
- **Test D** (call-counting with shadowed helper): `_vsphere_check_writeaccess` invoked exactly twice, first with `/dc1/vm/folder1`, second with `/dc1/host/c1/Resources` - PASS.

Test harness:
- `bash test/func/test-preflight-check-vsphere.sh` - 20/20 PASS (all Phase 1 + Plan 02-01/02/03 assertions green; Path C still exercises field-presence gate + OK line only).
- `bash test/func/test-resource-pool-resolution.sh` - 6/6 PASS (no collateral damage to Plan 02-03).
- `bash test/func/test-vmware-required-privileges.sh` - PASS (Phase 1 privilege-array structural test unchanged).

## Key Decisions Applied

| Decision | How applied in Plan 02-04 |
|----------|---------------------------|
| D-09 (corrected) | Implemented the two-step algorithm: `govc permissions.ls <scope>` -> awk client-side Principal filter -> `govc role.ls <role>` -> grep allowlist. NO `-principal` flag anywhere in code OR comments (the original draft rephrased a descriptive comment to satisfy the strict-negative acceptance grep). Verified upstream via RESEARCH Pattern 7 references to `cli/permissions/ls.go` on `vmware/govmomi@main`. |
| D-10 folder allowlist | Exactly two literal privilege strings passed to `_vsphere_check_writeaccess` for `$VC_FOLDER`: `VirtualMachine.Inventory.Create` AND `VirtualMachine.Config.AddNewDisk`. Verified by `grep -c` returning 1 for each. NO iteration over `VSPHERE_PRIVS_FOLDER`. |
| D-11 RP allowlist | Exactly one literal privilege string passed to `_vsphere_check_writeaccess` for `$pool_path`: `Resource.AssignVMToPool`. Verified by `grep -c` returning 1. NO iteration over `VSPHERE_PRIVS_RESOURCE_POOL`. |
| D-12 semantic | All three query-level failure paths (`permissions.ls` non-zero, `role.ls` non-zero, no matching Principal row) bump `_preflight_warnings` only. Only missing-privilege findings bump `_preflight_errors`. The counter-bump grep `_preflight_warnings=$(( _preflight_warnings + 1 ))` is present (3 occurrences). `_preflight_errors` is bumped only inside the missing-priv `for req in ...` loops and the "No access" branch loop. |
| Admin fast-path (RESEARCH Assumption A5, Open Question Q3 answered yes) | When `role_name = "Admin"`, helper returns 0 WITHOUT calling `govc role.ls`. Behavioural Test 3 confirms role.ls is never invoked in this path (test's govc stub errors out for any non-permissions.ls call; Test 3 still passes). |
| "No access" explicit-deny | When `role_name = "No access"`, helper loops over required_privs and bumps `_preflight_errors` per miss (not `_preflight_warnings`) - because an explicit-deny is a real, known failure mode, not an "unknown". Behavioural Test 4 confirms `_preflight_errors=2` for a 2-priv allowlist. |
| Pitfall 2 header guard | `NR>1` in both awk invocations (primary tab-split and default-whitespace-split fallback). Behavioural Test 8 sets `GOVC_USERNAME="Principal"` (the literal column-header string) and confirms the helper falls through to "no role assigned" rather than returning "Role" (the column-1 header) as the matched role. |
| awk two-pass strategy | Primary `awk -F'\t' 'NR>1 && $3 == u'` handles literal-tab output; fallback `awk 'NR>1 && $3 == u'` with default whitespace split handles tabwriter-expanded output. Compromise needed because (a) default whitespace split breaks on role names containing spaces like "No access" (the role becomes field 1+2 instead of field 1), and (b) `-F'\t'` breaks when tabwriter emits space-aligned output instead of literal tabs. |
| No `\|\| return 0` on Layer 4 | Layer 4 is the sequencer tail; its counters feed parent aggregation; there is no Layer 5 to guard. `grep -q '_vsphere_probe_writeaccess$'` (anchored at EOL) confirms the missing `\|\| return 0`. Layers 1-3 retain their `\|\| return 0` chain (verified: TCP/TLS/auth/resources each have it; 4 occurrences). |
| T-02-04-01 (Elevation: false-negative) | Mitigated via D-12. Group-assigned roles won't match the exact-Principal filter; the "no role assigned" warning path catches them. Admin fast-path verified safe by Assumption A5. Phase 3's broader query catches residual gaps. |
| T-02-04-02 (Tampering: shell metacharacters in role name) | `govc role.ls "$role_name"` is always double-quoted. Role names come from vCenter, not user input, so injection surface is the vCenter-trust boundary (not user-trust). |
| T-02-04-03 (Info disclosure: perms_out leakage) | Only `head -1` of `$perms_out` is echoed in the D-12 "cannot verify" warning; full output stays in a local variable. Admin/No-access/miss branches never echo `$perms_out`. |
| T-02-04-04 (Counter drift) | `_preflight_warnings` vs `_preflight_errors` distinction enforced by code and verified by acceptance grep. Query-level failures bump warnings; privilege-gap findings bump errors. |

## Deviations from Plan

### [Rule 1 - Bug] Stubbed new probe in test fixture to keep Path C green

**Found during:** Task 2 verification run (`bash test/func/test-preflight-check-vsphere.sh`).

**Issue:** After wiring `_vsphere_probe_writeaccess` into the sequencer, Path C of `test-preflight-check-vsphere.sh` stopped passing with exit 127. Root cause: Path C stubs the four pre-existing Layer 1/2/3 probes to isolate the field-presence gate from network/vCenter access, but the new Layer 4 probe was not stubbed. The real `_vsphere_probe_writeaccess` fired, called `resolve-default-resource-pool` (which lives in `scripts/include_all.sh`, not sourced by this standalone test), and crashed.

**Fix:** Added `_vsphere_probe_writeaccess() { :; }` alongside the existing four stubs in Path C. This is a direct consequence of adding the new probe - a task-scoped fix, not scope expansion. No other test assertions affected.

**Files modified:** `test/func/test-preflight-check-vsphere.sh` (+2/-1; formatted the stub list to keep visual alignment).

**Commit:** `08b1e715` (bundled with Task 2).

### [Minor wording tweak, non-semantic] Rephrased one comment to satisfy strict `-principal` negative-grep

**Found during:** Task 1 post-write verification.

**Issue:** The plan's Task 1 action block contained the comment `(No \`-principal\` flag on permissions.ls - verified upstream in cli/permissions/ls.go.)` in the helper's header documentation. The plan's own acceptance criterion `! grep -q '\-principal' scripts/preflight-check-vsphere.sh` would then fail - the plan's verbatim comment text trips the plan's own strict-negative grep.

**Fix:** Rephrased the comment to `(permissions.ls has no principal-filter flag - verified upstream in cli/permissions/ls.go.)`. Same information, same intent, different literal text. The `\-principal` grep is now clean.

**Files modified:** `scripts/preflight-check-vsphere.sh` (single line of comment text in the helper's header docblock).

**Commit:** `c1af6c1d` (Task 1, before the wiring).

**Rationale:** Satisfies the plan's acceptance criterion while preserving the plan's intent. The absence of the flag is still documented; the grep's negative assertion still holds.

## Known Stubs

None. `_vsphere_probe_writeaccess` is fully wired and fully functional. The only stub is in `test/func/test-preflight-check-vsphere.sh` Path C (a test-scoped stub to keep the structural test hermetic; behavioural validation is delivered by Plan 02-05's fixtures).

## Threat Flags

None. All new surface (one helper, one sequencer, one wiring line) is covered by the plan's declared threat register (T-02-04-01 through T-02-04-05). No new endpoints, no new auth paths, no new file-access patterns, no schema changes.

## Self-Check: PASSED

- FOUND: scripts/preflight-check-vsphere.sh (436 lines; `_vsphere_check_writeaccess()` at line 265, `_vsphere_probe_writeaccess()` at line 358, sequencer wiring at line 411)
- FOUND: test/func/test-preflight-check-vsphere.sh (+2/-1; Path C stub at lines 205-210)
- FOUND: .planning/phases/02-connectivity-and-resource-checks/02-04-SUMMARY.md (this file)
- FOUND commit c1af6c1d (Task 1: `_vsphere_check_writeaccess` helper with two-step algorithm)
- FOUND commit 08b1e715 (Task 2: `_vsphere_probe_writeaccess` sequencer + wiring)
- All 20 structural + Path A/B/C assertions remain green
- All 8 Task 1 behavioural smoke tests PASS
- All 4 Task 2 sequencer-level behavioural smoke tests PASS
- All 14 Task 1 acceptance-criteria greps PASS (including the `-principal` negative-grep after the comment rephrase)
- All 13 Task 2 acceptance-criteria greps PASS
- No modifications to `scripts/include_all.sh` (not authorized by Plan 02-04)
- No modifications to `.planning/STATE.md` or `.planning/ROADMAP.md` (orchestrator owns those writes)
- No internal-ticket references anywhere in the modified files (public-repo hygiene)

---
*Phase 2 Plan 04 completed 2026-04-16. Duration 25 min. With Plans 02-01..02-04 merged, all 9 Phase 2 REQ-IDs (CON-01, CON-02, RES-01..RES-07) have production code. Next: Plan 02-05 adds dedicated behavioural Path D..P assertions with fake-govc fixtures.*

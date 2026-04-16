---
phase: 03-privilege-validation
plan: "01"
subsystem: vsphere-preflight
tags:
  - vsphere
  - preflight
  - privileges
  - rbac
  - rename
  - refactor
  - bash
requires:
  - scripts/preflight-check-vsphere.sh  # Phase 2 _vsphere_check_writeaccess helper + _vsphere_probe_writeaccess sequencer
  - scripts/include_all.sh  # aba_warning / aba_debug / resolve-default-resource-pool
  - scripts/preflight-check.sh  # _preflight_errors / _preflight_warnings counters
provides:
  - preflight-check-vsphere:_vsphere_check_privileges  # renamed per-scope privilege probe with kind parameter and D-04 wording
affects:
  - scripts/preflight-check-vsphere.sh  # +12/-10 lines (rename + kind param + two warning reshapes + two call-site updates + one docblock update)
  - test/func/test-preflight-check-vsphere.sh  # +5/-5 lines (Path N + Path P greps updated to D-04/D-16 wording)
tech-stack:
  added: []  # pure rename; no new runtime deps
  patterns:
    - Per-scope two-step RBAC probe (permissions.ls -> awk Principal filter -> role.ls -> allowlist grep) - carried verbatim from Phase 2 Plan 02-04
    - Admin fast-path (built-in Admin has all privileges by construction) - unchanged
    - No-access explicit-deny branch (every required priv is missing, bumps _preflight_errors per priv) - unchanged, wording updated only
    - awk tab-split with default-whitespace-split fallback + NR>1 header guard - unchanged
    - out=$(cmd 2>&1) stderr-variable capture idiom - unchanged
    - D-16 lowercase scope-kind token ('folder', 'resource pool') passed as explicit first arg to the helper
key-files:
  created:
    - .planning/phases/03-privilege-validation/03-01-SUMMARY.md
  modified:
    - scripts/preflight-check-vsphere.sh
    - test/func/test-preflight-check-vsphere.sh
decisions:
  - D-03 (applied): Renamed _vsphere_check_writeaccess to _vsphere_check_privileges. All occurrences (1 definition + 2 call sites + 1 docblock mention) updated.
  - D-04 (applied): Replaced "missing VM-create privilege '$req'" with "missing privilege '$req'" in both warning-emitting aba_warning lines (step 3b No-access branch + step 4 grep-miss branch). The "VM-create" substring is fully eradicated from scripts/preflight-check-vsphere.sh.
  - D-16 (applied): Added kind as the new first positional arg so every warning starts with a lowercase scope-kind token ("folder" or "resource pool" for the two current call sites). Signature now takes kind as $1, scope_path as $2, shift 2. Warning format is "vSphere: $kind '$scope_path' missing privilege '$req' [(user has role 'No access')]".
  - D-02 (honoured): _vsphere_probe_writeaccess sequencer (outer Layer 4 dispatcher) is NOT renamed in this plan. The follow-up plan replaces its body; keeping the sequencer name stable minimizes this plan's blast radius.
  - Phase 2 helper internals preserved byte-identical beyond the three changes above: govc permissions.ls call, query-failure warning + _preflight_warnings bump, awk two-pass Principal filter, no-role warning, Admin fast-path + aba_debug, role.ls call + role.ls-failure warning + _preflight_warnings bump, grep -qxF allowlist loop, counter semantics.
metrics:
  duration_seconds: 185
  duration_minutes: 3
  completed: 2026-04-16
  tasks_completed: 1
  files_modified: 2
  test_assertions_green: "33/33 Phase 2 paths (Paths A-P) + Phase 1 vmware-required-privileges array test"
---

# Phase 3 Plan 01: vSphere Privilege Probe Rename Summary

Pure surgical rename + wording reshape that unlocks the follow-up plan's 7-scope sequencer rewrite. Renamed Phase 2's per-scope RBAC helper `_vsphere_check_writeaccess` to `_vsphere_check_privileges`, added an explicit `kind` parameter (D-16) so every warning emits a lowercase scope-kind token, and reworded both `aba_warning` lines from "missing VM-create privilege" to "missing privilege" (D-04). Helper internals (govc permissions.ls call, awk two-pass Principal filter + NR>1 header guard, Admin fast-path, No-access explicit-deny loop, govc role.ls call, grep -qxF allowlist loop, query-level-failure vs missing-priv counter split) are preserved byte-identical. The outer `_vsphere_probe_writeaccess` sequencer (not renamed per D-02) passes the new kind arg ("folder", "resource pool") so the file stays parseable and Phase 2's 33-path test suite stays green.

## What Shipped

### `scripts/preflight-check-vsphere.sh` (+12/-10 lines)

**Helper rename + kind parameter.** The helper definition at line 282 renamed from `_vsphere_check_writeaccess()` to `_vsphere_check_privileges()`. Signature-capture block changed from `local scope_path="$1"; shift; local -a required_privs=("$@")` to `local kind="$1"; local scope_path="$2"; shift 2; local -a required_privs=("$@")`. The `$1/$2/$@` docblock two lines above updated to document the new shape (kind = lowercase scope label; scope_path = absolute vSphere inventory path; $@ = allowlist). The function-level one-line description swapped from "Per-scope write-access probe (RES-07, Phase 2 corrected two-step algorithm per 02-RESEARCH.md):" to "Per-scope privilege probe (Phase 3 privilege validation; reuses Phase 2's two-step RBAC algorithm verbatim):". All 8 enumerated docblock lines retained unchanged.

**D-04 wording swap on both warning branches.** Step 3b (No-access explicit-deny, previous line 331):
```
- aba_warning "vSphere: $scope_path missing VM-create privilege '$req' (user has role 'No access')"
+ aba_warning "vSphere: $kind '$scope_path' missing privilege '$req' (user has role 'No access')"
```
Step 4 (grep-miss per-priv loop, previous line 351):
```
- aba_warning "vSphere: $scope_path missing VM-create privilege '$req'"
+ aba_warning "vSphere: $kind '$scope_path' missing privilege '$req'"
```
The "VM-create" substring is fully eradicated from the production script.

**Call-site updates in `_vsphere_probe_writeaccess`.** Lines 371 and 376 changed to pass the new kind arg as positional $1:
```
- _vsphere_check_writeaccess "$VC_FOLDER" \
+ _vsphere_check_privileges "folder" "$VC_FOLDER" \
        VirtualMachine.Inventory.Create \
        VirtualMachine.Config.AddNewDisk
- _vsphere_check_writeaccess "$pool_path" \
+ _vsphere_check_privileges "resource pool" "$pool_path" \
        Resource.AssignVMToPool
```

**Docblock touch-up.** Line 360 comment `# Layer 4 sequencer. Runs _vsphere_check_writeaccess against VC_FOLDER ...` swapped to `_vsphere_check_privileges`. Required to satisfy the plan's strict-negative acceptance grep `! grep -q '_vsphere_check_writeaccess' scripts/preflight-check-vsphere.sh` (documented as a deviation below; see Deviation 1).

**Unchanged (byte-identical to Phase 2):**
- Step 1 `govc permissions.ls` call + perms_rc handling + the 4-line "cannot verify write-access" warning + `_preflight_warnings++`. The stale closing phrase "Skipping RES-07 for this scope; Phase 3 privilege query may still catch gaps." is NOT touched in this plan (the follow-up plan reshapes the Layer 4 sequencer and can reword if needed).
- Step 2 awk tab-split + whitespace-split fallback + NR>1 header guard + "no role assigned" warning + `_preflight_warnings++`.
- Step 3a Admin fast-path (`aba_debug` + return 0).
- Step 3c `govc role.ls` call + role.ls-failure warning + `_preflight_warnings++`.
- Counter-bump idioms: `_preflight_warnings=$(( _preflight_warnings + 1 ))` and `_preflight_errors=$(( _preflight_errors + 1 ))`.
- `return 0` at end of helper.
- Outer sequencer body structure (`resolve-default-resource-pool` + two allowlist calls + `return 0`).

### `test/func/test-preflight-check-vsphere.sh` (+5/-5 lines)

**Path N grep updates (lines 516-518).** All three No-access-role assertion greps updated from the old "missing VM-create privilege '<priv>' (user has role 'No access')" wording to the new D-04/D-16 form:
```
- folder_missing="missing VM-create privilege 'VirtualMachine.Inventory.Create' (user has role 'No access')"
+ folder_missing="folder '/GoodDC/vm/folder' missing privilege 'VirtualMachine.Inventory.Create' (user has role 'No access')"
- disk_missing="missing VM-create privilege 'VirtualMachine.Config.AddNewDisk' (user has role 'No access')"
+ disk_missing="folder '/GoodDC/vm/folder' missing privilege 'VirtualMachine.Config.AddNewDisk' (user has role 'No access')"
- pool_missing="missing VM-create privilege 'Resource.AssignVMToPool' (user has role 'No access')"
+ pool_missing="resource pool '/GoodDC/host/GoodCluster/Resources' missing privilege 'Resource.AssignVMToPool' (user has role 'No access')"
```

**Path P grep updates (lines 547-548).** Both "VMBuilder custom role" assertion greps updated:
```
- missing="missing VM-create privilege 'VirtualMachine.Config.AddNewDisk'"
+ missing="folder '/GoodDC/vm/folder' missing privilege 'VirtualMachine.Config.AddNewDisk'"
- present="missing VM-create privilege 'VirtualMachine.Inventory.Create'"
+ present="folder '/GoodDC/vm/folder' missing privilege 'VirtualMachine.Inventory.Create'"
```

These are grep-only updates to track the new wording of the same test paths - NOT new test paths. New Path Q-Z coverage (ROOT priv gap, multi-scope gaps, missing-object skip, ISO_DATASTORE dedup, Admin fast-path across 7 scopes, role-ls failure downgrade) lands in a later plan.

## Commits

| Task | Hash      | Summary                                                                  |
| ---- | --------- | ------------------------------------------------------------------------ |
| 1    | b6a58b99  | refactor(03-01): rename _vsphere_check_writeaccess to _vsphere_check_privileges |

## Verification Results

Static:
- `bash -n scripts/preflight-check-vsphere.sh` - PASS.
- `bash -n test/func/test-preflight-check-vsphere.sh` - PASS.
- `grep -q '_vsphere_check_privileges()' scripts/preflight-check-vsphere.sh` - PASS (new definition present).
- `! grep -q '_vsphere_check_writeaccess' scripts/preflight-check-vsphere.sh` - PASS (old name fully eradicated from production script - including docblock mention).
- `grep -c '_vsphere_check_privileges' scripts/preflight-check-vsphere.sh` reports 4 (1 definition + 2 call sites + 1 docblock mention on line 360). Plan's expected count was 3; docblock-mention update is captured as Deviation 1 below.
- `! grep -q 'missing VM-create privilege' scripts/preflight-check-vsphere.sh` - PASS (D-04 old wording eradicated).
- `grep -c "missing privilege '" scripts/preflight-check-vsphere.sh` reports 2 - PASS (step 3b No-access branch + step 4 grep-miss branch).
- `grep -cF "vSphere: \$kind '\$scope_path' missing privilege" scripts/preflight-check-vsphere.sh` reports 2 - PASS (D-16 kind+path form on both warning lines).
- `grep -q 'local kind="\$1"' scripts/preflight-check-vsphere.sh` - PASS.
- `grep -q 'shift 2' scripts/preflight-check-vsphere.sh` - PASS.
- `grep -qP '^\t[^\t ]' scripts/preflight-check-vsphere.sh` - PASS (tabs-for-indent).
- No space-indented non-comment lines - PASS.
- No banned `(( var++ ))` / `(( var-- ))` - PASS.
- `! grep -E '\b[A-Z]{4,7}-[0-9]+\b' scripts/preflight-check-vsphere.sh` - PASS (no internal-ticket tokens; see Deviation 2 for the `PRIV-01/PRIV-02` text I removed from the docblock header to satisfy this negative grep).
- `grep -c '_vsphere_check_privileges "folder"' scripts/preflight-check-vsphere.sh` = 1 - PASS.
- `grep -c '_vsphere_check_privileges "resource pool"' scripts/preflight-check-vsphere.sh` = 1 - PASS.
- `grep -q 'Resource.AssignVMToPool' scripts/preflight-check-vsphere.sh` - PASS (allowlist privilege names preserved at call site).
- Counter-bump idioms (`_preflight_errors=$(( _preflight_errors + 1 ))` and `_preflight_warnings=$(( _preflight_warnings + 1 ))`) present - PASS.
- No em-dashes introduced: `! grep -Pn '\x{2014}' scripts/preflight-check-vsphere.sh test/func/test-preflight-check-vsphere.sh` - PASS.
- No trailing whitespace - PASS.
- Commit message has no internal-ticket tokens: `git log -1 --pretty=%B | grep -qE '\b[A-Z]{4,7}-[0-9]+\b'` exits non-zero - PASS.
- `git log -1 --name-only` lists exactly `scripts/preflight-check-vsphere.sh` and `test/func/test-preflight-check-vsphere.sh` - PASS.

Behavioural:
- `bash test/func/test-preflight-check-vsphere.sh` - 33/33 PASS (all Paths A-P still green, including Path N + Path P with the D-04/D-16 wording).
- `bash test/func/test-vmware-required-privileges.sh` - All PASS (Phase 1 array-count test untouched and unaffected).

## Key Decisions Applied

| Decision | How applied in Plan 03-01 |
|----------|---------------------------|
| D-03 rename | `_vsphere_check_writeaccess` -> `_vsphere_check_privileges`. Applied at the definition site (line 283) and both call sites inside `_vsphere_probe_writeaccess` (lines 371, 376). The docblock mention on line 360 also updated (see Deviation 1). All occurrences of the old name in the production script are eradicated. |
| D-04 wording swap | "missing VM-create privilege '<priv>'" -> "missing privilege '<priv>'" on both warning-emitting lines (step 3b No-access branch + step 4 grep-miss branch). `grep -c "missing VM-create privilege"` now returns 0; `grep -c "missing privilege '"` returns exactly 2. |
| D-16 kind parameter | Helper signature now takes `kind` as `$1` (lowercase scope label), `scope_path` as `$2`, with `shift 2` before array-capture. Warning format is `"vSphere: $kind '$scope_path' missing privilege '$req' [(user has role 'No access')]"`. Both call sites pass the kind literal: "folder" for `$VC_FOLDER` and "resource pool" for the resolved RP path. |
| D-02 (honoured) | Outer sequencer `_vsphere_probe_writeaccess` NOT renamed in this plan. Its body is unchanged except for the two call-site name updates. The follow-up plan replaces the sequencer body with the 7-scope iteration. |
| Phase 2 body preserved byte-identical | The 91-line helper body's pre-existing Phase 2 logic (govc permissions.ls, awk two-pass Principal filter, NR>1 header guard, no-role warning, Admin fast-path, No-access branch, role.ls call, role.ls-failure warning, grep -qxF allowlist loop, counter semantics) is carried forward without semantic change. Only 4 lines changed in the helper body: the one-line function description, the `$1/$2` docblock, the signature-capture, and the two warning strings. |
| T-03-01-03 (counter drift) mitigation | Counter-bump sites on lines 332 and 352 are NOT moved. Both `_preflight_errors=$(( _preflight_errors + 1 ))` bumps remain directly below their respective `aba_warning` lines, inside the same for-loop scope. Verified by acceptance grep. |
| T-03-01-04 (review-surface) mitigation | Plan 03-01's diff is 12 insertions / 10 deletions in the production script and 5/5 in the test. Every changed line is either a rename, a signature, a docblock, a warning string, or a call-site - there is no algorithm change. |

## Deviations from Plan

### [Rule 1 - Bug] Docblock mention of the old name must also be updated to satisfy the strict-negative grep

**Found during:** post-edit acceptance-grep verification (before commit).

**Issue:** The plan's action step 5 narrowly lists the two call-site lines (371, 376) as the places the old name lives outside the helper definition, but line 360 also contains the old name in a descriptive comment above the outer sequencer definition (`# Layer 4 sequencer. Runs _vsphere_check_writeaccess against VC_FOLDER ...`). Leaving that line untouched would make `! grep -q '_vsphere_check_writeaccess' scripts/preflight-check-vsphere.sh` fail (the plan's own acceptance criterion).

**Fix:** Updated the comment on line 360 to reference the renamed helper. One-line pure-comment change; semantically identical; documentation accuracy improved.

**Files modified:** `scripts/preflight-check-vsphere.sh` (single comment line).

**Commit:** `b6a58b99` (bundled with Task 1).

**Rationale:** The plan's strictest criterion (full-eradication of the old name) wins over the softer 3-occurrence count. Occurrence count is 4 (1 definition + 2 call sites + 1 docblock mention) instead of the plan's expected 3. The extra occurrence is a single comment word; no behavioural consequence.

### [Rule 1 - Bug] Rephrased the function-level docblock summary to strip requirement-id tokens

**Found during:** post-edit acceptance-grep verification (before commit).

**Issue:** The plan's action step 1 prescribed replacing the helper's function-level summary line with `"Per-scope privilege probe (Phase 3 PRIV-01/PRIV-02; reuses Phase 2's two-step RBAC algorithm verbatim):"`. The substring `PRIV-01/PRIV-02` matches the plan's own strict-negative regex `\b[A-Z]{4,7}-[0-9]+\b` (4-letter prefix, dash, digits). It also contradicts the user's global memory rule that aba ships to a public repo and must not reference planning-artefact IDs in code. The plan's text self-contradicts its acceptance criterion.

**Fix:** Rephrased the docblock summary to `"Per-scope privilege probe (Phase 3 privilege validation; reuses Phase 2's two-step RBAC algorithm verbatim):"`. Same information, same intent ("this is Phase 3's privilege-validation helper"), different literal text without matching the forbidden regex.

**Files modified:** `scripts/preflight-check-vsphere.sh` (single comment line in the helper's header docblock).

**Commit:** `b6a58b99` (bundled with Task 1).

**Rationale:** Satisfies the plan's explicit acceptance criterion AND the project's public-repo hygiene policy (MEMORY.md + CLAUDE.md). No information loss - the phrase "Phase 3 privilege validation" conveys the same scoping context a reader needs; specific requirement IDs belong in `.planning/` (gitignored), not in the shipped code.

### [Rule 2 - Critical, NOT applied] Stale "Skipping RES-07 ... Phase 3 privilege query may still catch gaps" phrase left untouched by design

**Observation:** The helper's step-1 query-failure `aba_warning` closes with `"Skipping RES-07 for this scope; Phase 3 privilege query may still catch gaps."`. Phase 3 is now itself the privilege query, so the message is stale - it still reads as if Phase 3 were a future milestone. The plan explicitly instructs (action step 4) to "keep the line verbatim to minimize the diff" and to let the follow-up plan reword during the 7-scope sequencer rewrite.

**Action:** Kept the line verbatim. Documented here so the follow-up plan's executor has a pointer - the stale reference is a known issue, not a carry-over miss.

## Known Stubs

None. The helper is fully wired and fully functional under its new name. The outer sequencer `_vsphere_probe_writeaccess` still dispatches through it with the narrow 2-scope allowlists; behaviour is unchanged from Phase 2 Plan 02-04's verified state.

## Threat Flags

None. Every STRIDE threat in the plan's `<threat_model>` (T-03-01-01 through T-03-01-04) was either accepted or mitigated as declared:
- T-03-01-01 (ANSI/control chars in role names): aba_warning does not re-interpret escapes; no change to that call path.
- T-03-01-02 (shell metachars in `$GOVC_USERNAME` / `$role_name`): `$role_name` remains double-quoted in the `govc role.ls "$role_name"` call (unchanged from Phase 2).
- T-03-01-03 (counter drift): counter bumps on lines 332 and 352 unchanged; verified by acceptance grep.
- T-03-01-04 (review surface): commit diff is 17 insertions / 15 deletions total; each line is rename/signature/docblock/wording/call-site; zero algorithm change.

No new trust boundaries, no new network surface, no new auth paths, no new file-access patterns, no schema changes. The helper's attacker-controlled inputs (`$GOVC_USERNAME`, role names and privilege strings from `govc permissions.ls` / `govc role.ls`) are interpolated into `aba_warning` in the same positions and quoting as Phase 2.

## Self-Check: PASSED

- FOUND: scripts/preflight-check-vsphere.sh (437 lines; `_vsphere_check_privileges()` at line 283, `_vsphere_probe_writeaccess()` at line 366, call sites at lines 371 + 376, docblock mention at line 360)
- FOUND: test/func/test-preflight-check-vsphere.sh (+5/-5; Path N greps at lines 516-518, Path P greps at lines 547-548)
- FOUND: .planning/phases/03-privilege-validation/03-01-SUMMARY.md (this file)
- FOUND commit b6a58b99 (Task 1: rename + kind parameter + D-04 wording + call-site updates + Path N/P test-grep updates)
- All 33 Phase 2 structural + Path A-P behavioural assertions remain green after the rename
- All 17 Phase 1 vmware-required-privileges array-count assertions remain green
- All 18 Task 1 acceptance-grep criteria PASS (including both deviation-triggered adjustments)
- No modifications to `scripts/include_all.sh`, `scripts/preflight-check.sh`, `scripts/vmware-required-privileges.sh`, or any Makefile (scope honoured)
- No modifications to `.planning/STATE.md` or `.planning/ROADMAP.md` (orchestrator owns those writes)
- No internal-ticket references (`[A-Z]{4,7}-[0-9]+`) in production script or commit message
- No em-dashes introduced in any modified file
- No banned `(( var++ ))` arithmetic introduced
- No stderr-suppression patterns introduced
- Single atomic commit on the feature/vmware-preflight branch

---
*Phase 3 Plan 01 completed 2026-04-16. Duration 3 min. The follow-up plan can now start from a file where only the outer Layer 4 sequencer (`_vsphere_probe_writeaccess`) body needs reshaping - the per-scope helper is already named, parameterised, and worded for the 7-scope iteration.*

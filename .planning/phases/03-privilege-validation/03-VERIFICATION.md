---
phase: 03-privilege-validation
verified: 2026-04-16T00:00:00Z
status: passed
score: 5/5 must-haves verified
overrides_applied: 0
re_verification:
  previous_status: none
  previous_score: none
  gaps_closed: []
  gaps_remaining: []
  regressions: []
---

# Phase 3: Privilege Validation - Verification Report

**Phase Goal:** Deliver vSphere privilege validation against a curated 7-scope list (PRIV-01, PRIV-02, PRIV-04, PRIV-05, UX-02).
**Verified:** 2026-04-16
**Status:** PASSED
**Re-verification:** No - initial verification

---

## Goal Achievement

### Observable Truths (Phase Requirements)

| #   | Requirement | Status | Evidence |
| --- | ----------- | ------ | -------- |
| 1 | **PRIV-01**: effective-privilege query across the 7 curated scopes (ROOT, DATACENTER, CLUSTER, DATASTORE, NETWORK, FOLDER, RESOURCE_POOL) | PASS | `scripts/preflight-check-vsphere.sh:403` sources `scripts/vmware-required-privileges.sh`; lines 416/424/435/446/459/468/479/490 invoke `_vsphere_check_privileges` against each scope path; scope-to-path mapping matches D-10. Test Paths Q (ROOT), R (DC), Y (DATASTORE + ISO DATASTORE) cover per-scope dispatch. |
| 2 | **PRIV-02**: compare observed privileges against the curated VSPHERE_PRIVS_* arrays | PASS | All 7 arrays referenced at `preflight-check-vsphere.sh:416-490` with quoted expansion `"${VSPHERE_PRIVS_<SCOPE>[@]}"`; helper `_vsphere_check_privileges` at lines 310-385 uses `govc role.ls` + `grep -qxF` loop against required_privs. Path N asserts 83-missing count (flat sum of array lengths 11+30+5+3+1+28+5) proving full-array iteration. |
| 3 | **PRIV-04**: itemised gaps per scope with one `aba_warning` per missing privilege | PASS | Lines 360 and 380 emit one `aba_warning` per gap: `"vSphere: $kind '$scope_path' missing privilege '$req'[...]"`; counter-bump `_preflight_errors=$(( _preflight_errors + 1 ))` follows each warning at lines 361 and 381. Because scopes iterate sequentially (lines 416-496), gaps appear grouped by scope naturally. Path R verifies 30 DC-scope warnings in one group; Path N verifies 83 warnings across 7 scopes. |
| 4 | **PRIV-05**: distinguish "privilege not granted" from "object not found" | PASS | 7 file-scope found-flags declared at `preflight-check-vsphere.sh:26-32`; populated by `_vsphere_probe_resources` (lines 216, 221, 226, 233, 243, 249, 270). Six non-ROOT scopes gate on `_vsphere_<scope>_found=1` (lines 422/433/444/466/477/488); missing-object path emits `aba_debug "vSphere: skipping privilege check for missing <kind> '<path>'"` (lines 429/440/451/473/484/495) with NO counter bump. Path S asserts aba_debug (not aba_warning) and errors=0 on found=0. |
| 5 | **UX-02**: copy-pasteable categorised output + "next step" hint | PASS | D-17 summary block at `preflight-check-vsphere.sh:501-505` emits ONE multi-line `aba_warning`: (a) headline `"vSphere: $gap_count privilege gap(s) across $scopes_with_gaps scope(s)"`, (b) "Next: review the curated list at scripts/vmware-required-privileges.sh and the OpenShift docs linked in its header.", (c) "Grant the missing privileges to the vCenter user or role and re-run aba install." Gated by `if [ "$gap_count" -gt 0 ]` for quiet-on-success. Paths Q/R/X/Y verify headline content and count; Paths U/W verify summary suppressed on clean pass; Path Z verifies D-18 no-double-count. |

**Score:** 5/5 requirements verified

---

## Required Artifacts

| Artifact | Expected | Status | Details |
| -------- | -------- | ------ | ------- |
| `scripts/preflight-check-vsphere.sh` | 7-scope sequencer `_vsphere_probe_privileges`, 7 found-flags, D-17 summary block | PASS | 564 lines. `_vsphere_probe_privileges()` at line 401; 7 found-flags at lines 26-32; D-17 summary at lines 501-505. |
| `test/func/test-preflight-check-vsphere.sh` | Paths A-Z covering 7-scope behaviour | PASS | 820 lines. 17 structural + 26 behavioural assertions (A-Z) all passing. |
| `scripts/vmware-required-privileges.sh` | 7 curated arrays, untouched in Phase 3 | PASS | Read-only in Phase 3. 7 arrays: ROOT=11, DATACENTER=30, CLUSTER=5, DATASTORE=3, NETWORK=1, FOLDER=28, RESOURCE_POOL=5. |

---

## Key Link Verification

| From | To | Via | Status | Details |
| ---- | -- | --- | ------ | ------- |
| `_vsphere_probe_privileges` | `scripts/vmware-required-privileges.sh` | `source` + array expansion | WIRED | Line 403 sources curated arrays; lines 416/424/435/446/459/468/479/490 expand `"${VSPHERE_PRIVS_<SCOPE>[@]}"`. |
| `_vsphere_probe_resources` | `_vsphere_probe_privileges` | 7 file-scope found-flags (`_vsphere_<scope>_found`) | WIRED | Probe sets flags on success (lines 216/221/226/233/243/249/270); sequencer reads them (lines 422/433/444/466/477/488). |
| `preflight_check_vsphere` | `_vsphere_probe_privileges` | final sequencer dispatch after resource probe | WIRED | Line 562 dispatches to `_vsphere_probe_resources || return 0`; line 563 calls `_vsphere_probe_privileges`. Stale "Phase 3 will add..." comment removed per D-01. |
| `_vsphere_probe_privileges` | `_vsphere_check_privileges` | function call with kind + path + required_privs | WIRED | 8 call sites (7 scopes + ISO DATASTORE conditional) at lines 416/424/435/446/459/468/479/490. |

---

## Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
| -------- | ------------- | ------ | ------------------ | ------ |
| `_vsphere_probe_privileges` output | `_preflight_errors` delta | Per-priv `aba_warning` fires from `_vsphere_check_privileges` step 3b/step 4 | Yes - Paths N/P/Q/R/X/Y assert concrete error counts | FLOWING |
| `_vsphere_probe_privileges` output | `scopes_with_gaps` counter | Incremented per scope when `_preflight_errors` grew during that scope (lines 418/426/437/448/461/470/481/492) | Yes - Path R asserts "30 across 1"; Path Y asserts "17 across 3" | FLOWING |
| `_vsphere_probe_resources` found-flags | `_vsphere_<scope>_found` | Set on `_vsphere_object_exists` success or inline RP probe | Yes - Path S asserts flag=0 produces aba_debug skip; default probe scenarios (Path C flow) set all 7 flags | FLOWING |
| D-17 summary line | `$gap_count`, `$scopes_with_gaps` | Arithmetic diff of `_preflight_errors` before/after + scope-increment counter | Yes - Paths Q/R/X/Y assert headline with concrete N/M substitutions | FLOWING |

---

## Behavioural Spot-Checks

| Behavior | Command | Result | Status |
| -------- | ------- | ------ | ------ |
| Full test suite passes | `bash test/func/test-preflight-check-vsphere.sh` | Exit 0; 17 structural + 26 behavioural (A-Z) PASS; "=== All Tests Passed ===" footer | PASS |
| Curated privileges unit test passes | `bash test/func/test-vmware-required-privileges.sh` | Exit 0; all array-count and element-presence assertions PASS | PASS |
| Production script parses | `bash -n scripts/preflight-check-vsphere.sh` | Exit 0 | PASS |
| No "VM-create" wording remains (D-04) | `grep -n "VM-create" scripts/preflight-check-vsphere.sh` | No matches | PASS |
| No banned `(( var++ ))` arithmetic | `grep -Pn '\(\(\s*\w+\s*(\+\+|--)\s*\)\)' scripts/preflight-check-vsphere.sh` | No matches | PASS |
| No internal-ticket tokens | `grep -E '\b[A-Z]{4,7}-[0-9]+\b' scripts/preflight-check-vsphere.sh` | No matches | PASS |
| No em-dashes | `grep -Pn '\x{2014}' scripts/preflight-check-vsphere.sh` | No matches | PASS |
| No banned `2>&1 \| grep` pipeline | `grep -n "2>&1 *\| *grep" scripts/preflight-check-vsphere.sh` | No matches (comment references only, not the actual forbidden pipeline) | PASS |
| Tabs for indentation | Inspected file; no space-indented code lines | PASS | PASS |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
| ----------- | ----------- | ----------- | ------ | -------- |
| PRIV-01 | 03-02 / 03-03 | Query effective privileges across 7 scopes | SATISFIED | `source scripts/vmware-required-privileges.sh` + 7 dispatch calls to `_vsphere_check_privileges` across ROOT, DC, CLUSTER, DS (+ISO), NET, FOLDER, RP. Test Paths Q/R/Y confirm per-scope dispatch. |
| PRIV-02 | 03-02 / 03-03 | Compare observed privileges vs curated VSPHERE_PRIVS_* arrays | SATISFIED | All 7 arrays passed to helper; per-priv `grep -qxF` loop at lines 378-383. Path N asserts 83 (flat sum of all arrays) on No-access role. |
| PRIV-04 | 03-02 / 03-03 | Itemized gaps per scope, not opaque message | SATISFIED | Lines 360/380: `aba_warning "vSphere: $kind '$scope_path' missing privilege '$req'[...]"` emits one line per missing privilege; sequential scope iteration groups them naturally. Paths P/R demonstrate grouping. |
| PRIV-05 | 03-02 / 03-03 | Distinguish privilege-not-granted from object-not-found | SATISFIED | Found-flag gating at lines 422/433/444/466/477/488; missing-object emits `aba_debug` (not `aba_warning`) and does NOT bump counters. Path S asserts this directly. |
| UX-02 | 03-02 / 03-03 | Categorized copy-pasteable output + next-step hint | SATISFIED | D-17 summary at lines 501-505 emits three-line summary with curated-list reference + grant-and-re-run hint, gated on `gap_count > 0`. Paths Q/R/X/Y/U/W/Z cover emission, suppression, and no-double-count. |

All 5 phase requirements SATISFIED.

---

## Anti-Patterns Found

None.

| File | Line | Pattern | Severity | Impact |
| ---- | ---- | ------- | -------- | ------ |

Scans performed:
- `VM-create` wording (D-04): eradicated from production script
- `(( var++ ))` / `(( var-- ))`: none introduced
- Internal-ticket tokens (`[A-Z]{4,7}-[0-9]+`): none introduced in production or test code
- Em-dashes (U+2014): none introduced
- Stderr-suppression pipelines (`2>&1 | grep`): none (only comment-references to the ban)
- Space-indented code lines: none
- AI-assistance references (Claude, co-authored, AI assistance): none in commits, production, or test

Note on line 328 stale comment: `_vsphere_check_privileges` still contains a step-1 `aba_warning` closing line "Skipping RES-07 for this scope; Phase 3 privilege query may still catch gaps." Phase 3 IS now the privilege query, so the phrase is stale (self-referential). This was explicitly deferred by design per 03-01-SUMMARY "Deviation not applied" and is not a gap - it is a documentation-refresh task appropriate for a later plan, not a failure of the Phase 3 goal.

---

## Human Verification Required

None. All behaviour is asserted by the automated test suite (Paths A-Z) without requiring real vCenter or runtime inspection.

---

## Gaps Summary

No gaps.

All 5 phase requirements (PRIV-01, PRIV-02, PRIV-04, PRIV-05, UX-02) are satisfied with concrete code references and covered by the 43-assertion test suite. The production script is syntactically valid, adheres to all project coding rules (tabs, no banned arithmetic, no stderr suppression, no internal-ticket tokens, no em-dashes, no AI references), and integrates correctly with the Phase 2 layered preflight sequencer. The found-flag gating mechanism enforces PRIV-05's no-conflation contract; the D-17 summary block delivers UX-02's next-step hint; the 7-scope iteration over the curated VSPHERE_PRIVS_* arrays delivers PRIV-01/PRIV-02/PRIV-04.

---

## Commits Verified

| Hash | Subject |
| ---- | ------- |
| b6a58b99 | refactor(03-01): rename _vsphere_check_writeaccess to _vsphere_check_privileges |
| 7187f6b9 | docs(03-01): add Plan 03-01 SUMMARY for _vsphere_check_privileges rename |
| 3a785e73 | feat(03-02): replace Phase 2 write-access layer with full 7-scope privilege layer |
| 6f36a1c8 | docs(03-02): add Plan 03-02 SUMMARY for 7-scope privilege sequencer |
| 23802c91 | test(03-03): cover Phase 3 7-scope privilege layer (Paths Q-Z + M/N/O/P repair) |
| d3db739b | docs(03-03): add Plan 03-03 SUMMARY for Phase 3 behavioural test coverage |

All commits checked for:
- No internal-ticket tokens in message bodies
- No AI-assistance references
- Angular commit convention
- Correct scope (03-01 / 03-02 / 03-03)

---

_Verified: 2026-04-16_
_Verifier: Claude (gsd-verifier)_

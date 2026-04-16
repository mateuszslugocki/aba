---
phase: 03-privilege-validation
plan: "02"
subsystem: vsphere-preflight
tags:
  - vsphere
  - preflight
  - privileges
  - rbac
  - sequencer
  - bash
requires:
  - scripts/preflight-check-vsphere.sh  # Phase 2 Layer 4 sequencer + Plan 03-01 renamed helper
  - scripts/vmware-required-privileges.sh  # VSPHERE_PRIVS_<SCOPE> curated arrays
  - scripts/include_all.sh  # aba_warning / aba_debug
  - scripts/preflight-check.sh  # _preflight_errors / _preflight_warnings counters
provides:
  - preflight-check-vsphere:_vsphere_probe_privileges  # 7-scope privilege sequencer (renamed from _vsphere_probe_writeaccess)
  - preflight-check-vsphere:_vsphere_<scope>_found  # 7 file-scope found-flags set by _vsphere_probe_resources
affects:
  - scripts/preflight-check-vsphere.sh  # +148/-22 lines (7 flag inits + sequencer rewrite + call-site rename + found-flag wiring + D-17 summary)
requirements:
  - PRIV-02  # compare observed privileges against curated 7-scope list
  - PRIV-04  # itemized per-scope grouping with gap-count summary
tech-stack:
  added: []  # no new runtime deps; imports existing VSPHERE_PRIVS_<SCOPE> arrays
  patterns:
    - 7-scope D-14 iteration: root (unconditional), DC, cluster, DS, ISO DS (D-11 guarded), network, folder, resource pool
    - Found-flag gating: missing objects yield aba_debug skip, not a conflated privilege-gap warning
    - D-17 summary block: single multi-line aba_warning with gap count + curated-list reference + grant-and-re-run hint
    - Existing per-scope helper (_vsphere_check_privileges) invoked per scope with kind + path arguments
---

# Plan 03-02 SUMMARY: 7-Scope Privilege Sequencer

## Commit

- `3a785e73` feat(03-02): replace Phase 2 write-access layer with full 7-scope privilege layer

## Scope

Replaces the Phase 2 narrow 2-scope (VC_FOLDER + resolved resource pool) Layer 4 sequencer with the full D-14 7-scope privilege probe. Delivers PRIV-02 and PRIV-04 behaviour. Sequencer renamed from `_vsphere_probe_writeaccess` to `_vsphere_probe_privileges`.

## What Changed

- **7 file-scope found-flags** added at the top of the file (lines 26-32):
  `_vsphere_dc_found`, `_vsphere_cluster_found`, `_vsphere_datastore_found`, `_vsphere_iso_datastore_found`, `_vsphere_network_found`, `_vsphere_folder_found`, `_vsphere_resource_pool_found`. Populated by `_vsphere_probe_resources` on each successful existence probe.
- **Layer 4 sequencer rewritten** — now iterates 7 scopes referencing `VSPHERE_PRIVS_<SCOPE>` arrays from `scripts/vmware-required-privileges.sh`. Each non-root scope is gated on its Layer 3 `_vsphere_<scope>_found` flag; absent objects become `aba_debug` skips rather than privilege-gap warnings.
- **D-17 summary block** — emits one multi-line `aba_warning` with gap count, scope count, curated-list reference, and grant-and-re-run hint when at least one gap exists.
- **Call-site updated** — `_vsphere_probe_privileges` invoked from `preflight_check_vsphere` per the D-03 name.

## Diff Stats

- `scripts/preflight-check-vsphere.sh`: +148 / -22 lines
- Sourcing `scripts/vmware-required-privileges.sh`: 1 new line
- 7 found-flag initialisations, 10 `VSPHERE_PRIVS_<SCOPE>` array references verified via grep

## Test Suite Regression State (EXPECTED per plan line 614)

The plan explicitly expects behavioural Paths M/N/O/P to regress between 03-02 and 03-03; Plan 03-03 repairs them and adds Paths Q-Z.

- `bash -n scripts/preflight-check-vsphere.sh`: PASS
- `test/func/test-preflight-check-vsphere.sh` up to Path M: PASS (all structural + Paths A-M pass)
- Path N: FAIL (observed: `folder=1 disk=1 pool=1 errors=83`). Prior Phase 2 assertion was based on the 2-scope VM-create allowlist; 7-scope full iteration now counts missing privileges across all enabled scopes with the No-access role. Plan 03-03 will update the assertion (or replace Path N entirely) and add Paths Q-Z.
- Paths O and P: not observed (suite short-circuited on Path N under `set -e`).

## Deviations

None intentional. Agent execution was interrupted by an API 500 error after the feat commit landed but before SUMMARY.md was created; SUMMARY.md was authored by the orchestrator from the plan + commit state.

## Threat-Model Notes

- Double-quoting: all scope-path expansions in the sequencer use `"$VAR"` form.
- No shell-interpolation of curated arrays; they are expanded via `"${VSPHERE_PRIVS_<SCOPE>[@]}"` into the helper's allowlist argument list only.
- No commit contains internal-ticket references, em-dashes, or AI-assistance markers.

## Downstream

Plan 03-03 repairs the test suite (Paths M/N/O/P) and adds behavioural coverage for the 7-scope sequencer.

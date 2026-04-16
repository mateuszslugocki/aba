---
phase: 02-connectivity-and-resource-checks
plan: "02"
subsystem: vsphere-preflight
tags:
  - vsphere
  - preflight
  - resources
  - govc
  - bash
requires:
  - scripts/preflight-check-vsphere.sh  # Plan 02-01 helpers (_vsphere_parse_govc_url, _vsphere_probe_tcp/tls/auth)
  - scripts/preflight-check.sh  # counters + hook at line 201
  - scripts/include_all.sh  # aba_info/info_ok/warning/abort/debug (READ-ONLY in this plan)
provides:
  - preflight-check-vsphere:layer3-resources  # _vsphere_probe_resources sequencer (RES-01..05)
  - preflight-check-vsphere:object-exists     # _vsphere_object_exists generic helper (D-05 canonical)
  - preflight-check-vsphere:network-on-cluster # _vsphere_probe_resources_network_on_cluster (D-08)
affects:
  - scripts/preflight-check-vsphere.sh  # +107 lines net; 3 new private helpers, sequencer extended
  - test/func/test-preflight-check-vsphere.sh  # Path C stub block extended to no-op _vsphere_probe_resources
tech-stack:
  added: []  # no new runtime deps; still pure bash + govc
  patterns:
    - govc object.collect -s <absolute path> name  (D-05 canonical single-object probe)
    - govc find -i -type h <clusterPath>  (HostSystem moref listing for attachment check)
    - Per-line set-intersect via `echo "$set" | grep -qxF -- "$item"`  (host moref overlap)
    - DC-missing short-circuit via `if ! _vsphere_object_exists datacenter ...; then return 1`  (D-07)
    - Collect-all mid-layer via `... || :`  (D-04)
    - ISO_DATASTORE dedup guard: `[ -n "${ISO_DATASTORE:-}" ] && [ "$ISO_DATASTORE" != "$GOVC_DATASTORE" ]`
key-files:
  created: []
  modified:
    - scripts/preflight-check-vsphere.sh
    - test/func/test-preflight-check-vsphere.sh
decisions:
  - D-04 (carry-forward): Collect-all within a layer; short-circuit between layers - applied via `... || :` on sibling probes and `_vsphere_probe_resources || return 0` on the outer sequencer.
  - D-05 (carry-forward): Every existence check uses `govc object.collect -s <absolute path> name`. No `govc ls | grep` pattern anywhere.
  - D-06 + Pitfall 5: ISO_DATASTORE is probed as a DISTINCT RES-03 check only when set, non-empty, AND different from GOVC_DATASTORE. Same path -> dedup (single probe).
  - D-07 (DC cascade): Missing datacenter emits exactly two lines - one `aba_warning` "vSphere: datacenter '/DC' not found" AND one `aba_info` "vSphere: skipping cluster/datastore/network/folder/resource-pool checks until datacenter resolves". The info note is NOT a second error; counter bumps exactly once via `_vsphere_object_exists`.
  - D-08: Network probe cross-checks attachment to the configured cluster via set-intersect on HostSystem morefs. Soft-skips on govc read failure to avoid double-counting vs. the basic existence probe.
  - RES-06 (resource pool) deferred to Plan 02-03 per plan boundary; include_all.sh untouched in this plan.
metrics:
  duration_seconds: 201
  duration_minutes: 3
  completed: 2026-04-16
  tasks_completed: 2
  files_modified: 2
  test_assertions_green: "20/20 (Phase 1 paths A/B/C intact; Path C stub block extended for Layer 3)"
---

# Phase 2 Plan 02: vSphere Resource-Existence Probes Summary

Extended `preflight_check_vsphere` in `scripts/preflight-check-vsphere.sh` with Layer 3 (resource existence) covering RES-01..05 (datacenter, cluster, datastore primary + optional ISO, network + attachment cross-check, folder). Resource pool (RES-06) remains deferred to Plan 02-03 where the `resolve-default-resource-pool` helper lands in `include_all.sh`.

## What Shipped

Three new private helpers, wired into the existing Layer 1/2 sequencer:

- `_vsphere_object_exists <kind> <absolute-path>` - Generic single-object existence probe. Uses the D-05 canonical `govc object.collect -s <path> name`; captures stderr into a variable (allowed `out=$(cmd 2>&1)` idiom, not the banned `cmd 2>&1 | grep` pipeline). On success, silent aba_debug; on failure emits exactly one `aba_warning "vSphere: <kind> '<path>' not found"` and bumps `_preflight_errors` by one.
- `_vsphere_probe_resources_network_on_cluster` - RES-04 D-08 attachment cross-check. Reads the portgroup's `host` property (HostSystem morefs) with `govc object.collect -s "/$GOVC_DATACENTER/network/$GOVC_NETWORK" host`, reads the cluster's hosts with `govc find -i -type h "/$GOVC_DATACENTER/host/$GOVC_CLUSTER"`, and set-intersects per line via `grep -qxF`. On a real gap emits `"vSphere: network '<name>' is not attached to any host in cluster '<name>'"` + bumps counter. Soft-skips (debug only, no counter bump, returns 0) when either govc call fails, because the basic existence probe is the primary gate and we must not double-count.
- `_vsphere_probe_resources` - Layer 3 sequencer. DC first (D-07 cascade: missing DC emits exactly 1 warning + 1 info "skipping..." note + bumps counter exactly once, then short-circuits Layer 3). Otherwise runs cluster, datastore primary, optional ISO_DATASTORE (with D-06 dedup guard), network (+ attachment only if existence passes), and folder - all in collect-all mode via `... || :`.

Sequencer wiring inside `preflight_check_vsphere`:

```bash
_vsphere_probe_tcp       || return 0
_vsphere_probe_tls       || return 0
_vsphere_probe_auth      || return 0
_vsphere_probe_resources || return 0
```

The `|| return 0` on `_vsphere_probe_resources` fires only on the DC-missing cascade (the only path where that helper returns non-zero); everything else is signalled via `_preflight_errors`.

## Commits

| Task | Hash      | Summary                                                                |
| ---- | --------- | ---------------------------------------------------------------------- |
| 1    | 59f98e72  | feat(02-02): add _vsphere_object_exists and network-on-cluster helpers |
| 2    | cc4ffe7f  | feat(02-02): wire Layer 3 resource-existence sequencer with DC cascade |

## Verification Results

Static:
- `bash -n scripts/preflight-check-vsphere.sh` - PASS
- `grep -q '_vsphere_object_exists()'` - PASS (line 139)
- `grep -q '_vsphere_probe_resources_network_on_cluster()'` - PASS (line 161)
- `grep -q '_vsphere_probe_resources()'` - PASS (line 195)
- `grep -q 'govc object.collect -s "$path" name'` - PASS (line 142)
- `grep -q 'govc find -i -type h'` - PASS (line 164)
- `grep -q 'aba_info "vSphere: skipping cluster/datastore/network/folder/resource-pool'` - PASS (line 200)
- `grep -q 'ISO_DATASTORE" != "$GOVC_DATASTORE"'` - PASS (line 212)
- `grep -q '_vsphere_object_exists datacenter "/$GOVC_DATACENTER"'` - PASS (line 199)
- `grep -q '_vsphere_object_exists cluster "/$GOVC_DATACENTER/host/$GOVC_CLUSTER"'` - PASS (line 205)
- `grep -q '_vsphere_object_exists network "/$GOVC_DATACENTER/network/$GOVC_NETWORK"'` - PASS (line 219)
- `grep -q '_vsphere_object_exists folder "$VC_FOLDER"'` - PASS (line 224)
- `grep -q '_vsphere_probe_resources || return 0'` - PASS (line 284)
- No `govc ls` anywhere - PASS
- No `(( var++ ))` / `(( var-- ))` - PASS
- No internal-ticket refs (`[A-Z]{4,7}-[0-9]+`) - PASS
- No trailing whitespace - PASS
- `grep -c '_vsphere_object_exists' scripts/preflight-check-vsphere.sh` = 8 (>= 6 required)
- `grep -c '_vsphere_probe_' scripts/preflight-check-vsphere.sh` = 12 (>= 7 required)

Behavioural (in-process smoke tests with govc/aba_* stubs):

Task 1 - helpers:
- `_vsphere_object_exists` found case: silent + no counter bump - PASS
- `_vsphere_object_exists` not-found case: exact "not found" warning + 1 counter bump - PASS
- `_vsphere_probe_resources_network_on_cluster` overlap present: silent + no counter bump - PASS
- `_vsphere_probe_resources_network_on_cluster` no overlap: attachment warning + 1 counter bump - PASS
- `_vsphere_probe_resources_network_on_cluster` govc read fail: soft-skip (debug only) + no counter bump - PASS

Task 2 - sequencer:
- DC missing cascade -> 1 WARN + 1 INFO + `_preflight_errors=1` (exactly) - PASS
- DC exists + 4 other siblings missing -> 4 warnings + `_preflight_errors=4` (collect-all, D-04) - PASS
- ISO_DATASTORE unset -> 1 datastore probe - PASS
- ISO_DATASTORE == GOVC_DATASTORE -> 1 datastore probe (dedup) - PASS
- ISO_DATASTORE != GOVC_DATASTORE -> 2 datastore probes - PASS
- Network exists + attachment no-overlap -> existence OK + attachment WARN - PASS
- `_vsphere_probe_resources` returns 0 when DC exists, 1 only on DC-missing cascade - PASS

Test suite:
- `bash test/func/test-preflight-check-vsphere.sh` - 20/20 PASS
- `bash test/func/test-preflight-check.sh` - PASS (integration parent still green)
- `bash test/func/test-vmware-required-privileges.sh` - PASS (no collateral damage)

## Key Decisions Applied

| Decision | How applied in Plan 02-02 |
|----------|----------------------------|
| D-04 collect-all within layer | Sibling probes in `_vsphere_probe_resources` are `... || :` so one failure does not skip the next. Between layers, `_vsphere_probe_resources || return 0` in the outer sequencer preserves the short-circuit contract. |
| D-05 govc object.collect canonical | `_vsphere_object_exists` is the single chokepoint and uses `govc object.collect -s "$path" name` verbatim. No `govc ls` anywhere in the file. |
| D-06 datastore dedup | Guard `[ -n "${ISO_DATASTORE:-}" ] && [ "$ISO_DATASTORE" != "$GOVC_DATASTORE" ]` skips the second probe when paths match. Two distinct paths each get their own `_vsphere_object_exists datastore` call. |
| D-07 DC cascade | `if ! _vsphere_object_exists datacenter ...; then aba_info "...skipping..."; return 1; fi`. The "skipping" line is `aba_info` (NOT `aba_warning`) and does NOT bump `_preflight_errors` - the counter stays at exactly 1 for the root "datacenter not found" warning. |
| D-08 network-on-cluster | Attachment cross-check runs only after the basic existence probe passes (nested `if _vsphere_object_exists network ...; then _vsphere_probe_resources_network_on_cluster || :; fi`). Soft-skip on govc read failure avoids double-counting. |
| T-02-02-01 (Tampering: path construction) | All govc paths are double-quoted (`"/$GOVC_DATACENTER/host/$GOVC_CLUSTER"` etc.). No `eval`. |
| T-02-02-02 (Info disclosure: captured stderr) | `$out` is captured into a variable to keep the warning clean (`"vSphere: <kind> '<path>' not found"`) - raw govc stderr never reaches the user. |
| T-02-02-05 (Repudiation: counter mismatch) | DC cascade emits exactly 1 warning + 1 info and bumps the counter exactly once - verified by the T1 behavioural smoke test above and by the greps that confirm the cascade line is wrapped in `aba_info`, not `aba_warning`. |

## Deviations from Plan

### Rule 3 (blocking) - Path C smoke test needed `_vsphere_probe_resources` stub

**Found during:** Task 2 verification.

**Issue:** After Task 2 wired `_vsphere_probe_resources` into the sequencer, the Phase 1 Path C assertion (platform=vmw + all 7 required fields set to literal `"x"`) started failing: with `govc() { :; }` (returns 0 for every subcommand), `_vsphere_probe_resources` walked past the DC gate, called the network attachment sub-probe, found `cluster_hosts` empty (the generic stub produces no output for `govc find`), concluded "no overlap", and emitted the attachment warning. Path C then saw `_preflight_errors=1` and failed.

**Fix:** Extended the Path C stub block in `test/func/test-preflight-check-vsphere.sh` to also no-op `_vsphere_probe_resources`, matching the Plan 02-01 precedent that already no-ops `_vsphere_probe_tcp`, `_vsphere_probe_tls`, `_vsphere_probe_auth`. Path C's intent is a smoke test of the field-presence gate + OK line flow; the probes themselves will be covered by the dedicated Path D..P behavioural assertions delivered in Plan 02-05.

**Files modified:** `test/func/test-preflight-check-vsphere.sh` (Path C stub block only; committed together with Task 2).

**Rationale:** Test files live in the freely-editable zone per CLAUDE.md. The change is minimal-surgery - it does not change Path C's assertion or intent, it just extends the stub list to match the current sequencer.

## Threat Flags

None. All new surface (DC/cluster/datastore/network/folder probes + attachment cross-check) is covered by the plan's declared threat register (T-02-02-01 through T-02-02-05). Resource pool (RES-06) surface is explicitly deferred to Plan 02-03; no RP-related files, paths, or govc calls are touched in this plan.

## Self-Check: PASSED

- FOUND: scripts/preflight-check-vsphere.sh (289 lines; +107 net lines of new Layer 3 code)
- FOUND: test/func/test-preflight-check-vsphere.sh (222 lines; 1-line stub added)
- FOUND commit 59f98e72 (Task 1: _vsphere_object_exists + _vsphere_probe_resources_network_on_cluster helpers)
- FOUND commit cc4ffe7f (Task 2: _vsphere_probe_resources sequencer + preflight_check_vsphere wiring + Path C stub)
- All 20 existing Phase 1 assertions remain green
- All 12 Task 2 acceptance-criteria greps PASS
- All 5 Task 1 behavioural smoke tests PASS
- All 7 Task 2 behavioural smoke tests PASS
- No modifications to scripts/include_all.sh (RES-06 / D-13 reserved for Plan 02-03)
- No modifications to .planning/STATE.md or .planning/ROADMAP.md (orchestrator owns those writes)

---
*Phase 2 Plan 02 completed 2026-04-16. Duration 3 min. Next: Plan 02-03 adds `resolve-default-resource-pool` to include_all.sh and extends `_vsphere_probe_resources` with the RES-06 probe.*

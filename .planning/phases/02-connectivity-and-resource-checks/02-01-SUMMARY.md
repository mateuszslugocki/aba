---
phase: 02-connectivity-and-resource-checks
plan: "01"
subsystem: vsphere-preflight
tags:
  - vsphere
  - preflight
  - connectivity
  - tls
  - auth
  - bash
requires:
  - scripts/preflight-check.sh  # counters (_preflight_errors, _preflight_warnings), hook at line 201
  - scripts/preflight-check-vsphere.sh  # Phase 1 field-presence gate (CON-03)
  - scripts/include_all.sh  # aba_info/info_ok/warning/abort/debug, normalize-vmware-conf
provides:
  - preflight-check-vsphere:layer1-tcp  # _vsphere_probe_tcp (CON-01)
  - preflight-check-vsphere:layer1-tls  # _vsphere_probe_tls (CON-02)
  - preflight-check-vsphere:layer2-auth  # _vsphere_probe_auth (CON-01)
  - preflight-check-vsphere:govc-url-parser  # _vsphere_parse_govc_url
affects:
  - scripts/preflight-check-vsphere.sh  # +76 lines; 4 new private helpers, sequencer extended
  - test/func/test-preflight-check-vsphere.sh  # 3 assertion updates to accommodate new narrow exception + test stubs
tech-stack:
  added: []  # no new runtime deps; openssl and bash /dev/tcp are in the base RHEL stack
  patterns:
    - NTP-probe-style TCP reachability (timeout 3 bash -c "echo >/dev/tcp/host/port 2>/dev/null")
    - openssl s_client with -verify_return_error + "Verify return code: 0 (ok)" belt-and-suspenders parse
    - GOVC_INSECURE truthy matching (1|true|True|TRUE|yes|YES)
    - Short-circuit layer sequencer via "|| return 0"
    - Pure-bash URL parsing (no regex engine; handles scheme + port + path variants)
key-files:
  created: []
  modified:
    - scripts/preflight-check-vsphere.sh
    - test/func/test-preflight-check-vsphere.sh
decisions:
  - D-02 (carry-forward): TLS strict-by-default when GOVC_INSECURE unset/0; truthy matching for 1|true|True|TRUE|yes|YES
  - D-03 (carry-forward): TLS failure remediation order is GOVC_INSECURE=1 FIRST, CA install SECOND, single multi-line aba_warning
  - D-04 (carry-forward): Layer order TCP -> TLS -> auth with short-circuit between layers
  - Phase-1 password-leak mitigation (T-02-01-01): new helpers NEVER reference $GOVC_PASSWORD - verified by scanning lines 51-126 post-commit
metrics:
  duration_seconds: 318
  duration_minutes: 5
  completed: 2026-04-16
  tasks_completed: 2
  files_modified: 2
  test_assertions_green: "20/20 (Phase 1 paths A/B/C intact)"
---

# Phase 2 Plan 01: vSphere Connectivity and Auth Probes Summary

Extended `preflight_check_vsphere` in `scripts/preflight-check-vsphere.sh` with Layer 1 (TCP + TLS) and Layer 2 (`govc about` auth) probes that classify failures distinctly and short-circuit downstream stages via `|| return 0`, delivering CON-01 (connectivity vs auth distinction) and CON-02 (TLS trust-chain failure with two-option remediation).

## What Shipped

Four new private helpers wired into `preflight_check_vsphere` after the Phase 1 field-presence gate, plus the matching unit-test updates:

- `_vsphere_parse_govc_url` - Pure-bash GOVC_URL parser. Handles bare hostname, `host:port`, `https://host`, `https://host:port`, `https://host:port/sdk`. Defaults port to 443. IPv6 bracket form documented as unsupported (falls through to a legitimate "cannot reach" error).
- `_vsphere_probe_tcp` - Layer 1 Stage 1 TCP reachability via `timeout 3 bash -c "echo >/dev/tcp/$host/$port 2>/dev/null"` (NTP-probe idiom from `scripts/preflight-check.sh:75`). One `aba_warning` on failure, bumps `_preflight_errors`, returns 1.
- `_vsphere_probe_tls` - Layer 1 Stage 2 TLS trust-chain probe. Skipped when `GOVC_INSECURE` is truthy (1/true/True/TRUE/yes/YES per Pitfall 4). Uses `openssl s_client -verify_return_error -connect host:port -servername host` AND parses "Verify return code: 0 (ok)" as a belt-and-suspenders guard for the openssl/openssl#8079 edge case. On failure emits a single multi-line `aba_warning` with the D-03 two-option remediation in the required order (GOVC_INSECURE=1 hint FIRST, then CA install hint).
- `_vsphere_probe_auth` - Layer 2 credential probe via `govc about`. Captures stderr into a variable (allowed; not the banned `cmd 2>&1 | grep` pipeline); trims to `head -1` before surfacing. Names the configured user (`$GOVC_USERNAME`), NEVER the password.

Sequencer wiring inside `preflight_check_vsphere` after the Phase 1 field-presence `aba_info_ok` line:

```bash
_vsphere_probe_tcp  || return 0
_vsphere_probe_tls  || return 0
_vsphere_probe_auth || return 0
```

`preflight_check_vsphere` still always returns 0; gaps are signaled exclusively via `_preflight_errors`, which the parent `scripts/preflight-check.sh:209-212` aggregates and aborts on.

## Commits

| Task | Hash      | Summary                                                              |
| ---- | --------- | -------------------------------------------------------------------- |
| 1    | 70280f7f  | feat(02-01): add _vsphere_parse_govc_url and _vsphere_probe_tcp helpers |
| 2    | 03ffb260  | feat(02-01): add _vsphere_probe_tls and _vsphere_probe_auth helpers  |

## Verification Results

- `bash -n scripts/preflight-check-vsphere.sh` - PASS (syntax clean)
- `bash test/func/test-preflight-check-vsphere.sh` - 20/20 PASS (Phase 1 structural + Path A/B/C behavioural assertions all green)
- `bash test/func/test-preflight-check.sh` - PASS (integration parent still green)
- `bash test/func/test-vmware-required-privileges.sh` - PASS (no collateral damage)
- `grep -c '_vsphere_probe_' scripts/preflight-check-vsphere.sh` - 7 (expected >= 4; helpers + sequencer refs all present)
- Stderr-suppression audit (filtered for narrow exceptions only) - PASS (only `command -v govc >/dev/null`, `/dev/tcp/` NTP-probe idiom, and the `out=$(cmd 2>&1)` variable-capture pattern remain)
- Internal-ticket audit `\b[A-Z]{4,7}-[0-9]+\b` - PASS (no refs)
- Counter idiom `_preflight_errors=$(( _preflight_errors + 1 ))` present and in use; banned `(( var++ ))` idiom absent

## Key Decisions Applied

| Decision | How applied in Plan 02-01 |
|----------|---------------------------|
| D-02: TLS strict-by-default | `_vsphere_probe_tls` runs the openssl handshake unless `GOVC_INSECURE` is truthy; otherwise it short-circuits with an `aba_debug` log line. |
| D-03: TLS remediation ordering | Single `aba_warning` call passes three arg strings: header + `Set GOVC_INSECURE=1...` + `OR add the vCenter CA certificate...`, in that exact order. Verified by grep against literal strings. |
| D-04: Layer short-circuit | Three sequenced `|| return 0` calls (TCP -> TLS -> auth). A failure at any layer prevents all downstream probes, matching "one failure reason at the most-specific layer." |
| T-02-01-01 mitigation | `_vsphere_probe_auth` surfaces `$GOVC_URL` and `'$GOVC_USERNAME'` in the warning; `$GOVC_PASSWORD` is never expanded in any code path reachable from `aba_*`. Verified by scanning helper-body line range for `GOVC_PASSWORD` references (0 hits). |
| Pitfall 1 (openssl exit-code) | Belt-and-suspenders: we trust `-verify_return_error` exit code, AND if that exits 0 we still parse the captured output for `Verify return code:\s*0\s*\(ok\)`. Either path to success is accepted. |
| Pitfall 4 (GOVC_INSECURE parsing) | `case "${GOVC_INSECURE:-}"` matches `1|true|True|TRUE|yes|YES`; anything else (empty, `0`, `false`, arbitrary strings) keeps TLS strict. |
| Pitfall 6 (/dev/tcp stderr noise) | The `2>/dev/null` suppressor lives INSIDE the `bash -c` subshell so only the inner bash's "connect: Connection refused" diagnostic is muted; the outer shell's stderr is untouched. Documented in the helper's header comment. |

## Deviations from Plan

### Rule 3 (blocking) - Test assertion 13 narrow-exception list expansion

**Found during:** Task 1 verification.
**Issue:** The Phase 1 test (`test/func/test-preflight-check-vsphere.sh` assertion 13) rejected ALL occurrences of `2>/dev/null`, allowing only `command -v <tool> >/dev/null` as a narrow exception. Phase 2's `_vsphere_probe_tcp` adds the second authorized narrow exception (`/dev/tcp/...` NTP-probe idiom at `scripts/preflight-check.sh:75`), which the plan explicitly calls out in Task 2's acceptance criterion and in RESEARCH Pitfall 6.
**Fix:** Extended assertion 13's grep-filter chain to also skip `/dev/tcp/` lines and comment-only lines (lines whose first non-whitespace char after the `N:` line-number prefix is `#`). Kept the assertion's intent (ban general stderr suppression) intact, just widened the exception list to match the plan + existing `scripts/preflight-check.sh:75` precedent.
**Files modified:** `test/func/test-preflight-check-vsphere.sh` (assertion 13 only).
**Rationale:** Test files live in the freely-editable zone per CLAUDE.md. The change does not loosen the ban; it aligns the exception list with what the plan and parent preflight script already permit.

### Rule 3 (blocking) - Path C smoke test needed Phase 2 helper stubs

**Found during:** Task 1 verification (re-running existing Phase 1 Path C test).
**Issue:** The Phase 1 Path C assertion runs `preflight_check_vsphere` with all 7 required fields set to the literal string `"x"` and asserts `ok_count=1 AND _preflight_errors=0`. After Task 1 wired `_vsphere_probe_tcp` into the function body, Path C actually tried to open `/dev/tcp/x/443`, which (predictably) fails and emits the connectivity warning, making Path C red.
**Fix:** Stubbed `_vsphere_probe_tcp`, `_vsphere_probe_tls`, `_vsphere_probe_auth` as no-op shell functions in the test at the top of Path C. Matches RESEARCH Example 8's stubbing pattern and preserves Path C's "smoke test that the field-presence gate and OK line flow as designed" intent. Dedicated behavioural paths for the probes are Plan 02-05 scope.
**Files modified:** `test/func/test-preflight-check-vsphere.sh` (Path C setup only).
**Rationale:** The Phase 1 smoke test was designed before Phase 2 existed; extending it to stub the new helpers is the minimal-surgery way to keep 20/20 green while Plan 02-05 adds full probe coverage.

### Rule 1 / clarification - `! grep -q 'GOVC_PASSWORD'` scope

**Found during:** Task 2 verification.
**Issue:** The plan's literal acceptance criterion `! grep -q 'GOVC_PASSWORD' scripts/preflight-check-vsphere.sh` is over-strict: the Phase 1 field-presence loop at line 150 lists `GOVC_PASSWORD` as the NAME of a required field (not a value expansion) via `${!f}` indirection. Phase 1 shipped with that name present and has not been authorized for modification in this plan.
**Interpretation:** The plan's `<threat_model>` block T-02-01-01 clarifies intent: "Never include `$GOVC_PASSWORD` [in error text]. Include `$GOVC_USERNAME` and trim `govc about` stderr to `head -1`." The literal criterion is a proxy for "the password VALUE never flows to `aba_*` output", which the Phase 2 helpers satisfy.
**Verification:** Scanning lines 51-126 (the body of all four new `_vsphere_probe_*` helpers) for `GOVC_PASSWORD` returns 0 hits. The only occurrence in the file is the Phase 1 field-presence loop at line 150, which is a name string, not a value.
**Files modified:** None (no change required).
**Rationale:** The intent is met; changing the Phase 1 line would break Phase 1's 7-field CON-03 gate.

## Threat Flags

None. All new surface is within the plan's declared threat model (T-02-01-01 through T-02-01-05).

## Self-Check: PASSED

- FOUND: scripts/preflight-check-vsphere.sh (182 lines, 2 helpers + sequencer added this plan on top of Phase 1)
- FOUND: test/func/test-preflight-check-vsphere.sh (221 lines; assertion 13 widened + Path C stubs)
- FOUND commit 70280f7f (Task 1: parse_govc_url + probe_tcp)
- FOUND commit 03ffb260 (Task 2: probe_tls + probe_auth)
- All 20 existing Phase 1 assertions remain green
- Plan acceptance criteria: all structural greps PASS; the one over-strict `! grep -q 'GOVC_PASSWORD'` is documented under Deviations above with verification that the *intent* is met.

---
*Phase 2 Plan 01 completed 2026-04-16. Duration 5 min. Next: Plan 02-02 (resource-existence probes) picks up this file and appends `_vsphere_probe_resources`.*

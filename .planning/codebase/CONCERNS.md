# Codebase Concerns

**Analysis Date:** 2026-04-15

## Run-Once Framework Fragility

**Status:** Multiple race conditions identified and partially mitigated. Core mechanism is flock-solid, but edge cases persist.

**Affected files:**
- `scripts/include_all.sh` (run_once function, lines 1500-2200)
- `scripts/run-once.sh` (wrapper for Makefiles)
- `ai/RUN_ONCE_RELIABILITY.md`, `ai/RUN_ONCE_TTL_REMOVAL.md`, `ai/RUN_ONCE_VALIDATION_RACE_CONDITION_FIX.md`, `ai/RUN_ONCE_WAIT_MODE_RACE_FIX.md`

**Known issues:**

1. **Stale .exit file after crash** - If `run_once` is killed between writing `.exit` and cleanup, stale result files persist. On reboot, PID may be reused for different process. Lock prevents concurrent execution but doesn't validate whether the .pid is still the correct process.

2. **Validation re-read race condition** - Concurrent readers of the same .exit file during validation could see partial writes.

3. **TTL semantics** - `-t <seconds>` expires results by mtime, but file mtime may not reflect actual task completion time (especially with network delays or background processing).

4. **No automatic cleanup** - `~/.aba/runner/` directory accumulates lock/pid/exit/log files indefinitely. No mechanism to clean up completed tasks.

**Mitigations in place:**
- Flock-based mutual exclusion prevents concurrent runs of same task
- Trap handler (`aba_runtime_cleanup`) kills tasks on INT/TERM
- Lock file is released when task completes (FD closes)

**Improvement path:**
- Add PID validation before trusting .exit file (check `kill -0 $pid` to verify process still exists)
- Implement automatic cleanup of completed runner files (e.g. after 24h)
- Add optional result file validation checksum to detect partial writes
- Consider daemon-based runner service instead of shell-script coordination

---

## Error Suppression Sprawl

**Status:** Active cleanup underway. Historic `2>/dev/null` and `&>/dev/null` suppress both errors and legitimate output.

**Affected files:**
- `tui/abatui.sh` (lines 208-220, 801, 988, 1384, 2403, 2683-2689): catalog/version fetches completely silent
- `scripts/include_all.sh` (multiple locations): error message suppression in validation functions
- `scripts/reg-verify.sh` (line 56): certificate validation checks suppressed
- `ai/ERROR_SUPPRESSION_AUDIT.md`, `ai/ERROR_SUPPRESSION_FIXES.md`, `ai/E2E_ERROR_SUPPRESSION_AUDIT.md`

**Impact:**
- Users see no feedback when background catalog downloads fail (lines 2683-2689 in tui/abatui.sh)
- Validation errors in pull-secret and catalog checks hidden from TUI dialogs
- Operator catalog prefetch failures masked completely

**Examples of problematic patterns:**

```bash
# tui/abatui.sh line 801: Catalog download with zero feedback on failure
download_all_catalogs "$version_short" 86400 >/dev/null 2>&1

# tui/abatui.sh lines 2683-2689: Background version fetches completely silent
run_once -i "ocp:stable:latest_version_previous" -- bash -lc '...' >/dev/null 2>&1
```

**Remediation approach:**
1. Separate legitimate "expected failure" suppression (`if [ -f file ] 2>/dev/null || true`) from "we don't care about errors" patterns
2. For critical operations (catalog download, version fetch), log errors to a temp file and surface summary to user
3. Replace blanket `>/dev/null 2>&1` with targeted `2>/dev/null` (keep stdout) for validation functions
4. TUI should query `~/.aba/runner/*/log` files to show error summaries when operations fail silently

---

## Quay Mirror Registry Reliability

**Status:** Two distinct bugs documented and partially mitigated. Both affect reliability of mirrored deployments.

**Affected files:**
- `scripts/reg-install-quay.sh`
- `scripts/reg-install-docker.sh`
- `bundles/v2/common.sh` (cleanup_orphaned_quay_services)
- `ai/QUAY_STALE_SERVICE_BUG.md`, `ai/QUAY_PASTA_HAIRPIN.md`

**Issue 1: Stale Systemd Service from Version Upgrade**

When mirror-registry upgrades from PostgreSQL-backed (Dec 2024) to SQLite-backed (Jan 2026), the old `quay-postgres.service` unit file is abandoned. On next install, it crash-loops (1700+ restarts observed) and disrupts pasta networking, causing external connections to fail while localhost works.

- **Symptom:** Connection reset by peer on host's own external IP/hostname, but `localhost:8443` works
- **Trigger:** Multiple install/uninstall cycles with different mirror-registry versions
- **Mitigation:** `cleanup_orphaned_quay_services()` in `bundles/v2/common.sh` removes all stale `quay-*` services after uninstall
- **Gap:** Mitigation only in bundle build, not in main ABA install path. Production deployments using `aba -d mirror install` directly are vulnerable

**Issue 2: Pasta Networking Hairpin Route Missing**

After registry installation on a host with no default route (e.g. internet disconnected for bundle building), Quay pod cannot handle "hairpin" connections (host connecting to its own external IP). Pasta requires a default route at pod-creation time.

- **Symptom:** `curl bastion.example.com:8443` fails from the host, but works from external machines
- **Trigger:** Installation after `int_down` (e.g. in bundle phases 05-go-offline.sh → 06b-install-registry.sh)
- **Mitigation:** Add temporary default route via internal gateway before install, remove after
- **Location:** `bundles/v2/phases/06b-install-registry.sh`
- **Gap:** Not applied in standalone `aba -d mirror install` flow

**Remediation path:**
1. Add `cleanup_orphaned_quay_services()` to main `scripts/reg-install-quay.sh` before install
2. Add temporary default route logic to `scripts/reg-install-quay.sh` and `scripts/reg-install-docker.sh` (detect offline state via `ip route | grep default`)
3. Document in `Troubleshooting.md` the symptoms and manual remediation steps
4. Consider adding pre-flight check to `preflight-check.sh` that warns if default route will be removed

---

## OC-Mirror v2 Idiosyncrasies and Limitations

**Status:** Documented but requires explicit workarounds in ABA code. Bitmask exit codes available in OCP 4.18+ but not fully utilized.

**Affected files:**
- `scripts/reg-save.sh` (line ~150: hardcoded `--since 2025-01-01`)
- `scripts/reg-sync.sh` (oc-mirror invocation)
- `scripts/reg-load.sh` (oc-mirror invocation)
- `ai/OC-MIRROR-INTERNALS.md` (comprehensive source-code analysis)

**Issue 1: --since Hardcoded, Not User-Configurable**

Current: `--since 2025-01-01` is hardcoded in `reg-save.sh`. This forces complete archives (all blobs) but is not documented and not configurable.

- **Impact:** Users can't choose differential archives (smaller, but requires all previous loads) vs complete archives (larger, works on fresh registry)
- **Gap:** No config option to disable `--since` for incremental workflows
- **Fix:** Make `OC_MIRROR_SINCE` configurable in `~/.aba/config`, default empty (differential mode)

**Issue 2: Error File Detection Now Redundant**

All three scripts check for `mirroring_errors_*.txt` files after oc-mirror completes. In oc-mirror v2 (OCP 4.18+), bitmask exit codes are the reliable signal.

- **Refactoring:** Remove error file checks, rely solely on exit codes
- **Improvement:** Decode bitmask to identify which category failed (release, operator, additional, helm)
- **Benefit:** Clearer user feedback on what failed and why
- **Size:** ~70 lines of similar retry logic in three files (210 total) can be extracted to shared function

**Issue 3: Imageset-config Must Always List All Operators**

The `imageset-config.yaml` is the complete truth per `save/load` round. Removing an operator from config drops it from OperatorHub, even if its images remain in registry.

- **Impact:** Users can't incrementally add operators across multiple save/load cycles
- **User confusion:** "I loaded operator X in round 1, why did it disappear in round 2?" (answer: removed from config in round 2)
- **Mitigation needed:** Document clearly in config template and `--help` text

**Remediation approach:**
1. Extract shared `_run_oc_mirror_with_retry()` function to `scripts/include_all.sh` (avoid 210 lines of copy-paste)
2. Add bitmask decoding: exit 1 (generic/pre-batch), 2 (release), 4 (operator), 8 (additional), 16 (helm), combinations via OR
3. Log decoded exit code + history of exit codes across retries so user can see progress (or lack thereof)
4. Remove `mirroring_errors_*.txt` detection
5. Add `OC_MIRROR_SINCE` to `~/.aba/config` template with detailed comments

---

## Catalog Download Reliability and TUI Flakiness

**Status:** Multiple race conditions and design issues cause TUI timeouts and test flakiness.

**Affected files:**
- `scripts/prefetch-catalogs.sh`
- `scripts/download-and-wait-catalogs.sh`
- `scripts/download-catalogs-start.sh`
- `tui/abatui.sh` (lines 801, 1384, 2683-2689)
- `ai/BACKLOG.md` (Enhancement: TUI early catalog prefetch)

**Issue 1: I/O Contention When Prefetching Wrong Version**

Prefetch-after-pull-secret was proposed to avoid user waiting at operators screen. But at pull-secret time, no version is selected yet. Prefetch downloads (e.g.) `stable:latest`, but user selects `fast` — wrong catalogs are downloaded and discarded, consuming bandwidth and saturating I/O.

- **Symptom:** Operators screen takes >240s to load instead of normal <120s (3 concurrent podman pulls for wrong version + 3 for correct version = 6 total)
- **Impact:** TUI tests timeout with 2x scaling
- **Trigger:** Backlog item proposes prefetch after pull-secret, but version mismatch makes it worse

**Issue 2: Parallel Catalog Download Stalling**

With `CATALOG_MAX_PARALLEL=3` (default), concurrent `podman pull` operations for multiple catalogs can stall or timeout individually, and the wait logic doesn't always detect completion.

- **Test impact:** `suite-tui-*.sh` tests fail intermittently with "Timeout waiting for catalogs"
- **Root cause:** Wait-for-all-catalogs uses file existence checks, not actual download status checks
- **Location:** `scripts/download-and-wait-catalogs.sh`

**Issue 3: Catalog Download Failures Hidden in TUI**

`tui/abatui.sh` line 1384: `wait_for_all_catalogs` wrapped in `>/dev/null 2>&1`. If catalogs fail to download, TUI shows no error and operators screen never appears.

**Remediation path:**
1. Move prefetch to **after version selection** (backlog approach B), not after pull-secret
2. Add per-catalog timeout and retry logic, not just global wait timeout
3. Expose catalog download errors to TUI dialog instead of silent failure
4. Consider sequential download with progress bar instead of 3-parallel for first catalog set
5. Add `CATALOG_MAX_PARALLEL` injection to TUI test helper (prototyped in BACKLOG.md)

---

## Symlink Dependency Chain Fragility

**Status:** Known issue. `make -s init` guards fix immediate symptom but architecture is fragile.

**Affected files:**
- `scripts/aba.sh` (lines 1031-1080: VM lifecycle commands guarded with `make -s init`)
- `scripts/vmw-delete.sh`, `scripts/vmw-start.sh`, `scripts/vmw-stop.sh`, `scripts/vmw-kill.sh` (all use `source scripts/include_all.sh`)
- `templates/Makefile.cluster` (creates symlinks: `scripts -> ../scripts`, `templates -> ../templates`, `Makefile -> ../templates/Makefile.cluster`)
- `ai/BACKLOG.md` (Architecture: Review symlink dependency and consider /opt/aba)

**Problem:**

Cluster directories (e.g. `e2e-sno1/`) rely on symlinks to access shared code. When symlinks are removed (e.g. by `aba clean`), scripts fail with "No such file or directory". Workaround is `make -s init` before every lifecycle command, but this is a band-aid.

**Vulnerabilities:**

1. **`aba clean` breaks VM commands** - Removes `scripts/`, `templates/`, `.init` symlinks. Subsequent `aba -d cluster start` fails until `make init` recreates them.

2. **Relative path brittleness** - Symlinks use `../scripts` which only works if cluster is one level below ABA root. Nested or relocated directories break.

3. **Bundle/tar portability** - Archives created without `-h` flag have broken symlinks. Users reporting "scripts/ not found" after extracting bundles.

4. **Contributor confusion** - New developers don't expect `scripts/` in cluster dir to be a symlink.

**Long-term alternative proposed:**

Install static files to `/opt/aba/` (requires versioning strategy, backward-compatibility path, install step changes). Cluster directories would reference `$ABA_SCRIPTS` or `/opt/aba/scripts/` instead of symlinks. Cleaner but touches most Makefiles and scripts.

**Current mitigation:**

Added `make -s init` guard before each VM lifecycle call in `aba.sh`. Prevents most failures but doesn't address the design fragility.

**Audit needed:**

1. Search for all `source scripts/include_all.sh` and `source templates/` patterns (already partially done in BACKLOG.md lines 130-134)
2. Review `|| exit 0` patterns on `start)`, `kill|poweroff)` — currently mask real errors
3. Decide: accept symlink approach with better guards, or plan `/opt/aba` migration

---

## E2E Test Coverage Gaps and Fragility

**Status:** Multiple coverage gaps and brittle test infrastructure. E2E failures often mask real ABA bugs.

**Affected files:**
- `test/e2e/` (entire E2E framework)
- `ai/E2E_COVERAGE_GAPS.md`, `ai/E2E_TEST_AUDIT.md`, `ai/BUGS-FROM-E2E-20260328.md`, `ai/E2E_PROPOSED_CORE_CHANGES.md`

**Issue 1: No ESXi-Direct E2E Coverage**

All E2E suites hardcode `--platform vmw` (vCenter mode, `VC=1`). No coverage for ESXi-direct installs (`VC=` empty), which is a supported ABA deployment mode.

- **Risk:** ESXi-direct bugs (e.g. the doubled `resourcePool` path bug) go undetected until user reports
- **Fix:** Add one pool with ESXi-only `vmware.conf` and run suites against it
- **Backlog:** BACKLOG.md item "Enhancement: E2E tests MUST support ESXi-direct API"

**Issue 2: IP Conflict False Positive on Existing Clusters**

`preflight_check_ip_conflicts()` in `scripts/preflight-check.sh` treats every responding IP as a conflict. When regenerating ISO for an existing cluster (day-2 reconfig, version upgrade, or E2E test-install), running nodes respond to arping and cause hard failure.

- **Symptom:** E2E bundle-maker test fails at ISO generation
- **Workaround:** User can use `aba --verify conf` to skip, but this also skips DNS/NTP checks
- **Fix options:** 
  - A) Skip IP checks if cluster already exists (marker file or VM check)
  - B) Downgrade to warning for IPs matching our cluster's expected MAC prefix
  - C) Query cluster API to verify ownership

**Issue 3: E2E Cleanup Bypasses ABA Uninstall with Brute-Force `rm -rf`**

`_cleanup_dis_aba()` in `test/e2e/runner.sh` line 383 uses `sudo rm -rf ~/quay-install` to brute-force remove registry data, bypassing `aba uninstall`. This hides bugs in ABA's cleanup code and doesn't work (Quay SQLite DB has attributes that survive `rm -rf`).

- **Impact:** Tests can't properly clean up between runs. Mirror-sync suite fails 3/3 on "save and load (should reinstall registry)" with PermissionError on stale SQLite file.
- **Design violation:** Tests should use `aba uninstall` (eat their own dog food), not bypass it
- **Fix:** Rely on `aba -y -d mirror uninstall`, fail the test if it doesn't clean up properly (that's a real ABA bug)

**Issue 4: Quay Infra Pod Not Excluded in Registry-Removed Test**

`suite-mirror-sync.sh` line 166-167 tests that Quay registry is removed by checking `podman ps` and filtering out lines with "quay". But the Quay infra container has a hash name (e.g. `66b889d3339e-infra`) that doesn't match "quay", so it's not excluded.

- **Impact:** Test fails even though Docker registry was properly removed
- **Fix:** Also exclude lines containing "infra" or use a more specific filter (check for `registry` container name only)

**Issue 5: Dispatcher Can Override Manual Suite Assignments**

When `run.sh restart --suite X --pool N` is used while dispatcher is running, the dispatcher may reassign the pool to a different suite from its queue.

- **Impact:** Operator trying to manually fix a failing suite accidentally has work reassigned
- **Fix:** Have restart command register with dispatcher, or document that manual restarts require stopping dispatcher first

**Covered extensively in:**
- `ai/BUGS-FROM-E2E-20260328.md` (bugs 1-4)
- `ai/E2E_COVERAGE_GAPS.md` (coverage analysis)
- `ai/E2E_PROPOSED_CORE_CHANGES.md` (framework fixes and proposed ABA changes)

---

## Security and Trust-Store Surface

**Status:** Multiple trust-store and signature verification paths. Sigstore option-B support added recently.

**Affected files:**
- `scripts/verify-release-image.sh` (lines 40-56: skopeo inspect with registry auth, line 62: HACK for IDMS extraction)
- `templates/aba-sigstore-config.yaml` (sigstore configuration)
- `scripts/day2.sh` (applies additionalTrustedCA patch)
- `ai/sigstore-test-report.md` (sigstore option B test results)

**Issue 1: Release Image Verification Can Fail Silently in Offline Mode**

`verify-release-image.sh` lines 37-56: When `verify_conf=conf` or `verify_conf=off`, skips connectivity check completely. If registry is actually unreachable, user gets no warning and install will fail later when trying to pull release image.

- **Risk:** User doesn't discover registry auth issues until cluster bootstrap begins
- **Impact:** No feedback at ABA configuration time, only at install time (hours wasted)
- **Improvement:** Even in offline mode, attempt the connectivity check but show warning instead of hard failure

**Issue 2: IDMS Extraction Hack**

Line 62: `# HACK` — comment explains this is a workaround for oc versions before 4.14 that don't support IDMS. The code uses `|| true` to ignore errors, hiding real failures.

- **Cleanup needed:** Require oc 4.14+ or add proper version check
- **Current**: Masked error makes debugging harder (silent failure)

**Issue 3: Pull-Secret Merge Logic Vulnerable**

The pull-secret merge for adding mirror registry credentials is not documented. If multiple operators merge pull-secrets or credentials are invalid JSON, behavior is undefined.

- **Risk:** Silent credential loss during mirror registration
- **Location:** `scripts/reg-register.sh` (pull-secret handling)
- **Audit needed:** Verify merge logic is safe, has error checking, and is tested

**Issue 4: Sigstore Verification Coverage**

Sigstore option B (keyless signatures) added recently. Needs testing on:
- OCP 4.21+ (sigstore enabled)
- OCP 4.20 (backport)
- Mirrored + offline scenarios (does sigstore work with mirrored release images?)
- Key rotation scenarios

**Remediation path:**
1. Add proper version check for oc (require 4.14+), remove `|| true` hack
2. In offline/conf mode, still attempt connectivity check but show warning, don't hard-fail
3. Audit pull-secret merge logic in `reg-register.sh`, add JSON validation
4. Add sigstore E2E test coverage (option A vs B, mirrored vs connected)
5. Document trust-store chain: registry CA → pull-secret → sigstore verification
6. Consider adding `aba verify sigstore` command

---

## Pending Architectural Migrations

**Status:** Multiple design changes proposed but not yet implemented. Block new feature work and create tech debt.

**Items from BACKLOG.md and other ai/ docs:**

1. **MTU 9000 Migration** (`ai/PLAN-MTU-9000-MIGRATION.md`)
   - Change cluster networking to use 9000-byte MTU instead of 1500
   - Affects all cluster creation, day-2 reconfiguration
   - Impacts: VLAN bonding, network plugin config, bootstrap networking

2. **Output Tidying** (`ai/PLAN-OUTPUT-TIDYING.md`)
   - Redundant output from `aba startup` (nodes listed 4 times)
   - VCenter/ESXi paths should show VM name only, not full path
   - User-facing output cleanup

3. **Config-Time Variables** (`ai/PLAN-CONFIG-TIME-VARS.md`)
   - Support setting variables at cluster creation time without cluster.conf file edits
   - Currently: `aba cluster -n X -I proxy` silently ignored if cluster.conf exists (BACKLOG.md bug: CLI flags silently ignored)
   - Fix: Selective override of config values from CLI flags

4. **Data Directory Consolidation** (`ai/DATA_DIR_CONSOLIDATION.md`)
   - Consolidate `data/`, `data-mirror/`, `data-<name>/` into single `data/` directory
   - Requires Makefile refactoring

5. **Docker Registry First-Class Support** (`ai/DESIGN-docker-registry-first-class.design`)
   - Currently: Quay is first-class, Docker registry is second-class
   - Proposal: Full parity in installer, day-2 config, operators support
   - Impact: Multiple ABA paths need Docker-specific handling

6. **Makefile Simplification** (`ai/MAKEFILE_SIMPLIFICATION.md`)
   - Consolidate cluster Makefile targets and phases
   - Reduce duplication across `Makefile.cluster`, `Makefile.mirror`, `Makefile.bundle`

7. **Install Cleanup Strategy** (`ai/INSTALL_CLEANUP_STRATEGY.md`)
   - `aba clean` currently removes too much (symlinks, config). Should preserve cluster.conf
   - Better cleanup semantics needed

**Impact:**

These migrations will touch significant portions of the codebase. Starting any major feature work while these are pending risks conflicts and rework. Priority should be established and migrations should be scheduled.

---

## CLI Flag Processing Dual-Path Bug

**Status:** Documented and analyzed. Multi-level interaction bug between aba.sh flag processing and cluster.conf creation.

**Affected files:**
- `scripts/aba.sh` (lines 58-104: first pass does `cd` for `-d`, lines ~530-900: second pass processes all other flags)
- `scripts/setup-cluster.sh` (line 47: `create_cluster_cmd` omits values, lines 28-30: commented-out full overwrite code)
- `scripts/create-cluster-conf.sh` (line 21: early exit on existing cluster.conf)
- `templates/Makefile.cluster` (line 115: passes values to setup-cluster.sh)

**Problem:**

When using `aba cluster -n mycluster -I proxy`, CLI flags are silently ignored if `cluster.conf` already exists.

| Approach | Works? | Why |
|----------|--------|-----|
| `aba -d mycluster -I proxy install` | ✅ YES | `-d` does `cd` early, flag handlers find `cluster.conf` and apply |
| `aba cluster -n mycluster -I proxy` | ❌ NO | No `-d`, CWD stays in ABA root, flag handler checks `[ -f cluster.conf ]` (fails), adds to `$BUILD_COMMAND`, but `create-cluster-conf.sh` exits early if file exists |

**Affected flags (14 total):**
All CLI-passable values: `--api-vip`, `--ingress-vip`, `--master-cpu`, `--master-memory`, `--worker-cpu`, `--worker-memory`, `--starting-ip`, `--data-disk`, `--int-connection`, `--num-workers`, `--num-masters`, `--vlan`, `--ssh-key`, `--proxy`, `--no-proxy`

**Design question:**

Should `aba cluster` overwrite existing `cluster.conf`? Three options considered:

- **A) Full overwrite** - Destroys manual edits (rejected, code commented out)
- **B) Selective override** - Keep existing file, apply only explicit CLI flags (recommended)
- **C) Warn and skip** - Print warning, don't apply flag (unhelpful)

**Recommendation:** Option B (selective override), matches how `aba.conf` and `mirror.conf` already work.

**Fix options:**

1. In `setup-cluster.sh` after `create_cluster_cmd`, apply non-empty CLI values via `replace-value-conf`
2. Rework `create-cluster-conf.sh` to merge CLI values into existing file instead of exiting
3. In `aba.sh`, detect cluster.conf existence and cd early (elegant but requires careful ordering)

---

## Catalog Download and Podman Extraction Reliability

**Status:** Multiple fallback mechanisms in place but integration is fragile.

**Affected files:**
- `scripts/prefetch-catalogs.sh`
- `scripts/download-catalogs-start.sh`
- `scripts/download-and-wait-catalogs.sh`
- `ai/PODMAN_CATALOG_EXTRACTION.md`

**Issue 1: Podman Extraction Is Implicit Fallback**

When `oc-mirror` is unavailable (e.g. user hasn't run `aba -d cli install`), catalog downloads fall back to `podman` extraction from images. This fallback is not user-facing and not explicitly tested.

- **Risk:** If podman breaks or is unavailable, user gets confusing errors instead of clear "oc-mirror not available, trying podman" message
- **Coverage:** Fallback tested in old E2E, not in new suite framework
- **Improvement:** Make fallback explicit, add debug logging, consider warning user

**Issue 2: Parallel Download Coordination**

Multiple scripts coordinate catalog downloads via file-based signaling. Race condition window exists between download start and `.ready` file creation.

---

## Static Analysis: Code Hotspots (FIXME/TODO Count)

**Highest concentration by file:**

| File | Count | Notes |
|------|-------|-------|
| `scripts/aba.sh` | 16 | Design issues (option flags, light mode), symlink dependency (line 117) |
| `test/test1-basic-sync-test-and-save-load-test.sh` | 11 | Old test file, mostly cleanup/deprecation notes |
| `scripts/include_all.sh` | 5 | Error handling simplification, wait_show timer issue |
| `scripts/cluster-graceful-shutdown.sh` | 3 | Certificate expiration display (commented out FIXME) |
| `test/test5-airgapped-install-local-reg.sh` | 5 | RPM pre-existence test, config override patterns |
| `scripts/generate-image.sh` | 4 | PXE support, cluster config output |
| `scripts/vmw-upload.sh` | 2 | govc log handling, error parsing |
| `scripts/day2.sh` | 1 | Mirror availability check scoping |
| `scripts/reg-load.sh` | 1 | Trust store CA installation ordering |
| `scripts/verify-release-image.sh` | 1 | IDMS extraction hack |

**Patterns in FIXME/TODO comments:**

1. **Design uncertainty** - "Should we do X or Y?" (lines in aba.sh about light mode, option flags)
2. **Deprecation markers** - "Remove after X is done" (backup.sh, test files)
3. **Test infrastructure notes** - "FIXME: This test doesn't actually test X" (test files)
4. **Error handling simplification** - "FIXME: Simplify this!" (include_all.sh line 736)
5. **Async coordination** - "Wait for background task with proper timeout" (aba_wait_show timer freeze)

**Hottest file:** `scripts/aba.sh` (16 FIXME/TODO comments + symlink dependency issue on line 117 = 17 open design questions)

---

## Missing Critical Features from Backlog

**Status:** Actively tracked in BACKLOG.md. Impact blocks user workflows.

1. **Shutdown Timeout and Failure Handling** (URGENT hotfix on main)
   - VM power-off timeout too short (5 min → need 40 min)
   - Timeout should trigger `aba_abort`, not just warning
   - Fix exists on dev, needs cherry-pick to main

2. **Mirror Registry Identity Change Warning**
   - When `reg_host`, `reg_port`, `reg_vendor` change in mirror.conf after install, should warn user
   - Currently silent, can confuse users during day-2 reconfig

3. **TUI Early Catalog Prefetch** (partially analyzed, blocked by design issues)
   - Download operator catalogs after pull-secret to avoid wait at operators screen
   - Version mismatch and I/O contention make this non-trivial

4. **Dot-Waiting Loop Replacement**
   - Replace hand-rolled `echo -n .` loops with `aba_wait_show()`
   - Affects: day2-config-osus.sh (2 locations), cluster-startup.sh, day2-osus.sh
   - Also fix: cluster-startup.sh curl checks dumping raw HTTP 503 headers (noted as fixed with `>/dev/null`)

5. **oauth-proxy Imagestream Deletion Still Needed?**
   - After `additionalTrustedCA` patch, is imagestream deletion still necessary on OCP 4.14+?
   - 15 retries with backoff are time-consuming if unnecessary
   - Needs testing on fresh 4.14+ mirrored install

6. **Bare `sudo` in SSH Commands**
   - `reg-uninstall.sh` line 127 and `reg-uninstall-remote.sh` lines 81-82 use bare `sudo` in SSH commands
   - Remote host may not have sudo installed
   - Should detect availability first: `$_ssh "which sudo >/dev/null && echo sudo"`

7. **aba_wait_show Timer Freezes**
   - Timer display freezes while polled command executes
   - Proposal: run command in background, update timer every second
   - Improves UX for long-running polls (e.g. `curl --connect-timeout 10`)

---

## File Deletion and Symlink Incident

**Status:** Investigated and documented. Root cause was manual git state manipulation.

**Affected file:**
- `ai/GIT_FILE_DELETION_INVESTIGATION.md`

**What happened:**

User deleted script files from git (`git rm`) without understanding the implications. ABA code continued to reference the deleted files, causing failures.

**Resolution:**

Files were restored. Investigation documented which scripts are entry points and must never be deleted.

**Lesson:**

Better documentation of script dependencies and entry points needed. Consider linting to catch references to deleted files at build/test time.

---

## VM Lifecycle Script Audit Needed

**Status:** Partial completion. `aba clean` removal of symlinks fixed, but broader audit incomplete.

**Affected files:**
- `scripts/vmw-delete.sh`, `scripts/vmw-start.sh`, `scripts/vmw-stop.sh`, `scripts/vmw-kill.sh`, `scripts/vmw-refresh.sh` (all use `source scripts/include_all.sh`)
- `scripts/aba.sh` (lines 1031-1080: VM lifecycle commands now guarded with `make -s init`)
- `templates/Makefile.cluster` (creates symlinks)
- `ai/BACKLOG.md` (Audit section, lines 119-140)

**Remaining audit items:**

1. Search codebase for all `source scripts/include_all.sh` and `source templates/` callers — are there scripts other than VM lifecycle that assume symlinks?
2. Review `|| exit 0` on lifecycle commands — currently mask real errors, should be more targeted
3. Review `|| echo "No vm(s)."` on `aba ls` — is this the right fallback?
4. Decide: accept symlink approach with better guards, or plan `/opt/aba` migration

---

## Summary: Priority-Ordered Concerns

**CRITICAL (blocks major workflows):**
1. **run_once race conditions** — Stale .exit files, validation re-reads (mitigations in place, needs hardening)
2. **Quay registry reliability** — Stale systemd services, pasta hairpin (affects all mirrored deployments)
3. **E2E coverage gaps** — No ESXi-direct testing, IP conflict false positives (hides real bugs)
4. **CLI flag silently ignored** — `aba cluster -n X -I proxy` doesn't apply proxy flag (user confusion)

**HIGH (affects features):**
5. **Error suppression sprawl** — TUI silent failures, version fetches with zero feedback
6. **OC-Mirror --since hardcoded** — Not configurable, no choice for differential archives
7. **E2E cleanup bypasses aba uninstall** — Hides ABA bugs, causes test failures

**MEDIUM (technical debt):**
8. **Symlink dependency fragility** — `aba clean` breaks VM commands, portable bundle issues
9. **Catalog download reliability** — TUI I/O contention, version mismatch on prefetch, stalling
10. **FIXME/TODO code hotspot** — aba.sh has 16+ open design questions

**LOW (nice to have):**
11. **Shutdown VM timeout** — Already has fix, needs cherry-pick to main
12. **Output tidying** — Redundant node listings, path display
13. **VM notes/descriptions** — Better annotations for VMware/KVM

---

*Concerns audit: 2026-04-15*

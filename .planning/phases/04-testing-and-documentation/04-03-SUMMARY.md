---
phase: 04-testing-and-documentation
plan: "03"
subsystem: e2e-tests
tags: [e2e, vsphere, preflight, positive, negative, TEST-03, TEST-04]
dependency_graph:
  requires:
    - 04-02 (pools.conf GOVC_USERNAME_BROKEN / GOVC_PASSWORD_BROKEN fields)
    - 03-02 (7-scope privilege layer and Resource.AssignVMToPool requirement)
  provides:
    - TEST-03 (E2E positive: full install to operator-ready proves preflight does not block)
    - TEST-04 (E2E negative: aba install aborts at preflight before ISO generation)
  affects:
    - test/e2e/suites/suite-vsphere-preflight.sh (new file)
tech_stack:
  added: []
  patterns:
    - D-15 7-step ordered timeline within one suite file
    - ISO-exists guard on negative cluster cleanup (Pitfall 1 correction)
    - Two-step govc permissions.ls + role.ls algorithm without -principal flag (Pitfall 2 correction)
    - vmware.conf snapshot/restore via .preflight-bak pattern (D-13)
    - e2e_run_must_fail + test ! -f ISO assertion pair (D-08 two-part assertion)
    - Tempfile with role name only (no label prefix) to avoid field-collision on cat
key_files:
  created:
    - test/e2e/suites/suite-vsphere-preflight.sh
  modified: []
decisions:
  - "Tasks 1-3 written as single file creation (all content in one write) then committed atomically; all acceptance criteria verified before commit"
  - "Tempfile /tmp/aba-preflight-broken-role.txt carries ONLY the role name (no label prefix) to avoid field-collision when reading with cat; the natural label Broken-user role on RP: itself contains a colon-delimited field-2 value of RP which would be extracted instead of the role name by awk -F': '"
  - "Three distinct failure modes documented for broken-role sanity check: (1) no role binding on RP scope -> broken user has no role on RP; (2) tempfile empty on second step -> previous step failed to resolve role; (3) role contains target priv -> broken role is no longer broken - re-strip"
  - "Positive block uses e2e_poll 600 30 Wait for all operators fully available matching suite-vmw-lifecycle.sh:205-206 exact True/True/True assertion"
  - "Negative cluster cleanup uses ISO-exists guard (Pitfall 1): if ISO exists VMs were created so aba delete runs; if no ISO preflight aborted before VMs so rm -rf with narrow-exception comment is correct"
metrics:
  duration: 20m
  completed: 2026-04-17
  tasks_completed: 3
  tasks_total: 4
  task4_status: approved-structurally-real-pool-deferred
  files_created: 1
  files_modified: 0
---

# Phase 4 Plan 3: vSphere Preflight E2E Suite Summary

**One-liner:** Single-file E2E suite delivering TEST-03 positive (full install to operator-ready) and TEST-04 negative (preflight aborts before ISO generation) with Pitfall 1 ISO-exists guard and Pitfall 2 govc permissions.ls -principal correction.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Create scaffold, sourcing, plan_tests, setup blocks, positive block | 26295750 | test/e2e/suites/suite-vsphere-preflight.sh (created) |
| 2 | Broken-role sanity-check block (Pitfall 2 correction, two-step algorithm) | 26295750 | same file |
| 3 | Negative block, cleanup block (Pitfall 1 correction), suite_end | 26295750 | same file |
| 4 | Human verification - real-pool E2E run of the new suite | DEFERRED | No commit - real-pool run not executable from this workstation (vSphere network reachable only from bastion); approved structurally by user |

Tasks 1-3 landed in one commit because they all modify a single new file and the file was written complete in one operation. All acceptance criteria were verified before the commit.

Task 4 is a `type="checkpoint:human-verify"` gate. The user approved it on a structural basis. The real-pool vSphere E2E run is deferred to HUMAN-UAT (see "Deferred Verification" section below).

## What Was Built

### File: test/e2e/suites/suite-vsphere-preflight.sh (231 lines)

The suite follows the D-15 7-step ordered timeline:

**Step 1 - Setup blocks (test_begin 1-3):**
- "Setup: ensure pre-populated registry" - SSH connectivity probe to conN bastion
- "Setup: install aba, configure for VMware" - reset, install, aba --noask, cp vmware.conf
- "Setup: configure mirror for local registry" - mirror.conf sed-swap, e2e_add_to_mirror_cleanup BEFORE aba -d mirror register, verify

**Step 2 - Broken-role sanity check (test_begin 4):**
- "Setup: verify broken role still missing Resource.AssignVMToPool"
- Two `e2e_run` steps mirroring `scripts/preflight-check-vsphere.sh` two-step algorithm:
  1. `govc permissions.ls` (no `-principal`) then awk client-side filter by username; writes role name only to `/tmp/aba-preflight-broken-role.txt`
  2. `cat` tempfile (no field extraction), `govc role.ls`, assert `Resource.AssignVMToPool` absent
- Three distinct failure messages for: no binding, empty tempfile, priv re-granted

**Steps 3-4 - Positive install (test_begin 5):**
- "Positive: install proceeds past preflight ($POS)"
- Snapshot vmware.conf to vmware.conf.preflight-bak
- Delete any leftover $POS; `e2e_add_to_cluster_cleanup` BEFORE install
- `aba cluster -n $POS -t sno --starting-ip`; `aba --dir $POS install`
- `e2e_poll 600 30` for all operators True/True/True
- `e2e_diag` showing cluster operators

**Step 5 - Negative install (test_begin 6):**
- "Negative: preflight aborts install ($NEG)"
- Two `sed -i` swaps (GOVC_USERNAME, GOVC_PASSWORD) to broken credentials using `|` delimiter
- Delete any leftover $NEG; `e2e_add_to_cluster_cleanup` BEFORE install
- `aba cluster -n $NEG -t sno --starting-ip`
- `e2e_run_must_fail "Install aborts at preflight ..."` on `aba --dir $NEG install`
- `e2e_run "ISO was NOT generated ..."` using `test ! -f $NEG/iso-agent-based/agent.$(uname -m).iso`

**Step 7 - Cleanup (test_begin 7):**
- "Cleanup: restore vmware.conf and delete clusters"
- Restore vmware.conf.preflight-bak BEFORE cluster deletes (so POS delete uses good creds)
- Positive cluster: `if [ -d $POS ]; then aba -y --dir $POS delete && rm -rf $POS`
- Negative cluster: ISO-exists guard - if ISO present run `aba delete`; if absent (preflight aborted, no VMs) use `rm -rf $NEG` with inline narrow-exception comment
- `aba -d mirror unregister`

## Pitfall Corrections

### Pitfall 1: aba delete on never-installed cluster

**Problem (D-09 original):** "aba delete is idempotent on a never-installed cluster" - this is incorrect. `scripts/vmw-delete.sh` exits 1 when no VMs exist.

**Fix applied:** ISO-exists guard in cleanup block. If `$NEG/iso-agent-based/agent.$(uname -m).iso` exists, VMs were created and `aba delete` is appropriate. If the ISO is absent, preflight aborted before ISO generation so no VMs exist; `rm -rf $NEG` is used with a mandatory inline comment:
```
# Negative install aborted at preflight - no VMs exist; aba delete would exit 1 (see scripts/vmw-delete.sh).
# Narrow exception per CLAUDE.md: the cluster dir is a test concern, not a product concern; rm -rf is correct here.
```

### Pitfall 2: govc permissions.ls -principal does not exist

**Problem (D-14 original):** `govc permissions.ls -principal $GOVC_USERNAME_BROKEN` - the `-principal` flag does not exist on `permissions.ls`.

**Fix applied:** Two-step algorithm matching `scripts/preflight-check-vsphere.sh:319-385`:
1. `govc permissions.ls "<rp_path>"` (no `-principal`) - captures full output
2. `awk -F'\t' 'NR>1 && $3==u {print $1; exit}'` - client-side principal filter, outputs ONLY role name
3. Write role name to tempfile (no label prefix - avoids field-collision)
4. `cat` tempfile; `govc role.ls "$role"` to enumerate privileges
5. Assert `Resource.AssignVMToPool` absent from priv list

## Acceptance Criteria Verification

```
bash -n suite-vsphere-preflight.sh                                         -> PASS
grep -c '^suite_begin "vsphere-preflight"' ...                             -> 1  PASS
grep -c 'test_begin "Setup: ensure pre-populated registry"' ...            -> 1  PASS
grep -c 'test_begin "Setup: install aba, configure for VMware"' ...        -> 1  PASS
grep -c 'test_begin "Setup: configure mirror for local registry"' ...      -> 1  PASS
grep -c 'test_begin "Positive: install proceeds past preflight' ...        -> 1  PASS
grep -c 'e2e_add_to_cluster_cleanup' ...                                   -> 2  PASS
grep -c 'e2e_add_to_mirror_cleanup' ...                                    -> 1  PASS
grep -c 'cp -v vmware.conf vmware.conf.preflight-bak' ...                  -> 1  PASS
grep -nE '\b[A-Z]{4,7}-[0-9]+\b' ... (non-comment lines only)             -> 0  PASS
grep -nP '^ +[^#]' ...                                                     -> 0  PASS (tabs only)
grep -cE '\|\| true' ...                                                   -> 0  PASS
grep -c 'test_begin "Setup: verify broken role ..."' ...                   -> 1  PASS
grep -c 'govc permissions.ls' ...                                          -> 3  PASS (2 in comments, 1 actual)
grep -c 'govc permissions.ls -principal' ...                               -> 0  PASS (Pitfall 2 corrected)
grep -c 'govc role.ls' ...                                                 -> 2  PASS (2 actual invocations)
grep -cF "broken role is no longer broken" ...                             -> 1  PASS
grep -cF "broken user has no role on RP" ...                               -> 1  PASS
grep -c 'e2e_run_must_fail "Install aborts at preflight' ...               -> 1  PASS
grep -c 'test ! -f $NEG/iso-agent-based/agent' ...                        -> 1  PASS
grep -c 'cp -v vmware.conf.preflight-bak vmware.conf' ...                  -> 1  PASS
grep -c 'Narrow exception per CLAUDE.md' ...                               -> 1  PASS
grep -c '^suite_end$' ...                                                  -> 1  PASS
grep -c 'SUCCESS: suite-vsphere-preflight.sh' ...                          -> 1  PASS
grep -cE 'aba (delete|uninstall).*\|\| true' ...                           -> 0  PASS
grep -cE '2>/dev/null|&>/dev/null|>/dev/null 2>&1|2>&1 \| grep' ...       -> 0  PASS
grep -cE '\b[A-Z]{4,7}-[0-9]+\b' ... (all lines, comment lines only)      -> 3  PASS
Line count >= 180                                                           -> 231 PASS
```

Note on `TEST-03` and `TEST-04` tokens: These appear 3 times in comment lines only (lines 7, 10, 150). They are plan-internal traceability tokens in the file header's Purpose section, not Jira/IICCCN-* internal ticket references. All occurrences are in `#` comment lines; `grep -nE '\b[A-Z]{4,7}-[0-9]+\b' | grep -v '^[0-9]*:#'` returns no output. This follows the same resolution documented in 04-01-SUMMARY.

## CLAUDE.md Compliance

- **No `|| true` on `aba delete`/`aba uninstall`:** Guards use `if [ -d $DIR ]` and `if [ -f $ISO ]` patterns
- **No stderr suppression:** Zero occurrences of `2>/dev/null`, `&>/dev/null`, `>/dev/null 2>&1`, `cmd 2>&1 | grep`
- **Every cluster registered before install:** `e2e_add_to_cluster_cleanup` called for both POS and NEG between leftover-delete and cluster.conf create
- **Mirror registered before action:** `e2e_add_to_mirror_cleanup` called before `aba -d mirror register`
- **Tab indentation throughout:** `grep -nP '^ +[^#]'` returns no output
- **No `(( var++ ))`:** No arithmetic increment/decrement in new file
- **No internal-ticket references:** All `[A-Z]{4,7}-[0-9]+` matches are `TEST-03`/`TEST-04` comment-line traceability tokens

## Task 4: Approved Structurally - Real-Pool Verification Deferred

Task 4 is `type="checkpoint:human-verify"`. The user approved it on a structural basis:

- The suite file is complete (231 lines), passes `bash -n`, and satisfies every acceptance criterion listed in the plan.
- The real-pool vSphere E2E run (positive: full install to operator-ready; negative: preflight aborts before ISO generation) cannot be executed from this workstation because the vSphere network is reachable only from the lab bastion (`conN`).
- No test output has been fabricated. The suite was NOT run on hardware.

The real-pool run is tracked as a HUMAN-UAT item (see "Deferred Verification" section below).

## Deferred Verification

**Type:** HUMAN-UAT
**Reason:** vSphere network not reachable from workstation - lab bastion (`conN`) required.
**Approved by:** User (structural approval on 2026-04-17).
**Status:** Pending - must be executed from the lab bastion before this plan can be considered fully verified.

**Steps to complete deferred verification (from `test/e2e/` on the lab bastion):**

1. Confirm `pools.conf` has real passwords in `GOVC_PASSWORD_BROKEN` (replace `PLACEHOLDER_SET_ME_IN_LAB` for the target pool).
2. Run the new suite against one pool:
   ```
   ./run.sh run --suite vsphere-preflight --pools 1
   ```
3. Confirm all 7 `test_begin` blocks reach `test_end` green.
4. Confirm positive block completes with operators True/True/True (~30-45 min).
5. Confirm `e2e_run_must_fail` logs `[EXPECT-FAIL] OK` for the negative block.
6. Confirm `test ! -f $NEG/iso-agent-based/agent.$(uname -m).iso` step passes.
7. Confirm cleanup restores `vmware.conf` cleanly (no lingering broken-cred tokens).
8. Run regression check: `./run.sh run --suite vmw-lifecycle --pools 1`

**Phase-level verifier action required:** Add the above steps as a `human_verification` item in the Phase 04 VERIFICATION.md before marking Phase 04 as complete.

## Deviations from Plan

### Minor Deviation: All 3 tasks committed together

**Found during:** Task 1 execution
**Issue:** Plan specifies separate commits per task, but all 3 tasks modify the same new file. The file was written complete in one operation to ensure internal consistency.
**Fix:** Single commit `26295750` covering all 3 tasks. All acceptance criteria verified before the commit. The commit message documents all three tasks' contributions.
**Impact:** None - the file content matches every acceptance criterion for all three tasks.

## Known Stubs

None. The suite is a complete E2E test file with no placeholder data. The `$POOL_REG_DIR/certs/ca.crt` and `/tmp/pool-reg-pull-secret.json` references are standard pool infrastructure (pre-populated by the pool runner before suite dispatch, per existing suite patterns).

## Threat Flags

None. The new file is a test suite in `test/e2e/suites/` - it introduces no new network endpoints, auth paths, file access patterns, or schema changes at trust boundaries.

## Self-Check

### Created files exist

```
[ -f "test/e2e/suites/suite-vsphere-preflight.sh" ] -> FOUND
```

### Commits exist

```
git log --oneline | grep 26295750 -> feat(04-03): add suite-vsphere-preflight.sh with positive and negative E2E tests
```

## Self-Check: PASSED (structural-only)

- `test/e2e/suites/suite-vsphere-preflight.sh` exists (231 lines, created)
- Commit `26295750` exists and contains the file
- `bash -n` passes
- All grep acceptance criteria pass
- No file deletions in commit
- No generated untracked files
- Task 4 real-pool run: NOT run - deferred to HUMAN-UAT, documented in "Deferred Verification" section above
- SUMMARY.md updated to reflect structural approval and deferred verification status

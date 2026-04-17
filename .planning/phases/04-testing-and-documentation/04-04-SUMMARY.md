---
phase: 04-testing-and-documentation
plan: "04"
subsystem: documentation
tags: [docs, troubleshooting, vsphere, preflight]
dependency_graph:
  requires: []
  provides: [DOC-01, DOC-02]
  affects: [Troubleshooting.md, README.md]
tech_stack:
  added: []
  patterns: [markdown-relative-links, github-anchor-slugs, four-space-indent-shell-blocks]
key_files:
  created: []
  modified:
    - Troubleshooting.md
    - README.md
decisions:
  - Operator subsection precedes admin subsection per D-17
  - Used four-space indented shell block (not triple-fence) for govc example to avoid fence collision
  - Cross-link uses exact relative path scripts/vmware-required-privileges.sh in both subsections
  - README insertion uses blank-line separator between existing Troubleshooting link and new vSphere line
metrics:
  duration: "~10 minutes"
  completed: "2026-04-17T03:14:15Z"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 2
---

# Phase 04 Plan 04: vSphere Preflight Documentation Summary

**One-liner:** Operator-facing vSphere preflight docs with three-line-type explanation, govc role.create example using all 7 VSPHERE_PRIVS_* arrays, and a README cross-link to the new Troubleshooting.md anchor.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Append vSphere Preflight Validation section to Troubleshooting.md | 34cb4719 | Troubleshooting.md (+97 lines) |
| 2 | Add one-line cross-link in README.md after line 594 | 5e4f082e | README.md (+2 lines) |

## What Was Built

### Task 1 - Troubleshooting.md

Appended a new `## vSphere Preflight Validation` H2 section (lines 173-268) containing:

**Operator subsection** (`### For operators: reading the output`):
- Example failing output (fenced code block) showing two `missing privilege` lines and the summary line
- Line-by-line explanation of all three distinct line types:
  - `missing privilege 'X'` - RBAC gap; ask admin to grant or bind a role
  - `not found` - configuration problem, not RBAC; check vmware.conf field spelling
  - `cannot verify write-access` - warning, missing `System.Read`; not counted as error
- Summary line (`N privilege gap(s) across M scope(s)`) explanation
- Remediation: re-run `aba install`; no flag needed
- Cross-link to `scripts/vmware-required-privileges.sh`

**Admin subsection** (`### For vSphere admins: granting the privileges`):
- Cross-link to `scripts/vmware-required-privileges.sh` with array-per-scope description
- Worked `govc role.create aba-installer` example with all 7 array expansions:
  - `${VSPHERE_PRIVS_ROOT[@]}`, `${VSPHERE_PRIVS_DATACENTER[@]}`, `${VSPHERE_PRIVS_CLUSTER[@]}`
  - `${VSPHERE_PRIVS_DATASTORE[@]}`, `${VSPHERE_PRIVS_NETWORK[@]}`, `${VSPHERE_PRIVS_FOLDER[@]}`
  - `${VSPHERE_PRIVS_RESOURCE_POOL[@]}`
- `govc permissions.set` example binding the role on the resource pool scope
- Note that re-running the commands picks up any future privilege additions automatically
- Third cross-link occurrence in the closing sentence

### Task 2 - README.md

Inserted one blank line + one `**vSphere-specific:**` line immediately after the existing
Troubleshooting cross-link at line 594, pointing at `Troubleshooting.md#vsphere-preflight-validation`.

The GitHub anchor slug `#vsphere-preflight-validation` is the automatic lowercase+hyphen
conversion of `## vSphere Preflight Validation` (GitHub renders "vSphere" as "vsphere").

## Acceptance Grep Counts (post-commit)

| Check | Expected | Actual |
|-------|----------|--------|
| `grep -c '^## vSphere Preflight Validation' Troubleshooting.md` | 1 | 1 |
| `grep -c '^### For operators: reading the output' Troubleshooting.md` | 1 | 1 |
| `grep -c '^### For vSphere admins: granting the privileges' Troubleshooting.md` | 1 | 1 |
| `grep -cF '[scripts/vmware-required-privileges.sh](...)' Troubleshooting.md` | >=2 | 3 |
| `grep -c 'govc role.create aba-installer' Troubleshooting.md` | 1 | 1 |
| `grep -c 'govc permissions.set' Troubleshooting.md` | >=1 | 2 |
| `grep -c 'VSPHERE_PRIVS_' Troubleshooting.md` | 7 | 7 |
| `grep -c 'privilege gap(s) across' Troubleshooting.md` | >=1 | 2 |
| `grep -c 'not found' Troubleshooting.md` | >=1 | 2 |
| `grep -cE '\b[A-Z]{4,7}-[0-9]+\b' Troubleshooting.md` | 0 | 0 |
| `grep -ciE 'web client\|vsphere client\|vsphere ui' Troubleshooting.md` | 0 | 0 |
| `grep -ciE '^\|.*Privilege.*\|' Troubleshooting.md` | 0 | 0 |
| trailing newline (0a) | yes | yes |
| `grep -cF '[vSphere Preflight Validation](Troubleshooting.md#vsphere-preflight-validation)' README.md` | 1 | 1 |
| `grep -cF '**vSphere-specific:**' README.md` | 1 | 1 |
| `grep -cF '[Troubleshooting](Troubleshooting.md)' README.md` | 1 | 1 |
| `grep -cE '\b[A-Z]{4,7}-[0-9]+\b' README.md` | unchanged | 0 |

## Deviations from Plan

None - plan executed exactly as written.

## Known Stubs

None. Both files contain real, actionable documentation with verified cross-links.

## Threat Flags

None. No new network endpoints, auth paths, file access patterns, or schema changes.

## Self-Check: PASSED

- `Troubleshooting.md` exists and contains `## vSphere Preflight Validation`: confirmed
- `README.md` contains `[vSphere Preflight Validation](Troubleshooting.md#vsphere-preflight-validation)`: confirmed
- Commit 34cb4719 exists: Troubleshooting.md task
- Commit 5e4f082e exists: README.md task
- No internal-ticket tokens in either file: confirmed (grep returns 0)
- No markdown table duplicating priv arrays: confirmed
- No UI walkthrough content: confirmed
- File ends with newline: confirmed (0x0a)

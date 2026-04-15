# Testing Patterns

**Analysis Date:** 2026-04-15

## Testing Layers

ABA has two distinct test layers:

1. **Functional Tests** (`test/func/`) — Fast, local, runnable without VMs
2. **E2E Tests** (`test/e2e/`) — Integration tests on vSphere pools with isolated networks

## Functional Tests

### Overview

Located in `test/func/`, these are unit and functional tests for individual components.
Runnable locally on any RHEL/Linux host. No VMs, no infrastructure required.

**Runner:**
- `test/func/run-all-tests.sh` — Executes all `test-*.sh` scripts
- Exit code 0 = all pass, non-zero = failure
- Output: colored pass/fail summaries per test

### Test File Organization

**Location:**
- All functional tests in `test/func/*.sh`
- Co-located with code they test (e.g., `test-run-once-reliability.sh` tests `scripts/include_all.sh`)

**Naming:**
- Format: `test-<component>.sh` or `test-<feature>.sh`
- Examples:
  - `test-run-once-reliability.sh` — Task manager (`run_once()`)
  - `test-preflight-check.sh` — Preflight validation
  - `test-e2e-framework.sh` — E2E coordinator (`run.sh`)
  - `test-tui-v2-01-wizard.sh` — TUI wizard flow

**Structure:**
```
test/func/
├── run-all-tests.sh              # Test runner
├── test-*.sh                     # Individual test files
├── suite-*.sh                    # (Legacy) some tests use suite pattern
└── *-test-lib.sh                 # Shared helpers
```

### Test Structure

**Shebang & setup:**
```bash
#!/bin/bash
# Test description
# Purpose and prerequisites

set -e  # Exit on first failure

# Source common functions
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ABA_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
source "$ABA_ROOT/scripts/include_all.sh"
```

**Simple test pattern:**
```bash
#!/bin/bash
# Test: Check syntax of scripts

cd "$(dirname "$0")/../.."

test_pass() { echo "✓ PASS: $1"; }
test_fail() { echo "✗ FAIL: $1"; exit 1; }

# Syntax check
bash -n scripts/my-script.sh && test_pass "Syntax valid" || test_fail "Syntax invalid"

# Verify expected functions
grep -q '^my_function()' scripts/my-script.sh && test_pass "Function exists" || test_fail "Function missing"

echo "All tests passed"
```

**Assertion helpers:**
```bash
_assert() {
    local desc="$1"; shift
    if "$@" >/dev/null 2>&1; then
        echo "  ✓ PASS: $desc"
    else
        echo "  ✗ FAIL: $desc"
        exit 1
    fi
}

_assert_output() {
    local desc="$1" pattern="$2"; shift 2
    local out=$("$@" 2>&1) || true
    if echo "$out" | grep -qE "$pattern"; then
        echo "  ✓ PASS: $desc"
    else
        echo "  ✗ FAIL: $desc (expected /$pattern/ in output)"
        exit 1
    fi
}

_assert_file_exists() {
    local file="$1"
    [ -f "$file" ] || { echo "✗ FAIL: File not found: $file"; exit 1; }
    echo "  ✓ PASS: File exists: $file"
}
```

### Key Test Examples

**`test-run-once-reliability.sh`** — Tests task manager:
- Basic execution and wait
- Idempotency (caching)
- Crash recovery (kill scenarios)
- TTL (time-to-live) caching
- Parallel validation
- Exit code capture

**`test-preflight-check.sh`** — Validates script standards:
- Syntax check
- Tab indentation (no spaces)
- No `$ABA_ROOT` usage (relative paths only)
- No trailing whitespace
- Function existence
- Executable permissions

**`test-e2e-framework.sh`** — Tests E2E coordinator without VMs:
- Deploy to specific pool
- Status detection (idle, running)
- Stop isolation (doesn't touch other pools)
- RC file generation
- Suite completion detection
- Requires SSH access to conN (test pool VM)

### Running Functional Tests

```bash
# Run all tests
test/func/run-all-tests.sh

# Run individual test
test/func/test-run-once-reliability.sh

# Run with specific TEST_DIR (for run_once tests)
TEST_DIR=/tmp/my-test test/func/test-run-once-reliability.sh
```

## E2E Tests

### Overview

End-to-end tests exercise the full ABA workflow on vSphere infrastructure.
Each test runs on an isolated pool (conN = connected bastion, disN = air-gapped bastion).
Tests run in parallel across multiple pools; each pool runs one suite at a time.

**Architecture:**
- `test/e2e/run.sh` — Coordinator (CLI parsing, dispatch, status monitoring)
- `test/e2e/runner.sh` — Runs on conN (executes suites, cleanup, RC file generation)
- `test/e2e/suites/suite-*.sh` — Individual test suites
- `test/e2e/lib/framework.sh` — Core test framework (e2e_run, assertions, lifecycle)
- `test/e2e/lib/` — Support libraries (pool management, VM helpers, SSH, config)

### Configuration

**Location:**
- `test/e2e/config.env` — Defaults (channel, OCP version, RHEL version, SSH users, timeouts)
- `test/e2e/pools.conf` — Per-pool overrides (datastore, folder, VM template, disk size)

**Precedence (highest to lowest):**
1. CLI flags: `--pools 4`, `--os rhel9`, `--suite X`
2. Per-pool in `pools.conf`: `VM_DISK_EXTRA_GB=30`, `CON_SSH_USER=root`
3. Defaults in `config.env`: `TEST_CHANNEL=stable`, `OCP_VERSION=p`

**Key settings:**
```bash
# config.env
TEST_CHANNEL=stable           # OCP channel: fast | stable | candidate
OCP_VERSION=p                 # p = previous, l = latest, or explicit e.g. 4.17.12
INT_BASTION_RHEL_VER=rhel8   # RHEL version for conN/disN VMs
CON_SSH_USER=steve            # SSH user on conN (or root)
DIS_SSH_USER=steve            # SSH user on disN (or root)
VM_DISK_EXTRA_GB=0            # Extra disk space (0 = no expansion)
VMWARE_CONF=~/.vmware.conf   # govc configuration
VM_SNAPSHOT=aba-test          # Snapshot to revert before clone
VM_DATASTORE=Datastore4-1    # vSphere datastore (per-pool in pools.conf)
VC_FOLDER=/Datacenter/vm/aba-e2e/poolN  # vCenter folder (per-pool)
```

### Pool Structure

Each pool contains:
- **conN** (connected bastion) — Internet access, runs `runner.sh`, SSH server
- **disN** (disconnected bastion) — No internet (by design), used for air-gapped tests
- **Cluster VMs** — Install targets for test suites

**Network layout:**
```
Pool 1: con1 (10.0.2.10) + dis1 (10.0.2.11) + cluster1 (10.0.2.12+)
Pool 2: con2 (10.0.2.20) + dis2 (10.0.2.21) + cluster2 (10.0.2.22+)
...
VLAN 10.10.20.0/24: conN-to-disN link (static IPs, no DHCP)
```

### Run Commands

**Dispatcher:**
```bash
# Run all suites across N pools (parallel dispatch, one suite per pool at a time)
./test/e2e/run.sh run --all --pools 4

# Run specific suite on pool 2
./test/e2e/run.sh run --suite cluster-ops --pool 2

# Force dispatch (works with running dispatcher, hot-deploy)
./test/e2e/run.sh run --suite cluster-ops --pool 2 --force -y

# Resume last suite, skip already-passed tests
./test/e2e/run.sh run --pool 2 --resume

# Developer mode (push local source to ~/aba on conN)
./test/e2e/run.sh run --all --pools 4 --dev

# Show status of all pools
./test/e2e/run.sh status --pools 4

# List available suites
./test/e2e/run.sh list

# Attach to pool 1's tmux session (live output)
./test/e2e/run.sh attach con1

# Stop and clean up pool 2
./test/e2e/run.sh stop --pool 2 --clean

# Deploy code without running suites
./test/e2e/run.sh deploy --pools 4
```

### Suite File Organization

**Location:**
```
test/e2e/
├── suites/
│   ├── suite-connected-public.sh        # Public registry (proxy/direct modes)
│   ├── suite-mirror-sync.sh             # Mirror create, sync, load
│   ├── suite-airgapped-local-reg.sh     # Local registry on disN
│   ├── suite-airgapped-existing-reg.sh  # Pre-existing registry
│   ├── suite-cluster-ops.sh             # Cluster install, day2 ops, operators
│   ├── suite-vmw-lifecycle.sh           # VMware cluster lifecycle
│   ├── suite-kvm-lifecycle.sh           # KVM cluster lifecycle
│   ├── suite-network-advanced.sh        # VLAN-based installs
│   ├── suite-create-bundle-to-disk.sh   # Bundle creation and transfer
│   ├── suite-negative-paths.sh          # Error handling, edge cases
│   ├── suite-cli-validation.sh          # CLI argument validation
│   └── suite-config-validation.sh       # Config file validation
├── lib/
│   ├── framework.sh        # e2e_run, e2e_run_remote, assertions, lifecycle
│   ├── constants.sh        # Shared constants
│   ├── config-helpers.sh   # Pool IP/domain calculation
│   ├── remote.sh           # SSH helpers (_essh, e2e_run_remote)
│   ├── pool-lifecycle.sh   # VM cloning, network, dnsmasq, firewall
│   ├── vm-helpers.sh       # govc wrappers
│   └── setup.sh            # Pool infrastructure setup
├── run.sh                  # Coordinator
├── runner.sh               # Runs on conN
├── config.env              # Defaults
└── pools.conf              # Per-pool overrides
```

### Suite Structure

**Minimal suite template:**
```bash
#!/usr/bin/env bash

set -u

_SUITE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$_SUITE_DIR/../lib/framework.sh"          # Core framework
source "$_SUITE_DIR/../lib/config-helpers.sh"    # Pool config helpers

# Pool-unique names (avoid collisions across parallel pools)
CLUSTER1="$(pool_cluster_name cluster1)"

e2e_setup                                         # Initialize logging, state files

plan_tests \
    "Setup: install ABA" \
    "Configure" \
    "Install cluster" \
    "Verify" \
    "Cleanup: delete cluster"

suite_begin "my-suite"

# ============================================================================
# Test 1: Setup
# ============================================================================
test_begin "Setup: install ABA"

e2e_run "Install ABA" "curl ... | bash"
cd ~/aba

test_end

# ============================================================================
# Test 2: Configure
# ============================================================================
test_begin "Configure"

e2e_run "Create cluster config" "aba cluster -n $CLUSTER1 ..."
e2e_run "Set DNS" "sed -i 's/^dns_servers=.*/dns_servers=$(pool_dns_server)/' aba.conf"

test_end

# ============================================================================
# Test 3: Install
# ============================================================================
test_begin "Install cluster"

e2e_add_to_cluster_cleanup "$PWD/$CLUSTER1"      # Register for cleanup
e2e_run -r 2 10 "Install" "aba -d $CLUSTER1 install"  # Retry 2x, 10s delay

test_end

# ============================================================================
# Test 4: Verify
# ============================================================================
test_begin "Verify"

e2e_poll 600 30 "Wait for operators" \
    "aba -d $CLUSTER1 run | grep -c 'True.*True.*True'"
e2e_run "Show status" "aba -d $CLUSTER1 run"

test_end

# ============================================================================
# Test 5: Cleanup
# ============================================================================
test_begin "Cleanup: delete cluster"

e2e_run "Delete cluster" "aba -d $CLUSTER1 delete"

test_end

suite_end
```

### E2E Framework API

**Test lifecycle:**
- `e2e_setup` — Initialize logging, state files, error traps
- `plan_tests "name1" "name2" ...` — Register test names for progress display
- `suite_begin "name"` — Start suite (logs, resources registration)
- `test_begin "name"` — Start test block (progress, resume point)
- `test_end` — End test block (record PASS/FAIL/SKIP)
- `suite_end` — Finalize suite (print summary, notify)

**Command execution:**
- `e2e_run "description" "command"` — Run on conN, fail on error
- `e2e_run -r N T "desc" "cmd"` — Retry N times with T-second delay
- `e2e_run_remote "description" "command"` — Run on disN (via SSH)
- `e2e_run_must_fail "description" "command"` — Command MUST fail (inverse assertion)
- `e2e_diag "description" "command"` — Diagnostic (exit code ignored, output preserved)
- `e2e_poll TIMEOUT INTERVAL "desc" "condition"` — Poll until condition returns 0

**Assertions:**
- `assert_file_exists "path"` — Verify file exists
- `assert_dir_exists "path"` — Verify directory exists
- `assert_equals "expected" "actual"` — String comparison

**Cleanup registration:**
- `e2e_add_to_cluster_cleanup "/path/to/cluster"` — Register cluster for deletion
- `e2e_add_to_mirror_cleanup "mirror-name"` — Register mirror for uninstall
- Cleanup happens in explicit `test_begin "Cleanup: ..."` block before `suite_end`

**Remote execution:**
- `e2e_run_remote "desc" "cmd"` — Run on disN (via _essh)
- `_essh hostname command` — SSH wrapper (timeouts, batch mode, no known_hosts check)

### Golden Rules for E2E Tests

**1. Tests MUST fail on error (never mask issues)**
- Let failures propagate
- Never use `|| true` on lifecycle commands (aba delete, aba uninstall)
- If cleanup fails, that IS the bug — stop and investigate

**2. NEVER suppress stderr**
- **Forbidden patterns:**
  - `cmd 2>/dev/null` — hides error entirely
  - `cmd >/dev/null 2>&1` — silences both streams
  - `cmd &>/dev/null` — same problem
  - `cmd 2>&1 | grep` — stderr merged into pipe
- **Correct alternatives:**
  - `cmd >/dev/null` — stdout silenced, stderr visible
  - `cmd || true` — error visible, exit code swallowed
  - Use `e2e_diag` for diagnostic commands (exit code ignored)
- **Narrow exceptions (must have a comment):**
  - `command -v foo >/dev/null` — existence check
  - `kill -0 $pid 2>/dev/null` — process probe
  - `tmux ... 2>/dev/null` — session may not exist
  - `[ "$x" -gt 0 ] 2>/dev/null` — arithmetic guard

**3. Resource lifecycle**
- SNO clusters: MUST delete (free resources)
- Compact/Standard clusters: MUST delete (large, hold VIPs)
- Mirrors: MUST uninstall from same host that installed
- Every suite must have explicit `test_begin "Cleanup: ..."` block before `suite_end`
- Register resources immediately before install: `e2e_add_to_cluster_cleanup "$PWD/$CLUSTER"`

**4. Test hygiene**
- NEVER create ABA-internal files directly (e.g., `cat > imageset-config-save.yaml`)
- NEVER call ABA-internal functions directly (e.g., `run_once()`, `download_all_catalogs()`)
- Use `aba` CLI or `make` targets instead
- NEVER use `aba reset` as mid-process cleanup (only for 100% fresh repo)
- For mid-process cleanup use `aba clean`, `aba uninstall`, or `aba delete`

**5. No air-gap violations**
- disN has NO INTERNET by design
- All artifacts must arrive via bundle transfer from conN
- If something is missing on disN, the bundle creation failed — never fetch from internet

**6. Pool awareness**
- Always pass `--pools N` when not all pools are active
- Pool-unique cluster names via `pool_cluster_name()` helper
- Per-pool IP/domain calculation via `pool_dns_server()`, `pool_domain()`, `pool_sno_ip()`

### Running E2E Tests

```bash
cd test/e2e

# Full test run
./run.sh run --all --pools 4

# Single suite on specific pool
./run.sh run --suite cluster-ops --pool 2

# With developer mode (push local source)
./run.sh run --all --pools 4 --dev

# Check status
./run.sh status --pools 4

# Attach to live output
./run.sh attach con1

# Stop and clean
./run.sh stop --pools 4 --clean
```

### Test Output & Logs

**Log files:**
- Suite logs: `test/e2e/logs/suite-<name>-pool<N>-<timestamp>.log`
- Summary log: `test/e2e/logs/summary.log`
- Per-test state: `test/e2e/logs/suite-<name>-pool<N>.rc`

**Monitoring:**
```bash
# Live tail of current suite
tail -f test/e2e/logs/latest.log

# Multi-pool dashboard
./run.sh live 4

# Read-only summary
./run.sh dash 4
```

### Coverage & Responsibility

**Functional tests cover:**
- Component-level functionality (task manager, config loading, CLI tools)
- Static validation (syntax, style, conventions)
- Fast feedback loop (run locally in seconds)

**E2E tests cover:**
- Full workflows (install, cluster ops, mirror management)
- Integration between components
- Infrastructure interactions (vSphere, networking, storage)
- Parallel execution (multiple pools, same tests concurrently)

**What's NOT tested:**
- ESXi-direct installs (currently vCenter-only; see BACKLOG.md)
- Physical bare-metal (only KVM/VMware in labs)
- Production-scale deployments (limited to SNO/Compact/Standard on test infra)

## Test Requirements Before Commit

**From `.cursor/rules/rules-of-engagement.mdc`:**
- Run `build/pre-commit-checks.sh` before every code commit (updates ABA_BUILD, checks syntax)
- For code changes: use full `build/pre-commit-checks.sh`
- For docs-only: use `build/pre-commit-checks.sh --skip-version`
- **NEVER commit code without automated test coverage** (unless user manually verifies)
- Show user what changed before committing — always wait for explicit approval

---

*Testing analysis: 2026-04-15*

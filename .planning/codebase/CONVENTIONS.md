# Coding Conventions

**Analysis Date:** 2026-04-15

## Language

**Primary:**
- Bash — All scripts in `scripts/`, `test/`, and utilities
  - Shell: POSIX-compatible, Bash 4.2+ required for test framework
  - Interpreter: `#!/bin/bash` (not `#!/bin/sh`)

**Secondary:**
- Make — Task orchestration (Makefiles in project root and subdirectories)
- Configuration files — `.conf` format (key=value, used as shell sources)

## Indentation & Whitespace

**Indentation:**
- **Tabs only** — Never spaces (enforced by `test/func/test-preflight-check.sh` and pre-commit checks)
- Example:
  ```bash
  if [ "$x" ]; then
  	echo "good"  # <-- tab indent

  	echo "still good"  # <-- empty line has NO characters
  fi
  ```

**Empty Lines:**
- Must have **zero characters** (no spaces, no tabs)
- Trailing whitespace on any line is forbidden

**Enforcement:**
- Pre-commit check (`build/pre-commit-checks.sh`) runs `bash -n` on all scripts
- Tests check with `grep -Pn '\s+$'` for trailing whitespace
- Tab requirement verified by functional tests

## Naming Conventions

**File naming in `scripts/`:**
- Kebab-case with descriptive prefixes: `cli-download-all.sh`, `cluster-config.sh`, `check-version-mismatch.sh`
- Entry point: `aba.sh` (with tab-stamped `ABA_BUILD` timestamp)
- Configuration loaders: `normalize-*.sh` functions (e.g., `normalize-aba-conf`, `normalize-mirror-conf`)
- Task runners: `ensure_*.sh` patterns (e.g., `ensure_oc`, `ensure_oc_mirror`, `ensure_openshift_install`, `ensure_govc`)

**File naming in `test/`:**
- Functional tests: `test-*.sh` (e.g., `test-run-once-reliability.sh`, `test-preflight-check.sh`)
- E2E suites: `suite-*.sh` (e.g., `suite-connected-public.sh`, `suite-cluster-ops.sh`)
- Test helpers: `*-test-lib.sh` (e.g., `tui-test-lib.sh`)

**Function naming:**
- All lowercase with underscores: `normalize_aba_conf`, `verify_conf`, `check_cluster_installed`
- Private/internal functions: Prefix with underscore: `_color_echo`, `_essh`, `_e2e_run`
- Color echo functions: `echo_red`, `echo_green`, `echo_yellow`, etc. (bright variants: `echo_bright_red`)
- ABA framework functions: `aba_info`, `aba_info_ok`, `aba_warning`, `aba_abort`, `aba_debug`
- E2E framework: `e2e_run`, `e2e_run_remote`, `e2e_run_must_fail`, `e2e_diag`, `test_begin`, `test_end`

**Variable naming:**
- Config variables: UPPERCASE (e.g., `OCP_VERSION`, `PLATFORM`, `VMWARE_CONF`)
- Internal shell variables: lowercase (e.g., `ret`, `elapsed`, `exit_code`)
- Global task IDs in `run_once()`: dot-separated (e.g., `task:id`, `test:simple`, `test:kill`)

**Configuration file naming:**
- Main config: `aba.conf` (single source of truth for platform/channel/version/domain)
- Platform-specific: `vmware.conf`, `kvm.conf` (loaded conditionally based on `platform` variable)
- Mirror config: `mirror.conf` (created by `aba mirror.conf`)
- Test harness: `config.env` (E2E defaults), `pools.conf` (per-pool overrides)

## Code Style & Structure

**Shebang & Setup:**
- Always `#!/bin/bash` (not `#!/bin/sh`)
- Source `scripts/include_all.sh` immediately (provides color echo, error handlers, strict mode)
- Set error handling: `set -e`, `set -u`, `trap 'show_error' ERR` (via include_all.sh by default)

**Error Handling:**
- Use `aba_abort` in scripts (exits with message to stderr):
  ```bash
  if ! some_command; then
      aba_abort "Operation failed" \
          "Additional context line 1" \
          "Additional context line 2"
  fi
  ```
- **NEVER use `aba_abort` in TUI code** — it calls `exit 1` and kills the dialog interface
- In TUI, use dialog boxes: `dialog --msgbox "Error\nDetails" 0 0` and `return`

**Standard output vs stderr:**
- **Stdout:** Human-readable output (`ls`, `grep`, normal command results) OR structured data
- **Stderr:** Error messages, warnings, progress info (via `>&2`)
- **Exception:** `aba bundle --out -` command must output **only tar data to stdout** — all messages go to `>&2`

**Output redirection:**
- Avoid `2>/dev/null` and `>/dev/null 2>&1` unless there's a documented reason
- Use conditionals instead of suppression:
  ```bash
  # ✅ GOOD
  if [[ -f "$file" ]]; then
      pid=$(<"$file")
  fi

  # ❌ BAD - may hide real errors
  pid=$(<"$file" 2>/dev/null)
  ```

**Comments:**
- Add comments for non-obvious bash features (indirect expansion, arithmetic, array ops)
- Comment ALL narrow exceptions to stderr suppression rules:
  ```bash
  # ✅ existence check (no stderr output expected)
  command -v foo >/dev/null

  # ✅ process probe (ENOENT noise suppressed)
  kill -0 $pid 2>/dev/null

  # ✅ tmux session may not exist yet
  tmux kill-session -t '$SESSION' 2>/dev/null || true
  ```
- Code comments are the primary source of truth (keep them with the code, not in docs)

**Color echo helpers:**
- Use from `scripts/include_all.sh`: `echo_red`, `echo_green`, `echo_yellow`, `echo_cyan`, etc.
- For ABA framework messages: `aba_info` (white), `aba_info_ok` (green), `aba_warning` (red), `aba_debug` (magenta with timestamp)
- For E2E framework: `_e2e_red`, `_e2e_green`, `_e2e_yellow` (colors always enabled for log viewing)

## Critical Bash Pitfalls

**NEVER use `(( var++ ))` or `(( var-- ))`:**
- When `var` is 0, `(( 0 ))` returns exit code 1 and crashes under `set -e` or ERR trap
- **CORRECT:** `var=$(( var + 1 ))` or `var=$((var + 1))`
- **WRONG (banned):** `(( var++ )) || true` — don't band-aid with `|| true`

**Bug: `$(<"file" 2>/dev/null)` returns empty in bash 5.1.8+:**
- Read file content without suppression: `pid=$(<"$pid_file")`
- Check existence first if it may not exist: `if [[ -f "$file" ]]; then pid=$(<"$file"); fi`

**SSH piping corrupts files:**
- Piping file content through multi-hop SSH can corrupt binary data (SSH key corruption, `authorized_keys` failures)
- Use scp or write on target host directly instead

## Task Management (`run_once()`)

**Pattern:**
- `run_once -i "task:id" -- command args` — Start task
- `run_once -w -i "task:id"` — Wait for completion with optional message
- `run_once -w -m "Message..." -i "task:id"` — Wait with visible message
- `run_once -p -i "task:id"` — Peek: check if task is done (exit 0 if complete)
- `run_once -e -i "task:id"` — Get stderr from failed task
- `run_once -t 3600 -i "task:id" -- command` — Cache result for 1 hour (TTL)

**Why not direct access to `~/.aba/runner/`:**
- `run_once()` manages locking, PID files, output redirection, exit codes
- Direct access bypasses task queue and crash recovery
- All task state must go through `run_once()`

**Error handling pattern:**
```bash
if run_once -w -m "Waiting for download..." -i "task:id"; then
    aba_info_ok "Task completed successfully"
else
    error_output=$(run_once -e -i "task:id" | head -20)
    aba_abort "Task failed" \
        "Error details:" \
        "$error_output"
fi
```

## Configuration Loading (`normalize_*()`)

**Pattern:**
- Functions in `scripts/include_all.sh`: `normalize-aba-conf`, `normalize-mirror-conf`, `normalize-cluster-conf`
- Output **only values from config files** (with defaults for backwards compat)
- **Derived/computed values belong in the calling script, NOT in normalize functions**
- Example:
  ```bash
  # ✅ CORRECT - normalize_aba_conf outputs raw values
  export registry_host=$(normalize_aba_conf | grep ^registry_host | cut -d= -f2)

  # ✅ CORRECT - caller computes derived value
  regcreds_dir="$registry_host-creds"  # Derived, belongs here

  # ❌ WRONG - don't put computed values in normalize_aba_conf
  # export regcreds_dir="$registry_host-creds"  # Should NOT be in normalize function
  ```

**Config file as single source of truth:**
- `aba.conf` is authoritative for `platform`, `ocp_version`, `ocp_channel`, `domain`, etc.
- Supporting files (`vmware.conf`, `kvm.conf`) loaded **conditionally** based on `platform` variable
- **File presence MUST NOT be used to infer settings** — only the config variable is authoritative
- Example:
  ```bash
  # ✅ CORRECT - check the platform variable
  if [ "$platform" = "vmw" ]; then
      source vmware.conf
  fi

  # ❌ WRONG - don't check file existence
  # if [ -f vmware.conf ]; then  # This is unreliable
  ```

## CLI Tool Installation (`ensure_*()`)

**Pattern:**
- `ensure_oc` — Install OC CLI
- `ensure_oc_mirror` — Install oc-mirror
- `ensure_openshift_install` — Install openshift-install
- `ensure_govc` — Install govc (vSphere CLI)
- Located in `scripts/include_all.sh`, callable from any directory
- Idempotent: `ensure_oc` twice only installs once (uses `command -v` to check)

## Marker Files & Makefile Discipline

**Marker files (created by Makefiles, NOT by scripts):**
- `.available` — Resource is ready (cluster running, mirror installed)
- `.unavailable` — Resource is known to be missing or failed
- `.cleanup` — Cluster registered for cleanup (test framework)
- `.mirror-cleanup` — Mirror registered for cleanup (test framework)

**Why scripts don't manage markers:**
- Makefiles have centralized dependency tracking
- Scripts calling scripts would double-trigger on file changes
- Marker timestamps represent Makefile contract, not script output

**Scripts called from Makefiles:**
- NEVER call scripts directly from shell: `scripts/my-script.sh`
- ALWAYS use Makefile targets: `make -C mydir target`
- This ensures markers are set/cleared correctly

## Relative Paths

**Use relative paths everywhere EXCEPT in `scripts/aba.sh`:**
- Bad: `source "$ABA_ROOT/scripts/include_all.sh"`
- Good: `source scripts/include_all.sh` (assumes caller is in project root or proper subdir)
- Exception: `scripts/aba.sh` uses `$ABA_ROOT` since it's the entry point that may be invoked from anywhere

## Module Exports & Barrel Files

**Scripts are not modules — each one is an executable unit:**
- No "barrel files" or `source` chains in user-facing scripts
- Each script sources only what it needs from `scripts/include_all.sh`
- E2E framework scripts source from `test/e2e/lib/` as needed

## Version & Build Stamping

**How versioning works:**
- `VERSION` file: Manual SemVer (e.g., "0.9.0")
- `ABA_BUILD` in `scripts/aba.sh`: Automatic timestamp (YYYYMMDDHHMMSS), updated by `build/pre-commit-checks.sh`
- `build/release.sh`: Bumps VERSION, updates CHANGELOG, runs pre-commit checks, creates git tag

**Version display:**
```bash
aba --aba-version
# Output: aba version 0.9.0 (build 20260120220637)
#         Git: dev @ 0da6892
```

**Pre-commit workflow (for code changes):**
```bash
build/pre-commit-checks.sh           # Updates ABA_BUILD, checks syntax, validates RPM lists
git add scripts/ tests/              # Stage changed files
git status --short                   # Show what's staged
git commit -m "type(scope): message" # Explicit approval required
git push origin dev                  # Push (ALWAYS together with commit)
```

**For docs-only commits:**
```bash
build/pre-commit-checks.sh --skip-version  # Skip ABA_BUILD update
git add docs/
git commit -m "docs: ..."
git push origin dev
```

---

*Convention analysis: 2026-04-15*

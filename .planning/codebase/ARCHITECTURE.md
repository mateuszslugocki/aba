# Architecture

**Analysis Date:** 2026-04-15

## Pattern Overview

**Overall:** Make-orchestrated bash architecture with cross-process task coordination

**Key Characteristics:**
- **Three-tier execution model:** CLI (`aba` symlink) → Makefiles (per-domain) → scripts (implementation) → shared library functions (`include_all.sh`)
- **File-based Make dependencies** for build/packaging tasks (CLI binaries, mirror registries, bundle creation)
- **Cross-process task coordination** via `run_once()` mechanism for long-running operations (catalog downloads, registry startup, image mirroring)
- **Configuration as source of truth:** `aba.conf` (main), `mirror.conf`, `cluster.conf`, `vmware.conf`, `kvm.conf` are canonical; file presence never infers settings
- **Connected → Bundle → Disconnected pipeline:** All artifacts downloaded on connected workstation, packaged into bundle, deployed to air-gapped bastion with zero internet access

## Layers

**CLI Layer (User-facing):**
- Location: `scripts/aba.sh` (entry point via `aba` symlink), `tui/abatui.sh` (TUI via `abatui` symlink)
- Purpose: Argument parsing, environment setup, directory management, workflow orchestration
- Responsibilities: Parse `--dir/-d`, `--debug/-D` flags; change to target directory; source `include_all.sh`; dispatch to Makefiles or scripts
- Depends on: None (sets up environment for everything else)
- Used by: User invocation, `install` bootstrap script

**Make Layer (Dependency management):**
- Location: Root `Makefile`, `cli/Makefile`, `mirror/Makefile`, `bundles/v2/Makefile`, `test/Makefile`, subdirectory Makefiles
- Purpose: Declare file-based dependencies, orchestrate builds, manage idempotency via file timestamps
- Responsibilities: Download CLI binaries (oc, openshift-install, oc-mirror, govc), build mirror registries, create bundles, manage RPM dependencies
- Depends on: `scripts/` for implementation, config files for parameterization
- Used by: `scripts/aba.sh`, `scripts/setup-mirror.sh`, `scripts/setup-cluster.sh`, bundle creation pipeline

**Script Layer (Workflow implementation):**
- Location: `scripts/` (~70 scripts), `bundles/v2/scripts/` (8 pipeline stages), `test/e2e/suites/` (test cases)
- Purpose: High-level orchestration of business logic (create mirror registry, sync images, install cluster, configure day2 ops)
- Responsibilities: Call functions from `include_all.sh`, invoke Makefiles, manage state, handle user interaction
- Depends on: `include_all.sh`, Makefiles, configuration files
- Used by: CLI layer, Make rules, other scripts

**Library Layer (Reusable functions):**
- Location: `scripts/include_all.sh` (~2500 lines of bash functions)
- Purpose: Centralized repository of functions callable from CLI, TUI, tests, and scripts
- Responsibilities: Task coordination (`run_once()`), CLI tool installation (`ensure_oc()`, `ensure_govc()`, etc.), config normalization (`normalize_aba_conf()`, `normalize_mirror_conf()`, etc.), operator catalog management (`download_all_catalogs()`, `wait_for_all_catalogs()`), output formatting (color functions, logging)
- Depends on: External tools (curl, docker, podman, oc-mirror) that must already be installed or downloaded
- Used by: All scripts, CLI, TUI, tests

## Data Flow

**Connected Workstation → Disconnected Bastion Pipeline:**

1. **User invokes `aba` on connected workstation** (has internet access)
   - `scripts/aba.sh` parses arguments, sources `include_all.sh`
   - Calls `normalize_aba_conf()` to load/validate `aba.conf`

2. **Download phase (connected only)**
   - `make -C cli` downloads OpenShift CLI tools (oc, openshift-install, oc-mirror, govc)
   - `download_all_catalogs()` starts 3 catalog downloads in background via `run_once -i`
   - Happens early, overlaps with other work

3. **Wait phase (when needed)**
   - `wait_for_all_catalogs()` blocks until all 3 catalogs present
   - Explicit dependency in Makefiles: `$(SCRIPTS)/save_imageset.sh` depends on `catalogs-download catalogs-wait`
   - Prevents image save from starting before catalogs ready

4. **Mirror create/sync phase (connected)**
   - User runs `aba -d mirror save` or `aba -d mirror sync`
   - `make -C mirror` with `run_once` wrapping starts mirror registry
   - `reg-save.sh` or `reg-sync.sh` invokes `oc-mirror` to download/copy images
   - Creates tarball (save) or pushes to registry (sync)

5. **Bundle creation (connected)**
   - User runs `aba bundle` or `aba tar`
   - `scripts/make-bundle.sh` or `backup.sh` packages repo + mirrors + CLIs
   - Creates single transferable archive

6. **Transfer to disconnected bastion** (physical: USB, network appliance, etc.)
   - No internet on bastion by design
   - All artifacts already in bundle

7. **Unpack and install (disconnected)**
   - `test/e2e/` suites unpack bundle, install registry, load images, create cluster
   - Or user manually: untar bundle, run `aba -d mirror install`, run `aba -d mirror load`
   - Registry and cluster operate entirely from local mirrors

**State Management:**

- **Config files** (`aba.conf`, `mirror.conf`, `cluster.conf`): Written by CLI, read by scripts; represent desired state
- **Run-once state** (`~/.aba/runner/<task-id>/`): Tracks background task status (lock, pid, exit code, output)
- **Marker files** (`.init`, `.available`, `.unavailable`): Track Makefile target completion state
- **Mirror data** (`~/.aba/mirror/<name>/`): Per-mirror registry credentials and state

## Key Abstractions

**run_once() - Cross-Process Task Coordination:**
- Purpose: Deduplicate long-running operations; allow "start early, wait late" pattern
- Location: `scripts/include_all.sh` (defined lines ~1400-1650)
- Pattern: `run_once -i <task-id> -- command` (start background), `run_once -w -i <task-id> -- command` (wait for completion)
- Usage examples:
  - Catalog downloads: `run_once -i "catalog:4.19:redhat-operator" -- ...download...`
  - CLI installation: `run_once -i "cli:install:oc-mirror" -- make -C cli oc-mirror`
  - Connectivity checks: `run_once -i "cli:check:api.openshift.com" -t 600 -- curl https://api.openshift.com/`
- Key feature: TTL support (`-t seconds`) for non-file-based caching (e.g., version fetches)

**ensure_* Functions - Tool Installation:**
- Location: `scripts/include_all.sh`
- Pattern: `ensure_oc()`, `ensure_govc()`, `ensure_oc_mirror()`, etc.
- Behavior: Check if tool installed; if not, trigger download via Makefile + run_once; wait for completion
- Called from: Scripts that need specific tools before proceeding

**normalize_* Functions - Configuration Validation:**
- Location: `scripts/include_all.sh`
- Examples: `normalize_aba_conf()`, `normalize_mirror_conf()`, `normalize_cluster_conf()`
- Behavior: Source config file, validate required fields, convert relative paths to absolute, reject invalid platforms
- Called from: Every workflow entry point to ensure valid state

**download_all_catalogs() / wait_for_all_catalogs():**
- Location: `scripts/include_all.sh`
- Pattern: Start 3 catalog downloads in parallel via background `run_once`, wait when needed
- Catalogs: `redhat-operator`, `certified-operator`, `community-operator` (NOT marketplace)
- Called from: Image save workflows, bundle creation

**Makefiles - File-Based Dependency Declaration:**
- Pattern: Declare output files as targets; Make only re-runs rules if targets missing or sources changed
- Usage in ABA:
  - `cli/Makefile`: Download/extract CLI binaries (file targets: `~/bin/oc`, `~/bin/openshift-install`)
  - `mirror/Makefile`: Create mirror registry, imagesets, load/save images (file targets: `mirror-registry`, `save/imageset-config-save.yaml`)
  - `bundles/v2/Makefile`: 8-stage pipeline with marker files (`.done-00-setup`, `.done-01-install-aba`, etc.)

**Symlinks - Same Paths Everywhere:**
- `aba → scripts/aba.sh`: CLI entry point accessible from any directory
- `abatui → tui/abatui.sh`: TUI entry point accessible from any directory
- Within subdirectories (e.g., `mirror/`): Symlinks to `../scripts`, `../templates`, `../cli`, `../aba.conf`
- Purpose: Allow scripts to use `scripts/...` relative paths regardless of invocation context

## Entry Points

**CLI Entry Point (`aba` symlink → `scripts/aba.sh`):**
- Location: `scripts/aba.sh` (~1600 lines)
- Triggers: User runs `aba [command] [args]`
- Responsibilities:
  - Parse `--dir`, `--debug` flags
  - Change to target directory (or stay in current dir)
  - Set `$ABA_ROOT` (only place this variable is set)
  - Source `include_all.sh`
  - Dispatch to appropriate Makefile or script based on subcommand

**TUI Entry Point (`abatui` symlink → `tui/abatui.sh`):**
- Location: `tui/abatui.sh` (~2000 lines)
- Triggers: User runs `abatui` or `./abatui`
- Responsibilities:
  - Interactive configuration wizard (dialog-based)
  - Guide user through channel → version → operators selection
  - Generate/update `aba.conf` and imageset config
  - Prevent TUI crashes from function errors (all functions return, don't exit)

**Makefile Entry Points:**
- Root `make aba`, `make cluster`, `make mirror`: High-level targets
- `make -C cli`: Download/extract CLI tools
- `make -C mirror save/load/sync/verify`: Mirror operations
- `make -C bundles/v2`: Bundle creation pipeline

**Script Entry Points:**
- `scripts/setup-mirror.sh`: Create mirror directory and configure mirror.conf
- `scripts/setup-cluster.sh`: Create cluster directory and configure cluster.conf
- `scripts/day2.sh`: Post-install cluster configuration
- `test/e2e/run.sh`: E2E test coordinator (manages vSphere/KVM pools, dispatches test suites)

## Error Handling

**Strategy:** Functions return error codes (0 = success, 1 = error); top-level scripts can abort via `aba_abort()`

**Patterns:**

1. **In functions (library code):**
   - Use `return 1` on error (never `exit`)
   - Output error message to stderr: `echo "Error: ..." >&2`
   - Callers check: `if ! function_name; then handle_error; fi`

2. **In scripts (executable code):**
   - Call functions and check return codes
   - Can use `aba_abort "message"` for critical failures (exits script + shows red error)
   - Make automatically stops on non-zero exit from called scripts

3. **In Makefiles:**
   - Non-zero exit from recipe causes Make to stop
   - Can use `set -e` at top of recipe to fail on any error
   - Use `|| true` to allow specific commands to fail without stopping

4. **TUI (special case):**
   - TUI sources `include_all.sh` but cannot afford function crashes (would kill TUI)
   - All functions called from TUI must use `return` (never `exit`)
   - TUI checks return codes and shows error dialogs

## Cross-Cutting Concerns

**Logging:**
- Function: `aba_debug()` prints to stdout if `$DEBUG_ABA=1` set
- Pattern: `aba_debug "Message"` for diagnostic output
- Files: No persistent logging in most functions; `test/e2e/` writes to `test/e2e/logs/`

**Validation:**
- Early validation: `normalize_*_conf()` functions validate config files at start
- Field-level: `verify_release_image()`, `check_cluster_installed()` validate specific states
- Preflight: `preflight-check.sh` validates environment (OS, tools, disk space, memory)

**Authentication:**
- Pull secret: `create-containers-auth.sh` formats pull secret JSON for oc-mirror
- Credentials: `regcreds/` directory holds per-mirror pull secrets and CA certs
- Platform authority: `platform=` variable in config is only way to determine active platform (vmware/kvm/none)

**Configuration Loading:**
- Single responsibility: `normalize_aba_conf()`, `normalize_mirror_conf()`, `normalize_cluster_conf()`
- Called early: Before any real work happens
- Prevents: Using file presence (e.g., `vmware.conf` exists) to infer settings
- Canonical: Config variables are always the source of truth, not file system state

**Marker File System (.available, .init, .unavailable):**
- Purpose: Make target state tracking; also used as "is task complete?" flag
- Pattern: Make rule creates `target.rule` action; at end, touches `.available` or `.unavailable` to signal completion
- Used in `mirror/Makefile`: `.init` (directory initialized), `.available` (registry ready), `.unavailable` (registry failed init)
- Cleaned by: `make clean` (remove outputs), `make reset` (remove outputs + run_once state)

## Connected-Workstation → Bundle → Disconnected-Bastion Pipeline

**Phase 1: Configuration (Connected, Internet)**
1. User runs `aba` or `abatui` to create `aba.conf`
2. User selects channel (stable/fast/candidate) and version (e.g., 4.19.1)
3. User selects operators and creates `mirror/mirror.conf`

**Phase 2: Download (Connected, Internet)**
1. CLI binaries downloaded (oc, openshift-install, oc-mirror, govc) in parallel via Make
2. Operator catalogs downloaded in background (3 catalogs: redhat-operator, certified-operator, community-operator)
3. Mirror registry tarball downloaded (Quay or Docker depending on architecture)
4. All happen via Make targets with proper dependency ordering

**Phase 3: Image Mirroring (Connected, Internet)**
1. User runs `aba -d mirror save` to create mirror archives
2. `oc-mirror` collects images matching imageset-config (releases, operators, additional images)
3. Creates tarball(s) containing images + manifests + metadata
4. Output: `mirror/save/*.tar` files (can be multi-part depending on size)

**Phase 4: Bundle Creation (Connected, Internet)**
1. User runs `aba bundle` to create transfer archive
2. `backup.sh` or `make-bundle.sh` packs everything:
   - Entire ABA repo (scripts, templates, Makefiles)
   - Downloaded CLI binaries
   - Mirror tarballs
   - Config files
3. Output: Single `aba-bundle-*.tar` file (size: 10-100+ GB depending on operator count)

**Phase 5: Transfer (Network/Physical)**
- Copy bundle to disconnected bastion via USB, network appliance, etc.
- No internet required; bundle is completely self-contained

**Phase 6: Unpack and Install (Disconnected, No Internet)**
1. Unpack bundle: `tar xf aba-bundle-*.tar`
2. CD into unpacked directory
3. Load images into registry: `aba -d mirror install && aba -d mirror load`
4. Create cluster: `aba -d cluster install`
5. All operations read from local mirrors, never reach internet

**Key Properties:**
- **Idempotent:** Bundle contains everything needed; repeated unpacking/loading is safe
- **Standalone:** Individual bundle files can be transferred separately (e.g., only `mirror/save/*.tar` on USB)
- **Verifiable:** `aba -d mirror verify` checks that all images in registry match imageset-config

## TUI Architecture

**Entry Point:** `tui/abatui.sh` (wizard-based dialog interface)

**Design:** State machine with three main screens
1. **Channel Selection** (stable/fast/candidate)
2. **Version Selection** (fetch latest, or choose specific)
3. **Operator Selection** (choose which operators to include)
4. **Summary** (review choices, apply)

**Key Pattern:** All functions return codes (never exit); TUI checks return codes and shows error dialogs

**Sourced Files:**
- `scripts/include_all.sh`: Library functions
- `tui/tui-strings.sh`: UI string constants

**Config Output:**
- Creates/updates `aba.conf` with user selections
- Creates/updates `mirror/mirror.conf` with operator selections
- Creates `mirror/data/imageset-config.yaml` ready for `aba -d mirror save`

**Safety:**
- Detects if wizard interrupted and removes incomplete config
- Logs all actions to `~/.aba/logs/aba-tui.log`
- Handles dialog cancellations gracefully

## E2E Harness Architecture

**Coordinator:** `test/e2e/run.sh` (thin dispatcher, ~1500 lines)

**Purpose:** Manage vSphere/KVM infrastructure pools; dispatch test suites; collect results

**Key Components:**

1. **Pool Management (`test/e2e/lib/pool-lifecycle.sh`):**
   - vSphere resources: Define "pools" (con1-conN connected hosts, dis1-disN disconnected hosts)
   - KVM option: Use local libvirt on single host
   - File: `test/e2e/pools.conf` defines pool topology

2. **vSphere Integration (`test/e2e/lib/vm-helpers.sh`):**
   - Govc-based VM control: power on/off, snapshots, file upload
   - Tmux session per host: conN runs `test/e2e/lib/runner.sh` for suite execution

3. **Test Suite Dispatch:**
   - Each suite (e.g., `suite-airgapped-local-reg.sh`) is a bash script
   - Copied to conN via scp before execution
   - Runs in tmux session: `tmux send-keys -t <session> "bash suite.sh" Enter`
   - Results written to `.rc` file when complete

4. **Concurrency Model:**
   - Multiple pools can run suites in parallel
   - One suite per pool at a time (pool is "busy")
   - Run.sh polls `.rc` files to detect completion
   - When pool finishes, next queued suite dispatches automatically

5. **State Files:**
   - `.rc` files in suite directory: Exit code and summary
   - `.log` files: Full stdout/stderr capture
   - `.snap` files: Golden snapshot state (for comparisons)

6. **Error Handling:**
   - Suite crashes? `.rc` file not written → timeout (wait forever). Future: add liveness check
   - Network loss? Retry scp, retry tmux commands
   - Pool broken? Mark unavailable, move to next pool

## Marker File System

**Purpose:** Track Makefile target completion and operation state

**Locations and Meanings:**

1. **In mirror directory (`mirror/.init`, `mirror/.available`):**
   - `.init`: Directory initialized with symlinks, created regcreds path
   - `.available`: Registry successfully installed and ready
   - `.unavailable`: Registry installation failed

2. **In bundle build (`bundles/v2/build/<VER>-<NAME>/.done-NN-*):**
   - `.done-00-setup`: Infrastructure verified
   - `.done-01-install-aba`: ABA repo prepared
   - `.done-02-configure`: aba.conf and imageset-config generated
   - `.done-03-image-save`: Images downloaded to disk
   - `.done-04-bundle-tar`: Bundle tarball created
   - `.done-05-offline`: Switched to offline mode for testing
   - `.done-06a-unpacked`: Bundle unpacked in test environment
   - `.done-06b-registry-installed`: Registry installed in test environment
   - `.done-06c-registry-loaded`: Images loaded into test registry
   - `.done-06d-tests-passed`: Cluster created and tests passed

3. **In run_once state (`~/.aba/runner/<task-id>/`):**
   - `.lock`: Flock-held lock file (blocks concurrent runs)
   - `.pid`: Process ID of running task
   - `.exit`: Exit code from completed task
   - `.log`: Captured output

4. **Make dependency pattern:**
   - Rule creates marker file at end: `@touch .available`
   - Later rules depend on marker: `target: .available ...`
   - Marker survives intermediate failures (rule fails, marker missing)
   - `make clean` removes markers + outputs
   - `make reset` removes markers + outputs + run_once state

---

*Architecture analysis: 2026-04-15*

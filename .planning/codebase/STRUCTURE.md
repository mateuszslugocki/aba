# Codebase Structure

**Analysis Date:** 2026-04-15

## Directory Layout

```
/home/mslugocki/forks/aba/
├── Makefile                  # Root orchestration (init, cluster, mirror, reset, clean)
├── VERSION                   # Semantic version string
├── aba                       # Symlink to scripts/aba.sh (CLI entry point)
├── abatui                    # Symlink to tui/abatui.sh (TUI entry point)
├── install                   # Bootstrap script (git clone + run aba)
├── README.md                 # User documentation
├── CHANGELOG.md              # Release notes
├── Troubleshooting.md        # Common problems and solutions
│
├── scripts/                  # Core implementation (~70 scripts, ~5000 lines total)
│   ├── aba.sh               # CLI entry point: parsing, dispatch, env setup
│   ├── include_all.sh       # Library: functions, colors, validation (~2500 lines)
│   ├── run-once.sh          # Wrapper to call run_once() from Makefiles
│   │
│   ├── setup-mirror.sh      # Create mirror directory and mirror.conf
│   ├── setup-cluster.sh     # Create cluster directory and cluster.conf
│   ├── init.sh              # Bootstrap initialization
│   ├── install-rpms.sh      # Install OS packages (libvirt, jq, etc.)
│   │
│   ├── reg-*.sh             # Registry operations (17 scripts)
│   │   ├── reg-common.sh           # Shared registry functions
│   │   ├── reg-install.sh          # Choose and install registry (quay or docker)
│   │   ├── reg-install-quay.sh     # Install mirror-registry (quay)
│   │   ├── reg-install-docker.sh   # Install docker registry
│   │   ├── reg-install-remote.sh   # Configure existing remote registry
│   │   ├── reg-save.sh             # Mirror images to disk (oc-mirror save)
│   │   ├── reg-load.sh             # Load images from disk to registry
│   │   ├── reg-sync.sh             # Sync internet → registry (oc-mirror m2m)
│   │   ├── reg-verify.sh           # Check images in registry
│   │   ├── reg-register.sh         # Register external registry
│   │   ├── reg-uninstall.sh        # Uninstall and clean registry
│   │   ├── reg-unregister.sh       # Unregister external registry
│   │   └── ... (other registry variants)
│   │
│   ├── vmw-*.sh             # VMware/vSphere operations (9 scripts)
│   │   ├── vmw-create.sh           # Create VM on vSphere
│   │   ├── vmw-delete.sh           # Delete VM
│   │   ├── vmw-exists.sh           # Check if VM exists
│   │   ├── vmw-ls.sh               # List VMs
│   │   ├── vmw-on.sh               # Power on VM
│   │   ├── vmw-kill.sh             # Power off VM
│   │   └── ... (other vSphere ops)
│   │
│   ├── kvm-*.sh             # KVM/libvirt operations (10 scripts)
│   │   ├── kvm-create.sh           # Create VM on local libvirt
│   │   ├── kvm-delete.sh           # Delete VM
│   │   ├── kvm-exists.sh           # Check if VM exists
│   │   ├── kvm-ls.sh               # List VMs
│   │   ├── kvm-on.sh               # Power on VM
│   │   └── ... (other libvirt ops)
│   │
│   ├── cluster-*.sh         # Cluster operations (7 scripts)
│   │   ├── cluster-config.sh       # Generate cluster config
│   │   ├── cluster-info.sh         # Show cluster info
│   │   ├── cluster-startup.sh      # Boot and wait for bootstrap
│   │   ├── cluster-graceful-shutdown.sh  # Shut down cluster cleanly
│   │   ├── create-cluster-conf.sh  # Create cluster.conf
│   │   └── ...
│   │
│   ├── day2-*.sh            # Day2 operations (2 scripts)
│   │   ├── day2.sh                 # Main day2 menu
│   │   ├── day2-config-ntp.sh      # Configure NTP
│   │   └── day2-config-osus.sh     # Configure OSUS
│   │
│   ├── cli-*.sh             # CLI tools operations (2 scripts)
│   │   ├── cli-download-all.sh     # Download all CLI tools
│   │   └── cli-install-all.sh      # Install all CLI tools
│   │
│   ├── Catalog/image ops:
│   │   ├── download-catalogs-start.sh      # Start parallel catalog downloads
│   │   ├── download-catalogs-wait.sh       # Wait for catalogs
│   │   ├── download-catalog-index.sh       # Download single catalog
│   │   ├── add-operators-to-imageset.sh    # Modify imageset-config
│   │   ├── list-operators.sh               # Show available operators
│   │   └── prefetch-catalogs.sh            # Prefetch for offline use
│   │
│   ├── Utility/validation:
│   │   ├── preflight-check.sh              # Check environment readiness
│   │   ├── verify-release-image.sh         # Verify release image exists
│   │   ├── verify-config.sh                # Check config completeness
│   │   ├── check-cluster-installed.sh      # Check cluster status
│   │   ├── show-cluster-login.sh           # Show kubeadmin password
│   │   ├── monitor-bootstrap.sh            # Watch bootstrap progress
│   │   ├── monitor-install.sh              # Watch installation progress
│   │   └── ...
│   │
│   ├── Bundle/transfer:
│   │   ├── make-bundle.sh                  # Create transfer bundle
│   │   ├── backup.sh                       # Tar repo for offline use
│   │   └── reset-gate.sh                   # Check before reset
│   │
│   └── Other:
│       ├── aba-get-version.sh              # Fetch version from internet
│       ├── install-govc.sh                 # Install govc tool
│       ├── install-vmware.conf.sh          # Wizard for vmware.conf
│       ├── install-kvm.conf.sh             # Wizard for kvm.conf
│       └── ...
│
├── tui/                      # Terminal UI (interactive wizard, ~2000 lines)
│   ├── abatui.sh            # TUI entry point: wizard state machine
│   └── tui-strings.sh       # UI strings (labels, messages, help text)
│
├── cli/                      # CLI binaries download + extract (Makefile-driven)
│   ├── Makefile             # Download oc, openshift-install, oc-mirror, govc
│   └── (no other files; outputs go to ~/bin)
│
├── mirror/                   # Mirror registry workflows (Makefile-driven)
│   ├── Makefile             # orchestrates: init, registry install, image save/load/sync
│   ├── mirror.conf          # Config: registry type, ports, sizing
│   ├── data/                # Working directory for mirrors
│   │   ├── imageset-config.yaml       # oc-mirror input: what to mirror
│   │   ├── imageset-config-save.yaml  # oc-mirror input for incremental saves
│   │   └── ... (oc-mirror working files)
│   └── save/                # Archival outputs (when doing mirror save)
│       ├── imageset-config-save.yaml  # Config used for save operation
│       ├── mirror_*.tar                # Image tarball outputs
│       └── ...
│
├── bundles/                  # Bundle creation pipeline
│   ├── v2/                  # Current bundle version
│   │   ├── Makefile         # 8-stage pipeline (setup, install-aba, configure, save, bundle, offline, load, test)
│   │   ├── bundle.conf      # Config: work directory, bundle naming
│   │   ├── common.sh        # Shared functions for bundle stages
│   │   ├── scripts/         # 8 stage scripts (.sh files)
│   │   │   ├── 00-setup-connectivity.sh        # Verify internet connectivity
│   │   │   ├── 01-install-aba-from-git.sh     # Clone ABA from GitHub
│   │   │   ├── 02-configure-aba-and-imageset.sh  # Run aba/abatui to config
│   │   │   ├── 03-save-images-to-disk.sh      # Run mirror save
│   │   │   ├── 04-create-bundle-tar.sh        # Tar everything
│   │   │   ├── 05-go-offline-for-testing.sh   # Disconnect network
│   │   │   ├── 06a-unpack-bundle-tar.sh       # Extract in offline env
│   │   │   ├── 06b-install-mirror-registry.sh # Install registry offline
│   │   │   ├── 06c-load-images-to-registry.sh # Load images offline
│   │   │   └── 06d-test-cluster-and-operators.sh  # Create test cluster offline
│   │   └── templates/       # Bundle configuration templates
│   │
│   └── templates/          # Bundle content templates
│
├── test/                    # Test suites (functional and E2E)
│   ├── Makefile            # Placeholder (most cleanup targets)
│   │
│   ├── func/               # Functional tests (~40 bash test scripts)
│   │   ├── run-all-tests.sh           # Test runner
│   │   ├── tui-test-lib.sh            # TUI testing helpers
│   │   ├── test-aba-root-only-in-aba-sh.sh          # Verify $ABA_ROOT usage
│   │   ├── test-backup-repo-dir-name.sh             # Check backup naming
│   │   ├── test-bundle-*.sh           # Bundle creation tests
│   │   ├── test-catalog-*.sh          # Catalog download tests
│   │   ├── test-cli-*.sh              # CLI download/install tests
│   │   ├── test-connectivity-checks.sh               # Network check tests
│   │   ├── test-docker-registry.sh                   # Docker registry tests
│   │   ├── test-e2e-*.sh              # E2E framework tests
│   │   ├── test-mirror-save-workflow.sh              # Mirror save tests
│   │   ├── test-run-once-*.sh         # run_once() function tests
│   │   ├── test-symlinks-exist.sh                    # Symlink validation
│   │   ├── test-tui-v2-*.sh           # TUI wizard tests
│   │   └── ...
│   │
│   ├── e2e/                # E2E test framework and suites
│   │   ├── run.sh                     # Coordinator: pool manager, suite dispatcher
│   │   ├── pools.conf                 # Pool topology (con/dis host definitions)
│   │   │
│   │   ├── lib/                       # E2E library (shared functions)
│   │   │   ├── constants.sh           # Pool names, timeouts, paths
│   │   │   ├── framework.sh           # Test case helpers (assert_*, setup_*, etc.)
│   │   │   ├── config-helpers.sh      # Config file generators
│   │   │   ├── pool-lifecycle.sh      # VM power management
│   │   │   ├── vm-helpers.sh          # Govc and libvirt helpers
│   │   │   ├── remote.sh              # SSH and scp helpers
│   │   │   └── setup.sh               # Environment initialization
│   │   │
│   │   ├── suites/                    # Test suite implementations
│   │   │   ├── suite-airgapped-local-reg.sh        # Full offline workflow
│   │   │   ├── suite-airgapped-existing-reg.sh     # Offline with external registry
│   │   │   ├── suite-cli-validation.sh             # CLI tool validation
│   │   │   ├── suite-config-validation.sh          # Config file validation
│   │   │   ├── suite-cluster-ops.sh                # Cluster lifecycle tests
│   │   │   ├── dummy/                              # Quick dummy suites (for testing harness)
│   │   │   └── ...
│   │   │
│   │   ├── scripts/                   # E2E helper scripts (scp'd to pools)
│   │   │   └── setup-pool-registry.sh # Install registry on pool host
│   │   │
│   │   ├── logs/                      # Test output logs
│   │   │   └── *.log                  # Suite results and summaries
│   │   │
│   │   └── examples/                  # Example test configurations
│   │
│   ├── compact/            # Compact cluster test topology (SNO)
│   ├── sno/                # Single-node cluster tests
│   ├── standard/           # Standard 3-node cluster tests
│   ├── ex/                 # Extended test configurations
│   └── misc/               # Miscellaneous test utilities
│
├── templates/              # Configuration file templates
│   ├── aba.conf           # Main config template (platform, versions, defaults)
│   ├── mirror.conf        # Mirror config template
│   ├── cluster.conf       # Cluster config template
│   ├── vmware.conf        # vSphere config template
│   ├── kvm.conf           # KVM config template
│   └── ... (other templates)
│
├── build/                  # Release and build utilities
│   ├── pre-commit-checks.sh          # Format + lint checks
│   ├── release.sh                    # Version bump and tagging
│   └── ...
│
├── rpms/                   # OS package provisioning
│   ├── Makefile           # Install RPMs based on OS version
│   └── ...
│
├── ai/                     # Architecture documentation (for Claude)
│   ├── ARCHITECTURE_VISION.md        # Future refactoring plans
│   ├── DECISIONS.md                  # Key design decisions
│   ├── OC-MIRROR-INTERNALS.md        # oc-mirror v2 source analysis
│   ├── CONCURRENCY_PROTECTION.md     # Concurrency analysis
│   ├── RUN_ONCE_RELIABILITY.md       # run_once() reliability analysis
│   ├── RULES_OF_ENGAGEMENT.md        # Development guidelines
│   ├── TUI_BUTTON_STANDARDS.md       # TUI UI/UX guidelines
│   └── ... (other design docs, ~40+ files)
│
├── images/                 # Container image utilities
│   └── ... (Podman helpers, image build configs)
│
├── tools/                  # External tools/integrations
│   └── ... (utilities, helpers)
│
├── others/                 # Miscellaneous scripts
│   └── ... (archival, experimental)
│
└── .planning/              # Codebase mapping (generated by /gsd-map-codebase)
    └── codebase/
        ├── ARCHITECTURE.md     # Architecture analysis
        ├── STRUCTURE.md        # This file
        ├── CONVENTIONS.md      # Code style guide
        ├── TESTING.md          # Test patterns
        ├── STACK.md            # Technology stack
        ├── INTEGRATIONS.md     # External services
        └── CONCERNS.md         # Tech debt and issues
```

## Directory Purposes

**scripts/:**
- Purpose: Implementation of all business logic (mirror ops, cluster setup, day2 configs, etc.)
- Contains: ~70 executable bash scripts, plus `include_all.sh` (shared library)
- Key files: `aba.sh` (CLI dispatcher), `include_all.sh` (2500 lines of reusable functions)
- Entry point: Scripts called by Makefiles or CLI
- Callable from: Anywhere via `source scripts/include_all.sh` (all functions) or direct execution

**tui/:**
- Purpose: Interactive terminal UI for configuration wizard
- Contains: TUI entry point + string constants
- Key files: `abatui.sh` (state machine), `tui-strings.sh` (UI text)
- No hard dependencies on specific infrastructure (vmware/kvm)
- Output: Creates/updates config files (`aba.conf`, `mirror/mirror.conf`, imageset config)

**cli/:**
- Purpose: Download and extract OpenShift CLI tools
- Contains: Makefile orchestrating downloads (no script files)
- Managed by: `cli/Makefile` (file-based targets)
- Outputs: `~/bin/oc`, `~/bin/openshift-install`, `~/bin/oc-mirror`, `~/bin/govc`
- Called from: `ensure_oc()`, `ensure_govc()` functions via `run_once` wrapper

**mirror/:**
- Purpose: Registry creation and image mirroring workflows
- Contains: Makefile, mirror configuration, working directory structure
- Key files: `mirror/Makefile` (main orchestrator)
- Subdir `mirror/data/`: Working directory for oc-mirror (manifests, history, configs)
- Subdir `mirror/save/`: Archive outputs from `mirror save`
- Symlinks: `mirror/scripts`, `mirror/templates`, `mirror/cli`, `mirror/aba.conf` all point to parent

**bundles/v2/:**
- Purpose: Create self-contained transfer archives for air-gapped installs
- Contains: 8-stage pipeline Makefile, stage scripts, common utilities
- Key files: `bundles/v2/Makefile`, `bundles/v2/scripts/` (00-setup through 06d-tests)
- Outputs: `.done-*` marker files, bundle tarball (size: 10-100+ GB)
- Pattern: Each stage script checks previous `.done-*` file (idempotent)

**test/func/:**
- Purpose: Unit and functional tests of ABA components
- Contains: ~40 bash test scripts, test runner
- Examples: Test run_once reliability, catalog downloads, bundle creation, TUI workflows
- Invoked by: `./run-all-tests.sh`
- No external infrastructure needed (runs on single host)

**test/e2e/:**
- Purpose: End-to-end testing in multi-host environments (vSphere pools or KVM)
- Contains: Coordinator, library, suite definitions, pool configuration
- Key files: `run.sh` (coordinator), `pools.conf` (topology), `lib/framework.sh` (test helpers)
- Suites: Full workflows (save → bundle → unpack → load → install) in isolated pools
- Infrastructure: vSphere or KVM, one pool per concurrent test, tmux session per host
- Outputs: `.rc` files (exit codes), `.log` files (test output)

**templates/:**
- Purpose: Configuration file templates copied to working directories
- Contains: Default configs for aba.conf, mirror.conf, cluster.conf, etc.
- Used by: `setup-mirror.sh`, `setup-cluster.sh` to initialize new directories

**ai/:**
- Purpose: Architecture documentation for Claude assistants
- Contains: Design decisions, analysis docs, design patterns, future plans
- NOT execution code; reference only
- Examples: `ARCHITECTURE_VISION.md`, `DECISIONS.md`, `RUN_ONCE_RELIABILITY.md`

## Key File Locations

**Entry Points:**
- `scripts/aba.sh`: CLI entry point (symlinked as `aba`)
- `tui/abatui.sh`: TUI entry point (symlinked as `abatui`)
- `install`: Bootstrap script (git clone + first run)
- `test/e2e/run.sh`: E2E test coordinator
- `test/func/run-all-tests.sh`: Functional test runner

**Configuration:**
- `aba.conf`: Main config (channel, version, platform, ask/noask)
- `mirror/mirror.conf`: Registry config (type, ports, volumes)
- `cluster.conf`: Cluster config (node count, sizing, IPs)
- `vmware.conf`: vSphere config (vCenter, datacenter, datastore)
- `kvm.conf`: KVM config (libvirt URI, network)

**Core Logic:**
- `scripts/include_all.sh`: 2500 lines of reusable functions
- `mirror/Makefile`: Mirror workflows
- `cli/Makefile`: CLI tool downloads
- `bundles/v2/Makefile`: 8-stage bundle pipeline

**Testing:**
- `test/func/*.sh`: 40+ unit/functional tests
- `test/e2e/suites/*.sh`: Integration test suites
- `test/e2e/lib/framework.sh`: Test helpers

## Naming Conventions

**Files:**
- `reg-*.sh`: Registry operations (reg-save.sh, reg-load.sh, reg-sync.sh, etc.)
- `kvm-*.sh`: KVM/libvirt operations (kvm-create.sh, kvm-delete.sh, etc.)
- `vmw-*.sh`: VMware/vSphere operations (vmw-create.sh, vmw-delete.sh, etc.)
- `cluster-*.sh`: Cluster operations (cluster-config.sh, cluster-startup.sh, etc.)
- `day2-*.sh`: Day2 operations (day2.sh, day2-config-ntp.sh, etc.)
- `download-*.sh`: Catalog/image downloads (download-catalogs-start.sh, etc.)
- `ensure-*.sh` / `install-*.sh`: Tool installation (ensure-cli.sh, install-rpms.sh, etc.)
- `test-*.sh`: Functional tests (test-run-once-reliability.sh, test-catalog-helpers.sh, etc.)
- `suite-*.sh`: E2E test suites (suite-airgapped-local-reg.sh, etc.)

**Directories:**
- `mirror/`, `cluster/`: Per-operation working directories (created by `setup-mirror.sh`, `setup-cluster.sh`)
- `mirror/data/`: oc-mirror working files (manifests, history, configs)
- `mirror/save/`: Image archive outputs
- `~/.aba/`: User state directory (cache, logs, runner, mirrors)
  - `~/.aba/cache/`: Downloaded catalogs, cached versions
  - `~/.aba/runner/`: run_once() state files (lock, pid, exit, log)
  - `~/.aba/logs/`: TUI and E2E logs
  - `~/.aba/mirror/<name>/`: Per-mirror registry credentials

**Config Variables:**
- `platform=`: Active platform (vmware, kvm, or empty for local)
- `channel=`: Release channel (stable, fast, candidate)
- `version=`: OpenShift version (e.g., 4.19.1)
- `ask=`: Auto-accept prompts (true/false)

## Where to Add New Code

**New Mirror Operation (e.g., "mirror verify enhancement"):**
- Primary code: `scripts/reg-verify.sh` (modify existing or create new)
- Library functions: Add to `scripts/include_all.sh` if reusable
- Make target: Add to `mirror/Makefile` if needs file dependencies
- Tests: `test/func/test-mirror-verify.sh` (new functional test)
- E2E: Add step to `test/e2e/suites/suite-airgapped-*.sh` if critical path

**New Platform Support (e.g., "add AWS native VMs"):**
- Platform detection: Modify config templates, add `aws.conf` template
- VM operations: Create `scripts/aws-*.sh` (parallel to kvm-*.sh, vmw-*.sh)
- Entry point: Dispatch in `scripts/aba.sh` or add new Make target
- Tests: Create functional tests + E2E suite with AWS pool

**New Day2 Configuration:**
- Script: Create `scripts/day2-config-*.sh` (parallel to ntp, osus)
- Menu: Add to `scripts/day2.sh` menu
- Library: Add functions to `scripts/include_all.sh` if reusable
- Tests: Create test in `test/func/` and `test/e2e/` if infrastructure-dependent

**New Operator or Additional Image:**
- Config change: Edit `mirror/data/imageset-config.yaml`
- Verification: Run `aba -d mirror verify` to check if images exist
- Bundle: If including in bundle, rebuild via `bundles/v2/Makefile`

**New Test Suite:**
- Location: `test/e2e/suites/suite-*.sh`
- Library: Use existing helpers from `test/e2e/lib/framework.sh`
- Registration: Add to `run.sh` suite list (auto-detected by naming)
- Infrastructure: Define required pool topology in `test/e2e/pools.conf`

**New Functional Test:**
- Location: `test/func/test-*.sh`
- Library: Use helpers from `test/func/tui-test-lib.sh` if TUI-related
- Runner: Auto-detected by `test/func/run-all-tests.sh` (no registration needed)
- No infrastructure: Runs on single host

## Special Directories

**~/.aba/:**
- Purpose: User state directory (persistent across runs)
- Generated: Yes (created as needed)
- Committed: No (not in git)
- Cleanup: `make reset` with `aba reset -f` does global cleanup

**mirror/:**
- Purpose: Per-mirror working directory
- Generated: Yes (created by `setup-mirror.sh`)
- Committed: No (not in git; contains credentials)
- Multiple: Can have `mirror/`, `mirror-2/`, `mirror-prod/`, etc. (each named via `aba mirror --name`)

**cluster/:**
- Purpose: Per-cluster working directory
- Generated: Yes (created by `setup-cluster.sh`)
- Committed: No (not in git; contains cluster configs, credentials)
- Multiple: Can have `cluster/`, `cluster-prod/`, `cluster-test/`, etc. (each named via `aba cluster --name`)

**test/e2e/logs/:**
- Purpose: Test output capture and results
- Generated: Yes (during test runs)
- Committed: No (logs are large, not needed for reproduction)
- Content: `.log`, `.rc` files per suite per pool

**bundles/v2/build/:**
- Purpose: Bundle creation working directory
- Generated: Yes (created during `make VER=... NAME=...`)
- Committed: No (large, temporary)
- Structure: `<VER>-<NAME>/build/` with `.done-*` marker files

---

*Structure analysis: 2026-04-15*

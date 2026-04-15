# Technology Stack

**Analysis Date:** 2026-04-15

## Languages

**Primary:**
- Bash (4.x+) - Core orchestration, 12,974 total lines across 100+ scripts in `scripts/`, build utilities, and installation logic
- Make (GNU Make 4.x+) - Build/task orchestration via nested Makefiles: `Makefile` (root), `mirror/Makefile`, `cli/Makefile`, `rpms/Makefile`, `bundles/v2/Makefile`, `build/Makefile`, `test/Makefile`

**Secondary:**
- Python 3.x - Jinja2 template rendering for configuration generation
- YAML - Configuration files and OpenShift manifests (generated from Jinja2 templates)
- JSON - Secrets (pull-secret.json) and API payloads
- Shell script sourcing - Shared function library in `scripts/include_all.sh`

## Runtime

**Environment:**
- RHEL 8.x, 9.x (primary)
- CentOS Stream
- Fedora (latest versions, typically 39+)
- UBI 9 container image (for containerized deployment via `build/Containerfile`)

**Note:** macOS NOT supported (no oc-mirror available). Checked explicitly in `scripts/aba.sh:37` and `install:10`.

**Package Manager:**
- DNF (RHEL 8/9, CentOS Stream, Fedora)
- microdnf (UBI 9 containers)

**Lockfile:** None (bash/Make inherently lock package versions via downloaded tool tarballs)

## Frameworks & Core Tools

**CLI Tools (downloaded dynamically):**
- `oc` (oc client) - OpenShift CLI, version matched to OCP release, downloaded from `mirror.openshift.com`
- `openshift-install` - Agent-Based Installer (ABI), version matched to OCP release, downloaded from `mirror.openshift.com`
- `oc-mirror` v2 - Image mirroring tool, stable-4.21 branch from `mirror.openshift.com/pub/openshift-v4/[arch]/clients/ocp/stable-4.21`
- `butane` - Ignition config generator, downloaded from `mirror.openshift.com/pub/openshift-v4/clients/butane/latest`
- `govc` - VMware vSphere/ESXi API client, conditionally required for `platform=vmw`, downloaded from `github.com/vmware/govmomi/releases`

**Platform Tools:**
- Container Runtime: `podman` - Required for image extraction/inspection during catalog processing
- Image Tools: `skopeo` - OCI/Docker image inspection (used for container image operations)
- HTTP Tools: `curl` - All artifact downloads; `httpd-tools` (htpasswd) for registry auth
- Jinja2 Templating: `python3-jinja2` - Config file generation from `templates/*.j2`
- YAML Processing: `python3-pyyaml` - Configuration parsing and validation

**Build/Release:**
- `git` - Version control, clone/tag operations in `build/release.sh`
- `jq` - JSON parsing for registry credentials and API responses
- `ncurses` - TUI support (`tui/abatui.sh`)
- `dialog` - Interactive prompts for config values
- `make` (v4.x+) - Primary orchestration
- `file`, `hostname`, `diffutils`, `which` - System utilities for compatibility checks

## Key Dependencies

**Critical Runtime:**
- `python3` - Jinja2 template rendering for `aba.conf`, `mirror.conf`, `cluster.conf`, install-config, agent-config
- `podman` - Image handling during catalog downloads (uses podman extraction rather than oc-mirror)
- `skopeo` - Image registry authentication and metadata inspection
- `curl` - All network operations for CLI tool downloads, checksum verification (uses `--retry 8` for resilience)

**External RPM Sources:**
- From `templates/rpms-external.txt` (connected environment):
  ```
  make jq python3 python3-jinja2 python3-pyyaml ncurses which diffutils dialog podman httpd-tools skopeo
  ```
- From `templates/rpms-internal.txt` (disconnected environment):
  ```
  make jq python3 python3-jinja2 python3-pyyaml ncurses which file hostname diffutils podman bind-utils nmstate net-tools skopeo openssl coreos-installer httpd
  ```
- **Additional notes:** DNF module `subscription-manager` plugin disabled in containers (`microdnf --disableplugin=subscription-manager`)

**Container Deployment:**
- Base image: `registry.access.redhat.com/ubi9/ubi-minimal:latest` (defined in `build/Containerfile`)
- Container tools installed: oc, openshift-install, oc-mirror, butane
- Python packages: nmstate (built from source in container)

## Configuration Files & Schemas

**Global Configuration:**
- `aba.conf` (Jinja2 template: `templates/aba.conf.j2`) - Rendered at first run, sets:
  - OCP version/channel (`ocp_version`, `ocp_channel`)
  - Deployment platform (`platform=vmw|kvm|bm`)
  - Operator sets (`op_sets`, `ops`)
  - Network values (`domain`, `machine_network`, `dns_servers`, `next_hop_address`, `ntp_servers`)
  - Registry paths and credentials (pulled from mirror.conf, not this file)
  - Pull secret location (`pull_secret_file=~/.pull-secret.json`)
  - Advanced settings: `excl_platform`, `verify_conf`, `oc_mirror_version`

**Mirror Configuration:**
- `mirror/mirror.conf` (Jinja2 template: `templates/mirror.conf.j2`) - Per-mirror registry setup:
  - Registry FQDN, port, path, credentials (`reg_host`, `reg_port`, `reg_path`, `reg_user`, `reg_pw`)
  - Registry vendor (`reg_vendor=auto|quay|docker`)
  - Storage path (`data_dir`)
  - Remote SSH access (`reg_ssh_key`, `reg_ssh_user`)
  - Operator overrides (`op_sets`, `ops`)

**Cluster Configuration:**
- `cluster.conf` (Jinja2 template: `templates/cluster.conf.j2`) - Per-cluster deployment:
  - Cluster name, base domain, node counts
  - Network: `machine_network`, `starting_ip`, `api_vip`, `ingress_vip`, DNS, NTP servers
  - vSphere/ESXi (if `platform=vmw`): MAC prefix, CPU/memory per node type, data disk
  - KVM/libvirt (if `platform=kvm`): (handled in separate kvm.conf)
  - SSH key paths, proxy settings, mirror directory reference

**Platform-Specific Configs:**
- `vmware.conf` (template: `templates/vmware.conf`) - govc environment:
  - `GOVC_URL`, `GOVC_USERNAME`, `GOVC_PASSWORD`
  - `GOVC_DATASTORE`, `GOVC_NETWORK`, `GOVC_DATACENTER`, `GOVC_CLUSTER`
  - vCenter folder, resource pool, cert validation (`GOVC_INSECURE`)
  - Stored at `~/.aba/ssh.conf` (SSH config), user-provided for vSphere auth

- `kvm.conf` (template: `templates/kvm.conf`) - libvirt environment:
  - `LIBVIRT_URI=qemu+ssh://kvm-user@kvmhost.lan/system`
  - `KVM_STORAGE_POOL`, `KVM_NETWORK`

**OpenShift Configuration Schemas:**
- `install-config.yaml` (template: `templates/install-config.yaml.j2`) - ABI input (Agent-Based Installer)
- `agent-config.yaml` (template: `templates/agent-config.yaml.j2`) - Node provisioning details:
  - Network bonding variants: `agent-config-bond.yaml.j2`, `agent-config-vlan.yaml.j2`, `agent-config-vlan-bond.yaml.j2`
  - Optional NTP extension: `agent-config-vlan-bond.yaml.j2.with.NTPSource`
- `imageset-config.yaml` (template: `templates/imageset-config.yaml.j2`) - oc-mirror v2 ImageSetConfiguration:
  ```yaml
  kind: ImageSetConfiguration
  apiVersion: mirror.openshift.io/v2alpha1
  mirror:
    platform:
      architectures: [if non-amd64]
      channels: [OpenShift release channel]
    operators: [from operator sets]
  ```
- `image-content-sources.yaml` (template: `templates/image-content-sources.yaml.j2`) - ImageContentSourcePolicy for cluster
- `cm-additional-trust-bundle.j2` - CA certificate ConfigMap for disconnected registry

**Secrets & Authentication:**
- `pull-secret.json` (required user-provided from Red Hat) - Path: `~/.pull-secret.json`
- `pull-secret-mirror.json` (template: `templates/pull-secret-mirror.json.j2`) - Generated for internal mirror registry
- `~/.aba/ssh.conf` - SSH configuration (generated by `install` script):
  ```
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
  ConnectTimeout=15
  BatchMode=yes
  LogLevel=ERROR
  ```
- `~/.aba/config` - User-level ABA config (template: `templates/aba-home-config`)
- `~/.config/containers/registries.d/aba-sigstore.yaml` - Sigstore signature handling (template: `templates/aba-sigstore-config.yaml`)

**Operator Definitions:**
- `templates/operator-set-*` files define operator inclusion lists:
  - `operator-set-ocp` - OpenShift core operators
  - `operator-set-acm`, `operator-set-virt`, `operator-set-odf`, `operator-set-odfdr`, `operator-set-quay`
  - `operator-set-mesh2`, `operator-set-mesh3`, `operator-set-appdev`, `operator-set-ai`
  - `operator-set-sec`, `operator-set-dell`, `operator-set-gpu`, `operator-set-logging`
  - Format: operator names listed (used by oc-mirror to populate ImageSetConfiguration)

## Build & Release Tooling

**Release Workflow:**
- `build/release.sh` - Full release lifecycle:
  - Validates inputs, updates VERSION file, bumps ABA_VERSION in `scripts/aba.sh`
  - Creates annotated git tags, pushes to GitHub, creates GitHub releases
  - Supports `--dry-run`, `--ref <commit>` (release from past commit), `--hotfix` (from main)
  - Requires gh CLI (`github.com/cli/cli`)

**Pre-commit Checks:**
- `build/pre-commit-checks.sh` - Runs before release:
  - Stamps ABA_BUILD timestamp (YYYYMMDDHHMMSS format) into `scripts/aba.sh`
  - Syncs RPM lists from `templates/rpms-external.txt` and `templates/rpms-internal.txt` into `install` script
  - Syntax-checks all .sh files with `bash -n`
  - Verifies branch (dev) and pulls latest from remote
  - Exit code 0 = success, 1 = failure

**Version Management:**
- `VERSION` file - Human-readable release version
- `scripts/aba.sh:ABA_VERSION` - Semantic version (updated at release)
- `scripts/aba.sh:ABA_BUILD` - Build timestamp (updated at every pre-commit check)
- Version validation in `scripts/aba.sh:30`: ABA_BUILD must match format `^[0-9]{14}$`

**Container Build:**
- `build/Containerfile` - Multi-stage UBI 9 container with:
  - OpenShift client tools (oc, openshift-install, oc-mirror from stable-4.21)
  - Python (with Jinja2, PyYAML)
  - podman, skopeo, nmstate, bind-utils, jq, curl, git
  - User 1001 as non-root
  - WORKDIR: `/aba` (bind-mounted repo)
  - ENTRYPOINT: `/usr/local/bin/default-cmd.sh`

## Installation & Setup

**Automatic Installation:**
- `./install` script - Checks system requirements, installs missing RPMs via dnf, installs aba to $PATH
  - Supports `-q` flag for quiet mode (used internally by aba for auto-update checks)
  - Checks for required packages from merged external/internal RPM lists
  - Deploys `~/.aba/ssh.conf` and `~/.aba/config`
  - Deploys sigstore config to `~/.config/containers/registries.d/aba-sigstore.yaml`
  - Installs to first available dir in $PATH priority: `~/bin`, `/usr/local/bin`, `/usr/local/sbin`, `/usr/bin`, `/usr/sbin`

**Post-Install State:**
- `~/.aba/` directory structure:
  - `ssh.conf` - SSH client config (created)
  - `config` - User config (created if missing)
  - `runner/` - Task state (created by run_once, cleaned on fresh install)
  - `cache/` - Downloaded artifacts (cleaned on fresh install)
  - `mirror/` - Per-mirror credentials/state (created by mirror init)

## Platform Requirements

**Development/Connected Workstation:**
- OS: RHEL 8/9, CentOS Stream, Fedora
- Minimum: 4 CPU, 8 GB RAM (for podman and tool caching)
- Disk: 50 GB free (tool downloads, oc-mirror workspace)
- Network: Internet access for artifact downloads
- Required packages synced from `templates/rpms-external.txt`

**Disconnected Bastion:**
- OS: RHEL 8/9, CentOS Stream, Fedora (same as workstation)
- Minimum: 4 CPU, 16 GB RAM (registry + ABA repo)
- Disk: Depends on operator selection; OpenShift platform alone: ~50 GB; with operators: 100+ GB
  - Quay appliance uses: `$data_dir/quay-install`
  - oc-mirror cache: `$data_dir/.oc-mirror`
  - Mirror TAR files: `mirror/data/mirror_*.tar` (sequential numbering)
- Required packages synced from `templates/rpms-internal.txt`

**Deployment Targets:**
- vSphere/ESXi (via govc CLI)
- KVM/libvirt (via SSH + libvirt URI)
- Bare-metal (via Ironic or manual PXE boot from agent ISO)

---

*Stack analysis: 2026-04-15*

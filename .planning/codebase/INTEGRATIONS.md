# External Integrations

**Analysis Date:** 2026-04-15

## APIs & External Services

**Red Hat Registry:**
- `registry.redhat.io` - Official Red Hat container images
  - Used for: OpenShift release images, operator images, UBI base images
  - Auth: Red Hat pull-secret (required; sourced from `https://console.redhat.com/openshift/downloads#tool-pull-secret`)
  - Validation: `scripts/aba.sh:1531-1561` checks pull-secret auth against registry.redhat.io via curl
  - Env var: `pull_secret_file` in `aba.conf` (default: `~/.pull-secret.json`)

**Quay.io (Community/OpenShift Release):**
- `quay.io/openshift-release-dev` - OpenShift release images and operators
  - Auth: Included in Red Hat pull-secret
  - Sigstore signatures: Enabled for this registry in `templates/aba-sigstore-config.yaml`
  - Used by: oc-mirror v2 for platform and operator mirroring

**Docker Hub & Other Public Registries:**
- `docker.io` - Source for Docker registry image if Quay mirror-registry unavailable
  - File: `docker-reg-image.tgz` (best-effort download, warning if missing)
  - Fallback: On arm64 where mirror-registry (Quay appliance) not available
  - Location: `mirror/` directory

**OpenShift Mirror Server:**
- `mirror.openshift.com` - Official CLI tool downloads
  - oc client: `https://mirror.openshift.com/pub/openshift-v4/[arch]/clients/ocp/[version]/`
  - openshift-install: `https://mirror.openshift.com/pub/openshift-v4/[arch]/clients/ocp/[version]/`
  - oc-mirror v2: `https://mirror.openshift.com/pub/openshift-v4/[arch]/clients/ocp/stable-4.21/`
  - butane: `https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/`
  - Checksums: `sha256sum.txt` fetched alongside each tarball
  - Downloads: `cli/Makefile` orchestrates all downloads with retries (`--retry 8`)

**GitHub (VMware govmomi):**
- `github.com/vmware/govmomi/releases/latest` - govc CLI binary
  - File: `govc_Linux_[amd64|arm64].tar.gz`
  - Checksums: `checksums.txt` from same release
  - Conditionally required: Only when `platform=vmw` in `aba.conf`
  - Download: `cli/Makefile` (lines 293-307)

**Cincinnati (OpenShift Update Service):**
- API: `https://api.openshift.com/api/upgrades_info/v1/graph`
- Purpose: Fetch latest OCP version for selected channel
- Used by: `scripts/aba.sh` to determine target version if not specified in `aba.conf`
- Error handling: Validates DNS and network connectivity (`scripts/aba.sh:1322-1327`)

## Data Storage

**Databases:**
- None (ABA is stateless orchestration layer)
- Configuration stored as text files (aba.conf, mirror.conf, cluster.conf)
- Credentials stored as JSON (pull-secret.json) or environment variables

**File Storage:**
- Local filesystem only for bundle creation
  - Mirror TAR files: `mirror/data/mirror_*.tar` (sequential, numbered 000001-999999)
  - oc-mirror workspace: `mirror/data/.oc-mirror/` (internal cache)
  - Quay appliance data: `$data_dir/quay-install/` (user-configurable)
  - Docker registry data: `$data_dir/.aba/docker-registry/` (if used)
- SSH remote deployment: Registry installation can target remote host via `reg_ssh_key` and `reg_ssh_user` in mirror.conf

**Caching:**
- oc-mirror workspace: `mirror/data/.oc-mirror/` - Caches image blobs downloaded during save/sync/load
- Run-once task state: `~/.aba/runner/` - Tracks task completion across sessions
- ABA cache: `~/.aba/cache/` - Caches catalog indexes downloaded from `registry.redhat.io`

## Authentication & Identity

**Auth Provider:**
- Custom: Pull-secret-based authentication
  - Format: Standard Docker config JSON (base64-encoded auth)
  - Sources:
    1. Red Hat pull-secret: `~/.pull-secret.json` (user-provided, required)
    2. Mirror pull-secret: `templates/pull-secret-mirror.json.j2` (auto-generated for internal registry)
  - Deployment: `scripts/reg-register.sh` copies/merges secrets to `regcreds/` directory in mirror

**SSH Authentication (for remote hosts):**
- KVM/libvirt: Passwordless SSH access via libvirt URI (`LIBVIRT_URI=qemu+ssh://user@host/system`)
- Remote registry: SSH key path in mirror.conf (`reg_ssh_key=~/.ssh/id_rsa`)
- SSH config: Deployed to `~/.aba/ssh.conf` by `install` script:
  ```
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
  ConnectTimeout=15
  BatchMode=yes
  LogLevel=ERROR
  ```

**vSphere/ESXi Authentication:**
- Via `templates/vmware.conf` (user-provided, not version-controlled):
  - `GOVC_URL` - vCenter/ESXi host
  - `GOVC_USERNAME`, `GOVC_PASSWORD` - Credentials
  - `GOVC_INSECURE=true` - Disable cert validation (for labs/test environments)
- Credentials never stored in aba.conf (requires external vmware.conf)

## Monitoring & Observability

**Error Tracking:**
- None built-in (no SaaS integration)
- Local error logging via helper functions in `scripts/include_all.sh`:
  - `aba_abort()`, `aba_error()`, `aba_info()`, `aba_debug()`
  - Errors written to stderr; debug to stdout if `DEBUG_ABA=1`

**Logs:**
- Approach: File-based, local archival
  - Installation errors: `.dnf-install.log` in repo root
  - Tool execution: stdout/stderr from `scripts/` (captured by `run_once`)
  - Test harness: `test/e2e/` creates per-scenario log files in tmux sessions
  - Mirror operations: `reg-save.sh`, `reg-sync.sh`, `reg-load.sh` output to console
- Retention: Not automatically managed; user responsibility

**Signature Verification:**
- Sigstore (cosign) verification for container images
  - Config: `~/.config/containers/registries.d/aba-sigstore.yaml` (template: `templates/aba-sigstore-config.yaml`)
  - Enabled for: `quay.io/openshift-release-dev`, `registry.redhat.io`
  - Disabled globally by default; re-enabled per-registry
  - Used by: podman and oc-mirror during image pulls
  - Tool: podman built-in support (no external binary needed)

## CI/CD & Deployment

**Hosting:**
- No cloud hosting (ABA runs on user's infrastructure)
- GitHub: Repository hosting (github.com/sjbylo/aba)
- Mirror registries: User-managed (Quay appliance or Docker registry)

**CI Pipeline:**
- GitHub Actions (not visible in codebase) - Assumed for automated testing/release
- Pre-commit hooks: `build/pre-commit-checks.sh` (syntax validation, version stamping)
- Release automation: `build/release.sh` (version bumps, tag creation, GitHub release)

**Deployment Mechanisms:**
1. **Bundle-based (Connected -> Disconnected):**
   - `aba bundle` or `aba tar` creates portable archive
   - Bundle contents: CLI tools, mirror tarballs, operator catalogs, ABA repo
   - Transfer: Physical media or sneaker-net to disconnected environment
   - Unpacking: Manual extraction of bundle on bastion

2. **Mirror Registry Installation:**
   - Quay mirror-registry appliance: Downloaded from `mirror.openshift.com/pub/openshift-v4/[arch]/clients/mirror-registry/`
   - Docker registry: Downloaded from `docker.io/library/registry` or `docker-reg-image.tgz`
   - Installation: `scripts/reg-install.sh` (configures ports, credentials, persistent storage)
   - Remote installation: Via SSH (`reg_ssh_key`, `reg_ssh_user` in mirror.conf)

3. **OpenShift Agent-Based Installer (ABI):**
   - Entry point: `openshift-install` binary (downloaded from mirror.openshift.com)
   - Config inputs: `install-config.yaml`, `agent-config.yaml` (rendered from Jinja2 templates)
   - Operator images: Provided via `imageset-config.yaml` (oc-mirror v2 format)
   - Image content sources: Injected via `ImageContentSourcePolicy` in agent-config
   - Execution: `scripts/create-cluster-*.sh` invokes `openshift-install agent create cluster-manifests`

## Environment Configuration

**Required Env Vars (from aba.conf):**
- `ocp_version` - OpenShift version (e.g., 4.18.9)
- `ocp_channel` - Release channel (stable, fast, candidate, eus)
- `pull_secret_file` - Path to Red Hat pull-secret JSON
- `domain` - Base domain for cluster DNS
- `machine_network` - Network CIDR (e.g., 192.168.1.0/24)
- `dns_servers` - Comma-separated DNS server IPs
- `ntp_servers` - Comma-separated NTP servers or hostnames (critical if system clocks unsync'd)
- `next_hop_address` - Default gateway IP
- `platform` - Deployment target: vmw, kvm, or bm

**Optional Env Vars:**
- `op_sets` - Comma-separated operator set names (ocp, acm, virt, odf, appdev, mesh2, mesh3, odfdr, quay, sec, ai, logging, dell, gpu)
- `ops` - Individual operator names
- `excl_platform` - Exclude platform images from mirror (default: false)
- `verify_conf` - Validation level: all (default), conf, off
- `oc_mirror_version` - Hardcode oc-mirror v2 (defaults to v2 if unset)

**Secrets Location:**
- `~/.pull-secret.json` - Red Hat pull-secret (user-provided, not version-controlled)
- `vmware.conf` (user-provided) - vSphere/ESXi credentials
- `kvm.conf` (user-provided) - KVM SSH connection details
- `~/.aba/mirror/[mirror-name]/` - Per-mirror credentials (auto-generated or registered)
- `~/.aba/ssh.conf` - SSH client config (deployed by install script)

## Webhooks & Callbacks

**Incoming:**
- None (ABA is pull-only; no inbound webhooks)

**Outgoing:**
- GitHub releases: Created via gh CLI by `build/release.sh` (POST to `api.github.com/repos/sjbylo/aba/releases`)
- No external notifications or alerting

## oc-mirror v2 Integration

**ImageSetConfiguration Flow:**
1. User defines operators via `op_sets` or `ops` in aba.conf/mirror.conf
2. `scripts/add-operators-to-imageset.sh` expands operators to full YAML definitions
3. `templates/imageset-config.yaml.j2` renders ImageSetConfiguration with:
   - Platform images (OpenShift release + graph dependencies)
   - Operator catalogs (from `registry.redhat.io/redhat/*-index:v[major-version]`)
4. oc-mirror v2 execution:
   - `aba -d mirror save` - Downloads images to local disk (`mirror/data/mirror_*.tar`)
   - `aba -d mirror sync` - Streams images directly to internal registry (faster, no tar)
   - `aba -d mirror load` - Extracts tar files to internal registry (disconnected bastion)
5. Generated manifests:
   - `mirror/data/imageset-config.yaml` - Complete computed config
   - `mirror/data/image-content-sources.yaml` - ImageContentSourcePolicy (copied to agent-config)

**v2-Specific Notes** (from `templates/aba.conf.j2:44-53`):
- v2 requires all operator dependencies to be explicitly included
  - Example: `web-terminal` requires `devworkspace-operator` (both in `operator-set-ocp`)
- TAR files AND YAML files both required on bastion during load (v2 needs YAML for image references)
- Error reporting differs from v1; recommend verifying mirrored operators in Quay via UI
- Use `--retry` flag for resilience on network failures

## OperatorHub & CatalogSources

**Catalog Sources:**
- Redhat Operator Index (primary):
  - Catalog: `registry.redhat.io/redhat/redhat-operator-index:v[OCP-major-version]`
  - Used by: `scripts/add-operators-to-imageset.sh` to fetch operator metadata
  - Downloaded during: `aba mirror save/sync` (catalog download phase)

- Operator catalogs (per-operator):
  - Base: `registry.redhat.io/redhat/[operator-name]-index:v[OCP-major-version]`
  - Examples: `openstack-index`, `cnv-index`, `ocs-index`, `acm-operator-index`
  - Fetched when operator included in `op_sets`

**Day-2 Configuration:**
- `scripts/day2-config-osus.sh` - Post-install OperatorHub setup
  - Patches CatalogSources in OpenShift to point to internal mirror registry
  - Updates ImageContentSourcePolicy if not already present
  - Handles OSUS (OpenShift Update Service) subscription verification
  - Requires: Cluster connectivity via oc CLI + kubeconfig

- `scripts/day2-config-ntp.sh` - Post-install NTP configuration
  - Configures control plane nodes as NTP servers
  - Allows upstream NTP client connections
  - Validates all nodes have correct NTP sources configured

## Networking & Infrastructure Requirements

**DNS Requirements:**
- Cluster DNS (A records):
  - `api.<cluster-name>.<base-domain>` -> api_vip (API endpoint)
  - `*.apps.<cluster-name>.<base-domain>` -> ingress_vip (app ingress wildcard)
  - Registry FQDN: `<reg_host>` (e.g., registry.example.com:8443)
- Validation: `scripts/aba.sh` checks DNS resolution during init
- Tools: `bind-utils` (dig, host, nslookup) for troubleshooting

**NTP Requirements:**
- `ntp_servers` setting in aba.conf critical if system clocks unsync'd
- Used by: Agent bootstrap (`add_ntp_ignition_to_iso.sh`), day-2 config (`day2-config-ntp.sh`)
- Format: Comma-separated hostnames or IPs (NTP clients resolve to IPs automatically)
- Validation: `scripts/day2-config-ntp.sh` verifies all nodes reach configured targets

**Network Connectivity:**
- Connected workstation: Full internet access for artifact downloads
- Disconnected bastion: No outbound internet (air-gapped); internal registry access only
- Registry access: HTTPS (port 8443 for Quay, configurable for Docker registry)
- vSphere/ESXi: Network access from workstation to vCenter/ESXi API (port 443)
- KVM/libvirt: SSH access from workstation to KVM host (port 22 by default)

**Storage Considerations:**
- Quay appliance: Minimum 20 GB (platform) to 200+ GB (with operators)
  - Location: `$data_dir/quay-install/` (user-configurable in mirror.conf)
  - Persistent: Requires persistent storage between bastion reboots
- oc-mirror cache: `$data_dir/.oc-mirror/` - Temporary, can be pruned post-sync
- Bundle archive: Depends on operator selection; platform-only ~50-100 GB

## Red Hat Ecosystem Integration

**Pull-Secret Source:**
- URL: `https://console.redhat.com/openshift/downloads#tool-pull-secret`
- Required: RHEL Subscription or OpenShift trial account
- Format: JSON with auths section containing base64-encoded credentials for registry.redhat.io
- Validation: Checked against registry.redhat.io during `aba init` (scripts/aba.sh:1535)

**OpenShift Versions:**
- Channel support: stable, fast, candidate, eus (from Red Hat documentation)
- Version source: Cincinnati API (openshift.com) for auto-detection
- Release images: Mirrored from quay.io/openshift-release-dev and registry.redhat.io

**Operator Index Updates:**
- Operator catalogs updated by Red Hat asynchronously
- ABA mirrors snapshot at bundle creation time
- Day-2 operator additions: Require re-running `aba -d mirror load` with new imageset-config.yaml

---

*Integration audit: 2026-04-15*

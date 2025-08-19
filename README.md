# Raspberry Pi ClusterHAT Automation

Automated provisioning for a hybrid Raspberry Pi ClusterHAT environment (Pi Zero USB-boot nodes + standard Pis + optional management/storage Pi5) using Ansible.

---
## Contents
- [Architecture](#architecture)
- [Features](#features)
- [Inventory & Topology](#inventory--topology)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Playbooks](#playbooks)
- [Feature Flags](#feature-flags)
- [Core Variables](#core-variables)
- [Package Management Model](#package-management-model)
- [Slurm Configuration & Aliases](#slurm-configuration--aliases)
- [USB Boot Provisioning Flow](#usb-boot-provisioning-flow)
- [Roles Overview](#roles-overview)
- [Secrets Handling](#secrets-handling)
- [Common Operations](#common-operations)
- [Troubleshooting](#troubleshooting)
- [Roadmap / Ideas](#roadmap--ideas)
- [Detailed Variable Reference](#detailed-variable-reference)
- [k3s Cluster Details](#k3s-cluster-details)
- [Slurm Usage Examples](#slurm-usage-examples)
- [NFS Layout & Mounts](#nfs-layout--mounts)
- [Security Hardening Checklist](#security-hardening-checklist)
- [Performance & Scaling Notes](#performance--scaling-notes)
- [Customization Patterns](#customization-patterns)
- [Maintenance & Updates](#maintenance--updates)
- [Contributing](#contributing)
- [Glossary](#glossary)
- [Changelog Template](#changelog-template)

---
## Architecture
A single controller host (physical hostname `clusterhat`) running ClusterCTRL CNAT image manages:
- 4 Pi Zero nodes (`z1`..`z4`) via ClusterHAT v2.5 USB boot with CNAT networking (172.19.181.0/24)
- Optional Pi cluster nodes (`p1`..`p4`) on main LAN
- Optional management/storage node (`p5`, typically Pi5 with NFS export)
- Software stack includes optional Slurm workload manager, Docker runtime, and k3s (lightweight Kubernetes)

```
            +---------------------------+
            |        clusterhat         |  (controller Pi 4 with CNAT image)
            |  IP: 172.16.2.200 (LAN)   |  (aliases: controller, p0)
            |  USB Gadget: 172.19.181.254|
            +--+----+----+----+----+----+
               |    |    |    |    | USB (ClusterHAT v2.5)
               |    |    |    |
             z1   z2   z3   z4    (Pi Zero USB boot rootfs via CNAT)
       172.19.181.1-4              (automatically assigned by CNAT)

  LAN (172.16.2.0/24)
   p1  p2  p3  p4        (optional compute / k3s agents / Docker)
   p5  (k3s server + NFS server + management)
```

---
## Features
| Capability | Status | Notes |
|------------|--------|-------|
| USB boot prep for Pi Zero nodes | ✅ | Direct per-node extract + `usbboot-init` + idempotent markers |
| Slurm (controller + nodes) | ✅ | Aliases: `clusterhat`, `controller`, `p0` |
| Docker runtime | ✅ | Optional on zeros via `docker_runtime_on_zeros` |
| k3s cluster | ✅ | Server on `p5` (configurable), agents on others |
| NFS export (p5) | ✅ | `/opt/data` exported & mounted |
| Hostname management | ✅ | Role-based |
| Package normalization | ✅ | `apt_pkg_base` + per-group `apt_pkg_extra` |
| SSH key distribution | ✅ | Public key injection + per-host config template |
| Route provisioning for ZeroLAN | ✅ | Static file + runtime presence check |
| Shell quality-of-life aliases | ✅ | `exa` based `ll`, `la`, `lt` auto-installed |

---
## Inventory & Topology
Inventory file: `hosts`

Groups:
- `master`: `clusterhat`
- `zerocluster`: `z1`..`z4`
- `picluster`: `p1`..`p4` (optional for Slurm / k3s)
- `management`: `p5`
- Aggregate groups: `munge`, `slurm`, `containers`

### Slurm Host Aliases & k3s
The Slurm controller uses one `NodeName` with `NodeAlias=controller,p0` so jobs referencing any of those names resolve to the controller hardware without duplicating resources.

The k3s server has been moved to the management node `p5` (variable `k3s_cluster_server_host`). All other eligible nodes join as agents.

---
## Prerequisites
- Ansible >= 2.14 (for filter availability & collection compatibility)
- Python on control machine
- Collections (install via requirements file):
  - `community.crypto`
  - `community.general`
  - `ansible.utils`
- Target Controller: Raspberry Pi 4 with ClusterHAT v2.5 running **ClusterCTRL CNAT** image
- Target OS for other nodes: Debian / Raspberry Pi OS (Bookworm recommended)
- Access: SSH to all nodes (initial bootstrap for Zeros is via USB boot scaffolding)

**Important**: The controller must be running a ClusterCTRL image (CNAT or CBRIDGE) from the [official ClusterHAT downloads](https://clusterctrl.com/setup-software). Regular Raspberry Pi OS will not work as it lacks the ClusterCTRL software and USB gadget configuration.

Install collections:
```
ansible-galaxy collection install -r requirements.yml
```

Dry run check:
```
ansible-playbook -i hosts cluster.yml --check
```

---
## Quick Start
1. Adjust variables in `group_vars/all` (user, feature flags, NFS host) and host-specific overrides as needed.
2. Place / secure `files/munge/munge.key` (or use Ansible Vault).
3. (Optional) Update `cluster_public_ssh_key` in `group_vars/all`.
4. Provision full cluster (master + zeros + optional nodes):
```
ansible-playbook -i hosts cluster.yml
```
5. Provision only USB boot rootfs (if updating Zero images):
```
ansible-playbook -i hosts usbboot.yml
```
6. Enable Slurm or k3s by toggling flags (see below) and re-run playbook.

---
## Playbooks
| Playbook | Purpose |
|----------|---------|
| `cluster.yml` | Full cluster build (roles + tasks) |
| `master.yml` | Master/controller baseline only |
| `usbboot.yml` | Prepare or update Pi Zero USB boot rootfs images |
| `sdcboot.yml` | SD card imaging tasks (legacy / optional) |
| `ctrlssh.yml` | Bootstrap SSH config for Zero nodes |
| `control_nfs.yml` | Configure the Ansible control host itself as an NFS server (optional) |

---
## Feature Flags
Defined in `group_vars/all`:
- `slurmstack_enable` (bool) – install Slurm + Munge
- `slurmstack_include_picluster` (bool) – extend Slurm to `picluster`
- `docker_runtime_enable` (bool)
- `docker_runtime_on_zeros` (bool) – include Zeros in Docker install
- `k3s_cluster_enable` (bool)

---
## Core Variables
| Variable | Location | Description |
|----------|----------|-------------|
| `pi_user` / `pi_pwhash` | `group_vars/all` | User and pre-hashed password (userconf.txt for USB boot) |
| `cluster_public_ssh_key` | `group_vars/all` | Added to authorized_keys for uniform access |
| `nfs_server_host` / `nfs_export_path` | `group_vars/all` | Define NFS export source |
| `apt_pkg_base` | `group_vars/all` | Baseline packages cluster-wide |
| `apt_pkg_extra` | Per-group | Extend baseline packages group specifically |
| `usbboot_img` | `vars/vars.yml` | Source archive (tar.xz) for Pi Zero rootfs |
| `usbboot_*` | `vars/vars.yml` | USB boot behavior toggles (download, reextract, power control) |
| `clhat_mpnt` | `vars/vars.yml` | Mount point used for shared NFS data (default `/opt/data`) |

### Package Composition
`apt_pkg` is computed: `apt_pkg_base + apt_pkg_extra` (if defined). Avoid redefining `apt_pkg` directly; add group-specific extras in e.g. `group_vars/zerocluster.yml`.

> Tip: To add a package only for Pi Zeros create `group_vars/zerocluster.yml` with:
> ```yaml
> apt_pkg_extra:
>   - i2c-tools
> ```

---
## Package Management Model
Simplifies previous duplication: universal base list plus optional group-specific additions. Modify cluster-wide defaults in one place, keep deltas isolated.

---
## Slurm Configuration & Aliases
File: `files/slurm/slurm.conf`
- Controller: single line with `NodeName=clusterhat ... NodeAlias=controller,p0`
- Prevents double counting resources versus multiple duplicate NodeName entries.
- Zeros defined individually; `picluster` nodes commented for optional inclusion.

If adding `picluster` nodes to Slurm:
1. Uncomment or add `NodeName=p[1-4]` definitions with proper `CPUs=`.
2. Add a partition line, e.g.:
```
PartitionName=picluster Nodes=p[1-4] Default=NO MaxTime=INFINITE State=UP
```

---
## USB Boot Provisioning Flow
Playbook: `usbboot.yml` (logic in `tasks/usbboot.yml`). Revised (2025-08) to follow the official ClusterHAT documentation workflow:

0. (Optional) Uninstall: if `usbboot_uninstall=true`, remove `/var/lib/clusterctrl/nfs/pN` directories then end the play.
1. Downloads the official ClusterCTRL usbboot archive directly to the controller (not via control node caching).
2. For each Zero node directory `/var/lib/clusterctrl/nfs/pN`: 
   - Optionally cleans existing contents if re-extracting (`usbboot_clean_before_extract`)
   - Extracts the usbboot archive using `tar -axf` (following official documentation)
   - Runs `usbboot-init N` to configure the node properly for USB boot
3. Creates per-node marker file `.usbboot_extracted` for idempotency.
4. Creates `userconf.txt` with `<user>:<hash>` for first boot user setup (Bookworm format).
5. Enables SSH by creating the `ssh` flag file in `boot/firmware/`.
6. Powers on nodes using `clusterctrl on pN` commands if `power_on=true`.
7. (Optional) Waits for SSH connectivity on the CNAT network (172.19.181.X) addresses.

Key changes from previous version:
- Simplified to match official ClusterHAT workflow exactly
- Removed control-node caching complexity
- Downloads directly to controller using standard `wget`/`get_url`
- Uses official `tar -axf` extraction method
- Always runs `usbboot-init` as required by ClusterCTRL
- Uses CNAT network addressing (172.19.181.1-4) for SSH waits

Idempotency markers:
- `.usbboot_extracted` inside each `pN` rootfs directory.

Key USB variables (see `vars/vars.yml`):
| Variable | Purpose |
|----------|---------|
| `usbboot_img` | URL to official ClusterCTRL usbboot archive (now uses LITE version) |
| `usbboot_download_dir` | Directory on controller to download archive to |
| `usbboot_local_archive_path` | Computed path of downloaded archive (derived) |
| `usbboot_force_reextract` | Force removal + re-extract even if marker exists |
| `usbboot_clean_before_extract` | Clean target pN dir prior to extraction (default true) |
| `usbboot_enable_ssh` | Create SSH flag file after extract |
| `usbboot_wait_for_ssh` | Wait for SSH availability on CNAT network after powering |
| `usbboot_uninstall` | Remove prepared pN directories and exit |
| `power_on` | If true (default) attempt to power on Zero ports after provisioning |

To force re-extract: `-e usbboot_force_reextract=true`.
To uninstall prepared dirs: `-e usbboot_uninstall=true`.

Shared data mount: The NFS export mounted at `clhat_mpnt` (default `/opt/data`) now also hosts a `usbboot/` subdirectory created automatically; currently used for potential future caching/logs (directory ensured for consistency and extensibility).

### Zero Node Lifecycle (ClusterCTRL Workflow)
1. **Provision rootfs** (this playbook) → extracts usbboot archive to `/var/lib/clusterctrl/nfs/pN/`
2. **Configure node** → `usbboot-init N` sets up boot files and network configuration
3. **Power on node** → `clusterctrl on pN` enables USB power and gadget mode
4. **USB enumeration** → Pi Zero appears as USB gadget device to controller
5. **Boot process** → Zero loads kernel from USB gadget filesystem, gets IP via CNAT (172.19.181.N)
6. **First boot** → `userconf.txt` consumed (creates user), SSH enabled, ready for access
7. **Subsequent boots** → Normal operation, controlled via `clusterctrl` commands

The ClusterCTRL software handles USB gadget presentation, network bridging/NAT, and power management automatically.

---
## Roles Overview
| Role | Key Tasks |
|------|-----------|
| `hostname` | Set system hostname + /etc/hosts lines |
| `installpkg` | Install baseline packages (+ autoremove/clean) |
| `slurmstack` | Install Munge + Slurm (controller & nodes) + config deploy + service management |
| `docker_runtime` | Install & enable Docker across designated groups |
| `k3s_cluster` | Install k3s server (on `k3s_cluster_server_host`, default now `p5`) + agents (others) |
| `nfs_server` | Install & configure NFS export on designated host (typically p5) |
| `raspi-expand-rootfs` | Expand root filesystem (if free space found) |

---
## Secrets Handling
Sensitive items:
- `files/munge/munge.key`
- `pi_pwhash` (hashed password)

Recommendations:
1. Use **Ansible Vault**:
```
ansible-vault encrypt files/munge/munge.key
ansible-vault encrypt group_vars/all
```
2. Or external secret injection (CI variables -> `--extra-vars`).
3. Avoid committing real keys in a public repo.

---
## Common Operations
| Action | Command |
|--------|---------|
| Full provision | `ansible-playbook -i hosts cluster.yml` |
| Limit to controller | `ansible-playbook -i hosts cluster.yml --limit master` |
| Enable Slurm run | `ansible-playbook -i hosts cluster.yml -e slurmstack_enable=true` |
| Add picluster to Slurm | `-e slurmstack_enable=true -e slurmstack_include_picluster=true` |
| Provision USB boot only | `ansible-playbook -i hosts usbboot.yml` |
| Force USB re-extract | `ansible-playbook -i hosts usbboot.yml -e usbboot_force_reextract=true` |
| Wait disabled for USB boot | `-e usbboot_wait_for_ssh=false` |

Dry-run check any change:
```
ansible-playbook -i hosts cluster.yml --check --diff
```

---
## Troubleshooting
| Symptom | Check | Fix |
|---------|-------|-----|
| Slurm nodes show UNKNOWN | `systemctl status slurmd` / hostname mismatch | Ensure hostname matches `NodeHostname` or add alias; restart slurmd |
| Jobs submitted to `controller` fail | `scontrol show nodes clusterhat` | Confirm alias present via `NodeAlias` |
| USB nodes not booting | `clusterctrl status` | Verify rootfs extracted & `ssh` flag present |
| NFS mount missing | `mount | grep /opt/data` | Re-run play; ensure p5 reachable and export in `/etc/exports` |
| Docker not running on zeros | Flag value | Set `docker_runtime_on_zeros: true` and re-run |
| k3s agents fail to join | Token available? | Ensure play ran on controller first; re-run with `--limit master` then agents |

Logs of interest: `/var/log/slurm/*.log`, `journalctl -u slurmd -u slurmctld`, `/var/log/syslog` for USB/network issues.

---
## Roadmap / Ideas
- Add ansible-lint / CI workflow
- Vault integration by default (documented example)
- Dynamic Slurm node generation from inventory hostvars
- Optional Prometheus / monitoring stack role
- Automated test harness via Molecule
- Toggle for cgroup v2 adjustments

---
## License
Project-level license not yet specified (roles may contain their own licenses). Consider adding an overall LICENSE file.

---
## Disclaimer
Intended for lab / learning use. Harden security (host keys, vault secrets, trimmed package set) before production deployment.

---
## Detailed Variable Reference

Below is a non-exhaustive but expanded catalogue. Prefer searching the codebase (`grep` or IDE) for ultimate truth.

### USB Boot (`vars/vars.yml`)
| Variable | Default | Notes |
|----------|---------|-------|
| `usbboot_img` | URL | Official ClusterCTRL Bookworm LITE usbboot archive. |
| `usbboot_img_checksum` | '' | Optional `sha256:HEX` or `md5:HEX`. Empty = skip verify. |
| `usbboot_download_dir` | Control user home | Where to download archive on controller. |
| `usbboot_force_reextract` | `false` | Forces clean + extract per node. |
| `usbboot_clean_before_extract` | `true` | Delete contents of pN dir pre-extract. |
| `usbboot_enable_ssh` | `true` | Touch `ssh` flag. |
| `usbboot_wait_for_ssh` | `true` | Wait for openssh on CNAT network after prep. |
| `usbboot_uninstall` | `no` | Remove pN dirs and exit. |
| `usbboot_skip_prereq_install` | `false` | Set true if base image already has tools. |
| `usbboot_boot_firmware_dir` | `boot/firmware` | Adjust for older releases (`boot`). |
| `power_on` | `true` | Auto power on Zero ports if any off after provisioning. |

### Slurm
| Variable | Location | Purpose |
|----------|----------|---------|
| `slurmstack_enable` | `group_vars/all` | Master switch. |
| `slurmstack_include_picluster` | `group_vars/all` | Add `p[1-4]` nodes. |
| `slurm_partitions_extra` | (define) | Extend partitions via template logic (if added). |

### k3s
| Variable | Purpose |
|----------|---------|
| `k3s_cluster_enable` | Enable k3s role. |
| `k3s_cluster_server_host` | Hostname for server (currently `p5`). |
| `k3s_version` (if added) | Pin specific version (add in defaults). |

### NFS
| Variable | Purpose |
|----------|---------|
| `nfs_server_host` | Host exporting data (p5 or controller). |
| `nfs_export_path` | Exported directory root. |
| `nfs_client_mount` | Where clients mount export. |

### Users / SSH
| Variable | Notes |
|----------|-------|
| `pi_user` | Default login (created via userconf). |
| `pi_pwhash` | Pre-hashed password. Use `mkpasswd --method=sha512crypt`. |
| `cluster_public_ssh_key` | Appended to `authorized_keys`. |

---
## k3s Cluster Details
When enabled:
1. Server installed on `k3s_cluster_server_host` (p5 default).
2. Token retrieved and stored (consider vault for persistence).
3. Agents join automatically using server's advertised IP.

Common checks:
```
kubectl get nodes
kubectl -n kube-system get pods
```
If `kubectl` not present on control laptop, copy `/etc/rancher/k3s/k3s.yaml` from server and set `KUBECONFIG`.

Disable temporarily: run play with `-e k3s_cluster_enable=false` (does not uninstall binaries by default – add uninstall logic if desired).

---
## Slurm Usage Examples
Submit a test job:
```
sbatch -N1 -c1 --wrap='hostname; sleep 5'
```
Interactive session on controller partition:
```
salloc -N1 -w controller bash
```
Show nodes / partitions:
```
sinfo -N -l
scontrol show nodes
```
Cancel all jobs for current user:
```
scancel -u $USER
```

Add picluster dynamically (example extra vars):
```
ansible-playbook -i hosts cluster.yml -e slurmstack_enable=true -e slurmstack_include_picluster=true
```

---
## NFS Layout & Mounts
Default export path assumption: `/opt/data` on `nfs_server_host`.

Client mount (if using fstab):
```
<nfs_server_host>:/opt/data  /opt/data  nfs  defaults,nofail  0  0
```
Test manually:
```
showmount -e <nfs_server_host>
mount -t nfs <nfs_server_host>:/opt/data /mnt
```

Performance tuning ideas: enable `async`, consider `noatime`, increase rsize/wsize (e.g. 65536) for bulk throughput.

---
## Security Hardening Checklist
| Area | Action |
|------|--------|
| SSH | Replace password auth with key-only; disable root login. |
| Users | Rotate `pi_pwhash` or remove default `pi` user. |
| Secrets | Move munge key + hashes into Vault. |
| Slurm | Restrict `SubmitPlugins` / cgroup enforcement (already has cgroup config). |
| k3s | Enable NetworkPolicy; restrict service CIDRs; rotate token. |
| NFS | Export with `ro` where possible; limit CIDR; add `root_squash`. |
| Packages | Remove dev utils on Zeros to reduce footprint. |
| Logs | Ship to central log host (rsyslog, Loki) for auditing. |
| Firmware | Keep Pi EEPROM / bootloaders updated. |

---
## Performance & Scaling Notes
| Topic | Guidance |
|-------|----------|
| USB boot extract time | Large images: consider pre-pruning locales, docs before archiving. |
| Repeated extracts | Use one extract then `cp -al` hardlinks (future enhancement) to speed provisioning. |
| Slurm scheduling | Small cluster: single partition; more nodes → split CPU vs GPU (if any). |
| k3s etcd/storage | For durability move server to SSD-backed host. |
| NFS contention | Enable `actimeo` tuning or shift to local overlay FS for builds. |
| Monitoring | Add node exporter + Prometheus for capacity planning. |

---
## Customization Patterns
1. Add new role: create `roles/<name>/{tasks,defaults,meta}` then include via `cluster.yml` with a flag.
2. Per-host overrides: use `host_vars/<hostname>.yml` for CPU/memory specific tuning.
3. Conditional tasks: wrap with `when: <feature_flag> | bool` to keep idempotent.
4. Templating: create `templates/*.j2` and deploy with `template` module; reference variables from group/host vars.
5. Inventory layering: introduce a `lab` vs `prod` inventory file and reuse same roles.

---
## Maintenance & Updates
| Task | Recommendation |
|------|----------------|
| Update usbboot image | Download new tar.xz, set `usbboot_force_reextract=true`. |
| Rotate munge key | Replace file, restart Slurm components. |
| OS package updates | Run `ansible-playbook -i hosts cluster.yml -t pkg-update` (add tag). |
| k3s upgrade | Pin `INSTALL_K3S_VERSION` environment via role var (extend role). |
| Backup config | Archive `group_vars`, `host_vars`, and `/var/lib/clusterctrl/nfs/pN/etc`. |

---
## Contributing
1. Fork & branch: `feat/<short-desc>`.
2. Run lint (add ansible-lint soon).
3. Open PR with clear summary + before/after behavior.
4. Include test (Molecule) for new roles where practical.

Style: prefer explicit variable names, minimal magic defaults, add comments for non-obvious shell commands.

---
## Glossary
| Term | Definition |
|------|------------|
| ClusterHAT | Hardware board enabling USB boot of multiple Pi Zeros from one host. |
| Zero Node | A Raspberry Pi Zero managed via USB by the ClusterHAT. |
| Controller | The host physically attached to the ClusterHAT (inventory: `clusterhat`). |
| pN | Conventional larger Pi nodes (e.g. 3/4/5) on LAN. |
| usbboot | Mechanism to boot a Pi Zero without SD by presenting rootfs over USB mass storage. |
| userconf.txt | Raspberry Pi OS mechanism to define an initial user & password hash. |
| Munge | Authentication service used by Slurm. |
| k3s | Lightweight Kubernetes distribution by Rancher. |

---
## Changelog Template
Maintain a `CHANGELOG.md` (to be added) following Keep a Changelog style:
```
## [Unreleased]
### Added
- 
### Changed
- 
### Fixed
- 
### Removed
- 
```
Tag releases via git tags when major provisioning logic changes (e.g. switch from staging to direct extraction).

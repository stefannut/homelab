# Hardware

This document describes the physical host(s) underpinning this homelab's services layer — specs, virtualization approach, and how the available resources map to running workloads. It exists so that capacity questions ("can this host take one more service?") and recovery questions ("what am I rebuilding, exactly?") have a single place to be answered.

This file describes hardware and host-level virtualization only. Service definitions live under `/services`; deployment automation lives in the Ansible inventory. Network topology (VLANs, firewall, routing) is out of scope here and is tracked in the `opnsense` service folder.

---

## Host: `pve1`

### Hardware

| Component | Spec |
| --- | --- |
| CPU | Intel Core i3-10100F — 4 cores / 8 threads @ 4.30 GHz |
| GPU | NVIDIA GeForce GTX 1050 Ti — 4 GB VRAM |
| RAM | 8 GB DDR4 |
| Storage | 512 GB SSD |

**Capacity notes:**

* 8 GB of RAM is the primary constraint on this host. It sets a hard ceiling on how many concurrent LXC containers/VMs are practical, particularly with an ML workload (PyTorch/CUDA) in the mix — memory headroom, not CPU, is the first thing to check before adding a new service here.
* The GTX 1050 Ti's 4 GB VRAM limits model size/batch size for ML experimentation and is shared with Frigate if GPU-accelerated detection is enabled for the NVR — these two workloads compete for the same VRAM budget and shouldn't be assumed to coexist at full load without checking.
* 512 GB SSD is the single storage tier — there is currently no separate fast/slow tier, so backup jobs, Frigate's recording retention, and VM/container disk growth all draw from the same pool. Worth tracking usage per-workload if any one of them starts growing unpredictably (Frigate recordings are the most likely culprit).

### Software & Infrastructure

| Layer | Detail |
| --- | --- |
| Hypervisor OS | Proxmox VE 9.2 |
| Kernel | Linux 7.0 version pve |
| Networking | Tailscale (mesh VPN) |
| Virtualization | LXC containers & QEMU VMs |

**Notes:**

* Tailscale provides mesh connectivity for remote/cross-site access. This is the extent of networking documented here — firewall rules, subnet routing, and DNS remain in the `opnsense` service folder; only the fact that this host participates in the Tailscale mesh is noted here as a hardware/host-level detail.
* Proxmox is the virtualization boundary: services are expected to run inside LXC containers or QEMU VMs rather than directly on the Proxmox host itself, keeping the hypervisor layer clean and each workload isolated and independently recoverable.

### Usage Profile

This host currently serves four primary roles:

1. **Development environment** — Debian + XFCE, used as a general-purpose dev workspace.
2. **Machine learning experimentation** — CUDA/PyTorch, GPU-passthrough dependent on the GTX 1050 Ti above.
3. **Backup Server (NAS)** — backup target for this host's own VMs/containers.
4. **Home surveillance** — Frigate NVR, GPU acceleration shared with the ML role where applicable.

Running backup (PBS) and the thing being backed up on the same physical host is a known single point of failure — worth flagging explicitly rather than leaving implicit. See [Planned Improvements](https://www.google.com/search?q=../ROADMAP.md%23planned-improvements) for offsite/off-host backup direction.

---

## Host: `nas1`

### Hardware

| Component | Spec |
| --- | --- |
| CPU | Intel Celeron N2830 — 2 cores / 2 threads @ 2.16 GHz (burst up to 2.41 GHz) |
| GPU | Intel HD Graphics (Bay Trail) |
| RAM | 2 GB DDR3 |
| Storage | 500 GB HDD |

**Capacity notes:**

* 2 GB of RAM is a tight constraint, restricting this host strictly to lightweight storage and file-sharing tasks without heavy background applications or memory-intensive services.
* The Intel Celeron N2830 processor and single 500 GB HDD are optimized for low-power, centralized file storage and secondary backups rather than high-throughput or concurrent multi-user processing.

### Software & Infrastructure

| Layer | Detail |
| --- | --- |
| OS | OpenMediaVault (OMV) |
| File Sharing / Services | SMB / NFS |

**Notes:**

* OpenMediaVault provides a headless, lightweight storage-focused operating system designed to maximize the utility of constrained hardware.
* Functions as a dedicated secondary storage node, keeping backup targets and shared files off the primary hypervisor node.

### Usage Profile

This host serves as the dedicated network-attached storage (NAS) and secondary backup destination:

1. **Centralized Storage & File Sharing** — Providing local network file shares (SMB/NFS) for household devices and workflows.
2. **Secondary Backup Destination** — Acting as an off-host repository for homelab data backups, separating storage from the main Proxmox execution environment.

---

## Adding a New Host

When a second host joins the homelab, duplicate the `## Host: <name>` section above rather than merging specs into one table — each host gets its own hardware, software, and usage profile block. This keeps per-host capacity reasoning legible as the infrastructure grows past a single machine, and each section should stay traceable to its corresponding `host_vars/<hostname>.yml` entry in the Ansible inventory (see `CONTRIBUTING.md`, §5.1).

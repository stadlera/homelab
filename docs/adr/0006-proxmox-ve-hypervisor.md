# ADR-0006 — Use Proxmox VE as the hypervisor

* Status: accepted
* Date: 2026-05-25
* Issue: #8

## Context and Problem Statement

The homelab's compute substrate is two x86 mini-PCs, complemented by two Raspberry Pis and an OPNsense router. The two x86 nodes need a hypervisor that hosts the k3s VMs, the OPNsense VM, and occasional utility/lab VMs, and that can be driven from this repo's IaC layer (per ADR-0004). The hypervisor sits beneath every other workload here, so picking it once and well matters more than most other choices in this stack. Both nodes already run Proxmox VE 8; this ADR records the reasoning rather than introducing the change.

## Decision Drivers

* Cluster-aware out of the box — live migration, shared corosync state, and graceful node draining between the two x86 nodes without bolting together pacemaker/sanlock by hand
* First-class IaC story — a maintained OpenTofu provider covering VM lifecycle, cloud-init, templates, and snapshots, so VM creation is `tofu apply`, not point-and-click
* Web UI as break-glass — IaC owns the steady state, but recovery and forensics shouldn't require SSH-only tooling
* Free of charge and free of subscription gating — the homelab budget does not extend to vSphere licences and shouldn't depend on a vendor's free tier
* KVM/QEMU base — keeps the exit cost bounded; workloads stay portable to plain libvirt if the management plane ever proves to be a liability
* LXC and ZFS support are convenient but not decision drivers — k3s owns container workloads, and storage layout is a node-config concern below the hypervisor

## Considered Options

* Proxmox VE
* TrueNAS SCALE
* Plain Debian + KVM/libvirt
* XCP-ng
* ESXi / vSphere

## Decision Outcome

Chosen option: **Proxmox VE** (currently 8.x), because it is the only option that ships clustering, a mature IaC integration, and a usable web UI without a paid subscription. The KVM/QEMU foundation means we can fall back to plain libvirt on the same hosts if Proxmox's management plane ever becomes a liability, so the lock-in is shallow. TrueNAS SCALE prioritises storage with VMs as a secondary feature; plain Debian + libvirt gives up the management plane we actually rely on for break-glass; XCP-ng's Terraform story is materially weaker; and Broadcom retiring free ESXi in 2024 puts vSphere out of reach for a hobby budget on terms that could shift again.

### Terraform/OpenTofu provider

Two community providers exist for Proxmox: `Telmate/proxmox` and `bpg/proxmox`. We use **`bpg/proxmox`**. It has the more active maintainer cadence over the last several releases, covers both the Proxmox API and the SSH paths needed for operations the API does not expose (image format conversions, file uploads to node-local storage), and has first-class cloud-init resources (`proxmox_virtual_environment_file`, `proxmox_virtual_environment_vm` with `initialization` blocks). `Telmate/proxmox` is still maintained but trails on cloud-init ergonomics and has a history of long stalls. This provider choice belongs with the hypervisor, not with the IaC-engine choice in ADR-0004.

### Consequences

* Good, because the two-node cluster gives us live migration and graceful draining for OS upgrades without rebuilding workloads
* Good, because `bpg/proxmox` plus cloud-init covers the entire VM lifecycle from `tofu apply` — no out-of-band image-baking pipeline
* Good, because the AGPL-3 licence and Debian base have been stable for years, and the community is large enough that obscure issues are typically already answered
* Good, because Proxmox Backup Server (PBS) is a natural follow-on if we later want incremental, deduplicated VM backups
* Neutral, because PVE 9 (Debian 13 base) is on the horizon — the "8" in the current install is the deployed major version, not a freeze; the upgrade will be a separate, tracked change
* Bad, because we depend on Proxmox Server Solutions GmbH as a single upstream — mitigated by the option to drop to plain libvirt on the same KVM/QEMU stack
* Bad, because the no-subscription repo throws a "no valid subscription" warning at web-UI login; cosmetic but worth knowing
* Bad, because `bpg/proxmox` is community-maintained — a sustained loss of maintenance would force a fork or a fallback to Ansible-driven VM provisioning

### Confirmation

When VMs in `opentofu/proxmox/` are provisioned, confirm: (1) `tofu plan` / `tofu apply` is the only mechanism used to create or modify them — no manual web-UI edits, (2) cloud-init seeds the VM with the expected user, SSH key, and network config without console intervention, and (3) draining one node migrates running VMs to the other and returns to a healthy cluster state once the drained node is back. A scheduled node reboot should be a non-event for workloads.

## Pros and Cons of the Options

### Proxmox VE

Debian-based hypervisor distribution combining KVM (VMs), LXC (containers), a clustering layer (corosync), and a web UI / REST API.

* Good, because clustering, live migration, and HA are built in and free
* Good, because the REST API plus `bpg/proxmox` covers the full IaC story
* Good, because the underlying KVM/QEMU stack is the same one plain libvirt uses — exit cost is bounded
* Good, because the AGPL-3 licence and Debian base are predictable and well-understood
* Neutral, because LXC support is a bonus we do not currently use
* Bad, because the enterprise repo is gated behind a subscription; using the no-subscription repo is supported but throws a warning at login
* Bad, because direction is set by a single company, even if the source is fully open

### TrueNAS SCALE

iXsystems' Debian-based NAS OS with KVM-based VM support and a Kubernetes-derived (now Docker-based) "Apps" platform.

* Good, because ZFS is first-class — one box could be both storage and hypervisor
* Good, because the web UI is polished
* Neutral, because storage-first design suits a NAS appliance but pulls focus from VM lifecycle work
* Bad, because the Terraform/OpenTofu provider story is effectively non-existent for VM lifecycle
* Bad, because the Apps platform has churned (k3s → Docker) and would compete with the k3s cluster for the same role
* Bad, because VM clustering is not a peer feature of Proxmox's

### Plain Debian + KVM/libvirt

Bare Debian with `libvirtd`, `virsh`, and either Cockpit or virt-manager for the rare GUI need.

* Good, because there is no vendor and no extra management plane — maximum control
* Good, because libvirt is the long-term lingua franca of Linux virtualisation
* Neutral, because the `dmacvicar/libvirt` provider works but has fewer batteries-included primitives than `bpg/proxmox`
* Bad, because clustering, live migration, and HA require assembling pacemaker/corosync/sanlock by hand
* Bad, because we'd be rebuilding the management layer Proxmox already ships — effort better spent elsewhere in the stack

### XCP-ng

Open-source fork of Citrix Hypervisor (XenServer), managed via Xen Orchestra.

* Good, because Xen's dom0 isolation model has a strong security track record
* Good, because Xen Orchestra is a capable management plane
* Neutral, because XCP-ng on commodity x86 mini-PCs is well-trodden, just less so than Proxmox
* Bad, because the Terraform provider (`vatesfr/xenorchestra`) is less mature than `bpg/proxmox` — fewer cloud-init primitives and rougher edges
* Bad, because Xen Orchestra's "free if you build from sources" model adds friction compared to Proxmox's straight no-subscription repo

### ESXi / vSphere

Broadcom-owned (since 2023) Type-1 hypervisor with vCenter management.

* Good, because the enterprise feature set and tooling are mature
* Bad, because Broadcom retired the free ESXi tier in 2024 and made entry-level licensing materially less hobbyist-friendly
* Bad, because future licence terms are unpredictable in a way Proxmox's are not
* Bad, because the portfolio signal is wrong — picking a paid proprietary hypervisor for a homelab demonstrates neither cost-awareness nor pragmatism

## More Information

* Proxmox VE documentation: https://pve.proxmox.com/wiki/Main_Page
* `bpg/proxmox` provider: https://registry.terraform.io/providers/bpg/proxmox/latest
* `Telmate/proxmox` provider: https://registry.terraform.io/providers/Telmate/proxmox/latest
* Related: ADR-0004 (OpenTofu) — the IaC engine that consumes the `bpg/proxmox` provider
* Revisit this decision if: clustering primitives we rely on regress, `bpg/proxmox` becomes unmaintained, Proxmox subscription gating expands to features we currently use for free, or the compute footprint moves to ARM/SBCs in a way that changes the calculus toward libvirt directly

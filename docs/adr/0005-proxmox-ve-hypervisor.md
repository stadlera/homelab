# ADR-0005 — Use Proxmox VE 8 as the homelab hypervisor

* Status: proposed
* Date: 2026-05-24
* Issue: #8

## Context and Problem Statement

The homelab runs on two small x86 nodes that need to host a mix of long-running VMs (k3s control plane and workers, OPNsense, supporting services) and templates managed declaratively from OpenTofu. The hypervisor choice has to support cloud-init templates, snapshots, a stable API for IaC, and ideally a clustered control plane — without imposing a licence cost or vendor account on a personal-use environment. The hardware was already provisioned with Proxmox VE before this ADR was written; this records the decision rather than initiates it.

## Decision Drivers

* Mature web UI for emergency operations — when IaC is broken, the fix path can't itself require IaC
* Stable, well-documented REST API with a maintained OpenTofu provider — this is the primary interface from `opentofu/proxmox`
* Cloud-init template support for repeatable VM provisioning
* Free of charge for personal use; no paywall blocking basic functionality
* Linux/KVM under the hood — kernel, tooling, and recovery paths are standard, not vendor-proprietary
* Optional clustering for live migration and a shared management plane, even if HA quorum is deferred until a third participant is added
* Doubles as a portfolio reference — the choice should be defensible against the obvious alternatives, not just "what I had running"

## Considered Options

* Proxmox VE 8
* TrueNAS SCALE
* Debian + KVM/libvirt
* XCP-ng
* ESXi / vSphere

## Decision Outcome

Chosen option: **Proxmox VE 8**, because it offers the highest density of homelab-relevant features for the lowest operational ceremony: a mature web UI, KVM + LXC in one product, native ZFS, clustering and replication, snapshots, cloud-init, and a REST API with a maintained OpenTofu provider — all under AGPLv3 with a free `pve-no-subscription` repo. None of the alternatives matched on all of those points: TrueNAS SCALE is a NAS that also runs VMs (inverted priorities), Debian + libvirt would require reinventing the appliance layer, XCP-ng has a noticeably thinner IaC ecosystem, and ESXi's free tier was discontinued under Broadcom. VE 8.x is chosen over 9.x because 8 is the current production release at build time with stable provider support; the VE 9 upgrade path is tracked as a separate operational decision rather than a hypervisor change.

### Consequences

* Good, because cluster, replication, snapshots, backups, and IaC all work out of the box without a stack of add-ons
* Good, because the management plane is recoverable through a web UI when automation breaks — important for a homelab with no on-call rotation
* Good, because the underlying Debian + KVM + ZFS stack is standard and transferable; nothing about a VM is Proxmox-proprietary
* Bad, because the free no-subscription repo triggers a "no valid subscription" login nag and uses a separate `pve-no-subscription` apt source — both managed via Ansible to keep the nodes patched and the UI clean
* Bad, because the two-node cluster cannot achieve quorum on its own; live migration works, but HA failover requires a third quorum participant (full node or QDevice on one of the Pis) — tracked as a follow-up, explicitly out of scope for this ADR
* Bad, because AGPLv3 is acceptable for self-hosted use but would prevent building a redistributable product on top — not a constraint here, but worth recording

### Confirmation

A node is compliant with this ADR when:

* `pveversion` reports an 8.x release (e.g. `pve-manager/8.x.x/...`)
* `/etc/apt/sources.list.d/` contains `pve-no-subscription`, not `pve-enterprise`
* The OpenTofu `opentofu/proxmox` stack authenticates against the node and `tofu plan` reports no drift on baseline resources
* `pvecm status` lists both nodes as cluster members (quorum caveat tracked separately)

## Pros and Cons of the Options

### Proxmox VE 8

Debian-based virtualisation appliance combining KVM (VMs), LXC (containers), ZFS, Ceph, clustering, and a web UI behind a REST API.

* Good, because one product covers VMs, containers, storage, cluster, and backups — no integration glue required
* Good, because the OpenTofu provider `bpg/proxmox` is actively maintained and tracks the API closely
* Good, because cloud-init, snapshot, and template workflows are first-class — exactly what an IaC-driven homelab needs
* Good, because AGPLv3 with the `pve-no-subscription` repo means no paywall on core functionality
* Neutral, because enterprise support exists but is not used here — the community wiki and forum are sufficient at this scale
* Bad, because the no-subscription nag dialog and separate apt source add small friction (mitigated in Ansible)
* Bad, because Proxmox-specific abstractions (storage IDs, node names) leak into IaC and would need translation on a migration off

### TrueNAS SCALE

Debian-based NAS appliance from iXsystems that added KVM and Docker/Kubernetes workloads on top of a ZFS-first design.

* Good, because storage management is the strongest of any option — ZFS pool lifecycle is a first-class UI concept
* Good, because for a single combined NAS+hypervisor box it removes a layer
* Neutral, because the VM and container story has improved meaningfully but is still secondary to storage
* Bad, because clustering and live migration are not in the same league as Proxmox — VMs remain tied to one appliance
* Bad, because IaC support is thinner — no OpenTofu provider comparable to `bpg/proxmox` for VM lifecycle
* Bad, because the architectural framing ("NAS that also runs VMs") inverts this homelab's priorities ("hypervisor that also serves storage"); storage is a separate decision

### Debian + KVM/libvirt

Plain Debian with `qemu-kvm`, `libvirt`, and either `virsh`/`virt-manager` or Cockpit as a partial UI.

* Good, because there is no appliance layer between the operator and the kernel — everything is `apt`-installable and standard
* Good, because the licence story is uniformly permissive (no AGPL, no nags)
* Neutral, because the OpenTofu `libvirt` provider exists but covers fewer high-level concepts than `bpg/proxmox`
* Bad, because clustering, live migration, shared storage, and backups must all be assembled by hand — significant operational work for no portfolio payoff
* Bad, because no built-in web UI; Cockpit is partial and not a replacement for the Proxmox console under outage conditions
* Bad, because reinventing what Proxmox already packages is the opposite of "as-code first" leverage

### XCP-ng

Open-source fork of XenServer, Xen-based, with Xen Orchestra as the management UI.

* Good, because Xen has a strong security pedigree and XCP-ng's free tier is unconstrained
* Good, because Xen Orchestra is a polished UI; XO Lite ships free alongside the hypervisor
* Neutral, because the Xen vs KVM debate is largely a wash at homelab scale
* Bad, because the OpenTofu provider ecosystem is noticeably thinner — `vatesfr/xenorchestra` is workable but less mature than `bpg/proxmox` and updates more slowly
* Bad, because community momentum and third-party tooling skew toward KVM; Xen is no longer the default Linux hypervisor
* Bad, because LXC-equivalent containers are not first-class — would need to layer Docker/Podman inside VMs

### ESXi / vSphere

Broadcom/VMware's commercial hypervisor and management platform.

* Good, because it is the enterprise standard with the most mature feature set
* Bad, because the free ESXi tier was discontinued in 2024 — Broadcom's subscription-only model is inappropriate for a personal homelab
* Bad, because licence cost and account-binding are inconsistent with the "no vendor account required" stance taken elsewhere in this repo
* Bad, because the API and provider ecosystem assume vSphere, not bare ESXi — homelab-scale use is now actively discouraged by the vendor

## Provider note: `bpg/proxmox` over `Telmate/proxmox`

The OpenTofu provider choice is recorded here rather than in ADR-0004 because the comparison is Proxmox-specific:

* **`bpg/proxmox`** — community-maintained, very active, covers VMs, containers, templates, cloud-init, storage, and most newer Proxmox API surfaces; releases regularly; documentation is good
* **`Telmate/proxmox`** — the original community provider; development has slowed and it lags behind newer Proxmox API features, with long-standing rough edges around template cloning and cloud-init handling

`bpg/proxmox` is selected. The `opentofu/proxmox` stack must source `bpg/proxmox/proxmox` in `versions.tf`; this replaces the placeholder currently in that file.

## More Information

* Proxmox VE pricing and subscription model: https://www.proxmox.com/en/proxmox-virtual-environment/pricing
* `bpg/proxmox` provider: https://registry.terraform.io/providers/bpg/proxmox/latest/docs
* Revisit this decision when: a third quorum participant is added (HA story changes), Proxmox VE 9 has been on the stable channel long enough to plan an in-place upgrade, or a hardware refresh prompts re-evaluating the storage/hypervisor split

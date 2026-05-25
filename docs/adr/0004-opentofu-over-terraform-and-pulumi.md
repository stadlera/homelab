# ADR-0004 — Use OpenTofu for infrastructure provisioning

* Status: accepted
* Date: 2026-05-24
* Issue: #9

## Context and Problem Statement

The homelab needs an IaC tool to provision cloud-like resources: Proxmox VMs and templates, Cloudflare DNS records, Tailscale ACLs and devices, and on-prem network/firewall objects. The choice has to coexist with Ansible (host/service configuration) and Flux (in-cluster GitOps), so it only needs to own the provisioning layer — not config management or app delivery. HashiCorp's 2023 relicense of Terraform to the Business Source License (BSL) makes "just use Terraform" no longer a neutral default; an explicit choice is warranted.

## Decision Drivers

* OSI-approved, community-governed licence — the repo doubles as a portfolio and should reflect a deliberate stance on the BSL relicense, not inertia
* Declarative state model with `plan` / `apply` — Ansible alone cannot represent infra resources with a dependency graph and drift detection
* Provider coverage for Proxmox, Cloudflare, and Tailscale must be first-class
* Drop-in compatibility with existing HCL — modules, tutorials, and prior knowledge should transfer without rewrites
* Small-scale homelab — programmatic abstraction power is not a meaningful driver; readability and low ceremony are

## Considered Options

* OpenTofu
* Terraform
* Pulumi
* Ansible only

## Decision Outcome

Chosen option: **OpenTofu**, because it preserves every property that made Terraform the obvious IaC choice (HCL, providers, state model, plan/apply, ecosystem) while remaining under MPL 2.0 with Linux Foundation governance. The BSL relicense is not a legal problem for personal homelab use, but choosing the community-governed fork avoids depending on a vendor that has demonstrated willingness to change the rules. Pulumi's real-language model is genuinely compelling at larger scale but its abstraction premium is wasted on a two-node homelab. Ansible alone has no state model for infrastructure resources.

### Consequences

* Good, because the licence is MPL 2.0 — no BSL ambiguity, no vendor lock-in risk, governance under a neutral foundation
* Good, because HCL and the provider protocol are unchanged — existing Terraform modules, examples, and muscle memory transfer with no rewrite
* Good, because the toolchain around it (`tflint`, `terragrunt`, `terraform-docs`, IDE plugins) already targets both binaries
* Bad, because some new third-party providers still ship to the Terraform registry first; OpenTofu's registry mirrors most but not all, so occasional manual provider sourcing may be required
* Bad, because the `terraform/` directory in this repo is now misnamed — a follow-up rename to `opentofu/` (or a neutral `iac/`) is needed to avoid confusion

### Confirmation

When the migration is complete, confirm that: the `tofu` binary (not `terraform`) is used in all CI workflows, Makefiles, and runbooks; no `required_providers` block pulls from a registry that requires a Terraform-only licence; and `tofu fmt`, `tofu validate`, and `tofu plan` run cleanly against every stack in `terraform/`. The directory rename is tracked as a follow-up.

## Pros and Cons of the Options

### OpenTofu

Linux Foundation fork of Terraform 1.5.x under MPL 2.0, created in response to the BSL relicense. HCL- and provider-compatible.

* Good, because MPL 2.0 and foundation governance remove the BSL concern entirely
* Good, because it is a drop-in for existing HCL — zero learning cost on top of Terraform knowledge
* Good, because community momentum is healthy and growing; major providers (Proxmox via `bpg/proxmox`, Cloudflare, Tailscale) work unchanged
* Neutral, because long-term divergence from Terraform is possible but currently small
* Bad, because the registry ecosystem is still catching up — occasional providers need to be sourced from GitHub directly

### Terraform

HashiCorp's IaC tool, BSL 1.1 since August 2023.

* Good, because it has the largest community, most tutorials, and earliest provider releases
* Neutral, because BSL is not a legal blocker for personal/homelab use — the restriction targets commercial competitors
* Bad, because BSL signals that the licence terms can change again; choosing it is an implicit endorsement of that model
* Bad, because it offers no functional advantage over OpenTofu at this scale

### Pulumi

IaC using real programming languages (TypeScript, Python, Go, .NET) with a state backend model similar to Terraform.

* Good, because real languages enable strong typing, real refactoring, and first-class abstractions
* Good, because testing infra with standard unit-test frameworks is straightforward
* Neutral, because the value of programmatic abstractions scales with infra size — a two-node homelab does not stress what HCL can express
* Bad, because the provider set for Proxmox is community-maintained and less mature than the OpenTofu/Terraform equivalent
* Bad, because it adds a language runtime and package-manager dependency to the IaC layer
* Bad, because the free tier of Pulumi Cloud is the path of least resistance for state; self-hosting the backend is extra ceremony

### Ansible only

Use Ansible (already in the stack for configuration) to also provision infra via modules like `community.general.proxmox_kvm` and `community.general.cloudflare_dns_record`.

* Good, because it removes a tool from the stack — one engine, one inventory
* Bad, because Ansible has no dependency graph, no plan phase, and no managed state — drift is invisible until a play fails
* Bad, because cloud/provider modules in Ansible are noticeably thinner and slower-moving than the Terraform/OpenTofu provider ecosystem
* Bad, because mixing provisioning and configuration in the same tool blurs the boundary this repo has deliberately drawn

## More Information

* OpenTofu announcement and governance: https://opentofu.org/
* HashiCorp BSL FAQ: https://www.hashicorp.com/license-faq
* Follow-up: rename `terraform/` → `opentofu/` (or `iac/`) and update CI workflows to use the `tofu` binary — to be tracked as a separate issue
* Revisit this decision if OpenTofu governance falters, if a critical provider becomes Terraform-exclusive, or if the infra grows to a scale where Pulumi's abstraction model would meaningfully reduce duplication

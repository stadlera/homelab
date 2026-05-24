# ADR-0005 — Use Flux v2 over ArgoCD for GitOps

* Status: proposed
* Date: 2026-05-24
* Issue: #10

## Context and Problem Statement

The k3s cluster needs a GitOps controller to reconcile manifests from this repository — both infrastructure (`kubernetes/infrastructure/`) and applications (`kubernetes/apps/`). The choice has to fit a small two-node-plus-Pis homelab, work with SOPS-encrypted secrets (per ADR-0001), and avoid extra services to babysit.

## Decision Drivers

* Native SOPS decryption — secrets in git must be decrypted by the controller itself, with no extra plugin or sidecar
* Lightweight controller footprint — the cluster runs on Proxmox VMs alongside other workloads; every extra Deployment costs RAM
* Single cluster, single tenant — no need for multi-cluster fan-out or RBAC partitioning
* Pull-based reconciliation with drift correction — the cluster must self-heal from git, not depend on push pipelines
* Stays out of the way — the homelab already has Grafana/Prometheus for observability; the GitOps tool should not bring a heavy UI stack of its own

## Considered Options

* Flux v2
* ArgoCD
* Rancher Fleet
* Manual `helm upgrade` / `kubectl apply`

## Decision Outcome

Chosen option: **Flux v2**, because it is the only option that decrypts SOPS-encrypted manifests natively (matching ADR-0001 without adding a plugin), runs as a small set of single-purpose controllers that fit the homelab's resource budget, and reconciles directly from git without a central UI server. ArgoCD is the strongest alternative but loses on the SOPS integration and the controller footprint; the other options either solve a different problem (Fleet) or give up reconciliation entirely (manual).

### Consequences

* Good, because `.sops.yaml` from ADR-0001 is consumed directly by Flux's `kustomize-controller` with an age key in a cluster secret — no extra moving parts
* Good, because the controller set (source, kustomize, helm, notification) is small and each piece is independently restartable
* Good, because `flux bootstrap` writes its own manifests into this repo, making the cluster's GitOps config self-hosted from day one
* Neutral, because Flux's Image Automation Controllers are available if we later want git-committed image tag bumps; Renovate (ADR-0003) covers dependency updates for now
* Bad, because there is no built-in UI — day-to-day inspection is `flux` CLI and `kubectl`; a read-only dashboard (weave-gitops or Headlamp's Flux plugin) can be added later if it's missed
* Bad, because Flux's learning curve is steeper than ArgoCD's for newcomers — CRDs (`GitRepository`, `Kustomization`, `HelmRelease`) replace a clickable app view

### Confirmation

When `kubernetes/flux-system/` is populated by `flux bootstrap` and the first `Kustomization` reconciles an encrypted secret from `kubernetes/infrastructure/`, confirm: (1) the cluster decrypts SOPS without any extra controller or plugin, (2) drift introduced by `kubectl edit` is reverted within the reconciliation interval, and (3) the controllers' combined memory usage is materially lower than an equivalent ArgoCD install on the same cluster.

## Pros and Cons of the Options

### Flux v2

CNCF-graduated GitOps toolkit composed of focused controllers (`source`, `kustomize`, `helm`, `notification`, optional `image-*`).

* Good, because SOPS + age decryption is a first-class feature of `kustomize-controller` — the same `.sops.yaml` used by Ansible and Terraform works unchanged
* Good, because the controller set is lightweight and modular; unused controllers (e.g. image automation) can simply not be installed
* Good, because `flux bootstrap` produces a self-managing install committed to this repo
* Good, because OCI artifacts and Helm OCI repositories are supported natively as sources
* Bad, because there is no first-party UI; operators rely on CLI and metrics
* Bad, because troubleshooting requires understanding which controller owns which CRD

### ArgoCD

Single-binary-ish GitOps platform with a rich web UI, popular in larger organisations.

* Good, because the UI makes app status, sync history, and diffs immediately visible
* Good, because the `Application` and `ApplicationSet` model is approachable
* Good, because the community and ecosystem are larger than Flux's
* Neutral, because SOPS can be made to work via plugins (`argocd-vault-plugin`, `helm-secrets`, `ksops`), but each adds an out-of-tree component to install, upgrade, and trust
* Bad, because the standard install (`argocd-server`, `repo-server`, `application-controller`, Redis, optional Dex) is heavier than Flux's controller set for a single-cluster homelab
* Bad, because the UI/API server expands the attack surface and needs its own auth story even for one operator

### Rancher Fleet

Multi-cluster GitOps from the Rancher ecosystem; designed for fleets of downstream clusters.

* Good, because it scales to hundreds of clusters with a single control plane
* Neutral, because it would be a reasonable choice if Rancher were already in use
* Bad, because its multi-cluster model is overkill for one k3s cluster
* Bad, because SOPS is not a native primitive; secrets need a separate solution
* Bad, because adoption outside the Rancher ecosystem is thin, making troubleshooting harder

### Manual `helm upgrade` / `kubectl apply`

Run releases from a workstation or CI job, no in-cluster reconciler.

* Good, because there is nothing extra to install or learn
* Bad, because there is no drift detection — anything changed in-cluster stays changed
* Bad, because the cluster cannot self-heal after a node reboot or a partial apply
* Bad, because secrets handling reverts to ad-hoc scripts, undoing the ADR-0001 model

## More Information

Implementation — `flux bootstrap`, repo layout under `kubernetes/clusters/`, and the first `Kustomization` consuming a SOPS-encrypted secret — is tracked in #30. Cluster prerequisite is #29 (k3s bootstrap); manifest validation for PRs touching `kubernetes/` is #7. Revisit this decision if the homelab grows to multiple clusters (Fleet or ArgoCD ApplicationSets become proportionate), or if a UI becomes a hard requirement and weave-gitops / Headlamp prove insufficient.

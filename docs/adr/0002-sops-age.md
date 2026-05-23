# ADR-0002 — Use SOPS + age for secrets in git

* Status: accepted
* Date: 2026-05-23
* Issue: #3

## Context and Problem Statement

Secrets (API keys, passwords, kubeconfig tokens) must live alongside the infrastructure code that uses them. Storing them in plaintext is unacceptable. The solution must work uniformly across Flux v2, Ansible, and Terraform without requiring a running secrets-management service.

## Decision Drivers

* Flux v2 has first-class SOPS support — no extra tooling needed in the cluster
* Key management must be simple: no GPG web of trust, no daemon, no cloud dependency
* Encrypted files should remain valid YAML so diffs stay readable and auditable
* Must cover all three workloads: Kubernetes manifests (Flux), host vars (Ansible), tfvars (Terraform)

## Considered Options

* **SOPS + age** (chosen)
* git-crypt
* Sealed Secrets
* Vault + External Secrets Operator
* ansible-vault

## Decision Outcome

Chosen option: **SOPS + age**, because it is the only option that satisfies all four drivers with no additional infrastructure. Age replaces GPG with a simpler key model; SOPS encrypts only secret values (not structure), keeping diffs useful; and Flux's native `sops-age` decryption eliminates any in-cluster operator.

### Consequences

* Good: single `.sops.yaml` config covers Flux, Ansible, and Terraform with no per-tool plugins
* Good: age keys are plain files — easy to rotate, back up, and hand to CI as an env var
* Watch out: losing the private key without a backup means losing access to all secrets — key backup and rotation procedure is required (tracked in #3)
* Watch out: no dynamic secret leasing or automatic rotation; secrets are static until manually re-encrypted

## Options Considered

### SOPS + age

SOPS encrypts individual YAML/JSON values in place; age provides the key primitive. Flux v2 natively decrypts SOPS-encrypted manifests using a key stored as a cluster secret.

* Good, because Flux supports it natively — no extra controller
* Good, because age has no keyring, no expiry, no GPG trust model to manage
* Good, because encrypted YAML preserves structure; `git diff` shows which keys changed
* Good, because one tool covers Kubernetes manifests, Ansible vars, and Terraform files
* Bad, because key rotation requires re-encrypting all files (mitigated by scripting)

### git-crypt

Transparent git filter that encrypts files on push and decrypts on pull.

* Good, because it is invisible once configured — no per-file encrypt step
* Bad, because it encrypts entire files, making diffs unreadable
* Bad, because Flux has no native git-crypt support — would need a pre-deploy decrypt step
* Bad, because symmetric-key mode is fragile; GPG mode inherits GPG complexity

### Sealed Secrets

Bitnami controller encrypts Kubernetes Secrets into `SealedSecret` CRDs; the cluster holds the private key.

* Good, because it integrates cleanly with Kubernetes RBAC
* Bad, because you need a running cluster to encrypt — chicken-and-egg during bootstrap
* Bad, because it is Kubernetes-only; Ansible and Terraform secrets need a separate solution
* Bad, because the controller manages the private key, making backup and rotation harder

### Vault + External Secrets Operator

HashiCorp Vault stores secrets dynamically; ESO syncs them into Kubernetes Secrets at runtime.

* Good, because it supports dynamic secrets, leasing, and audit logging
* Good, because it is the industry standard for larger environments
* Bad, because it requires running and maintaining Vault as infrastructure
* Bad, because it is significant operational overhead for a homelab — disproportionate complexity
* Bad, because secrets are not in git at all, which complicates bootstrapping and DR

### ansible-vault

Built-in Ansible encryption for vars files and strings.

* Good, because it requires no extra tooling for Ansible users
* Bad, because it is Ansible-only — Flux and Terraform have no support
* Bad, because it uses a symmetric passphrase, making key rotation and CI integration awkward

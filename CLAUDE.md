# CLAUDE.md

Conventions for this repository — applies to human and agentic contributors alike.

## What this repo is

IaC for a self-hosted homelab: two Proxmox nodes, two Raspberry Pis, OPNsense router/firewall, Ubiquiti switch and AP. Stack: Terraform (provisioning), Ansible (configuration), Flux v2 (GitOps / k3s apps), SOPS + age (secrets). Serves as both running home infrastructure and a DevOps portfolio reference.

## Development workflow

Every change follows this sequence:

1. **Issue** — all work starts from a GitHub issue
2. **Review & refine** — read the issue; edit or comment until the scope is clear
3. **Discuss with agent** — surface open questions, adapt the issue until complete
4. **Plan** — agent proposes an implementation plan; discuss and adapt until approved
5. **Implement** — agent implements; may use sub-agents for independent workstreams
6. **PR review** — review the diff, request changes, iterate until ready
7. **Merge** — squash-merge to `main`
8. **Wrap up** — open or update follow-up issues as needed

> **Agents:** propose before implementing. Ask when unclear. State assumptions explicitly.

## Git

- **Never push to `main`** — all changes go through a reviewed PR
- Branch naming: `feat/`, `fix/`, `chore/`, `docs/` + short slug (e.g. `feat/snapcast-server-role`)
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): short description`
  - Valid types: `feat`, `fix`, `chore`, `docs`, `ci`, `refactor`, `security`
- PRs reference their issue: `Closes #N` in the PR description
- Keep PRs small and focused on one issue

## Repository layout

| Path | What lives here |
|---|---|
| `docs/adr/` | Architecture Decision Records — why decisions were made |
| `docs/runbooks/` | Break-glass and operational procedures |
| `terraform/` | Infrastructure provisioning (Proxmox, networking, DNS, Tailscale) |
| `ansible/` | Host and service configuration (Proxmox hosts, Pis, Snapcast) |
| `kubernetes/` | Flux-managed cluster workloads and infrastructure |
| `.github/workflows/` | CI — terraform plan, ansible-lint, manifest validation |

## CI

- **Pin GitHub Actions by SHA.** Use `uses: owner/action@<sha> # vN` — tags are mutable and can be silently overwritten. Resolve the current SHA with:
  ```
  git ls-remote https://github.com/owner/action refs/tags/vN
  ```
- **Always post new comments, never edit existing ones.** Updating a comment in-place hides history and makes the PR timeline confusing.

## Principles

- **As-code first.** If it can be automated, it should be. Manual steps are explicit exceptions with a documented reason.
- **No plaintext secrets.** SOPS + age for in-repo secrets; environment variables for CI credentials. When in doubt, stop and check before committing.
- **Decisions in ADRs.** Non-obvious architectural choices get an ADR — not a comment or a Slack message.
- **Small PRs.** One issue → one PR. Large PRs are hard to review and hard to revert.
- **Idempotent.** Ansible roles and Terraform modules must be safe to re-run without unintended side effects.

## ADRs

When recording an architectural decision:

- Copy `docs/adr/0000-template.md` and keep its structure (do not drop sections)
- File name: `NNNN-short-kebab-slug.md`, using the next free number from `ls docs/adr/`
- Reference the issue that prompted the decision in the header
- If a concurrently-merging PR took your number, rename the file and update any cross-references before merging

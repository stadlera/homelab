# ADR-0003 — Authenticate self-hosted Renovate via a custom GitHub App

* Status: accepted
* Date: 2026-05-24
* Issue: #4

## Context and Problem Statement

The Renovate GitHub Action (`renovatebot/github-action`) needs a token to push branches, open PRs, and update the Dependency Dashboard issue. The default workflow `GITHUB_TOKEN` is insufficient: it cannot carry the `workflow` scope required by Renovate's `github-actions` manager, and PRs it creates do not trigger `pull_request` workflow events — which would silently skip the `pr-title-lint` check on every Renovate PR. A different credential is required.

## Decision Drivers

* Renovate PRs must trigger downstream CI (currently `pr-title-lint`, future Terraform/Ansible/manifest validation)
* Token should not be tied to an individual user — the repo must keep working if any single contributor leaves
* Prefer short-lived credentials over long-lived ones (defence-in-depth if the token is leaked)
* Permissions should be scoped to this single repository
* No dependence on third-party hosted services for the auth path

## Considered Options

* Classic Personal Access Token
* Fine-grained Personal Access Token
* Custom GitHub App

## Decision Outcome

Chosen option: **Custom GitHub App**, because it is the only option that combines short-lived tokens (1 hour, minted fresh each run), repository-scoped permissions, independence from any user account, and PR-trigger semantics that propagate to downstream workflows.

### Consequences

* Good, because no long-lived secret exists in the workflow — only the App's private key, which is itself rotatable without disrupting workflows
* Good, because Renovate PRs appear as `<app-name>[bot]`, clearly distinguishing automated changes from human ones in audit history
* Good, because the App can be revoked or uninstalled independently of any user account
* Bad, because initial setup requires manual steps in the GitHub UI (App creation, key generation, installation, secret storage) — documented under "Confirmation" below
* Bad, because the App's private key is now an asset to manage — must be rotated periodically

### Confirmation

The workflow `.github/workflows/renovate.yml` references two secrets:

* `RENOVATE_CLIENT_ID` — the Client ID shown in the App's settings page (the `Iv1.xxxx` string, **not** the numeric App ID)
* `RENOVATE_APP_PRIVATE_KEY` — the contents of the `.pem` file generated for the App

Setup procedure (one-time, manual):

1. Create a new GitHub App at `https://github.com/settings/apps/new` (owner: personal account)
2. Permissions (Repository):
   * Contents: Read and write
   * Issues: Read and write
   * Pull requests: Read and write
   * Workflows: Read and write
   * Metadata: Read-only (default)
3. Subscribe to no events; uncheck "Active" under Webhook
4. Generate a private key — download the `.pem` file
5. Install the App on `stadlera/homelab` only (Install App → Only select repositories)
6. Add repo secrets at Settings → Secrets and variables → Actions:
   * `RENOVATE_CLIENT_ID` = the Client ID from the App's settings page (the `Iv1.xxxx` string, **not** the numeric App ID)
   * `RENOVATE_APP_PRIVATE_KEY` = full contents of the `.pem` (including `-----BEGIN/END-----` lines)
7. Trigger the workflow manually (Actions → Renovate → Run workflow) to verify

## Pros and Cons of the Options

### Classic Personal Access Token

A long-lived PAT with `repo` + `workflow` scopes, stored as a single repo secret.

* Good, because setup is the simplest (one secret, one workflow input)
* Bad, because the token is tied to a user account and long-lived
* Bad, because `repo` scope grants access to every repository the user can see, not just this one
* Bad, because rotation is manual and easy to forget

### Fine-grained Personal Access Token

A PAT scoped to `stadlera/homelab` only, with explicit per-permission grants.

* Good, because scope can be narrowed to this single repository
* Bad, because the token is still tied to a user and still long-lived
* Bad, because fine-grained PATs currently have a maximum lifetime measured in months, requiring periodic renewal

### Custom GitHub App

A GitHub App created in the owner's account, installed only on this repository. Each workflow run mints a fresh installation token via `actions/create-github-app-token`.

* Good, because tokens are short-lived (1 hour) and minted fresh each run
* Good, because permissions are scoped per-installation
* Good, because the App is owned by the account, not a user identity — survives any contributor leaving
* Neutral, because one private key must be stored — but it is the only long-lived asset and rotation requires no workflow change
* Bad, because initial setup is more involved than a PAT

## More Information

* `actions/create-github-app-token`: https://github.com/actions/create-github-app-token
* GitHub docs — Authenticating with a GitHub App in a workflow: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/making-authenticated-api-requests-with-a-github-app-in-a-github-actions-workflow
* Renovate Action — GitHub App example: https://github.com/renovatebot/github-action#self-hosted-github-app

## History

* 2026-05-24 — Initial decision (this ADR), issue #4

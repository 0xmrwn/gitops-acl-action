# GitHub Action — Sync Tailscale ACLs

Deterministic GitOps sync/tests for Tailscale ACLs using **pinned** versions of `gitops-pusher` and Go.
Avoids surprise toolchain downloads and works with newer ACL semantics like `autogroup:owner`.

> [!NOTE]
> **This is a maintained fork of `tailscale/gitops-acl-action`**.
> **Why a fork?**
>
> * Pin **both** Go (≥ 1.23) and `gitops-pusher` to current versions for deterministic CI.
> * Stop `go run` from downloading toolchains on every run (we set `GOTOOLCHAIN=local` and `go install` the CLI).
> * Remove redundant `actions/setup-go` when the caller already set up Go.
> * Provide built-in, keyed caching for Go modules and build cache.
> * Keep pace with recent Tailscale policy semantics (e.g. `autogroup:owner`) validated by an up-to-date `gitops-pusher`.

## Inputs
| Input                    | Required | Default           | Description                                                                                      |
|--------------------------|----------|-------------------|--------------------------------------------------------------------------------------------------|
| `tailnet`                | Yes      | —                 | Tailnet name (e.g., `example.com`)                                                               |
| `api-key`                | No       | —                 | Tailscale API key (expires in up to 90 days; prefer OAuth for long-lived setups)                 |
| `oauth-client-id`        | No       | —                 | Tailscale OAuth client ID (scope: `acl`). Use with `oauth-secret` instead of API key             |
| `oauth-secret`           | No       | —                 | Tailscale OAuth client secret. Use with `oauth-client-id` instead of API key                     |
| `policy-file`            | No       | `./policy.hujson` | Path to HuJSON policy file                                                                       |
| `action`                 | Yes      | —                 | Action to take: `test` or `apply`                                                                |
| `gitops-pusher-version`  | No       | `v1.86.4`         | Version/tag/commit for `tailscale.com/cmd/gitops-pusher`                                         |
| `go-version`             | No       | `1.23.11`         | Go version to ensure if needed (only sets up Go if your job hasn’t already)                      |

## Usage (GitHub Actions workflow)
Create `.github/workflows/tailscale.yml` in your **policy repo**:

```yaml
name: Sync Tailscale ACLs

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  acls:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4

      # Optional: if you already set up Go elsewhere in your job, the action won't run setup-go again.

      - name: Test on PRs
        if: github.event_name == 'pull_request'
        uses: YOUR_GH_ORG/your-gitops-acl-action@v1
        with:
          tailnet: ${{ secrets.TS_TAILNET }}
          api-key: ${{ secrets.TS_API_KEY }} # or use oauth-client-id / oauth-secret
          action: test

      - name: Apply on main
        if: github.event_name == 'push'
        uses: YOUR_GH_ORG/your-gitops-acl-action@v1
        with:
          tailnet: ${{ secrets.TS_TAILNET }}
          api-key: ${{ secrets.TS_API_KEY }}
          action: apply
````

> [!NOTE]
>
> * `gitops-pusher` CLI and usage come from the official module. We install it via `go install tailscale.com/cmd/gitops-pusher@<version>`.
> * **Newer ACL selectors** like `autogroup:owner` are supported by current Tailscale policy syntax and are validated by the latest `gitops-pusher`.
> * Prefer OAuth for unattended setups (API tokens expire within **1–90 days**, depending on how you configured them).

## Release process

1. Merge changes to `main`.
2. Tag a semver release, e.g. `git tag v1.0.0 && git push --tags`.
3. Create a GitHub release with notes.
4. Move the major tag: `git tag -f v1 v1.0.0 && git push -f origin v1`.

## Why this action?

* Pins **both** Go and `gitops-pusher` for repeatable CI.
* Avoids Go auto-toolchain downloads that wreck caching. ([Brandur][4])
* Only sets up Go when needed.

---

# 5) Example: using OAuth instead of API key

```yaml
- name: Apply with OAuth
  if: github.event_name == 'push'
  uses: YOUR_GH_ORG/your-gitops-acl-action@v1
  with:
    tailnet: ${{ secrets.TS_TAILNET }}
    oauth-client-id: ${{ secrets.TS_OAUTH_ID }}
    oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
    action: apply
````

(These env names map to what `gitops-pusher` expects under the hood.)

---

# 6) How this aligns with official docs

* We follow the **official GitOps guidance** (test on PRs, apply on main).
* We install `gitops-pusher` via `go install tailscale.com/cmd/gitops-pusher@<tag>` just like the docs suggest (but pinned, not `@latest`).
* We explicitly support modern selectors like **`autogroup:owner`** (documented under `autogroup:<role>`).
* We pick a sane default **Go 1.23.11** (latest patch in 1.23) which satisfies the ≥1.23 requirement and avoids unnecessary churn from 1.25 unless you want it.
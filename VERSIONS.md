# Version compatibility matrix

Each pleme-io action ships independent semver. Within a major (`v1`), inputs and outputs declared `v1_stable` in the action's typed declaration are immutable. Breaking changes bump the major.

## Initial-wave actions (v0.1.0)

All 11 actions ship at `v0.1.0` initially while the API is shaken out against real consumers. **Treat `v0.x.y` as unstable**: any minor bump may break input/output schema. Lock down `v1.0.0` happens once a downstream consumer's stability needs justify it.

| Action | Latest | v1.0.0 ETA |
|---|---|---|
| `terragrunt-apply` | `v0.1.0` | After first downstream consumer migration |
| `k8s-pause-verify` | `v0.1.0` | After first downstream consumer migration |
| `helmworks-render-check` | `v0.1.0` | After first downstream consumer migration |
| `argocd-app-sync` | `v0.1.0` | After first downstream consumer migration |
| `kubectl-wait` | `v0.1.0` | After first downstream consumer migration |
| `github-app-installation-token` | `v0.1.0` | After first downstream consumer migration |
| `slack-notify` | `v0.1.0` | After first downstream consumer migration |
| `eks-kubeconfig-update` | `v0.1.0` | After first downstream consumer migration |
| `flux-reconcile` | `v0.1.0` | After first downstream consumer migration |
| `tameshi-attest` | `v0.1.0` | After first downstream consumer migration |
| `multi-arch-image-release` | `v0.1.0` | After first downstream consumer migration |

## Pinning recommendations

| Pin | Behavior | Use when |
|---|---|---|
| `@v1` | Auto-tracks 1.x.y bug fixes | Most cases — recommended |
| `@v1.2` | Auto-tracks 1.2.x patches | Want minor stability + patch fixes |
| `@v1.2.3` | Exact version, immutable | Production where you control the upgrade window |
| `@<40-char-sha>` | Pinned to a specific commit | Max security — supply-chain audit needs |
| `@main` | **Never** — see anti-pattern below | — |

## Anti-pattern: `@main`

Pinning to `@main` is forbidden in pleme-io workflows. The branch can change at any time; consumers get unpredictable behavior; supply-chain attacks are mitigated by tag-pinning. The substrate's CI lints workflows for `@main` pins on pleme-io actions and fails the PR.

## Coordinated multi-action releases

If a substrate-level change in `pleme-actions-shared` requires every action to bump, releases happen in dependency order:

1. `pleme-actions-shared` → bump version + publish to crates.io
2. Each action's `Cargo.toml` → update `pleme-actions-shared = "<new>"`
3. Each action → tag a new minor (typically) or major (if the shared crate's change is breaking)

This is documented in [`CONTRIBUTING.md`](./CONTRIBUTING.md) under "Coordinated releases."

# pleme-actions

> The discovery hub for pleme-io's reusable GitHub Actions surface.

Each action is a published Rust binary + composite `action.yml`, generated from a typed `Action` declaration in [`arch-synthesizer/src/action_domain/`](https://github.com/pleme-io/arch-synthesizer/tree/main/src/action_domain) and built via substrate's [`rust-action-release-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/build/rust/action-release-flake.nix). Consumers reference via `uses: pleme-io/<action>@v1`.

**Canonical specification:** [`pleme-io/theory/CONSTRUCTIVE-ACTIONS.md`](https://github.com/pleme-io/theory/blob/main/CONSTRUCTIVE-ACTIONS.md).
**Implementation plan:** [`pleme-io/theory/PLEME-ACTIONS-PLAN.md`](https://github.com/pleme-io/theory/blob/main/PLEME-ACTIONS-PLAN.md).
**Skill:** [`pleme-actions`](https://github.com/pleme-io/blackmatter-pleme/blob/main/skills/pleme-actions/SKILL.md).

---

## The 11 published actions

### IaC

| Action | One-liner |
|---|---|
| [`pleme-io/terragrunt-apply`](https://github.com/pleme-io/terragrunt-apply) | Run terragrunt plan/apply/destroy with typed inputs and structured outputs (plan-summary, applied-resources, state-version) |

### Kubernetes

| Action | One-liner |
|---|---|
| [`pleme-io/kubectl-wait`](https://github.com/pleme-io/kubectl-wait) | Typed kubectl wait wrapper — pod ready, deployment available, CRD established |
| [`pleme-io/eks-kubeconfig-update`](https://github.com/pleme-io/eks-kubeconfig-update) | `aws eks update-kubeconfig` with identity + reachability checks |
| [`pleme-io/k8s-pause-verify`](https://github.com/pleme-io/k8s-pause-verify) | Assert pause-contract invariants on a paused-first ARC runner pool |

### GitOps

| Action | One-liner |
|---|---|
| [`pleme-io/argocd-app-sync`](https://github.com/pleme-io/argocd-app-sync) | Trigger an ArgoCD Application sync + wait for Synced/Healthy |
| [`pleme-io/flux-reconcile`](https://github.com/pleme-io/flux-reconcile) | Force a FluxCD reconcile + wait for Ready |

### Helm

| Action | One-liner |
|---|---|
| [`pleme-io/helmworks-render-check`](https://github.com/pleme-io/helmworks-render-check) | Pull a helmworks chart, render with operator values, fail on contract violations (pre-merge gate) |

### Auth + signaling

| Action | One-liner |
|---|---|
| [`pleme-io/github-app-installation-token`](https://github.com/pleme-io/github-app-installation-token) | Issue a GitHub App installation access token from App credentials (output is masked) |
| [`pleme-io/slack-notify`](https://github.com/pleme-io/slack-notify) | Typed Slack webhook notification with Block Kit payload |

### Release pipeline

| Action | One-liner |
|---|---|
| [`pleme-io/tameshi-attest`](https://github.com/pleme-io/tameshi-attest) | Compute BLAKE3 attestation hashes for release artifacts (files or directory trees) |
| [`pleme-io/multi-arch-image-release`](https://github.com/pleme-io/multi-arch-image-release) | Combine N per-arch OCI manifests into a single multi-arch image tag |

---

## How to consume

```yaml
- uses: pleme-io/<action>@v1   # auto-tracks 1.x.y bug fixes (recommended)
- uses: pleme-io/<action>@v1.2 # auto-tracks 1.2.x
- uses: pleme-io/<action>@v1.2.3              # exact version
- uses: pleme-io/<action>@<40-char-sha>       # max security
```

**Never `@main`** — pin a tag. Each action's release workflow auto-advances the moving major-version tag (`v1` → latest 1.x.y) on every release.

## Worked example

A typical pleme-io fleet workflow that 6 months ago took ~200 lines of inline shell + tool installs collapses to ~25 lines of declarative `uses:`:

```yaml
on: { workflow_dispatch: }
jobs:
  apply:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: ${{ secrets.DEPLOY_ROLE }}, aws-region: us-east-2 }
      - uses: pleme-io/eks-kubeconfig-update@v1
        with: { cluster-name: my-cluster, region: us-east-2 }
      - uses: pleme-io/terragrunt-apply@v1
        with:
          working-directory: leaves/my-leaf
          action: apply
          tf-vars: |
            {"github_token": "${{ secrets.MY_PAT }}"}
      - uses: pleme-io/argocd-app-sync@v1
        with: { app-name: my-app, kubectl-context: my-cluster }
      - uses: pleme-io/k8s-pause-verify@v1
        with: { namespace: my-ns, runner-set-label: my-pool }
      - if: failure()
        uses: pleme-io/slack-notify@v1
        with:
          webhook-url: ${{ secrets.OPS_WEBHOOK }}
          status: failure
          title: Deploy failed
          message: "${{ github.workflow }} failed on ${{ github.ref_name }}"
```

---

## Architecture

```
arch-synthesizer/src/action_domain/      ← the typed Action domain
   │   (Rust struct + render targets, mirrors the (defaction …) Lisp form)
   │
   ▼ renders to
pleme-io/<action-name>/                  ← per-action repo
   ├── flake.nix                         (consumes substrate's rust-action-release-flake)
   ├── Cargo.toml                        (depends on pleme-actions-shared)
   ├── src/main.rs                       (operator-authored binary logic)
   └── action.yml                        (composite, generated from typed declaration)
   │
   ▼ packaged + released by
substrate/lib/build/rust/action-release-flake.nix   ← the build/release primitive
   │
   ▼ at runtime, the binary reads inputs via
pleme-io/pleme-actions-shared            ← typed Input/Output/StepSummary/error/log
```

Adding a new action is one typed declaration + a `cargo new --bin` fill-in of `src/main.rs`. Everything else (action.yml, flake.nix, release workflow, README skeleton, repo posture) is rendered.

---

## Versioning

See [`VERSIONS.md`](./VERSIONS.md) for the version-compat matrix across all 11 actions.

Each action ships independent semver. Breaking changes bump the major (`v2` is opt-in). Within `v1`, the inputs and outputs declared as `v1_stable` in the action's typed declaration are immutable.

---

## Attestation

See [`ATTESTATION.md`](./ATTESTATION.md) for how to verify a downloaded action binary against its release attestation chain.

---

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md). Short version: edit the typed `Action` declaration → re-render → tag a new version. Never hand-author `action.yml` or `flake.nix`.

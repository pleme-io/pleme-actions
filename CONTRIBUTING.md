# Contributing to pleme-actions

> The factory pattern, in short: every action is a typed `Action` value in arch-synthesizer that renders mechanically to a per-action GitHub repo. Authoring a new action = one declaration + filling in `src/main.rs`.

## Prerequisites

- Working pleme-io dev environment (Nix + flakes)
- Read [`pleme-actions` skill](https://github.com/pleme-io/blackmatter-pleme/blob/main/skills/pleme-actions/SKILL.md) — the operator playbook
- Read [`theory/CONSTRUCTIVE-ACTIONS.md`](https://github.com/pleme-io/theory/blob/main/CONSTRUCTIVE-ACTIONS.md) — the canonical pattern spec

## Adding a new action

### 1. Declare the action in arch-synthesizer

Edit `arch-synthesizer/src/action_domain/fixtures.rs` to add a typed `Action` constructor. Run the existing tests as a regression gate:

```bash
cd ~/code/github/pleme-io/arch-synthesizer
cargo test action_domain
```

The constructor goes through the same `validate()` gate as the existing 11 fixtures — kebab-case naming, semver-policy ↔ inputs/outputs consistency, etc.

### 2. Scaffold the per-action repo

```bash
mkdir -p ~/code/github/pleme-io/<action-name>/src
cd ~/code/github/pleme-io/<action-name>
```

Files (same shape across all 11 existing actions; copy any one as a starting point):

- `Cargo.toml` — depends on `pleme-actions-shared`, picks `serde`/`serde_json`/`anyhow`/etc as needed
- `flake.nix` — consumes substrate's `rust-action-release-flake.nix`, declares the same `action` attrset that the `Action` typed declaration mirrors
- `src/main.rs` — the binary's logic (operator-authored)
- `README.md`, `.gitignore`, `.envrc`

### 3. Author `src/main.rs`

The shape every action follows:

```rust
use pleme_actions_shared::{ActionError, Input, Output, StepSummary};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Inputs {
    // declared inputs as fields, snake_case (matches kebab-case input
    // names via INPUT_<UPPER>__name uppercasing)
}

fn main() {
    pleme_actions_shared::log::init();
    if let Err(e) = run() {
        e.emit_to_stdout();
        if e.is_fatal() {
            std::process::exit(1);
        }
    }
}

fn run() -> Result<(), ActionError> {
    let inputs = Input::<Inputs>::from_env()?;
    // … action logic …
    let output = Output::from_runner_env()?;
    output.set("my-output", "value")?;
    let mut summary = StepSummary::from_runner_env()?;
    summary.heading(2, "my action").table(…);
    summary.commit()?;
    Ok(())
}
```

### 4. Test locally

```bash
cargo test
```

Pure-function logic gets unit-tested. External-tool integration (kubectl, aws, helm, regctl) is exercised in CI smoke tests against a real cluster — keep `src/main.rs` testable by extracting parsing + assertion logic into pure functions.

### 5. Register the repo

Per the [`pleme-io-github-posture`](https://github.com/pleme-io/blackmatter-pleme/blob/main/skills/pleme-io-github-posture/SKILL.md) skill:

```bash
$EDITOR ~/code/github/pleme-io/pangea-architectures/workspaces/pleme-io-opensource/org.yaml
# Append a `- name: <action-name>` entry mirroring an existing action's posture.

cd ~/code/github/pleme-io/pangea-architectures
nix run .#flow-plan-pleme-io-opensource    # confirm only your add appears
nix run .#flow-deploy-pleme-io-opensource  # creates the repo on github.com
```

### 6. Push the code

```bash
cd ~/code/github/pleme-io/<action-name>
git init -b main
git add -A
git commit -m "init: <action-name> — pleme-io GitHub Action"
git remote add origin git@github.com:pleme-io/<action-name>.git
git push -u origin main
```

### 7. Tag the first release

```bash
git tag v0.1.0
git push origin v0.1.0
```

The release workflow rendered by substrate's `rust-action-release-flake` builds cross-platform binaries + creates a GitHub release.

### 8. Add to the index

Edit `~/code/github/pleme-io/pleme-actions/README.md` — add a row in the appropriate domain table.

## Coordinated releases

When a `pleme-actions-shared` change requires bumping every action:

1. Bump `pleme-actions-shared` version, publish to crates.io
2. For each action: update `Cargo.toml`'s `pleme-actions-shared = "..."`, tag minor (or major if breaking)
3. Update [`VERSIONS.md`](./VERSIONS.md) with the new versions

## Lifting forge / substrate patterns

If a forge command or substrate primitive has reuse value beyond pleme-io's monorepos, it's a candidate for an action. See `tameshi-attest`, `flux-reconcile`, `multi-arch-image-release` — all lifted from forge's command modules into standalone actions.

The lift pattern: keep forge as the heavyweight implementation, the action as the thin reusable surface. The action's `src/main.rs` typically reproduces the forge command's logic concisely (no full forge dependency tree) and stays focused on the contract.

## Anti-patterns

- **Hand-authoring `action.yml`** — always rendered from the typed declaration
- **Cross-action dependencies** — each action is independent
- **Container actions** — composite + Rust binary download is the only approved shape
- **`@main` in consumer references** — see [`VERSIONS.md`](./VERSIONS.md)
- **Skipping the typed declaration** — registers the action in arch-synthesizer's typescape; without it, drift detection and version audits can't see the action

## Related

- [`pleme-actions` skill](https://github.com/pleme-io/blackmatter-pleme/blob/main/skills/pleme-actions/SKILL.md)
- [`repo-forge` skill](https://github.com/pleme-io/blackmatter-pleme/blob/main/skills/repo-forge/SKILL.md)
- [`pleme-io-github-posture` skill](https://github.com/pleme-io/blackmatter-pleme/blob/main/skills/pleme-io-github-posture/SKILL.md)
- [`claude-skills` skill](https://github.com/pleme-io/blackmatter-pleme/blob/main/skills/claude-skills/SKILL.md)

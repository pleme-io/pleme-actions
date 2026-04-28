# Verifying a pleme-io action binary

Each action's release ships an attestation chain — source SHA + build hash + binary BLAKE3 — surfaced in the GitHub Release notes. Consumers can verify a downloaded binary matches the documented source commit.

## What's attested

Every release of `pleme-io/<action>` carries:

| Field | Value | Verifier |
|---|---|---|
| `source_sha` | Git SHA of the source commit the binary was built from | `git rev-list --max-count=1 v<X.Y.Z>` |
| `build_hash` | BLAKE3 of the binary | `blake3sum <downloaded-binary>` |
| `release_tag` | The semver tag | matches the URL the consumer downloaded from |

The `pleme-io/tameshi-attest` action computes and emits these into the release body when it runs as part of the action's own release workflow.

## Verification flow (consumer-side)

1. Pin to a specific tag in the consumer workflow:

   ```yaml
   - uses: pleme-io/<action>@v0.1.0
   ```

2. Download the binary the action's action.yml resolves at runtime:

   ```bash
   gh release download v0.1.0 --repo pleme-io/<action>
   ```

3. Compute its BLAKE3 hash:

   ```bash
   blake3sum <action-name>-linux-amd64
   # → ea8f163db38682925e4491c5e58d4bb3506ef8c14eb78a86e908c5624a67200f  <action-name>-linux-amd64
   ```

4. Compare against the hash in the release body:

   ```bash
   gh release view v0.1.0 --repo pleme-io/<action> --json body --jq .body | grep blake3
   ```

5. Optionally, walk the source SHA:

   ```bash
   gh api repos/pleme-io/<action>/git/ref/tags/v0.1.0 --jq .object.sha
   # then verify the local build matches
   ```

## SHA pinning for max-security workflows

For consumers that need cryptographic pinning to a known-good build:

```yaml
- uses: pleme-io/<action>@<40-char-commit-sha>
```

GitHub guarantees the commit SHA is immutable; combined with the release-side BLAKE3 attestation, this gives end-to-end content-addressing of the binary the workflow runs.

## Cross-references

- [`pleme-io/tameshi-attest`](https://github.com/pleme-io/tameshi-attest) — the action that emits attestation hashes
- [`tameshi`](https://github.com/pleme-io/tameshi) — the core attestation library (BLAKE3, Merkle, compliance)
- [`sekiban`](https://github.com/pleme-io/sekiban) — K8s admission webhook gating deploys on attestation hashes

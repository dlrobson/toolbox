# toolbox

Central repository for shared infrastructure across all `dlrobson` projects. This repo owns and publishes the base Rust CI image, provides reusable GitHub Actions workflows, and runs Renovate dependency updates for all managed repositories.

## Contents

### Base Rust CI Image (`images/ci/`)

A Rust CI Docker image published to `ghcr.io/dlrobson/rust-ci`. It includes:

- Rust stable + nightly toolchains with `clippy`, `rust-analyzer`, `rustfmt`, and `llvm-tools-preview`
- Docker CLI + Buildx plugin (for building images inside CI)
- [`cargo-binstall`](https://github.com/cargo-bins/cargo-binstall), [`cargo-deny`](https://github.com/EmbarkStudios/cargo-deny), [`cargo-llvm-cov`](https://github.com/taiki-e/cargo-llvm-cov), and [`just`](https://github.com/casey/just)

#### Versioning

Images are tagged with an incrementing build number scoped to the Rust version (e.g. `1.93.1-0`, `1.93.1-1`). A new number is assigned each time any file under `images/ci/` changes — whether that's the Rust base version, a tool version, or anything else. This means:

- Every distinct build is immutable and permanently addressable by its full tag.
- The Rust version component tells you at a glance what toolchain is inside.
- A `sha-{hash}` tag is also applied for internal deduplication (content-identical rebuilds are skipped).
- On `main`, the mutable aliases `{rust-version}` (e.g. `1.93.1`) and `latest` are updated to point to the newest build.

| Tag | Mutability | Use case |
|---|---|---|
| `1.93.1-3` | Immutable | Pin to a specific known-good build |
| `sha-a3f2b1c4` | Immutable | Internal deduplication anchor |
| `1.93.1` | Mutable | "Latest build for this Rust version" |
| `latest` | Mutable | "Latest build overall" |

The image is automatically published on every push to `main` that changes files under `images/ci/`.

To use this image in another repository:

```dockerfile
# Pin to an immutable build (recommended for production)
FROM ghcr.io/dlrobson/rust-ci:1.93.1-3

# Or track the latest for a given Rust version
FROM ghcr.io/dlrobson/rust-ci:1.93.1
```

### Reusable Workflows (`.github/workflows/`)

These callable workflows can be referenced from any repository:

| Workflow | Purpose |
|---|---|
| `prepare-ci-image.yml` | Content-hash–based build/retag orchestrator for repo-specific CI images. Builds only when the Dockerfile context changes; retags to `latest` on `main`. |
| `build-image.yml` | Builds and pushes a Docker image with BuildKit layer caching. |
| `retag-image.yml` | Retags an existing image to a new tag using `imagetools`. |

Example — calling `prepare-ci-image.yml` from another repo to manage its own CI image:

```yaml
jobs:
  prepare-ci-image:
    uses: dlrobson/toolbox/.github/workflows/prepare-ci-image.yml@main
    permissions:
      packages: write
    with:
      registry: ghcr.io
      image-name: ${{ github.repository }}-ci
      dockerfile-context: images/ci
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

### Renovate (`.github/workflows/renovate.yml` + `.github/renovate.json`)

A self-hosted [Renovate](https://docs.renovatebot.com/) instance runs every 12 hours and manages dependencies across all listed repositories. The target repositories are declared in `.github/renovate.json` under the `repositories` key.

To add a repository to Renovate management, append it to the `repositories` array in `.github/renovate.json`:

```json
"repositories": [
    "dlrobson/toolbox",
    "dlrobson/rust-starter",
    "dlrobson/your-new-repo"
]
```

Renovate handles updates for:

- Rust toolchain (`rust-toolchain.toml`)
- Cargo dependencies (`Cargo.toml`)
- `cargo-binstall` version in Dockerfiles
- Binary tool versions in `binstall-versions.json` files
- GitHub Actions dependencies

**Required secret:** `RENOVATE_TOKEN` — a GitHub PAT with `repo` scope across all managed repositories.

## Adding a New Managed Repository

1. Add the repo name to `repositories` in `.github/renovate.json`.
2. In the new repo, remove any existing `renovate.json` and `renovate.yml` workflow.
3. In the new repo's CI workflow, call `prepare-ci-image` from this toolbox rather than maintaining a local copy.
4. If the new repo uses the shared Rust CI image, reference `ghcr.io/dlrobson/rust-ci:{tag}`.

## Devcontainer

> Devcontainer management is planned for a future iteration. Stay tuned.

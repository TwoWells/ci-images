# ci-images

Shared CI toolchain container images for the **TwoWells** self-hosted ARC pool
(`homeserver-pool`) — and portable to GitHub-hosted runners, because the toolchain travels with the
job via `container:` rather than being baked into the runner.

## Images

| Image                         | Contents                                                                                  | For                                            |
| ----------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `ghcr.io/twowells/rust-ci`    | Ubuntu 24.04 + build-essential, pkg-config, cmake, libssl-dev, git, rustup (stable + 1.95, rustfmt/clippy) | Rust projects (Lattice, Catenary, …)           |
| `ghcr.io/twowells/omnidsp-ci` _(planned)_ | `rust-ci` + Intel oneMKL + IPP (dynamic)                                       | OmniDSP Intel backends                         |

## Usage

```yaml
jobs:
  check:
    runs-on: homeserver-pool # ← swap to ubuntu-latest anytime; the image carries the toolchain
    container: ghcr.io/twowells/rust-ci:latest
    steps:
      - uses: actions/checkout@v4
      - run: rustc --version && cargo --version
```

Cargo tools (`cargo-nextest`, `cargo-deny`, `cargo-machete`, …) stay per-workflow via
`taiki-e/install-action` (prebuilt binaries) — not baked, to avoid version drift.

## Building

`.github/workflows/build-images.yml` builds and pushes to ghcr on push to `main` (or manual
dispatch), authenticating with the built-in `GITHUB_TOKEN` (`packages: write`). No PAT required.
Add a new image by dropping a `Dockerfile.<name>` and a matrix entry.

# ci-images

Shared CI toolchain container images for the **TwoWells** self-hosted ARC pool
(`homeserver-pool`) — and portable to GitHub-hosted runners, because the toolchain travels with the
job via `container:` rather than being baked into the runner.

## Images

| Image                                     | Contents                                                                                                       | For                                  |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| `ghcr.io/twowells/rust-ci`                | Ubuntu 24.04 + build-essential, pkg-config, cmake, libssl-dev, git, **curl**, **jq**, rustup (stable + 1.95, rustfmt/clippy) | Rust projects (Lattice, Catenary, …) |
| `ghcr.io/twowells/omnidsp-ci` _(planned)_ | `rust-ci` + Intel oneMKL + IPP (dynamic)                                                                        | OmniDSP Intel backends               |

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

## Publishing to crates.io — a copy-safe skip-guard

A publish job usually guards `cargo publish` with "is this version already on crates.io?" so re-runs
are idempotent (a green skip instead of a hard `already exists` error). Three non-obvious things bite
naïve versions of that guard — **all independent of the image**:

1. **crates.io's REST API has a User-Agent gate.** A bare `curl https://crates.io/api/v1/crates/<name>`
   returns **403** unless you send a descriptive `User-Agent`. The **sparse index**
   (`https://index.crates.io/…`) has no such gate — prefer it.
2. **A first-ever release returns 404.** The sparse index 404s for a crate that's never been
   published. Actions runs bash with `-eo pipefail`, so `curl -fsS … | grep` propagates curl's
   non-zero exit and `-e` kills the step — exactly during a crate's first release, when you least
   want it to.
3. **Fail-open vs fail-loud.** A blanket `|| true` swallows *every* curl error (5xx, DNS), not just
   the 404, so a transient hiccup reads as "not published → proceed." Usually benign (`cargo publish`
   hard-rejects a duplicate version anyway), but branch on the status code if you'd rather transient
   errors fail loud.

Sparse-index path prefix rule (cargo's): 1-char name → `1/<name>`, 2-char → `2/<name>`, 3-char →
`3/<c1>/<name>`, 4+ → `<c1c2>/<c3c4>/<name>` (e.g. `serde` → `se/rd/serde`, `catenary` →
`ca/te/catenary`).

### Canonical guard — sparse index, 404-safe, transient-loud

Uses `jq` + `curl` (both in `rust-ci`):

```yaml
      - name: Skip if version already on crates.io
        id: guard
        run: |
          name=$(cargo metadata --no-deps --format-version 1 | jq -r '.packages[0].name')
          ver=$( cargo metadata --no-deps --format-version 1 | jq -r '.packages[0].version')
          case ${#name} in
            1) dir=1/$name ;;
            2) dir=2/$name ;;
            3) dir=3/${name:0:1}/$name ;;
            *) dir=${name:0:2}/${name:2:2}/$name ;;
          esac
          code=$(curl -sS -o /tmp/idx -w '%{http_code}' "https://index.crates.io/$dir/$name" || echo 000)
          case "$code" in
            200) grep -q "\"vers\":\"$ver\"" /tmp/idx && pub=false || pub=true ;;  # already up?
            404) pub=true ;;                                  # never published → first release
            *)   echo "crates.io index returned $code"; exit 1 ;;  # transient → fail loud
          esac
          echo "publish=$pub" >> "$GITHUB_OUTPUT"
          echo "$name $ver → publish=$pub (index HTTP $code)"

      - name: cargo publish
        if: steps.guard.outputs.publish == 'true'
        run: cargo publish --locked
```

**Terser variant (fail-open)** — fine for a handful of crates, since `cargo publish` rejects
duplicates anyway:

```bash
curl -fsS "https://index.crates.io/$dir/$name" | grep -q "\"vers\":\"$ver\"" || true
```

**If you must use the REST API** instead of the sparse index, send a User-Agent:

```bash
curl -fsS -A 'twowells-ci (mwellsa@gmail.com)' "https://crates.io/api/v1/crates/$name"
```

## Building

`.github/workflows/build-images.yml` builds and pushes to ghcr on push to `main` (or manual
dispatch), authenticating with the built-in `GITHUB_TOKEN` (`packages: write`). No PAT required.
Add a new image by dropping a `Dockerfile.<name>` and a matrix entry.

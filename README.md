# trivy (mirror)

Public mirror of **[aquasecurity/trivy](https://github.com/aquasecurity/trivy)**, maintained by TrendSpider.

It keeps a full copy of the upstream source **and** builds our own Trivy binaries from that source, so our CI/CD can pull Trivy from a repository we control instead of depending on the upstream GitHub releases.

---

## Layout

- **`mirror`** — this repository's **default branch**. It is a service branch and holds only the two workflows below + this README. The sync never overwrites it, so the automation keeps running.
- **`main`, `release/*`, tags** — mirrored verbatim from upstream, **except** their `.github/workflows/` is stripped out (see below). The Go source is intact; only upstream CI is removed.

## Workflows

### 1. `mirror-sync` — keep the mirror up to date
- **Triggers:** schedule `0 3 * * *` and `0 15 * * *` (daily 03:00 & 15:00 UTC) + manual (`workflow_dispatch`).
- **What it does:** clones `aquasecurity/trivy` with `--mirror`, drops PR refs, **strips `.github/workflows/` from every branch and tag** using `git-filter-repo`, then force-pushes all branches + tags into this repo.
- **Why strip workflows:** otherwise our push would trigger Trivy's own CI (Test / Release Please / Canary …) on the mirrored branches. Removing the workflow files from the mirrored refs means there is simply nothing for GitHub to run — so Actions on `main` and other branches stay quiet. `build-release` builds from source and does not need those files.
- A safety-net step also disables any upstream workflow that might still be registered.

> Note: stripping workflows rewrites commit hashes, so the mirror's branch/tag hashes differ from upstream. That is expected and does not affect building (the version is taken from the tag name, not the commit).

### 2. `build-release` — build our own binaries
- **Triggers:** automatically via `workflow_run` after every successful `mirror-sync` (so a freshly synced tag is picked up right away) + manual (`workflow_dispatch` with optional `tag` and `scan_only` inputs).
- **Tag selection:** builds the **newest `vX.Y.Z` tag that does not yet have a Release**. If the latest tag already has a Release, it does nothing. Manual runs can target any tag via the `tag` input.
- **Go version:** built with the **latest patch of the Go minor read from the tag's `go.mod`** (e.g. `1.26.x`, not `stable`). Same minor keeps Trivy's experimental `encoding/json/v2` API compatible (so it compiles), while the newest patch already ships the stdlib security fixes — otherwise `govulncheck` would trip over already-patched standard-library CVEs.
- **Security scan (before building):** the source is checked by **five engines**, each preceded by a big banner (tool / version / GitHub / role). Three are blocking **gates**, two are informational; the build proceeds only if all gates pass.
  - **`go mod verify`** (Go toolchain) — **GATE**. Verifies the downloaded modules match `go.sum` bit-for-bit (dependency integrity / no tampering).
  - **`govulncheck`** (official Go `golang.org/x/vuln`) — **GATE**. Reachability-based: blocks only on vulnerabilities actually reachable in the code, at any severity (run with `-show verbose` so non-reachable module vulns are still listed for visibility).
  - **`Grype`** (Anchore) — **GATE**. Independent SCA with its own DB. Trivy's own `testdata/`/`integration/` fixtures (intentionally-vulnerable samples) are excluded; not reachability-aware, so kept at `--only-fixed --fail-on critical`.
  - **`osv-scanner`** (Google OSV) — **INFO**. Scans the root `go.mod` (real dependencies) against the OSV database. No reachability / severity threshold, so it also lists non-reachable vulns — reported, does not block.
  - **`capslock`** (Google) — **INFO**. Capability analysis: what the code + dependencies can actually DO (network, exec, filesystem, unsafe, reflection). Surfaces unexpected/malicious behaviour — reported, does not block.
- **Build:** `go build ./cmd/trivy/` with `GOEXPERIMENT=jsonv2` (Trivy uses the experimental `encoding/json/v2`), version injected via `-X …/pkg/version/app.ver=<ver>`, for **linux/amd64** and **linux/arm64**.
- **Publish:** creates a GitHub **Release** on the tag with assets named exactly like upstream — `trivy_<ver>_Linux-64bit.tar.gz`, `trivy_<ver>_Linux-ARM64.tar.gz`, `trivy_<ver>_checksums.txt`. Creation is idempotent: if the Release already exists it is skipped without error.

## Using it in CI/CD

Asset names match upstream, so the standard download flow works — just point at this repo:

```bash
VER=0.74.0
curl -sfL https://github.com/TrendSpider/trivy/releases/download/v${VER}/trivy_${VER}_Linux-64bit.tar.gz | tar xz
./trivy --version
```

## Requirements / setup

Both workflows push code that includes `.github/workflows/*`, which the default `GITHUB_TOKEN` is not allowed to push. So the automation uses a personal access token stored as the repo secret **`MIRROR_PAT`**:

- Classic PAT with scopes `repo` + `workflow`, **or**
- Fine-grained PAT scoped to this repo with **Contents: write**, **Workflows: write**, **Actions: write** (fine-grained tokens on an org must be **approved by an org owner**).

## Manual runs

- **Sync now:** Actions → **mirror-sync** → *Run workflow*.
- **Build a specific version:** Actions → **build-release** → *Run workflow* → set `tag` (e.g. `v0.73.0`).
- **Scan a version without building:** Actions → **build-release** → *Run workflow* → set `tag` and enable **`scan_only`** — runs the five security engines against that tag and stops (no build, no release). Useful to check whether a version passes the gates.

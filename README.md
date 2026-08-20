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
- **Triggers:** automatically via `workflow_run` after every successful `mirror-sync` (so a freshly synced tag is picked up right away) + manual (`workflow_dispatch` with an optional `tag` input).
- **Tag selection:** builds the **newest `vX.Y.Z` tag that does not yet have a Release**. If the latest tag already has a Release, it does nothing. Manual runs can target any tag via the `tag` input.
- **Security scan (before building):** the source is scanned by **three independent engines**, each preceded by a big banner (tool / version / GitHub). Two are blocking **gates**, one is informational:
  - **`govulncheck`** (official Go `golang.org/x/vuln`) — **GATE**. Reachability-based: reports only vulnerabilities actually reachable in the code, at any severity.
  - **`Grype`** (Anchore) — **GATE**. Independent SCA with its own DB. Trivy's own `testdata/`/`integration/` fixtures (intentionally-vulnerable samples) are excluded; not reachability-aware, so kept at `--only-fixed --fail-on critical` (govulncheck already covers reachable issues).
  - **`osv-scanner`** (Google OSV) — **INFO**. Scans only the root `go.mod` (real dependencies; `go.sum` is only checksums and isn't a recognised lockfile). It has no reachability and no severity threshold, so it also lists vulns that aren't reachable in the binary — kept **informational** (reported, but does not block); gives a third, independent database (OSV) for visibility.
  - The build proceeds only if both **gates** pass; the informational scanner never blocks.
- **Build:** `go build ./cmd/trivy/` with `GOEXPERIMENT=jsonv2` (Trivy uses the experimental `encoding/json/v2`), version injected via `-X …/pkg/version/app.ver=<ver>`, for **linux/amd64** and **linux/arm64**.
- **Publish:** creates a GitHub **Release** on the tag with assets named exactly like upstream — `trivy_<ver>_Linux-64bit.tar.gz`, `trivy_<ver>_Linux-ARM64.tar.gz`, `trivy_<ver>_checksums.txt`. Creation is idempotent: if the Release already exists it is skipped without error.

## Using it in CI/CD

Asset names match upstream, so the standard download flow works — just point at this repo:

```bash
VER=0.74.0
curl -sfL https://github.com/TrendSpider/trivy/releases/download/v${VER}/trivy_${VER}_Linux-64bit.tar.gz | tar xz
./trivy --version
```

The official `install.sh -r TrendSpider/trivy vX.Y.Z` also works.

## Requirements / setup

Both workflows push code that includes `.github/workflows/*`, which the default `GITHUB_TOKEN` is not allowed to push. So the automation uses a personal access token stored as the repo secret **`MIRROR_PAT`**:

- Classic PAT with scopes `repo` + `workflow`, **or**
- Fine-grained PAT scoped to this repo with **Contents: write**, **Workflows: write**, **Actions: write** (fine-grained tokens on an org must be **approved by an org owner**).

## Manual runs

- **Sync now:** Actions → **mirror-sync** → *Run workflow*.
- **Build a specific version:** Actions → **build-release** → *Run workflow* → set `tag` (e.g. `v0.73.0`).

# trivy (mirror)

Public mirror of **[aquasecurity/trivy](https://github.com/aquasecurity/trivy)**, maintained by TrendSpider.

## How it works

- The upstream source branches (`main`, `release/*`, …) and **all tags** are mirrored here as-is.
- Syncing is done by the [`mirror-sync`](.github/workflows/mirror-sync.yml) GitHub Actions workflow, which runs **daily (03:00 UTC)** and can also be triggered manually (`workflow_dispatch`).
- This **`mirror` branch is the repository's default branch** and holds only the sync workflow + this README. The sync never overwrites it, so the workflow keeps running.

## Setup / requirements

The sync pushes upstream code that includes `.github/workflows/*`. The default `GITHUB_TOKEN` is **not** allowed to push workflow files, so the workflow uses a personal access token stored as the repo secret **`MIRROR_PAT`**:

- Classic PAT with scopes `repo` + `workflow`, **or**
- Fine-grained PAT scoped to this repo with `Contents: write` + `Workflows: write`.

## Manual sync

Actions → **mirror-sync** → *Run workflow*.

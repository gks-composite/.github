# Docker Build and Push (GHCR)

Uniform container pipeline for the GateKeeper enterprise. One reusable workflow gives every
repository the same behaviour:

1. **Version** — computed by [`gks-composite/global-version`](https://github.com/gks-composite/global-version) from `v*` git tags (falls back to `0.1.<run_number>` for un-tagged repos).
2. **Secrets scan** — gitleaks scan of the checkout (diff-scoped on PRs, tree scan on pushes); fails on findings at every tier.
3. **Build** — a single-arch image is built and loaded for scanning.
4. **Image scan** — grype scan of the built image, release tier only (default branch / release/* / tags), fail on HIGH+CRITICAL.
5. **Push** — multi-tag push to GHCR. Pull requests also push with identifiable `pr-<n>` / `sha-<sha>` tags.
6. **Harden** — SBOM + provenance attestations (and optional cosign signing) on default-branch / release-tag builds.

## Authentication (no PATs)

The pipeline uses the built-in **`GITHUB_TOKEN`** only — for both the GHCR push and the in-Docker
dependency restore. There is no PAT and no fallback, so misconfigured access **fails the build**
loudly instead of silently. Two requirements:

- The calling job grants **`permissions: packages: write`** (shown below).
- The repo is **granted access to the packages** it uses — the image package it pushes, and any
  internal dependency packages it restores (Org → Packages → *package* → *Manage access* → add the repo).

## Adding it to a repository

1. Repo → **Actions** → **New workflow** → choose **Docker Build and Push (GHCR)**.
2. Set `image-name` to your image (published to `ghcr.io/<owner>/<image-name>`).
3. Point `dockerfile` / `context` / `target` at your build if they are not the defaults.

## Example — service at the repo root

```yaml
jobs:
  docker:
    uses: gks-composite/.github/.github/workflows/Docker-Build-Push.yml@main
    permissions:
      contents: read
      packages: write
      id-token: write
    with:
      image-name: fcm-proxy
      context: .
      dockerfile: Dockerfile
```

## Example — one Dockerfile, multiple images (build targets)

```yaml
jobs:
  server:
    uses: gks-composite/.github/.github/workflows/Docker-Build-Push.yml@main
    permissions:
      contents: read
      packages: write
      id-token: write
    with:
      image-name: licenseserver
      dockerfile: Dockerfile
      target: runtime-server
  admin:
    uses: gks-composite/.github/.github/workflows/Docker-Build-Push.yml@main
    permissions:
      contents: read
      packages: write
      id-token: write
    with:
      image-name: adminapi
      dockerfile: Dockerfile
      target: runtime-admin
```

## Example — monorepo of images (matrix)

```yaml
jobs:
  images:
    strategy:
      matrix:
        image: [mediamtx-ffmpeg, postgres-timeshift, azure-blob-services]
    uses: gks-composite/.github/.github/workflows/Docker-Build-Push.yml@main
    permissions:
      contents: read
      packages: write
      id-token: write
    with:
      image-name: ${{ matrix.image }}
      context: ${{ matrix.image }}
      dockerfile: Dockerfile
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `image-name` | *(required)* | Image name under `ghcr.io/<owner>/`. |
| `context` | `.` | Build context. |
| `dockerfile` | `Dockerfile` | Dockerfile path relative to the context. |
| `target` | `` | Build stage target. |
| `platforms` | `linux/amd64` | Platforms for the pushed image. |
| `version` | `` | Explicit version; empty computes via `global-version`. |
| `build-args` | `` | Newline-separated `KEY=value` build args. |
| `registry` | `ghcr.io` | Registry host. |
| `image-namespace` | *(owner)* | Registry namespace (lowercased). |
| `push` | `true` | Push (set false for build + scan only). |
| `fs-scan` / `image-scan` | `true` | Toggle the gitleaks secrets scan / release-tier grype image scan. |
| `scan-ignore-unfixed` | `false` | Ignore vulns with no fix. |
| `sbom` | `true` | SBOM + provenance on release-tier pushes. |
| `sign` | `false` | Cosign keyless sign on release-tier pushes. |
| `environment-tag` | `` | Extra raw tag for promotion. |
| `default-branch` | `main` | Trunk branch (`dev` for the docker monorepo). |
| `runs-on` | `ubuntu-latest` | Runner. |
| `free-disk-space` | `false` | Free runner disk for builds that run out of it (see below). GitHub-hosted Linux runners only. |
| `fetch-depth` | `0` | Checkout depth (0 for versioning). |

There are **no secrets** to pass — the workflow uses `GITHUB_TOKEN` from the calling job.

## Running out of disk (`free-disk-space`)

A release-tier build (default branch, `release/*`, tags) builds the image twice on one runner disk:
1. the single-arch image, loaded into Docker for the grype scan;
2. the push build for every platform in `platforms`.

A large multi-arch image can fill the disk. The job then dies with `No space left on device`, and sometimes
leaves no log at all: the run page shows `BlobNotFound`.

Set `free-disk-space: true` (App-Pipeline passes its own `free-disk-space` input through). It does two things:
- **Before anything else,** it deletes preinstalled toolchains this job never uses: Android SDK, .NET, Haskell,
  CodeQL, Swift, Boost and cached Docker images. PowerShell stays, because global-version runs on it. The job
  summary reports how much it freed.
- **After the release-tier scan,** it deletes the loaded scan image before the push build.

On a self-hosted or non-Linux runner it only logs a warning: deleting a persistent machine's toolchains would
outlive the job.

## Tag scheme

| Trigger | Tags produced |
|---|---|
| Pull request | `pr-<n>`, `sha-<sha>` |
| Branch push | `<branch>`, `sha-<sha>`, `<version>`, `<major>.<minor>`, `<major>` |
| Default branch | above **+** `latest` |
| Release tag `vX.Y.Z` | `<version>`, `<major>.<minor>`, `<major>`, `sha-<sha>` |

## Scan gating (tiered)

| Trigger | Severity gate |
|---|---|
| Pull request | fail on **CRITICAL** |
| Default branch / release tag | fail on **HIGH + CRITICAL** |
| Other branches | report only |

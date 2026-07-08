# Docker Build and Push (GHCR)

Uniform container pipeline for the GateKeeper enterprise. One reusable workflow gives every
repository the same behaviour:

1. **Version** — computed by [`gks-composite/global-version`](https://github.com/gks-composite/global-version) from `v*` git tags (falls back to `0.1.<run_number>` for un-tagged repos).
2. **Source scan** — Trivy filesystem scan of the build context (vuln + secret + misconfig).
3. **Build** — a single-arch image is built and loaded for scanning.
4. **Image scan** — Trivy scan of the built image.
5. **Push** — multi-tag push to GHCR. Pull requests also push with identifiable `pr-<n>` / `sha-<sha>` tags.
6. **Harden** — SBOM + provenance attestations (and optional cosign signing) on default-branch / release-tag builds.

## Adding it to a repository

1. Repo → **Actions** → **New workflow** → choose **Docker Build and Push (GHCR)**.
2. Set `image-name` to your image (published to `ghcr.io/<owner>/<image-name>`).
3. Point `dockerfile` / `context` / `target` at your build if they are not the defaults.

## Example — service at the repo root

```yaml
jobs:
  docker:
    uses: gks-composite/.github/.github/workflows/Docker-Build-Push.yml@main
    with:
      image-name: fcm-proxy
      context: .
      dockerfile: Dockerfile
    secrets:
      build-token: ${{ secrets.GH_PACKAGETOKEN }}   # private NuGet restore
```

## Example — one Dockerfile, multiple images (build targets)

```yaml
jobs:
  server:
    uses: gks-composite/.github/.github/workflows/Docker-Build-Push.yml@main
    with:
      image-name: licenseserver
      dockerfile: Dockerfile
      target: runtime-server
    secrets:
      build-token: ${{ secrets.GH_PACKAGETOKEN }}
  admin:
    uses: gks-composite/.github/.github/workflows/Docker-Build-Push.yml@main
    with:
      image-name: adminapi
      dockerfile: Dockerfile
      target: runtime-admin
    secrets:
      build-token: ${{ secrets.GH_PACKAGETOKEN }}
```

## Example — monorepo of images (matrix)

```yaml
jobs:
  images:
    strategy:
      matrix:
        image: [mediamtx-ffmpeg, postgres-timeshift, azure-blob-services]
    uses: gks-composite/.github/.github/workflows/Docker-Build-Push.yml@main
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
| `fs-scan` / `image-scan` | `true` | Toggle each Trivy scan. |
| `scan-ignore-unfixed` | `false` | Ignore vulns with no fix. |
| `sbom` | `true` | SBOM + provenance on release-tier pushes. |
| `sign` | `false` | Cosign keyless sign on release-tier pushes. |
| `environment-tag` | `` | Extra raw tag for promotion. |
| `default-branch` | `main` | Trunk branch (`dev` for the docker monorepo). |
| `runs-on` | `ubuntu-latest` | Runner. |
| `fetch-depth` | `0` | Checkout depth (0 for versioning). |

`secrets.build-token` (optional) is used for registry login and as the `github_token` BuildKit
secret; it falls back to `GITHUB_TOKEN`.

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

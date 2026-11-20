# Linux container package

The one way a FaceFirst component repository builds a Linux package. One invocation builds one
container image from the repository's `packaging/linux/Dockerfile` and packs it, for each entry in
`packages`, as

```text
FaceFirst-<Component>-linux-<arch>-v<version>.7z
```

holding `package.manifest`, the image as `image.oci.tar`, the licence texts under `LICENSES/` and
any declared configuration files. The archive is uploaded as a workflow artifact for the caller's
release job to attach beside the Windows packages. The package contract (manifest fields, naming,
what a package may carry) is ADR 0034 in the FaceFirst deployment repository; the header comment
of [`Linux-Package.yml`](../.github/workflows/Linux-Package.yml) documents every input and every
field of a `packages` entry.

## How it differs from Docker Build and Push

[`Docker-Build-Push.yml`](Docker-Build-Push.README.md) publishes an image to GHCR for something that
pulls it from a registry. This workflow produces a **file** an on-prem deployment client installs
with no registry in reach: the image travels inside the `.7z`, and the manifest pins it by digest.
The push to `ghcr.io/<owner>/<image_name>:<version>` on a releasable run is a by-product, and the
pushed digest is asserted equal to the one in the manifest.

## Rules a caller follows

- **Everything is built in a container.** The Dockerfile takes `VERSION` as a build argument and
  needs nothing installed on the runner; a package that is plain files (a shared runtime such as
  CudaRuntime) is still staged by a container build and copied out with `"image": "files"`.
- The image carries the third-party licence texts its contents require under `/licenses`.
- A pull request run builds, tests and uploads the archives and publishes nothing; only a run with
  `releasable: true` pushes the image and writes the layer cache.
- Pass `release_tag` when the caller releases under a tag, so a version that is already released is
  refused before anything is built.

## Authentication

The calling job grants `permissions: contents: read, packages: write`. The job logs in to
`ghcr.io` with the caller's `GH_PACKAGETOKEN` when it is set, else the built-in `GITHUB_TOKEN`.
`GH_READTOKEN` is optional, for a build that restores from a private feed.

Both tokens reach a `prepare_command` as the environment variables `GH_PACKAGETOKEN` and
`GH_READTOKEN`, and a Dockerfile only as build secrets, `id=github_token` and `id=read_token`,
mounted for the `RUN` that names them:

```dockerfile
RUN --mount=type=secret,id=read_token  TOKEN=$(cat /run/secrets/read_token) ./restore.sh
```

`secrets: inherit` passes them only when the caller is in the same organization or enterprise as
this repository. A caller outside it names them:

```yaml
    secrets:
      GH_PACKAGETOKEN: ${{ secrets.GH_PACKAGETOKEN }}
      GH_READTOKEN: ${{ secrets.GH_READTOKEN }}
```

## Signing

Linux packages carry nothing Authenticode can sign. The packageserver Sign Sweep is where they
are handled: it is being taught to recognise the `-linux-<arch>-v` asset name and store the
archive unchanged (FaceFirst-Engineering/packageserver#59).

## Example

```yaml
jobs:
  linux:
    needs: [versioner, check-branch]
    uses: gks-composite/.github/.github/workflows/Linux-Package.yml@main
    secrets: inherit
    permissions:
      contents: read
      packages: write
    with:
      version: ${{ needs.versioner.outputs.version }}
      releasable: ${{ needs.check-branch.outputs.is-releasable == 'true' }}
      release_tag: v${{ needs.versioner.outputs.version }}
      image_name: facefirst-edgex
      packages: >-
        [{"component": "FaceEncode", "serviceName": "ff-faceencode",
          "dependencies": ["MessageBroker"], "optionalDependencies": ["CudaRuntime"]}]
```

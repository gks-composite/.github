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

The built-in `GITHUB_TOKEN` with `packages: write`, or the caller's `GH_PACKAGETOKEN` when it is
set. `GH_READTOKEN` is optional and reaches a Dockerfile only as a build secret.

## Signing

Linux packages carry no Authenticode-signed files. The packageserver Sign Sweep recognises the
`-linux-<arch>-v` asset name and stores the archive unchanged.

## Example

```yaml
jobs:
  linux:
    needs: [versioner, check-branch]
    uses: gks-composite/.github/.github/workflows/Linux-Package.yml@main
    secrets: inherit
    with:
      version: ${{ needs.versioner.outputs.version }}
      releasable: ${{ needs.check-branch.outputs.is-releasable == 'true' }}
      image_name: facefirst-edgex
      packages: >-
        [{"component": "FaceEncode", "serviceName": "ff-faceencode",
          "dependencies": ["MessageBroker"], "optionalDependencies": ["CudaRuntime"]}]
```

# Version Manager (with override)

## Setup

Create a [new repo](https://github.com/new) and then clone it locally, then run the following:

```powershell
$tagPrefix = "v"
$firstVersion = "1.0.0.0"

$fulltag = "$($tagPrefix)$($firstVersion)"
git tag -a $fulltag
git push origin $fulltag
```

## Adding the action

Go to the repo you created and select "Actions" from the header bars. If there are no pipelines yet, scroll down until you see the FaceFirst-Engineering actions. Otherwise select `New Workflow` to get to the same menu. Alternatively, you can use the following URL with your repo subbed in:

```text
https://github.com/{OWNER}/{REPO}/actions/new
```

## Usage

This workflow supports both manual and automatic versioning. You can optionally provide a version number when running the workflow manually:

- **Manual version override:**
  - When triggering the workflow via "Run workflow", you can set the `version` input (e.g., `8.0.0.10`).
  - If left blank, the workflow will auto-generate a version.

- **Token override:**
  - If you set a repository secret named `OVERRIDE_TOKEN`, it will be used for repository checkout. Otherwise, the default `GITHUB_TOKEN` is used.
  - The workflow step name will indicate which token is being used.

Example step from the workflow:

```yaml
- name: ${{ secrets.OVERRIDE_TOKEN != '' && 'Checkout repo (override)' || 'Checkout repo (default)' }}
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
    token: ${{ secrets.OVERRIDE_TOKEN || secrets.GITHUB_TOKEN }}
    persist-credentials: false
```


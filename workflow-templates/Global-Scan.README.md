# Generic Global Scanner

## Setup

This workflow provides a generic, reusable security scanner for all repositories. To use it, add the workflow template to your repository's workflows directory or reference it from a central location.

## Adding the action

1. Go to your repository on GitHub.
2. Click on the "Actions" tab.
3. If there are no workflows yet, scroll down to find the available workflow templates, or click "New Workflow" to access the template menu.
4. Alternatively, you can use the following URL, replacing `{OWNER}` and `{REPO}` with your repository details:

   ```text
   https://github.com/{OWNER}/{REPO}/actions/new
   ```

5. Select the **Generic Global Scanner** workflow template to add it to your repository.

## Usage

This workflow is designed to be called as a reusable workflow using the `uses` keyword. It scans your repository for vulnerabilities using Trivy.

### Example workflow call

```yaml
name: Global Scan

on:
  workflow_call:
    inputs:
      working_dir:
        description: 'directory should be scanned'
        required: false
        type: string
        default: ""

jobs:
  scan:
    uses: gks-composite/.github/.github/workflows/global-scan.yml@main
    secrets: inherit
    with:
      working_dir: ${{ inputs.working_dir || '' }}
```

## Customization

- You can override the `working_dir` input to specify a subdirectory to scan. By default, the entire repository is scanned.

## Notes

- Make sure the referenced workflow file (`gks-composite/.github/.github/workflows/global-scan.yml@main`) is accessible and up to date.
- This workflow uses [Trivy](https://github.com/aquasecurity/trivy) for scanning and is intended to be generic for most repositories.
- The workflow requires appropriate permissions to read packages and update statuses.

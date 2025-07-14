# Global Lint (on pull request)

## Setup

This workflow is designed to provide a generic, reusable linter for all repositories. To use it, add the workflow template to your repository's workflows directory or reference it from a central location.

## Adding the action

1. Go to your repository on GitHub.
2. Click on the "Actions" tab.
3. If there are no workflows yet, scroll down to find the available workflow templates, or click "New Workflow" to access the template menu.
4. Alternatively, you can use the following URL, replacing `{OWNER}` and `{REPO}` with your repository details:

   ```text
   https://github.com/{OWNER}/{REPO}/actions/new
   ```

5. Select the **Global Lint (on pull request)** workflow template to add it to your repository.

## Usage

This workflow will automatically run on every pull request to any branch. It is designed to:

- Enforce linting standards across your codebase.
- Run as a reusable workflow using the `uses` keyword.
- Cancel in-progress runs for the same pull request branch to save resources.

### Example workflow call

```yaml
name: Global Lint (on pull request)

on:
  pull_request:
    branches:
      - '**'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    if: ${{ github.event_name == 'pull_request' }}
    uses: gks-composite/.github/.github/workflows/Global-Lint.yml@main
    secrets: inherit
    with:
      branch: ${{ github.event.pull_request.base.ref || github.base_ref || github.ref_name || 'main' }}
```

## Customization

- You can override the `branch` input if you want to lint against a specific base branch.
- The workflow is designed to inherit secrets from the calling repository.

## Notes

- Make sure the referenced workflow file (`gks-composite/.github/.github/workflows/Global-Lint.yml@main`) is accessible and up to date.
- This workflow is intended to be generic and should work with most repositories out of the box.

# Backport Workflow

Automatically creates cherry-pick PRs when labeled PRs are merged.

## Usage

```yaml
name: Backport

on: # zizmor: ignore[dangerous-triggers]
  pull_request_target:
    types: [closed]

# Permissions required by the reusable workflow
permissions:
  contents: write
  pull-requests: write

jobs:
  backport:
    uses: thpham/actions/.github/workflows/backport.yml@main
    # GITHUB_TOKEN is automatically available to reusable workflows
```

## Permissions

| Permission      | Purpose                          |
| --------------- | -------------------------------- |
| `contents`      | Write - Push cherry-pick commits |
| `pull-requests` | Write - Create backport PRs      |

## Inputs

| Input                    | Type    | Required | Default                | Description                 |
| ------------------------ | ------- | -------- | ---------------------- | --------------------------- |
| `label-pattern`          | string  | No       | `'^backport ([^ ]+)$'` | Regex for backport labels   |
| `check-snapshot-version` | boolean | No       | `true`                 | Warn if target not SNAPSHOT |
| `runner`                 | string  | No       | `'ubuntu-latest'`      | Runner to use               |

## How It Works

1. When a PR with `backport <branch>` labels is merged
2. The workflow creates cherry-pick PRs to each labeled branch
3. If `check-snapshot-version` is enabled, warns if target branch has non-SNAPSHOT version

## Label Format

Labels must match the pattern `backport <branch-name>`:

- `backport release/1.2`
- `backport main`

## SNAPSHOT Version Check

When enabled, the workflow checks if the target branch has a `-SNAPSHOT` version. If not, it adds a warning comment to the backport PR advising to wait for the SNAPSHOT bump.

## Security Note

This workflow uses `pull_request_target` for write permissions. Risk is mitigated by:

1. Only running on merged PRs from the same repository
2. The backport action doesn't execute untrusted PR code

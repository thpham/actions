# Suggest Backports Workflow

Automatically suggests backport labels for PRs containing fix or security commits.

## Usage

```yaml
name: Suggest Backports

on:
  pull_request_target:
    types: [opened, ready_for_review]

jobs:
  suggest:
    uses: thpham/actions/.github/workflows/suggest-backports.yml@v1
    secrets: inherit
```

## Inputs

| Input                  | Type   | Required | Default                        | Description                    |
| ---------------------- | ------ | -------- | ------------------------------ | ------------------------------ |
| `backportable-types`   | string | No       | `'^(fix\|security)(\(.+\))?:'` | Regex for backportable commits |
| `max-release-branches` | number | No       | `2`                            | Max branches to suggest        |
| `label-color`          | string | No       | `'5319e7'`                     | Label color (hex)              |
| `runner`               | string | No       | `'ubuntu-latest'`              | Runner to use                  |

## How It Works

1. When a PR is opened or marked ready for review
2. Analyzes commit messages for `fix:` or `security:` prefixes
3. Finds the latest release branches
4. Creates backport labels if they don't exist
5. Adds labels to the PR
6. Comments explaining the suggested backports

## Example Comment

```markdown
### Backport Labels Suggested

This PR contains `fix:` or `security:` commits that may need backporting:

- `fix(api): resolve null pointer exception`

**Suggested backport targets:**

- `backport release/1.2`
- `backport release/1.1`

> **To skip a backport**, remove the label before merging.
```

## Skipped Cases

The workflow skips:

- Draft PRs
- PRs from forks
- Backport PRs (branches starting with `backport-`)

## Security Note

Uses `pull_request_target` with mitigations:

1. Only runs on PRs from the same repository
2. Uses `persist-credentials: false` on checkout
3. Only uses GitHub API (doesn't execute PR code)

# Commitlint Workflow

Enforces Conventional Commits specification on PR commits.

## Usage

```yaml
name: Commitlint

on:
  pull_request:
    branches:
      - main
      - master
      - "release/**"

# Permissions required by the reusable workflow
permissions:
  contents: read
  pull-requests: read

jobs:
  commitlint:
    uses: thpham/actions/.github/workflows/commitlint.yml@main
    # No secrets needed for commitlint
```

## Permissions

| Permission      | Purpose                      |
| --------------- | ---------------------------- |
| `contents`      | Read - Access commit history |
| `pull-requests` | Read - Access PR commits     |

## Inputs

| Input             | Type    | Required | Default                   | Description            |
| ----------------- | ------- | -------- | ------------------------- | ---------------------- |
| `config-file`     | string  | No       | `'commitlint.config.mjs'` | Commitlint config file |
| `skip-dependabot` | boolean | No       | `true`                    | Skip Dependabot PRs    |
| `runner`          | string  | No       | `'ubuntu-latest'`         | Runner to use          |

## Configuration

Create `commitlint.config.mjs` in your repository:

```javascript
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "type-enum": [
      2,
      "always",
      [
        "feat", // New feature
        "fix", // Bug fix
        "security", // Security fix
        "docs", // Documentation
        "style", // Formatting
        "refactor", // Code refactoring
        "perf", // Performance
        "test", // Tests
        "build", // Build system
        "ci", // CI/CD
        "chore", // Maintenance
        "revert", // Revert
      ],
    ],
    "subject-max-length": [2, "always", 72],
    "body-max-line-length": [2, "always", 100],
    "header-max-length": [2, "always", 100],
  },
};
```

## Commit Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

Examples:

- `feat(api): add user authentication endpoint`
- `fix: resolve null pointer in data processor`
- `security(auth): patch JWT validation vulnerability`

## Dependabot Skip

By default, Dependabot PRs are skipped because their automated commits don't follow Conventional Commits format.

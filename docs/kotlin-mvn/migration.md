# Migration Guide

How to migrate from standalone workflows to reusable workflows with security best practices.

## Prerequisites

1. Access to `thpham/actions` repository
2. Configuration files in your repository (see below)

## Required Configuration Files

### Release Please

**`release-please-config.json`** (copy from `configs/kotlin-mvn/` and customize):

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "maven",
  "versioning": "always-bump-minor",
  "packages": {
    ".": {
      "package-name": "your-project",
      "changelog-path": "CHANGELOG.md"
    }
  },
  "plugins": [{ "type": "maven-workspace", "updateAllPackages": true }],
  "extra-files": ["modules/core/pom.xml", "modules/api/pom.xml"]
}
```

**`release-please-config-patch.json`** (same as above with `"versioning": "always-bump-patch"`)

**`.release-please-manifest.json`**:

```json
{
  ".": "1.0.0"
}
```

### Commitlint

**`commitlint.config.mjs`**:

```javascript
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "security",
        "docs",
        "style",
        "refactor",
        "perf",
        "test",
        "build",
        "ci",
        "chore",
        "revert",
      ],
    ],
    "subject-max-length": [2, "always", 72],
  },
};
```

### JReleaser (if using)

Copy and customize `jreleaser.yml` from your existing setup.

## Security Best Practices

Before migrating, understand these security principles:

### 1. Use `permissions: {}` at Workflow Level

Deny all permissions by default. The reusable workflow defines what it needs at the job level.

### 2. Avoid `secrets: inherit`

Never use `secrets: inherit` as it passes ALL repository secrets to the reusable workflow. Instead:

- **Most workflows need no secrets block** - `GITHUB_TOKEN` is automatically available
- **Only pass explicit secrets** when the workflow requires them (e.g., `SONAR_TOKEN`)

### 3. Reference Table

| Workflow                 | Secrets Needed                     |
| ------------------------ | ---------------------------------- |
| `kotlin-mvn-ci.yml`      | None (`GITHUB_TOKEN` is automatic) |
| `kotlin-mvn-release.yml` | None (optional: GPG, webhooks)     |
| `sonar.yml`              | `SONAR_TOKEN` (explicit)           |
| `backport.yml`           | None (`GITHUB_TOKEN` is automatic) |
| `suggest-backports.yml`  | None (`GITHUB_TOKEN` is automatic) |
| `commitlint.yml`         | None                               |
| `cleanup-registry.yml`   | None (`GITHUB_TOKEN` is automatic) |
| `lint-workflows.yml`     | None                               |

## Migration Steps

### Step 1: Create Caller Workflows

Replace your existing workflows with minimal, secure callers.

**`.github/workflows/ci.yml`**:

```yaml
name: CI

on:
  push:
    branches: [main, "release/**"]
    paths: [pom.xml, modules/**]
  pull_request:
    branches: [main, "release/**"]
    types: [opened, synchronize, reopened, labeled]
    paths: [pom.xml, modules/**]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Deny all permissions - reusable workflow defines what it needs
permissions: {}

jobs:
  ci:
    uses: thpham/actions/.github/workflows/kotlin-mvn-ci.yml@main
    with:
      docker-image-name: ${{ github.repository }}/your-api
      docker-context: modules/api
    # GITHUB_TOKEN is automatically available - no secrets block needed
```

**`.github/workflows/release.yml`**:

```yaml
name: Release

on:
  push:
    branches: [main, "release/**"]
  workflow_dispatch:

permissions: {}

jobs:
  release:
    uses: thpham/actions/.github/workflows/kotlin-mvn-release.yml@main
    with:
      docker-image-name: ${{ github.repository }}/your-api
      docker-context: modules/api
    # Optional secrets (uncomment if needed):
    # secrets:
    #   GPG_SECRET_KEY: ${{ secrets.GPG_SECRET_KEY }}
    #   GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
    #   TEAMS_WEBHOOK: ${{ secrets.TEAMS_WEBHOOK }}
```

**`.github/workflows/sonar.yml`**:

```yaml
name: SonarQube Analysis

on:
  push:
    branches: [main, "release/**"]
    paths: [pom.xml, modules/**]
  pull_request:
    branches: [main, "release/**"]
    paths: [pom.xml, modules/**]

concurrency:
  group: sonar-${{ github.ref }}
  cancel-in-progress: true

permissions: {}

jobs:
  sonar:
    uses: thpham/actions/.github/workflows/sonar.yml@main
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      # For self-hosted SonarQube:
      # SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

**`.github/workflows/backport.yml`**:

```yaml
name: Backport

on: # zizmor: ignore[dangerous-triggers]
  pull_request_target:
    types: [closed]

permissions: {}

jobs:
  backport:
    uses: thpham/actions/.github/workflows/backport.yml@main
```

**`.github/workflows/suggest-backports.yml`**:

```yaml
name: Suggest Backport Labels

on: # zizmor: ignore[dangerous-triggers]
  pull_request_target:
    types: [opened, ready_for_review]

permissions: {}

jobs:
  suggest:
    uses: thpham/actions/.github/workflows/suggest-backports.yml@main
```

**`.github/workflows/commitlint.yml`**:

```yaml
name: Commitlint

on:
  pull_request:
    branches: [main, "release/**"]

permissions: {}

jobs:
  commitlint:
    uses: thpham/actions/.github/workflows/commitlint.yml@main
```

**`.github/workflows/cleanup-registry.yml`**:

```yaml
name: Cleanup Container Registry

on:
  pull_request:
    types: [closed]
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch:

permissions: {}

jobs:
  cleanup:
    uses: thpham/actions/.github/workflows/cleanup-registry.yml@main
    with:
      image-name: ${{ github.repository }}/your-api
```

**`.github/workflows/lint-workflows.yml`**:

```yaml
name: Lint Workflows

on:
  push:
    branches: [main, "release/**"]
    paths: [".github/workflows/**", ".github/actionlint.yaml"]
  pull_request:
    branches: [main, "release/**"]
    paths: [".github/workflows/**", ".github/actionlint.yaml"]

concurrency:
  group: lint-workflows-${{ github.ref }}
  cancel-in-progress: true

permissions: {}

jobs:
  lint:
    uses: thpham/actions/.github/workflows/lint-workflows.yml@main
```

### Step 2: Create All Workflow Files

Create these files in `.github/workflows/`:

| File                    | Trigger                                            |
| ----------------------- | -------------------------------------------------- |
| `ci.yml`                | push, pull_request                                 |
| `release.yml`           | push to main/release/\*\*, workflow_dispatch       |
| `backport.yml`          | pull_request_target [closed]                       |
| `suggest-backports.yml` | pull_request_target [opened, ready_for_review]     |
| `commitlint.yml`        | pull_request                                       |
| `sonar.yml`             | push, pull_request                                 |
| `cleanup-registry.yml`  | pull_request [closed], schedule, workflow_dispatch |
| `lint-workflows.yml`    | push/pull_request to .github/workflows/\*\*        |

### Step 3: Configure Secrets

Only configure secrets that are actually needed:

- `SONAR_TOKEN` - Required for SonarQube analysis
- `SONAR_HOST_URL` - Only if using self-hosted SonarQube
- `GPG_*` secrets - Only if signing artifacts
- `TEAMS_WEBHOOK` / `SLACK_WEBHOOK` - Only if sending notifications

**Note:** `GITHUB_TOKEN` is automatically available to all workflows.

### Step 4: Test

1. Create a test PR to verify CI workflow
2. Check that commitlint validates commits
3. Verify Docker preview images (add `preview-image` label)
4. Test a release by merging to main

### Step 5: Clean Up

Remove old standalone workflow files (they're now replaced by callers).

## Troubleshooting

### Workflow Not Found

Ensure the actions repository is public or your repository has access.

### Permission Denied

Check that:

1. The reusable workflow has the required `permissions:` at job level
2. Required secrets are configured in your repository

### Docker Build Fails

Verify:

1. `docker-image-name` matches your package path
2. Dockerfile exists in `docker-context` directory
3. Build artifacts are in expected location

### Release Please Not Creating PR

Check:

1. Config files exist and are valid JSON
2. Conventional commits are being used
3. The reusable workflow has `contents: write` permission

### Zizmor Warnings

If you see `secrets-inherit` or `excessive-permissions` warnings, ensure you've applied the security best practices above (use `permissions: {}` and avoid `secrets: inherit`).

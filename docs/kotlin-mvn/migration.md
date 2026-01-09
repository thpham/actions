# Migration Guide

How to migrate from standalone workflows to platform-workflows.

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

## Migration Steps

### Step 1: Create Caller Workflows

Replace your existing workflows with minimal callers:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main, "release/**"]
  pull_request:
    branches: [main, "release/**"]

jobs:
  ci:
    uses: thpham/actions/.github/workflows/ci.yml@v1
    with:
      docker-image-name: ${{ github.repository }}/your-api
    secrets: inherit
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

Ensure these secrets are set in your repository:

- `SONAR_TOKEN` (if using SonarQube)
- `SONAR_HOST_URL` (if self-hosted)
- `GPG_*` secrets (if signing)
- `TEAMS_WEBHOOK` / `SLACK_WEBHOOK` (if notifying)

### Step 4: Test

1. Create a test PR to verify CI workflow
2. Check that commitlint validates commits
3. Verify Docker preview images (add `preview-image` label)
4. Test a release by merging to main

### Step 5: Clean Up

Remove old standalone workflow files (they're now replaced by callers).

## Example: Minimal Setup

For a simple project, you might only need:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: thpham/actions/.github/workflows/ci.yml@v1
    with:
      docker-image-name: ${{ github.repository }}/api
    secrets: inherit
```

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    uses: thpham/actions/.github/workflows/release.yml@v1
    with:
      docker-image-name: ${{ github.repository }}/api
    secrets: inherit
```

## Troubleshooting

### Workflow Not Found

Ensure the platform-workflows repository is public or your repository has access.

### Permission Denied

Check that `secrets: inherit` is used and required secrets are configured.

### Docker Build Fails

Verify:

1. `docker-image-name` matches your package path
2. Dockerfile exists in `docker-context` directory
3. Build artifacts are in expected location

### Release Please Not Creating PR

Check:

1. Config files exist and are valid JSON
2. Conventional commits are being used
3. `contents: write` permission is granted

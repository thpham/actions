# Kotlin/Maven Release Workflow

Automated releases with Release Please, multi-arch Docker builds, and JReleaser distribution for Kotlin/Maven/Spring Boot projects.

**Workflow file:** `kotlin-mvn-release.yml`

## Usage

```yaml
name: Release

on:
  push:
    branches: [main, master, "release/**"]
  workflow_dispatch:

# Permissions required by the reusable workflow
permissions:
  contents: write
  packages: write
  pull-requests: write
  actions: write

jobs:
  release:
    uses: thpham/actions/.github/workflows/kotlin-mvn-release.yml@main
    with:
      docker-image-name: ${{ github.repository }}/myservice-api
    # GITHUB_TOKEN is automatically available - no secrets block needed
    # Optional secrets (uncomment if needed):
    # secrets:
    #   GPG_SECRET_KEY: ${{ secrets.GPG_SECRET_KEY }}
    #   GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
```

## Permissions

| Permission      | Purpose                                     |
| --------------- | ------------------------------------------- |
| `contents`      | Write - Create tags, branches, and releases |
| `packages`      | Write - Push Docker images to registry      |
| `pull-requests` | Write - Create/merge release PRs            |
| `actions`       | Write - Cancel duplicate workflows          |

## Inputs

| Input                         | Type    | Required | Default                              | Description                 |
| ----------------------------- | ------- | -------- | ------------------------------------ | --------------------------- |
| `java-version`                | string  | No       | `'17'`                               | Java version                |
| `java-distribution`           | string  | No       | `'temurin'`                          | Java distribution           |
| `docker-registry`             | string  | No       | `'ghcr.io'`                          | Container registry          |
| `docker-image-name`           | string  | **Yes**  | -                                    | Docker image name           |
| `docker-context`              | string  | No       | `'.'`                                | Docker build context        |
| `docker-file`                 | string  | No       | `'Dockerfile'`                       | Dockerfile path             |
| `release-please-config`       | string  | No       | `'release-please-config.json'`       | Config for main             |
| `release-please-config-patch` | string  | No       | `'release-please-config-patch.json'` | Config for release/\*       |
| `release-please-manifest`     | string  | No       | `'.release-please-manifest.json'`    | Manifest file               |
| `auto-merge-snapshot`         | boolean | No       | `true`                               | Auto-merge SNAPSHOT PRs     |
| `create-release-branch`       | boolean | No       | `true`                               | Create release/X.Y branches |
| `jreleaser-enabled`           | boolean | No       | `true`                               | Run JReleaser               |
| `jreleaser-config`            | string  | No       | `'jreleaser.yml'`                    | JReleaser config            |
| `staging-deploy-dir`          | string  | No       | `'staging-deploy'`                   | Maven staging dir           |
| `artifact-path`               | string  | No       | `'**/target/*.jar'`                  | Artifact path               |
| `enable-arm64`                | boolean | No       | `true`                               | Enable ARM64 builds         |
| `gpg-signing`                 | boolean | No       | `false`                              | Enable GPG signing          |
| `notify-teams`                | boolean | No       | `false`                              | Teams notification          |
| `notify-slack`                | boolean | No       | `false`                              | Slack notification          |

## Outputs

| Output            | Description                   |
| ----------------- | ----------------------------- |
| `release_created` | Whether a release was created |
| `version`         | Released version (X.Y.Z)      |
| `tag_name`        | Release tag name              |

## Secrets

| Secret           | Required | Description       |
| ---------------- | -------- | ----------------- |
| `GPG_SECRET_KEY` | No       | GPG private key   |
| `GPG_PASSPHRASE` | No       | GPG passphrase    |
| `GPG_PUBLIC_KEY` | No       | GPG public key    |
| `TEAMS_WEBHOOK`  | No       | Teams webhook URL |
| `SLACK_WEBHOOK`  | No       | Slack webhook URL |

## Jobs

1. **release-please** - Create/update release PR, auto-merge SNAPSHOT PRs
2. **create-release-branch** - Create `release/X.Y` branch (main only)
3. **build** - Build Maven artifacts, stage for JReleaser
4. **build-docker** - Multi-arch Docker builds
5. **merge-docker** - Merge manifests, apply version tags
6. **distribute** - JReleaser distribution

## Branch-Based Versioning

The workflow uses different configs based on branch:

- **main/master**: `release-please-config.json` (minor bumps)
- **release/\***: `release-please-config-patch.json` (patch bumps)

This prevents version collisions when backporting fixes.

## Docker Tags

| Branch      | Tags Created               |
| ----------- | -------------------------- |
| main/master | `X.Y.Z`, `X.Y`, `latest`   |
| release/\*  | `X.Y.Z`, `X.Y` (no latest) |

## SNAPSHOT Auto-Merge

When Release Please creates a SNAPSHOT bump PR (labeled `autorelease: snapshot`), it's automatically merged. This enables immediate development on the release branch.

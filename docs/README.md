# Workflows Documentation

This directory contains detailed documentation for each reusable workflow.

## Workflow Reference

### Stack-Specific Workflows

| Workflow                                       | Stack        | Description                                  | Documentation                |
| ---------------------------------------------- | ------------ | -------------------------------------------- | ---------------------------- |
| [kotlin-mvn-ci.yml](workflows/ci.md)           | Kotlin/Maven | Build, test, lint, and Docker preview images | [View](workflows/ci.md)      |
| [kotlin-mvn-release.yml](workflows/release.md) | Kotlin/Maven | Release Please, Docker builds, JReleaser     | [View](workflows/release.md) |

### Stack-Agnostic Workflows

| Workflow                                                | Description                   | Documentation                          |
| ------------------------------------------------------- | ----------------------------- | -------------------------------------- |
| [backport.yml](workflows/backport.md)                   | Automated cherry-pick PRs     | [View](workflows/backport.md)          |
| [suggest-backports.yml](workflows/suggest-backports.md) | Backport label suggestions    | [View](workflows/suggest-backports.md) |
| [commitlint.yml](workflows/commitlint.md)               | Conventional Commits linting  | [View](workflows/commitlint.md)        |
| [sonar.yml](workflows/sonar.md)                         | SonarQube/SonarCloud analysis | [View](workflows/sonar.md)             |
| [cleanup-registry.yml](workflows/cleanup-registry.md)   | Container image cleanup       | [View](workflows/cleanup-registry.md)  |
| [lint-workflows.yml](workflows/lint-workflows.md)       | Workflow validation           | [View](workflows/lint-workflows.md)    |

## Migration Guide

See the [Migration Guide](migration.md) for instructions on migrating from existing workflows.

## Architecture & Concepts

- [Workflow Architecture](workflow-architecture.md) - Flow diagrams, job dependencies, versioning strategy, and scenario guides
- [GHE Cloud Compatibility](ghe-cloud-compatibility.md) - Enterprise migration guide and action compatibility report

### Caller-Reusable Pattern

```
┌───────────────────────────────────────────────────────────────────────┐
│                        Caller Repository                              │
├───────────────────────────────────────────────────────────────────────┤
│  .github/workflows/                                                   │
│  ├── ci.yml        → calls kotlin-mvn-ci.yml (or other stack)         │
│  ├── release.yml   → calls kotlin-mvn-release.yml (or other stack)    │
│  ├── backport.yml  → calls backport.yml                               │
│  └── ...                                                              │
├───────────────────────────────────────────────────────────────────────┤
│  Configuration Files                                                  │
│  ├── release-please-config.json                                       │
│  ├── release-please-config-patch.json                                 │
│  ├── .release-please-manifest.json                                    │
│  ├── commitlint.config.mjs                                            │
│  ├── jreleaser.yml                                                    │
│  └── Dockerfile                                                       │
└───────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────────────┐
│                    actions Repository                                 │
├───────────────────────────────────────────────────────────────────────┤
│  .github/workflows/                                                   │
│  ├── kotlin-mvn-ci.yml      (reusable, Kotlin/Maven stack)            │
│  ├── kotlin-mvn-release.yml (reusable, Kotlin/Maven stack)            │
│  ├── backport.yml           (reusable, stack-agnostic)                │
│  ├── suggest-backports.yml  (reusable, stack-agnostic)                │
│  ├── commitlint.yml         (reusable, stack-agnostic)                │
│  ├── sonar.yml              (reusable, stack-agnostic)                │
│  ├── cleanup-registry.yml   (reusable, stack-agnostic)                │
│  └── lint-workflows.yml     (reusable, stack-agnostic)                │
├───────────────────────────────────────────────────────────────────────┤
│  configs/kotlin-mvn/                                                  │
│  ├── release-please-config.json (template)                            │
│  ├── release-please-config-patch.json (template)                      │
│  └── commitlint.config.mjs (template)                                 │
└───────────────────────────────────────────────────────────────────────┘
```

## Release Flow

The workflows implement Release Flow branching strategy:

```
main (development)
  │
  ├── feature/xyz → PR → main (squash merge)
  │                   │
  │                   └── Release Please creates Release PR
  │                       │
  │                       └── Merge Release PR
  │                           │
  │                           ├── Tag created (v1.2.0)
  │                           ├── Docker images built (1.2.0, 1.2, latest)
  │                           ├── JReleaser distributes artifacts
  │                           └── release/1.2 branch created
  │
  └── release/1.x (maintenance branches)
        │
        ├── Backport PRs auto-created (for fix:/security: commits)
        ├── SNAPSHOT bump PR auto-merged
        └── Patch releases (1.1.1, 1.1.2, ...)
            └── Docker images (1.1.1, 1.1) - no 'latest' tag
```

## Image Tagging Strategy

| Source               | Primary Tag         | Rolling Tag | Notes                |
| -------------------- | ------------------- | ----------- | -------------------- |
| Release (main)       | `1.2.3`, `1.2`      | `latest`    | Full semver + latest |
| Release (release/\*) | `1.1.1`, `1.1`      | -           | No latest tag        |
| Main push            | `main-{ts}-{sha}`   | `edge`      | Snapshot builds      |
| PR preview           | `pr-{N}-{ts}-{sha}` | `pr-{N}`    | Requires label       |

## Security Best Practices

When calling reusable workflows, follow these security principles:

### 1. Grant Only Required Permissions

Reusable workflows define permissions at the job level, but the **caller workflow must grant at least those permissions** at the workflow level. Use `permissions: {}` only for workflows that need no permissions.

```yaml
# Grant permissions the reusable workflow needs
permissions:
  contents: read
  packages: write
  pull-requests: write
  actions: write

jobs:
  ci:
    uses: thpham/actions/.github/workflows/kotlin-mvn-ci.yml@main
```

### Permissions by Workflow

| Workflow                 | Required Permissions                                                           |
| ------------------------ | ------------------------------------------------------------------------------ |
| `kotlin-mvn-ci.yml`      | `contents: read`, `packages: write`, `pull-requests: write`, `actions: write`  |
| `kotlin-mvn-release.yml` | `contents: write`, `packages: write`, `pull-requests: write`, `actions: write` |
| `sonar.yml`              | `contents: read`                                                               |
| `backport.yml`           | `contents: write`, `pull-requests: write`                                      |
| `suggest-backports.yml`  | `contents: read`, `pull-requests: write`                                       |
| `commitlint.yml`         | `contents: read`, `pull-requests: read`                                        |
| `cleanup-registry.yml`   | `packages: write`, `pull-requests: read`                                       |
| `lint-workflows.yml`     | `contents: read`, `actions: read`, `security-events: write`                    |

### 2. Avoid `secrets: inherit`

Instead of passing all secrets blindly with `secrets: inherit`, explicitly pass only the secrets each workflow needs:

```yaml
# BAD - passes all secrets
secrets: inherit

# GOOD - explicit secrets only
secrets:
  SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

Most workflows only need `GITHUB_TOKEN`, which is **automatically available** to reusable workflows - no need to pass it explicitly.

### 3. Secrets by Workflow

| Workflow                 | Required Secrets                   | Notes                           |
| ------------------------ | ---------------------------------- | ------------------------------- |
| `kotlin-mvn-ci.yml`      | None                               | `GITHUB_TOKEN` is automatic     |
| `kotlin-mvn-release.yml` | None (optional: GPG\_\*, webhooks) | `GITHUB_TOKEN` is automatic     |
| `sonar.yml`              | `SONAR_TOKEN`                      | Required for SonarQube analysis |
| `backport.yml`           | None                               | `GITHUB_TOKEN` is automatic     |
| `suggest-backports.yml`  | None                               | `GITHUB_TOKEN` is automatic     |
| `commitlint.yml`         | None                               | No secrets needed               |
| `cleanup-registry.yml`   | None                               | `GITHUB_TOKEN` is automatic     |
| `lint-workflows.yml`     | None                               | No secrets needed               |

### 4. Example: Secure Caller Workflow

```yaml
name: CI

on:
  push:
    branches: [main, "release/**"]
  pull_request:
    branches: [main, "release/**"]

# Grant permissions required by the reusable workflow
permissions:
  contents: read
  packages: write
  pull-requests: write
  actions: write

jobs:
  ci:
    uses: thpham/actions/.github/workflows/kotlin-mvn-ci.yml@main
    with:
      docker-image-name: ${{ github.repository }}/api
    # No secrets: block needed - GITHUB_TOKEN is automatic
```

## Secrets Reference

| Secret           | Used By                | Required | Description           |
| ---------------- | ---------------------- | -------- | --------------------- |
| `GITHUB_TOKEN`   | All                    | Auto     | GitHub API access     |
| `SONAR_TOKEN`    | sonar.yml              | Optional | SonarQube/SonarCloud  |
| `SONAR_HOST_URL` | sonar.yml              | Optional | Self-hosted SonarQube |
| `GPG_SECRET_KEY` | kotlin-mvn-release.yml | Optional | Artifact signing      |
| `GPG_PASSPHRASE` | kotlin-mvn-release.yml | Optional | GPG passphrase        |
| `GPG_PUBLIC_KEY` | kotlin-mvn-release.yml | Optional | GPG public key        |
| `TEAMS_WEBHOOK`  | kotlin-mvn-release.yml | Optional | Teams notifications   |
| `SLACK_WEBHOOK`  | kotlin-mvn-release.yml | Optional | Slack notifications   |

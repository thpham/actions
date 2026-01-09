# Workflows Documentation

This directory contains detailed documentation for each reusable workflow.

## Workflow Reference

| Workflow                                                | Description                                  | Documentation                          |
| ------------------------------------------------------- | -------------------------------------------- | -------------------------------------- |
| [ci.yml](workflows/ci.md)                               | Build, test, lint, and Docker preview images | [View](workflows/ci.md)                |
| [release.yml](workflows/release.md)                     | Release Please, Docker builds, JReleaser     | [View](workflows/release.md)           |
| [backport.yml](workflows/backport.md)                   | Automated cherry-pick PRs                    | [View](workflows/backport.md)          |
| [suggest-backports.yml](workflows/suggest-backports.md) | Backport label suggestions                   | [View](workflows/suggest-backports.md) |
| [commitlint.yml](workflows/commitlint.md)               | Conventional Commits linting                 | [View](workflows/commitlint.md)        |
| [sonar.yml](workflows/sonar.md)                         | SonarQube/SonarCloud analysis                | [View](workflows/sonar.md)             |
| [cleanup-registry.yml](workflows/cleanup-registry.md)   | Container image cleanup                      | [View](workflows/cleanup-registry.md)  |
| [lint-workflows.yml](workflows/lint-workflows.md)       | Workflow validation                          | [View](workflows/lint-workflows.md)    |

## Migration Guide

See the [Migration Guide](migration.md) for instructions on migrating from existing workflows.

## Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                        Caller Repository                              │
├───────────────────────────────────────────────────────────────────────┤
│  .github/workflows/                                                   │
│  ├── ci.yml        → calls .github/workflows/ci.yml                   │
│  ├── release.yml   → calls .github/workflows/release.yml              │
│  ├── backport.yml  → calls .github/workflows/backport.yml             │
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
│  ├── ci.yml           (reusable)                                      │
│  ├── release.yml      (reusable)                                      │
│  ├── backport.yml     (reusable)                                      │
│  ├── suggest-backports.yml (reusable)                                 │
│  ├── commitlint.yml   (reusable)                                      │
│  ├── sonar.yml        (reusable)                                      │
│  ├── cleanup-registry.yml (reusable)                                  │
│  └── lint-workflows.yml (reusable)                                    │
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

### 1. Deny All Permissions by Default

Set `permissions: {}` at the workflow level to deny all permissions. The reusable workflow defines the specific permissions it needs at the job level.

```yaml
permissions: {}

jobs:
  ci:
    uses: thpham/actions/.github/workflows/ci.yml@main
```

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

| Workflow              | Required Secrets                  | Notes                                   |
| --------------------- | --------------------------------- | --------------------------------------- |
| `ci.yml`              | None                              | `GITHUB_TOKEN` is automatic             |
| `release.yml`         | None (optional: GPG\_\*, webhooks) | `GITHUB_TOKEN` is automatic             |
| `sonar.yml`           | `SONAR_TOKEN`                     | Required for SonarQube analysis         |
| `backport.yml`        | None                              | `GITHUB_TOKEN` is automatic             |
| `suggest-backports.yml` | None                            | `GITHUB_TOKEN` is automatic             |
| `commitlint.yml`      | None                              | No secrets needed                       |
| `cleanup-registry.yml`| None                              | `GITHUB_TOKEN` is automatic             |
| `lint-workflows.yml`  | None                              | No secrets needed                       |

### 4. Example: Secure Caller Workflow

```yaml
name: CI

on:
  push:
    branches: [main, "release/**"]
  pull_request:
    branches: [main, "release/**"]

# Deny all permissions - reusable workflow defines what it needs
permissions: {}

jobs:
  ci:
    uses: thpham/actions/.github/workflows/ci.yml@main
    with:
      docker-image-name: ${{ github.repository }}/api
    # No secrets: block needed - GITHUB_TOKEN is automatic
```

## Secrets Reference

| Secret           | Used By     | Required | Description           |
| ---------------- | ----------- | -------- | --------------------- |
| `GITHUB_TOKEN`   | All         | Auto     | GitHub API access     |
| `SONAR_TOKEN`    | sonar.yml   | Optional | SonarQube/SonarCloud  |
| `SONAR_HOST_URL` | sonar.yml   | Optional | Self-hosted SonarQube |
| `GPG_SECRET_KEY` | release.yml | Optional | Artifact signing      |
| `GPG_PASSPHRASE` | release.yml | Optional | GPG passphrase        |
| `GPG_PUBLIC_KEY` | release.yml | Optional | GPG public key        |
| `TEAMS_WEBHOOK`  | release.yml | Optional | Teams notifications   |
| `SLACK_WEBHOOK`  | release.yml | Optional | Slack notifications   |

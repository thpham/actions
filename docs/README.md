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

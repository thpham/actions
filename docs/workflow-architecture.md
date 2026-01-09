# Workflow Architecture

This document provides a comprehensive overview of the workflow architecture, including flow diagrams, job dependencies, and scenario guides.

## Table of Contents

- [Overview](#overview)
- [Workflow Architecture](#workflow-architecture)
- [Job Dependencies](#job-dependencies)
- [Versioning Strategy](#versioning-strategy)
- [Scenario Guides](#scenario-guides)
- [Troubleshooting](#troubleshooting)

---

## Overview

The reusable workflows work together to provide:

- **Continuous Integration**: Build, test, and lint on every PR and push
- **Automated Releases**: Semantic versioning with Release Please
- **Multi-Architecture Docker Builds**: Native amd64 and arm64 images
- **Backport Automation**: Cherry-pick fixes to maintenance branches
- **Code Quality**: SonarQube analysis and commit message validation
- **Container Registry Management**: Automatic cleanup of old images

| Workflow                | Purpose                                                       |
| ----------------------- | ------------------------------------------------------------- |
| `ci.yml`                | Build, test, and lint on PRs and pushes                       |
| `release.yml`           | Orchestrates releases, creates branches, builds Docker images |
| `backport.yml`          | Automatically creates backport PRs when labeled               |
| `suggest-backports.yml` | Suggests backport labels for fix/security PRs                 |
| `commitlint.yml`        | Enforces Conventional Commits specification                   |
| `sonar.yml`             | Code quality and coverage analysis                            |
| `lint-workflows.yml`    | Validates workflow syntax and security                        |
| `cleanup-registry.yml`  | Manages container image lifecycle                             |

---

## Workflow Architecture

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PULL REQUEST FLOW                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PR Opened ─┬─→ CI.build (compile & test)                                   │
│             ├─→ CI.lint (Spotless format check)                             │
│             ├─→ Commitlint (validate commit messages)                       │
│             ├─→ SonarQube (code quality analysis)                           │
│             ├─→ Suggest Backports (for fix/security PRs)                    │
│             └─→ CI.build-image (if "preview-image" label)                   │
│                       ↓                                                     │
│                 CI.merge-manifest (Docker manifest merge)                   │
│                                                                             │
│  PR Merged ──→ Backport (creates PRs for labeled backport branches)         │
│           └──→ Cleanup Registry (removes PR-specific images)                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         RELEASE FLOW (main/master branch)                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Push to main ─┬─→ CI.build + CI.lint + CI.build-image                      │
│                ├─→ SonarQube                                                │
│                └─→ Release.release-please (creates release PR)              │
│                                                                             │
│  Release PR merged ──→ Release.release-please (creates tag)                 │
│                    ├──→ Release.create-release-branch (release/X.Y)         │
│                    ├──→ Release.build (Maven artifacts)                     │
│                    ├──→ Release.build-docker (multi-arch images)            │
│                    ├──→ Release.merge-docker (manifest merge & tags)        │
│                    └──→ Release.distribute (JReleaser deployment)           │
│                                                                             │
│  New release branch ──→ Release.release-please (SNAPSHOT bump PR)           │
│                    └──→ Auto-merge SNAPSHOT PR                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        BACKPORT FLOW (release branches)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Fix merged to main with "backport release/X.Y" label                       │
│         ↓                                                                   │
│  Backport workflow creates cherry-pick PR to release/X.Y                    │
│         ↓                                                                   │
│  SNAPSHOT version check (warns if target not SNAPSHOT)                      │
│         ↓                                                                   │
│  Backport PR merged → Release workflow creates patch release                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Release Flow Branching Strategy

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

---

## Job Dependencies

### Release Workflow

```
                          release.yml
                              │
    ┌─────────────────────────┼────────────────────────┐
    │                         │                        │
    ▼                         ▼                        ▼
release-please ──────→ create-release-branch    (parallel jobs)
    │                         │
    │ release_created?        │ main/master only
    │                         │
    ▼                         ▼
  build ───────────────────────────────────────────────┐
    │                                                  │
    │ artifacts                                        │
    ▼                                                  ▼
build-docker ────────────────────────────────────→ merge-docker
(amd64)     │                                          │
            │                                          │
build-docker ──────────────────────────────────────────┘
(arm64)           (platform digests merged)
    │
    ▼
distribute (JReleaser)
```

### CI Workflow

```
                            ci.yml
                              │
    ┌─────────────────────────┼────────────────────┐
    │                         │                    │
    ▼                         ▼                    ▼
  build              lint (Spotless)          build-image
(compile & test)     (format check)           (if labeled)
    │                         │                    │
    │                         │              ┌─────┴─────┐
    │                         │              ▼           ▼
    │                         │           amd64       arm64
    │                         │              │           │
    │                         │              └─────┬─────┘
    │                         │                    ▼
    │                         │            merge-manifest
    │                         │                    │
    └─────────────────────────┴────────────────────┘
```

---

## Versioning Strategy

### Branch-Based Versioning

| Branch          | Config File                        | Version Bump | Example        |
| --------------- | ---------------------------------- | ------------ | -------------- |
| `main`/`master` | `release-please-config.json`       | Minor        | 1.9.0 → 1.10.0 |
| `release/*`     | `release-please-config-patch.json` | Patch        | 1.9.0 → 1.9.1  |

### SNAPSHOT Lifecycle

```
main: 1.10.0-SNAPSHOT (development)
        ↓ (release)
Tag: v1.10.0
        ↓
main: 1.11.0-SNAPSHOT (bumped to next minor)
        ↓
release/1.10 created from v1.10.0
        ↓
release/1.10: 1.10.1-SNAPSHOT (ready for patches)
```

### Conventional Commits

| Type                  | Version Bump | Changelog Section        |
| --------------------- | ------------ | ------------------------ |
| `feat`                | Minor        | Features                 |
| `fix`                 | Patch        | Bug Fixes                |
| `security`            | Patch        | Security                 |
| `perf`                | Patch        | Performance Improvements |
| `docs`                | None         | Documentation            |
| `ci`, `chore`, `test` | None         | Hidden                   |

---

## Scenario Guides

### Scenario 1: Normal Development Flow

1. **Create feature branch** from `main`
2. **Make commits** using Conventional Commits format
3. **Open PR** to `main`
4. **CI checks run** automatically:
   - Build and Tests
   - Lint check
   - Commitlint validation
   - SonarQube analysis (if configured)
5. **Review and merge** PR
6. **Release Please** creates/updates release PR
7. **Merge release PR** when ready to release

### Scenario 2: Creating a Release

1. **Merge release PR** created by Release Please
2. **Workflow automatically**:
   - Creates git tag (e.g., `v1.10.0`)
   - Creates `release/1.10` branch
   - Builds multi-arch Docker images
   - Pushes to GitHub Container Registry
   - Distributes via JReleaser
3. **SNAPSHOT bump** PR created on release branch
4. **Auto-merged** to prepare for patches

### Scenario 3: Backporting a Fix

1. **Create fix** on `main` branch with `fix:` commit type
2. **Suggest Backports workflow** suggests labels (or add manually)
3. **Add label** (e.g., `backport release/1.9`)
4. **Merge PR** to `main`
5. **Backport workflow** creates cherry-pick PR to `release/1.9`
6. **SNAPSHOT check** warns if target branch needs version bump
7. **Review and merge** backport PR
8. **Release workflow** creates patch release (e.g., `v1.9.1`)

### Scenario 4: Preview Docker Image for PR

1. **Open PR** with code changes
2. **Add label** `preview-image`
3. **CI workflow** builds Docker image
4. **Comment posted** with pull command:
   ```bash
   docker pull ghcr.io/owner/repo/api:pr-123
   ```
5. **Image deleted** when PR is closed

---

## Troubleshooting

### Common Issues

#### Release Please Not Creating PR

**Symptoms**: No release PR after merging commits

**Checks**:

1. Verify commits follow Conventional Commits format
2. Check `release-please-config.json` for `changelog-types`
3. Ensure branch is `main`, `master`, or `release/*`

**Manual Trigger**:

```bash
gh workflow run release.yml --ref main
```

#### Backport PR Version Conflict

**Symptoms**: Warning about non-SNAPSHOT version

**Cause**: Backport PR created before SNAPSHOT bump was merged

**Resolution**:

1. Wait for SNAPSHOT bump PR to be auto-merged
2. Re-run backport workflow or manually cherry-pick

#### Docker Build Failing on ARM

**Symptoms**: `build-docker` job fails for arm64

**Checks**:

1. Verify ARM runner available (`ubuntu-24.04-arm`)
2. Check Dockerfile compatibility with ARM
3. Review BuildKit cache configuration

#### SNAPSHOT Bump Not Auto-Merging

**Symptoms**: SNAPSHOT PR remains open

**Cause**: Auto-merge not enabled on repository

**Resolution**:

1. Enable auto-merge in repository settings
2. Or manually merge PR with:
   ```bash
   gh pr merge <PR_NUMBER> --squash
   ```

#### Workflow Dispatch Permission Denied

**Symptoms**: `HTTP 403` when triggering workflow

**Resolution**: The caller workflow should have `permissions: {}` at workflow level. The reusable workflow defines the permissions it needs at job level.

### Manual Interventions

#### Manually Create Release Branch

```bash
# Checkout the release tag
git checkout v1.10.0

# Create and push branch
git checkout -b release/1.10
git push origin release/1.10

# Trigger Release workflow for SNAPSHOT bump
gh workflow run release.yml --ref release/1.10
```

#### Manually Trigger Release

```bash
# From main branch
gh workflow run release.yml --ref main

# From release branch
gh workflow run release.yml --ref release/1.10
```

#### Clean Up Stale Images

```bash
# Manual cleanup trigger
gh workflow run cleanup-registry.yml

# Or delete specific image
gh api -X DELETE /user/packages/container/api/versions/VERSION_ID
```

#### Re-run Failed Backport

```bash
# List recent workflow runs
gh run list --workflow=backport.yml

# Re-run failed run
gh run rerun RUN_ID
```

### Recovery Procedures

#### Recover from Failed Release

1. **Check workflow logs** for error details
2. **Fix the issue** (e.g., test failure, Docker build error)
3. **Delete partial artifacts** if needed:
   ```bash
   # Delete failed release tag
   git push --delete origin v1.10.0
   git tag -d v1.10.0
   ```
4. **Re-run release workflow**:
   ```bash
   gh workflow run release.yml --ref main
   ```

#### Recover from Version Conflict

1. **Identify conflicting versions** in manifest and POMs
2. **Update `.release-please-manifest.json`**:
   ```json
   { ".": "1.10.0" }
   ```
3. **Update POM versions** if needed
4. **Commit and push** to trigger new release cycle

---

## Configuration Files Reference

| File                               | Purpose                               |
| ---------------------------------- | ------------------------------------- |
| `release-please-config.json`       | Main branch: minor version bumps      |
| `release-please-config-patch.json` | Release branches: patch version bumps |
| `.release-please-manifest.json`    | Tracks current release version        |
| `commitlint.config.mjs`            | Commit message validation rules       |
| `jreleaser.yml`                    | Release distribution configuration    |

---

## Related Documentation

- [Workflow Reference](README.md) - Individual workflow documentation
- [Migration Guide](migration.md) - Migrating from standalone workflows
- [Security Best Practices](README.md#security-best-practices) - Caller workflow security
- [GHE Cloud Compatibility](ghe-cloud-compatibility.md) - Enterprise migration guide

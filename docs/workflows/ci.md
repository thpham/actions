# Kotlin/Maven CI Workflow

Build, test, lint, and optionally build multi-arch Docker preview images for Kotlin/Maven/Spring Boot projects.

**Workflow file:** `kotlin-mvn-ci.yml`

## Usage

```yaml
name: CI

on:
  push:
    branches: [main, master, "release/**"]
    paths: ["pom.xml", "src/**", "modules/**"]
  pull_request:
    branches: [main, master, "release/**"]
    types: [opened, synchronize, reopened, labeled]
    paths: ["pom.xml", "src/**", "modules/**"]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Permissions required by the reusable workflow
permissions:
  contents: read
  packages: write
  pull-requests: write
  actions: write

jobs:
  ci:
    uses: thpham/actions/.github/workflows/kotlin-mvn-ci.yml@main
    with:
      docker-image-name: ${{ github.repository }}/myservice-api
    # GITHUB_TOKEN is automatically available - no secrets block needed
```

## Permissions

| Permission      | Purpose                                |
| --------------- | -------------------------------------- |
| `contents`      | Read - Checkout code                   |
| `packages`      | Write - Push Docker images to registry |
| `pull-requests` | Write - Comment on PRs with image info |
| `actions`       | Write - Cancel duplicate workflows     |

## Inputs

| Input                     | Type    | Required | Default              | Description                   |
| ------------------------- | ------- | -------- | -------------------- | ----------------------------- |
| `java-version`            | string  | No       | `'17'`               | Java version                  |
| `java-distribution`       | string  | No       | `'temurin'`          | Java distribution             |
| `maven-goals`             | string  | No       | `'verify'`           | Maven goals to run            |
| `spotless-check`          | boolean | No       | `true`               | Run Spotless format check     |
| `cancel-on-lint-failure`  | boolean | No       | `true`               | Cancel workflow if lint fails |
| `docker-enabled`          | boolean | No       | `true`               | Enable Docker builds          |
| `docker-registry`         | string  | No       | `'ghcr.io'`          | Container registry            |
| `docker-image-name`       | string  | **Yes**  | -                    | Docker image name             |
| `docker-context`          | string  | No       | `'.'`                | Docker build context          |
| `docker-file`             | string  | No       | `'Dockerfile'`       | Dockerfile path               |
| `docker-build-args`       | string  | No       | `''`                 | Additional build args         |
| `pr-preview-label`        | string  | No       | `'preview-image'`    | Label to trigger PR builds    |
| `artifact-path`           | string  | No       | `'**/target/*.jar'`  | Build artifact path           |
| `artifact-retention-days` | number  | No       | `1`                  | Artifact retention            |
| `enable-arm64`            | boolean | No       | `true`               | Enable ARM64 builds           |
| `amd64-runner`            | string  | No       | `'ubuntu-latest'`    | AMD64 runner                  |
| `arm64-runner`            | string  | No       | `'ubuntu-24.04-arm'` | ARM64 runner                  |

## Outputs

| Output        | Description                |
| ------------- | -------------------------- |
| `version`     | Maven project version      |
| `primary_tag` | Primary Docker image tag   |
| `rolling_tag` | Rolling Docker image tag   |
| `is_pr`       | Whether this is a PR build |

## Jobs

1. **build** - Compile, test, extract version, generate tags
2. **lint** - Spotless format check (optional)
3. **build-image** - Multi-arch Docker builds (conditional)
4. **merge-manifest** - Merge multi-arch manifests

## PR Preview Images

To build preview images for a PR, add the `preview-image` label.

The workflow will:

1. Build multi-arch images (amd64 + arm64)
2. Push to the container registry
3. Comment on the PR with pull commands

Tags created:

- `pr-{N}-{timestamp}-{sha}` (immutable)
- `pr-{N}` (rolling)

## Cache Strategy

- Maven cache restored for all jobs
- Maven cache saved only on push (not PRs) to prevent cache poisoning
- Docker layer cache per platform and branch

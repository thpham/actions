# Github Actions, Composite and Reusable Workflows

Reusable GitHub Actions workflows for organization-wide standardization, organized by technology stack.

## Available Stacks

| Stack          | Description                            | Documentation                                                            |
| -------------- | -------------------------------------- | ------------------------------------------------------------------------ |
| **kotlin-mvn** | Kotlin/Maven/Spring Boot microservices | [View Docs](docs/README.md) \| [Migration](docs/kotlin-mvn/migration.md) |

## Repository Structure

```
actions/
├── .github/workflows/           # Reusable workflows
│   ├── ci.yml
│   ├── release.yml
│   ├── backport.yml
│   ├── suggest-backports.yml
│   ├── commitlint.yml
│   ├── sonar.yml
│   ├── cleanup-registry.yml
│   └── lint-workflows.yml
├── configs/
│   └── kotlin-mvn/              # Stack-specific config templates
│       ├── release-please-config.json
│       ├── commitlint.config.mjs
│       └── ...
├── docs/
│   ├── README.md                # Documentation index
│   ├── workflows/               # Workflow-specific documentation
│   │   ├── ci.md
│   │   ├── release.md
│   │   └── ...
│   └── kotlin-mvn/              # Stack-specific guides
│       └── migration.md
└── README.md                    # This file
```

## Quick Start: kotlin-mvn

For Kotlin/Maven/Spring Boot microservices with Docker containers.

### CI Workflow

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
      docker-image-name: ${{ github.repository }}/myservice-api
    secrets: inherit
```

### Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main, "release/**"]
  workflow_dispatch:

jobs:
  release:
    uses: thpham/actions/.github/workflows/release.yml@v1
    with:
      docker-image-name: ${{ github.repository }}/myservice-api
    secrets: inherit
```

### All Available Workflows (kotlin-mvn)

| Workflow                | Purpose                             | Trigger                      |
| ----------------------- | ----------------------------------- | ---------------------------- |
| `ci.yml`                | Build, test, lint, Docker preview   | push, pull_request           |
| `release.yml`           | Release Please + Docker + JReleaser | push to main/release/\*\*    |
| `backport.yml`          | Auto cherry-pick PRs                | pull_request_target [closed] |
| `suggest-backports.yml` | Suggest backport labels             | pull_request_target [opened] |
| `commitlint.yml`        | Conventional Commits                | pull_request                 |
| `sonar.yml`             | Code quality analysis               | push, pull_request           |
| `cleanup-registry.yml`  | Container image cleanup             | PR close, schedule           |
| `lint-workflows.yml`    | Workflow validation                 | push/PR to workflows/\*\*    |

**Full documentation:** [Workflow Reference](docs/README.md) | [Migration Guide](docs/kotlin-mvn/migration.md)

## Features

- **Multi-architecture Docker builds** - Native amd64 + arm64 runners (no QEMU)
- **Release Flow support** - Minor bumps on main, patch bumps on release branches
- **Automated backporting** - Cherry-pick fixes to maintenance branches
- **PR preview images** - On-demand Docker images for testing
- **Code quality** - SonarQube integration with self-hosted support
- **Conventional Commits** - Enforced commit message format
- **Registry cleanup** - Automated container image lifecycle management

## Adding a New Stack

To add support for a new technology stack:

1. Add stack-specific inputs/conditionals to workflows in `.github/workflows/`
2. Create config templates: `configs/<stack-name>/`
3. Create stack-specific guides: `docs/<stack-name>/`
4. Update workflow documentation in `docs/workflows/`
5. Update this README with the new stack

## License

MIT

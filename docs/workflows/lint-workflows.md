# Lint Workflows

Validates GitHub Actions workflow syntax and security.

## Usage

```yaml
name: Lint Workflows

on:
  push:
    branches:
      - main
      - master
      - "release/**"
    paths:
      - ".github/workflows/**"
      - ".github/actionlint.yaml"
  pull_request:
    branches:
      - main
      - master
      - "release/**"
    paths:
      - ".github/workflows/**"
      - ".github/actionlint.yaml"

concurrency:
  group: lint-workflows-${{ github.ref }}
  cancel-in-progress: true

# Permissions required by the reusable workflow
permissions:
  contents: read
  actions: read
  security-events: write

jobs:
  lint:
    uses: thpham/actions/.github/workflows/lint-workflows.yml@main
    # No secrets needed for linting
```

## Inputs

| Input               | Type    | Required | Default                     | Description              |
| ------------------- | ------- | -------- | --------------------------- | ------------------------ |
| `actionlint-config` | string  | No       | `'.github/actionlint.yaml'` | Actionlint config        |
| `enable-zizmor`     | boolean | No       | `true`                      | Enable security scanning |
| `runner`            | string  | No       | `'ubuntu-latest'`           | Runner to use            |

## Jobs

### actionlint

Validates workflow syntax using [actionlint](https://github.com/rhysd/actionlint):

- Expression syntax
- Action inputs/outputs
- Runner compatibility
- Shell script validation

### zizmor (optional)

Security scanning using [zizmor](https://github.com/zizmorcore/zizmor):

- Dangerous triggers (`pull_request_target`)
- Credential exposure
- Injection vulnerabilities
- Results uploaded to GitHub Advanced Security

## Configuration

Create `.github/actionlint.yaml` for custom rules:

```yaml
self-hosted-runner:
  labels:
    - self-hosted
    - ubuntu-24.04-arm

config-variables:
  - MY_ORG_VAR
```

## Permissions

| Permission        | Purpose                             |
| ----------------- | ----------------------------------- |
| `contents`        | Read - Access workflow files        |
| `actions`         | Read - Access workflow run metadata |
| `security-events` | Write - Upload SARIF to GitHub GHAS |

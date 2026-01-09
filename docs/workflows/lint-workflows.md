# Lint Workflows

Validates GitHub Actions workflow syntax and security.

## Usage

```yaml
name: Lint Workflows

on:
  push:
    paths: [".github/workflows/**"]
  pull_request:
    paths: [".github/workflows/**"]

jobs:
  lint:
    uses: thpham/actions/.github/workflows/lint-workflows.yml@v1
    secrets: inherit
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

- `contents: read` (both jobs)
- `actions: read` (zizmor only)
- `security-events: write` (zizmor only)

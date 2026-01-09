# SonarQube Workflow

Code quality and coverage analysis with SonarQube or SonarCloud.

## Usage

```yaml
name: SonarQube

on:
  push:
    branches: [main, master, "release/**"]
    paths: ["pom.xml", "src/**", "modules/**"]
  pull_request:
    branches: [main, master, "release/**"]
    paths: ["pom.xml", "src/**", "modules/**"]

concurrency:
  group: sonar-${{ github.ref }}
  cancel-in-progress: true

jobs:
  sonar:
    uses: thpham/actions/.github/workflows/sonar.yml@v1
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

## Inputs

| Input               | Type    | Required | Default           | Description                 |
| ------------------- | ------- | -------- | ----------------- | --------------------------- |
| `java-version`      | string  | No       | `'17'`            | Java version                |
| `java-distribution` | string  | No       | `'temurin'`       | Java distribution           |
| `maven-goals`       | string  | No       | `'verify'`        | Maven goals before analysis |
| `sonar-args`        | string  | No       | `''`              | Additional Sonar arguments  |
| `skip-dependabot`   | boolean | No       | `true`            | Skip Dependabot PRs         |
| `skip-forks`        | boolean | No       | `true`            | Skip fork PRs               |
| `runner`            | string  | No       | `'ubuntu-latest'` | Runner to use               |

## Secrets

| Secret           | Required | Description                |
| ---------------- | -------- | -------------------------- |
| `SONAR_TOKEN`    | No\*     | SonarQube/SonarCloud token |
| `SONAR_HOST_URL` | No       | Self-hosted SonarQube URL  |

\*If `SONAR_TOKEN` is not configured, the workflow skips analysis gracefully.

## SonarCloud vs Self-Hosted

- **SonarCloud**: Leave `SONAR_HOST_URL` empty
- **Self-hosted**: Set `SONAR_HOST_URL` to your SonarQube server URL

## Skipped Cases

Analysis is skipped for:

- Dependabot PRs (no secret access)
- Fork PRs (no secret access)
- Missing `SONAR_TOKEN`

## Cache Strategy

- Maven cache restored for all runs
- SonarQube cache restored for all runs
- Caches saved only on push (not PRs) to prevent cache poisoning

# Cleanup Registry Workflow

Automated container image lifecycle management for GitHub Container Registry.

## Usage

```yaml
name: Cleanup Registry

on:
  pull_request:
    types: [closed]
  schedule:
    - cron: "0 3 * * *" # Daily at 3 AM UTC
  workflow_dispatch:

jobs:
  cleanup:
    uses: thpham/actions/.github/workflows/cleanup-registry.yml@v1
    with:
      image-name: ${{ github.repository }}/myservice-api
    secrets: inherit
```

## Inputs

| Input             | Type    | Required | Default           | Description                                          |
| ----------------- | ------- | -------- | ----------------- | ---------------------------------------------------- |
| `image-name`      | string  | **Yes**  | -                 | Full package path (e.g., `owner/repo/image-name`)    |
| `keep-n-tagged`   | number  | No       | `5`               | Tagged images to keep         |
| `delete-untagged` | boolean | No       | `true`            | Delete untagged manifests     |
| `preserve-semver` | boolean | No       | `true`            | Preserve version tags         |
| `preserve-edge`   | boolean | No       | `true`            | Preserve edge tag             |
| `runner`          | string  | No       | `'ubuntu-latest'` | Runner to use                 |

## Jobs

### cleanup-pr (on PR close)

Deletes images specific to the closed PR:

- `pr-{N}` (rolling tag)
- `pr-{N}-*` (timestamped tags)

### cleanup-scheduled (on schedule/dispatch)

Bulk cleanup that:

- Deletes old `main-*` timestamped tags (keeps N newest)
- Deletes orphaned PR images (closed PRs)
- Cleans untagged manifests
- Preserves:
  - `edge` tag
  - Semantic version tags (v1.2.3, etc.)
  - Open PR images

## Multi-Arch Safety

Uses `dataaxiom/ghcr-cleanup-action` which:

- Properly handles multi-architecture images
- Preserves manifest integrity
- Handles referrers/attestations

## Image Tagging Strategy

| Source    | Pattern                       | Cleanup Behavior            |
| --------- | ----------------------------- | --------------------------- |
| Release   | `1.2.3`, `1.2`, `latest`      | Preserved                   |
| Main push | `main-{ts}-{sha}`, `edge`     | Old removed, edge preserved |
| PR        | `pr-{N}-{ts}-{sha}`, `pr-{N}` | Removed on PR close         |

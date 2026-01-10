# GHCR Cleanup Workflow

Automated container image lifecycle management for GitHub Container Registry (GHCR).

**Workflow file:** `cleanup-ghcr.yml`

> **Note:** This workflow is specific to GitHub Container Registry (ghcr.io). For other OCI-compliant registries (ECR, ACR, GCR, Harbor), see [cleanup-oci.md](cleanup-oci.md).

## Usage

```yaml
name: Cleanup GHCR Images

on:
  pull_request:
    types: [closed]
  schedule:
    - cron: "0 3 * * *" # Daily at 3 AM UTC
  workflow_dispatch:

# Permissions required by the reusable workflow
permissions:
  packages: write
  pull-requests: read

jobs:
  cleanup:
    uses: thpham/actions/.github/workflows/cleanup-ghcr.yml@main
    with:
      image-name: ${{ github.event.repository.name }}/myproject-api
    # GITHUB_TOKEN is automatically available to reusable workflows
```

## Permissions

| Permission      | Purpose                         |
| --------------- | ------------------------------- |
| `packages`      | Write - Delete container images |
| `pull-requests` | Read - Access PR metadata       |

## Inputs

| Input             | Type    | Required | Default           | Description                                       |
| ----------------- | ------- | -------- | ----------------- | ------------------------------------------------- |
| `image-name`      | string  | **Yes**  | -                 | Full package path (e.g., `owner/repo/image-name`) |
| `keep-n-tagged`   | number  | No       | `5`               | Tagged images to keep                             |
| `delete-untagged` | boolean | No       | `true`            | Delete untagged manifests                         |
| `preserve-semver` | boolean | No       | `true`            | Preserve version tags                             |
| `preserve-edge`   | boolean | No       | `true`            | Preserve edge tag                                 |
| `runner`          | string  | No       | `'ubuntu-latest'` | Runner to use                                     |

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

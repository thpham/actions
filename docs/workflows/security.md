# Container Security

Comprehensive security features for container images including vulnerability scanning, image signing, and SBOM generation.

## Overview

Both CI and Release workflows support:
- **Trivy vulnerability scanning** with SARIF reports
- **Cosign image signing** (keyless OIDC or private key)
- **SBOM generation** (CycloneDX or SPDX format)
- **Per-architecture SBOM attestation** (SLSA best practice)
- **GitHub Security integration** via SARIF upload

## Security Inputs

### Vulnerability Scanning

| Input | Type | CI Default | Release Default | Description |
|-------|------|------------|-----------------|-------------|
| `security-scan-enabled` | boolean | `true` | `true` | Enable Trivy vulnerability scanning |
| `pr-scan-label` | string | `"scan-image"` | N/A | PR label that triggers scanning (CI only) |
| `scan-severity-threshold` | string | `"CRITICAL,HIGH"` | `"CRITICAL,HIGH"` | Minimum severity to report |
| `scan-fail-on-vuln` | boolean | `false` | `true` | Fail build if vulnerabilities found |
| `trivy-ignore-unfixed` | boolean | `true` | `false` | Ignore unfixed vulnerabilities |
| `trivy-skip-dirs` | string | `""` | `""` | Directories to skip (comma-separated) |

> **PR Scanning Behavior:** For pull requests, security scanning only runs if the PR has both
> the `preview-image` label (to build the image) AND the `scan-image` label. This allows faster
> iteration on PRs while still providing security scanning on-demand. Pushes to main/master
> always run security scans if `security-scan-enabled` is true.

### Image Signing

| Input | Type | CI Default | Release Default | Description |
|-------|------|------------|-----------------|-------------|
| `sign-images` | boolean | `false` | `true` | Enable Cosign image signing |
| `signing-method` | string | `"oidc"` | `"oidc"` | Signing method: `oidc` (keyless) or `key` |

### SBOM (Release Only)

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `sbom-enabled` | boolean | `true` | Generate container SBOM with Trivy |
| `sbom-format` | string | `"cyclonedx"` | SBOM format: `cyclonedx` or `spdx` |
| `sbom-attestation` | boolean | `true` | Attest SBOM to per-architecture image digest (SLSA best practice) |
| `sbom-upload-release` | boolean | `true` | Upload merged SBOM to GitHub release assets |

> **Note:** SBOM attestation follows SLSA best practices by attesting each architecture's SBOM
> to its specific image digest. A merged SBOM is also created and uploaded as a release asset
> for download convenience, but is not attested to the multi-arch manifest.

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `COSIGN_PRIVATE_KEY` | If `signing-method: key` | Cosign private key for image signing |
| `COSIGN_PASSWORD` | If `signing-method: key` | Cosign private key password |

## Required Permissions

Add these permissions when calling the workflows:

```yaml
permissions:
  contents: read
  packages: write
  pull-requests: write      # For PR comments
  actions: write            # For workflow cancellation
  security-events: write    # For SARIF upload
  id-token: write           # For OIDC keyless signing and SBOM attestation
```

## Usage Examples

### CI Workflow with Security Scanning

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  packages: write
  pull-requests: write
  actions: write
  security-events: write
  id-token: write

jobs:
  ci:
    uses: thpham/actions/.github/workflows/kotlin-mvn-ci.yml@main
    with:
      docker-image-name: ${{ github.repository }}/myservice
      # Security scanning behavior:
      # - PRs: only scans if 'scan-image' label is present (opt-in)
      # - Pushes to main: always scans
      pr-scan-label: scan-image     # Default label for PR scanning
      scan-fail-on-vuln: false      # Warn only (default for CI)
      sign-images: false            # Optional for CI
```

**PR Labels:**
- `preview-image` - Build Docker preview image
- `scan-image` - Also run Trivy vulnerability scan (requires `preview-image`)

### Release Workflow with Full Security

```yaml
name: Release

on:
  push:
    branches: [main, "release/**"]

permissions:
  contents: write
  packages: write
  pull-requests: write
  actions: write
  security-events: write
  id-token: write

jobs:
  release:
    uses: thpham/actions/.github/workflows/kotlin-mvn-release.yml@main
    with:
      docker-image-name: ${{ github.repository }}/myservice
      # All security features enabled by default for releases
      scan-fail-on-vuln: true        # Fail on vulnerabilities
      sign-images: true              # Sign with keyless OIDC
      sbom-enabled: true             # Generate SBOM
      sbom-attestation: true         # Attest SBOM to each architecture digest
      sbom-upload-release: true      # Upload merged SBOM to release
```

### Using Private Key Signing

```yaml
jobs:
  release:
    uses: thpham/actions/.github/workflows/kotlin-mvn-release.yml@main
    with:
      docker-image-name: ${{ github.repository }}/myservice
      sign-images: true
      signing-method: key           # Use private key instead of OIDC
    secrets:
      COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
      COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```

## Verification Commands

### Verify Image Signature (Keyless OIDC)

When using these reusable workflows, the certificate identity reflects the **workflow's repository**
(`thpham/actions`), not the calling repository. This is expected behavior since the signing happens
within the reusable workflow context.

```bash
# For images built with thpham/actions reusable workflows
cosign verify \
  --certificate-identity-regexp="https://github.com/thpham/actions/.*" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/owner/repo/image:tag

# Example: Verify a specific image
cosign verify \
  --certificate-identity-regexp="https://github.com/thpham/actions/.*" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/thpham/gha-mvn-rflow/myproject-api:v1.0.0
```

The signature metadata includes the actual calling repository context:
- `githubWorkflowRepository`: The repository that triggered the workflow
- `githubWorkflowRef`: The branch/tag that triggered the build
- `githubWorkflowTrigger`: The event type (push, pull_request, etc.)

### Verify Image Signature (Private Key)

```bash
cosign verify \
  --key cosign.pub \
  ghcr.io/owner/repo/image:tag
```

### Verify SBOM Attestation (Per-Architecture)

SBOM attestations are attached to each architecture-specific image digest:

```bash
# Get architecture-specific digest from manifest
docker manifest inspect ghcr.io/owner/repo/image:tag | jq '.manifests[] | select(.platform.architecture=="amd64") | .digest'

# Verify SBOM attestation on the architecture digest
cosign verify-attestation \
  --type cyclonedx \
  --certificate-identity-regexp="https://github.com/thpham/actions/.*" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  ghcr.io/owner/repo/image@sha256:abc123...
```

### Extract SBOM from Attestation

```bash
# Extract SBOM from an architecture-specific image
cosign download attestation ghcr.io/owner/repo/image@sha256:abc123... \
  | jq -r '.payload' \
  | base64 -d \
  | jq '.predicate'
```

### View Vulnerability Scan Results

1. Navigate to your repository on GitHub
2. Go to **Security** tab
3. Click **Code scanning alerts**
4. Filter by tool: **Trivy**

## Workflow Outputs

### CI Workflow

| Output | Description |
|--------|-------------|
| `scan_result` | Security scan result (passed/failed/skipped) |
| `image_signed` | Whether the image was signed |
| `manifest_digest` | Multi-arch manifest digest |

### Release Workflow

| Output | Description |
|--------|-------------|
| `image_signed` | Whether the image was signed |
| `manifest_digest` | Multi-arch manifest digest |

> **Note:** SBOM attestation status is not exposed as an output since attestation happens
> per-architecture in the scan-docker job. Verify attestation using the cosign commands above.

## Architecture

### CI Workflow Security Flow

```
build-image (matrix: amd64, arm64)
         │
         ▼
   scan-image (matrix: amd64, arm64)
   ├── Trivy vulnerability scan
   ├── SARIF upload to GitHub Security
   └── Table output for logs
         │
         ▼
   merge-manifest
   ├── Create multi-arch manifest
   └── Cosign sign (optional)
```

### Release Workflow Security Flow

```
build-docker (matrix: amd64, arm64)
         │
         ▼
   scan-docker (matrix: amd64, arm64)
   ├── Trivy vulnerability scan
   ├── SBOM generation (CycloneDX)
   ├── Cosign attest SBOM to arch digest (SLSA best practice)
   ├── SARIF upload to GitHub Security
   └── Upload SBOM artifacts
         │
         ▼
   merge-docker
   ├── Create multi-arch manifest
   ├── Cosign sign manifest
   ├── Merge architecture SBOMs (for download)
   └── Upload combined SBOM artifact
         │
         ▼
   distribute
   └── Upload merged SBOM to GitHub Release
```

## Disabling Security Features

All security features can be disabled if needed:

```yaml
jobs:
  ci:
    uses: thpham/actions/.github/workflows/kotlin-mvn-ci.yml@main
    with:
      docker-image-name: ${{ github.repository }}/myservice
      security-scan-enabled: false   # Skip vulnerability scanning
      sign-images: false             # Skip image signing
```

For releases:

```yaml
jobs:
  release:
    uses: thpham/actions/.github/workflows/kotlin-mvn-release.yml@main
    with:
      docker-image-name: ${{ github.repository }}/myservice
      security-scan-enabled: false   # Skip vulnerability scanning
      sign-images: false             # Skip image signing
      sbom-enabled: false            # Skip SBOM generation
      sbom-attestation: false        # Skip SBOM attestation
      sbom-upload-release: false     # Skip SBOM upload to release
```

## Troubleshooting

### OIDC Token Issues

If keyless signing fails with OIDC token errors:
1. Ensure `id-token: write` permission is set
2. Check that the workflow is running on GitHub Actions (not self-hosted without OIDC)
3. Verify the repository has GitHub Actions OIDC enabled

### Trivy Scan Timeouts

For large images, increase the timeout:
- The default Trivy timeout is 5 minutes
- Consider using `trivy-skip-dirs` to exclude non-essential directories

### SBOM Generation Failures

If SBOM generation fails:
1. Check that the image was successfully pushed to the registry
2. Verify registry authentication is working
3. Check Trivy action logs for specific errors

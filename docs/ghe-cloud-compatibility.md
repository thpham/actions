# GitHub Actions Compatibility Report

## GHE Cloud & Data Residency Impact Assessment

This report provides a comprehensive inventory of all GitHub Actions used in the reusable workflows and their compatibility with GitHub Enterprise Cloud (GHE Cloud) and data residency environments.

> **Note**: This analysis is based on source code inspection of vendored actions (see `vendir.yml`).

---

## Executive Summary

| Risk Level   | Count | Actions                                                                 |
| ------------ | ----- | ----------------------------------------------------------------------- |
| **Critical** | 1     | dataaxiom/ghcr-cleanup-action                                           |
| **High**     | 1     | wagoid/commitlint-github-action                                         |
| **Medium**   | 2     | googleapis/release-please-action, jreleaser/release-action              |
| **Low**      | 13    | All core GitHub actions, Docker actions, linting tools, backport-action |

### Key Findings

1. **1 action has hardcoded URLs** and will NOT work with GHE data residency without forking
2. **1 action pulls from Docker Hub** which may be blocked in enterprise environments
3. **2 actions need explicit URL configuration** via inputs
4. **13 actions are fully compatible** - they use environment variables set by GHE runners

### Source Code Verification

All findings below are verified by inspecting the actual vendored action source code, not just documentation.

---

## GHE Cloud & Data Residency Background

### URL Endpoint Changes

| Environment        | API URL                              | Registry URL        | Web URL                          |
| ------------------ | ------------------------------------ | ------------------- | -------------------------------- |
| **GitHub.com**     | `https://api.github.com`             | `https://ghcr.io`   | `https://github.com`             |
| **GHE Cloud**      | `https://api.YOUR-SUBDOMAIN.ghe.com` | Enterprise-specific | `https://YOUR-SUBDOMAIN.ghe.com` |
| **GHE Cloud (EU)** | `https://api.YOUR-SUBDOMAIN.ghe.com` | EU data center      | `https://YOUR-SUBDOMAIN.ghe.com` |

### How GHE Runners Handle URLs

GHE runners automatically set these environment variables:

```bash
GITHUB_SERVER_URL=https://github.mycompany.com
GITHUB_API_URL=https://github.mycompany.com/api/v3
GITHUB_GRAPHQL_URL=https://github.mycompany.com/api/graphql
ACTIONS_CACHE_URL=https://artifactcache.actions.mycompany.com/...
ACTIONS_RESULTS_URL=https://results.actions.mycompany.com/...
```

Well-designed actions use these variables with fallbacks to `github.com` defaults.

---

## Detailed Action Inventory

### Critical Risk Actions

#### 1. dataaxiom/ghcr-cleanup-action

| Property           | Value                |
| ------------------ | -------------------- |
| **Used In**        | cleanup-ghcr.yml     |
| **Risk Level**     | **CRITICAL**         |
| **GHE Compatible** | **NO**               |

**Source Code Analysis:**

```javascript
// Found in dist/index.js - HARDCODED URLs:
baseUrl: "https://api.github.com",
let githubUrl = 'https://api.github.com';
this.baseUrl = 'https://ghcr.io/';
```

**Problem**: This action has `ghcr.io` and `api.github.com` hardcoded in the compiled JavaScript. It does **NOT** read from `GITHUB_API_URL` environment variable.

**Impact**: Will fail on GHE Cloud data residency deployments - cannot be configured via inputs.

**Recommendation**:

- **Do not use** this action with GHE data residency
- Alternative: Use GitHub's built-in package retention policies
- Alternative: Fork and modify to use `GITHUB_API_URL` and configurable registry URL

---

### High Risk Actions

#### 2. wagoid/commitlint-github-action

| Property           | Value            |
| ------------------ | ---------------- |
| **Used In**        | commitlint.yml   |
| **Risk Level**     | **HIGH**         |
| **GHE Compatible** | Partial (Docker) |

**Source Code Analysis:**

```yaml
# From action.yml:
runs:
  using: docker
  image: docker://wagoid/commitlint-github-action:6.2.1
```

**Problem**: Pulls Docker image from Docker Hub (`docker.io`) at runtime.

**Impact in Enterprise Environments**:

- Fails if Docker Hub is blocked by firewall
- Fails in air-gapped environments
- No network access = no action execution

**Recommendation**:

- Mirror image to internal registry: `docker pull wagoid/commitlint-github-action:6.2.1 && docker tag ... && docker push ...`
- Fork action and update `action.yml` to point to internal registry:
  ```yaml
  image: docker://your-registry.company.com/commitlint-github-action:6.2.1
  ```

---

### Medium Risk Actions

#### 3. googleapis/release-please-action

| Property           | Value                  |
| ------------------ | ---------------------- |
| **Used In**        | kotlin-mvn-release.yml |
| **Risk Level**     | **MEDIUM**             |
| **GHE Compatible** | Yes (with config)      |

**Source Code Analysis:**

```javascript
// Found in dist/index.js:
exports.GH_API_URL = 'https://api.github.com';
exports.GH_GRAPHQL_URL = 'https://api.github.com';
const DEFAULT_GITHUB_SERVER_URL = 'https://github.com';

// BUT has configurable inputs:
githubApiUrl: core.getInput('github-api-url') || DEFAULT_GITHUB_API_URL,
changelogHost: core.getInput('changelog-host') || DEFAULT_GITHUB_SERVER_URL,
```

**Impact**: Uses hardcoded defaults BUT provides inputs to override.

**Required Configuration for GHE:**

```yaml
uses: ./vendor/googleapis/release-please-action
with:
  token: ${{ secrets.GITHUB_TOKEN }}
  github-api-url: ${{ env.GITHUB_API_URL }}
  github-graphql-url: ${{ env.GITHUB_GRAPHQL_URL }}
  changelog-host: ${{ env.GITHUB_SERVER_URL }}
```

---

#### 4. jreleaser/release-action

| Property           | Value                  |
| ------------------ | ---------------------- |
| **Used In**        | kotlin-mvn-release.yml |
| **Risk Level**     | **MEDIUM**             |
| **GHE Compatible** | Needs verification     |

**Source Code Analysis:**

This is a composite action that downloads and runs JReleaser CLI. JReleaser itself supports GitHub Enterprise via its configuration file.

**Potential Workaround:**

Configure JReleaser in `jreleaser.yml`:

```yaml
release:
  github:
    host: YOUR-SUBDOMAIN.ghe.com
    apiEndpoint: https://api.YOUR-SUBDOMAIN.ghe.com
```

Or via environment variables:

```yaml
env:
  JRELEASER_GITHUB_HOST: YOUR-SUBDOMAIN.ghe.com
  JRELEASER_GITHUB_API_ENDPOINT: https://api.YOUR-SUBDOMAIN.ghe.com
```

**Recommendation**: Test thoroughly; JReleaser CLI has enterprise support, but verify it receives correct URLs.

---

### Low Risk Actions (GHE Compatible)

These actions properly use GitHub-provided environment variables with fallbacks.

#### Official GitHub Actions

| Action                        | Source Code Verification                                           | GHE Support |
| ----------------------------- | ------------------------------------------------------------------ | ----------- |
| **actions/checkout**          | Uses `GITHUB_SERVER_URL`, `GITHUB_API_URL` env vars with fallbacks | ✅ Full     |
| **actions/cache**             | Uses `ACTIONS_CACHE_URL` set by runner                             | ✅ Full     |
| **actions/upload-artifact**   | Uses `ACTIONS_RESULTS_URL` set by runner                           | ✅ Full     |
| **actions/download-artifact** | Uses `ACTIONS_RESULTS_URL` set by runner                           | ✅ Full     |
| **actions/setup-java**        | Uses env vars; downloads from configurable sources                 | ✅ Full     |
| **actions/github-script**     | Uses `@actions/github` toolkit which respects `GITHUB_API_URL`     | ✅ Full     |

**Source Code Evidence (actions/checkout):**

```javascript
// dist/index.js:
this.apiUrl =
  (_a = process.env.GITHUB_API_URL) !== null && _a !== void 0
    ? _a
    : `https://api.github.com`;
this.serverUrl =
  (_b = process.env.GITHUB_SERVER_URL) !== null && _b !== void 0
    ? _b
    : `https://github.com`;
```

#### Docker Actions

| Action                         | Source Code Verification                          | GHE Support |
| ------------------------------ | ------------------------------------------------- | ----------- |
| **docker/login-action**        | Registry URL is configurable via `registry` input | ✅ Full     |
| **docker/build-push-action**   | Uses env vars; registry configurable              | ✅ Full     |
| **docker/metadata-action**     | Uses workflow context variables                   | ✅ Full     |
| **docker/setup-buildx-action** | Local Docker setup, no GitHub API calls           | ✅ Full     |

#### Linting & Quality Tools

| Action                       | Source Code Verification                           | GHE Support |
| ---------------------------- | -------------------------------------------------- | ----------- |
| **rhysd/actionlint**         | Local linting tool, no GitHub API calls            | ✅ Full     |
| **zizmorcore/zizmor-action** | Local security scanner, uploads to GHAS via runner | ✅ Full     |
| **korthout/backport-action** | Uses `@actions/github` toolkit (respects env vars) | ✅ Full     |

**Note on zizmor**: Requires GitHub Advanced Security (GHAS) to be enabled on your GHE instance for SARIF upload functionality.

---

## Migration Checklist

### Before Migration

- [ ] Verify GHE Cloud instance details (subdomain, region)
- [ ] Confirm GitHub Advanced Security availability
- [ ] Document current IP allowlists and network rules
- [ ] Identify enterprise-specific registry URL
- [ ] Mirror required Docker images to internal registry

### Required Changes

#### 1. Use cleanup-oci.yml Instead of cleanup-ghcr.yml

The `cleanup-ghcr.yml` workflow uses `dataaxiom/ghcr-cleanup-action` which has hardcoded `ghcr.io` URLs and cannot work with GHE data residency. Options:

- **Recommended:** Use `cleanup-oci.yml` instead, which uses `regctl` and supports any OCI-compliant registry
- Use GitHub's built-in package retention policies
- Fork and modify `dataaxiom/ghcr-cleanup-action` to use `GITHUB_API_URL`

#### 2. Mirror Docker Images for wagoid/commitlint-github-action

```bash
# Pull from Docker Hub
docker pull wagoid/commitlint-github-action:6.2.1

# Tag for internal registry
docker tag wagoid/commitlint-github-action:6.2.1 \
  your-registry.company.com/commitlint-github-action:6.2.1

# Push to internal registry
docker push your-registry.company.com/commitlint-github-action:6.2.1
```

Then fork the action and update `action.yml`:

```yaml
runs:
  using: docker
  image: docker://your-registry.company.com/commitlint-github-action:6.2.1
```

#### 3. Configure release-please-action

Add explicit URL inputs:

```yaml
uses: ./vendor/googleapis/release-please-action
with:
  token: ${{ secrets.GITHUB_TOKEN }}
  github-api-url: ${{ env.GITHUB_API_URL }}
  github-graphql-url: ${{ env.GITHUB_GRAPHQL_URL }}
  changelog-host: ${{ env.GITHUB_SERVER_URL }}
```

#### 4. Configure JReleaser

Add to `jreleaser.yml` or use environment variables for GHE endpoints.

### Post-Migration Verification

- [ ] CI workflow builds successfully
- [ ] Docker images push to GHE registry
- [ ] Release Please creates PRs correctly
- [ ] Backport automation functions
- [ ] Security scanning (zizmor) uploads to GHAS
- [ ] Commitlint runs with mirrored image

---

## Recommendations

### Immediate Actions

1. **Use cleanup-oci.yml** instead of cleanup-ghcr.yml for GHE data residency, or fork dataaxiom/ghcr-cleanup-action with fixes
2. **Mirror Docker images** for commitlint to internal registry
3. **Add explicit URL inputs** to release-please configuration
4. **Create organization-level variables** for GHE Cloud URLs

### Long-term Considerations

1. **Monitor action updates** for improved enterprise support
2. **Contribute upstream** - submit PRs to action maintainers for GHE support
3. **Consider alternatives** for problematic actions:
   - `dataaxiom/ghcr-cleanup-action` → GitHub retention policies or `gh` CLI script
   - `wagoid/commitlint-github-action` → Run commitlint directly via npm

---

## Appendix: Environment Variables Reference

### Automatically Set by GHE Runners

| Variable              | Description                        | Example                                    |
| --------------------- | ---------------------------------- | ------------------------------------------ |
| `GITHUB_SERVER_URL`   | Base URL for GitHub web interface  | `https://github.mycompany.com`             |
| `GITHUB_API_URL`      | REST API endpoint                  | `https://github.mycompany.com/api/v3`      |
| `GITHUB_GRAPHQL_URL`  | GraphQL API endpoint               | `https://github.mycompany.com/api/graphql` |
| `ACTIONS_CACHE_URL`   | Actions cache service endpoint     | Set by runner                              |
| `ACTIONS_RESULTS_URL` | Artifacts/results service endpoint | Set by runner                              |

### Well-Behaved Action Pattern

Actions should follow this pattern for GHE compatibility:

```javascript
const apiUrl = process.env.GITHUB_API_URL || "https://api.github.com";
const serverUrl = process.env.GITHUB_SERVER_URL || "https://github.com";
```

---

## References

- [GitHub Enterprise Cloud with Data Residency](https://docs.github.com/en/enterprise-cloud@latest/admin/data-residency/about-github-enterprise-cloud-with-data-residency)
- [Network Details for GHE.com](https://docs.github.com/en/enterprise-cloud@latest/admin/data-residency/network-details-for-ghecom)
- [GitHub Actions Variables](https://docs.github.com/en/enterprise-cloud@latest/actions/reference/workflows-and-actions/variables)
- [Using Actions in Enterprise](https://docs.github.com/en/enterprise-cloud@latest/admin/github-actions/getting-started-with-github-actions-for-your-enterprise/introducing-github-actions-to-your-enterprise)

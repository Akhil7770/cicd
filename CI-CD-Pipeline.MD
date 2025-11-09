# Cost Estimator Calc Service - CI/CD Pipeline Documentation

## 📋 Table of Contents

1. [Overview](#overview)
2. [Pipeline Architecture](#pipeline-architecture)
3. [CI Pipeline (Build & Test)](#ci-pipeline-build--test)
4. [CD Pipeline (Deployment)](#cd-pipeline-deployment)
5. [Docker Image Versioning](#docker-image-versioning)
6. [Environment Variable Injection](#environment-variable-injection)
7. [Complete Workflow Examples](#complete-workflow-examples)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Best Practices](#best-practices)

---

## Overview

The Cost Estimator Calc Service uses a **two-pipeline approach** for CI/CD:

- **CI Pipeline** (`ci.yaml`, `ci_pdev.yaml`) - Builds, tests, scans, and publishes Docker images to Artifactory
- **CD Pipeline** (`cd.yaml`) - Injects environment variables and deploys to Kubernetes

### Key Technologies

- **Language:** Python 3.10
- **Container Registry:** JFrog Artifactory (`cvsh.jfrog.io`)
- **Orchestration:** Kubernetes (GKE)
- **Deployment:** Helm
- **Version Control:** Git with Semantic Versioning
- **Security Scanning:** Snyk (SCA, SAST, Container Scanning)

---

## Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DEVELOPMENT FLOW                            │
└─────────────────────────────────────────────────────────────────────┘

Developer Commits Code
         ↓
   ┌─────────────┐
   │  Git Push   │
   │  to Branch  │
   └─────┬───────┘
         ↓
   ┌─────────────────────────────────────────────────────────────────┐
   │                     CI PIPELINE                                  │
   │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────────┐ │
   │  │  Setup   │→ │  Tests   │→ │  Security    │→ │   Build    │ │
   │  │          │  │  & Lint  │  │  Scans       │  │  & Publish │ │
   │  └──────────┘  └──────────┘  └──────────────┘  └─────┬──────┘ │
   │                                                         │        │
   │  Outputs:                                              │        │
   │  - Image Tag: 1.0.3 or abc123def...                   │        │
   │  - Image Location: cvsh.jfrog.io/...                  │        │
   │  - Git SHA: abc123def456...                           │        │
   └─────────────────────────────────────────────────────┬─┘        │
                                                          │          │
                                       Docker Image Pushed to        │
                                       Artifactory                   │
                                                          ↓          │
         ┌────────────────────────────────────────────────────────────┘
         │ (Manual or Auto-trigger)
         ↓
   ┌─────────────────────────────────────────────────────────────────┐
   │                     CD PIPELINE                                  │
   │  ┌──────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────┐│
   │  │ Detect   │→ │   Inject     │→ │   Commit     │→ │ Deploy  ││
   │  │ Changes  │  │   Env Vars   │  │   v1.yaml    │  │ to K8s  ││
   │  └──────────┘  └──────────────┘  └──────────────┘  └─────────┘│
   │                                                                  │
   │  - Loads 27 env vars from GitHub Environment                    │
   │  - Updates v1.yaml with configuration                           │
   │  - Deploys using Helm                                           │
   └─────────────────────────────────────────────────────────────────┘
                                  ↓
                        Application Running in
                     Kubernetes (pdev or prod)
```

---

## CI Pipeline (Build & Test)

### Workflows

#### 1. **ci_pdev.yaml** - Pdev Branch CI
- **Triggers:** Push to `pdev` branch (excluding config changes)
- **Purpose:** Build and test code for development environment
- **Reusable Workflow:** `python_docker.yaml@latest`

#### 2. **ci.yaml** - Master Branch CI  
- **Triggers:** Push to `master` branch (excluding config changes)
- **Purpose:** Build and test code for production environment
- **Reusable Workflow:** `python_docker.yaml@latest`

### CI Pipeline Jobs

```yaml
Jobs:
  1. setup              # Setup environment, install dependencies
  2. test               # Run unit tests, SonarQube scan
  3. lint               # Run linters (if configured)
  4. sast-scan          # Security scanning (Snyk - code vulnerabilities)
  5. sca-scan           # Dependency scanning (Snyk - package vulnerabilities)
  6. build-scan-publish # Build Docker image, scan container, push to Artifactory
  7. sbom               # Generate Software Bill of Materials
  8. cd-values-update   # (Optional) Auto-update deployment config
```

### Current Configuration

**File: `.github/workflows/ci_pdev.yaml`**
```yaml
name: Run Python Docker Pipelines For Pdev
on:
  push:
    branches: [pdev]
    paths-ignore:
      - 'deploy-configs/**'
      - 'README.md'
      - '.github/workflows/cd.yaml'

jobs:
  python_docker_build:
    uses: cvs-health-source-code/gha_workflow_actions/.github/workflows/python_docker.yaml@latest
    with:
      PYTHON_VERSION: "3.10"
      PYTHON_DEPS_COMMAND: "pip install pipenv==2024.4.1; pipenv lock; pipenv install --system --deploy --dev"
      DOCKERFILE_PATH: "./Dockerfile"
      EXTRA_PARAMETERS_SNYK: "--file=Pipfile"
      SKIP_UNRESOLVED_SNYK: "true"
    secrets: inherit
```

### What CI Pipeline Does

1. **Checkout Code** - Gets latest code from repository
2. **Install Dependencies** - Installs Python packages via pipenv
3. **Run Tests** - Executes pytest test suite
4. **Security Scans** - Runs Snyk scans (SAST, SCA)
5. **Determine Version** - Calculates image tag (see [Docker Image Versioning](#docker-image-versioning))
6. **Build Docker Image** - Creates containerized application
7. **Container Scan** - Scans Docker image for vulnerabilities
8. **Push to Artifactory** - Publishes image to `cvsh.jfrog.io`
9. **Generate SBOM** - Creates Software Bill of Materials

### CI Outputs

After successful CI build:

```yaml
outputs:
  imageid: "sha256:9a8b7c6d5e4f..."
  image-version: "1.0.3"  # or git SHA
  image-name: "cost-estimator-calc-service"
  image-location: "cvsh.jfrog.io/cvsdigital-docker/ghemu/cvs-health-source-code/cost-estimator-calc-service"
  git-sha: "abc123def456..."
  digest: "sha256:7f5e2a1b9c8d..."
```

---

## CD Pipeline (Deployment)

### Workflow: cd.yaml

**Triggers:**
- Push to `master` or `pdev` branches
- Changes to `**/values/v1.yaml` files
- Manual workflow dispatch

### CD Pipeline Jobs

```yaml
Jobs:
  1. detect-changes        # Detect which environment config changed
  2. inject-github-env-vars # Inject 27 environment variables
  3. deploy                # Deploy to Kubernetes using Helm
```

### Environment Detection

The CD pipeline detects which environment to deploy to:

```yaml
Monitors:
  - deploy-configs/hcb-edmlprvdraaece-prod/values/v1.yaml → Deploy to PROD
  - deploy-configs/hcb-edmlprvdraaece-pdev/values/v1.yaml → Deploy to PDEV
```

### What CD Pipeline Does

#### Job 1: detect-changes
```bash
# Detects which v1.yaml file changed
Changed: hcb-edmlprvdraaece-pdev/values/v1.yaml
Output: [{"ENV_NAME":"hcb-edmlprvdraaece-pdev","HELM_FILE_VERSION":"v1"}]
```

#### Job 2: inject-github-env-vars
```bash
# Loads GitHub Environment: hcb-edmlprvdraaece-pdev
# Reads 27 environment variables
# Creates injection template with all variables
# Replaces envVars section in v1.yaml
# Commits updated v1.yaml back to repository
```

#### Job 3: deploy
```bash
# Calls reusable Helm deployment workflow
# Passes environment name and configuration
# Deploys to GKE cluster
# Creates RFC ticket (for prod)
```

---

## Docker Image Versioning

### Version Strategy

The CI pipeline uses **Semantic Versioning** with **Git SHA fallback**.

### How Version is Determined

```
┌───────────────────────────────────────────────────────────────┐
│                 VERSION DETERMINATION FLOW                     │
└───────────────────────────────────────────────────────────────┘

Step 1: Check if IMAGE_TAG_VERSION is set
        ↓
        NO → Continue to Step 2

Step 2: Run Semantic Release
        ↓
        Analyzes commit messages since last tag
        ↓
        ┌─────────────────────────────────────────────┐
        │ Commit Type      → Version Bump              │
        ├─────────────────────────────────────────────┤
        │ fix: ...         → Patch (1.0.0 → 1.0.1)   │
        │ feat: ...        → Minor (1.0.0 → 1.1.0)   │
        │ feat!: ...       → Major (1.0.0 → 2.0.0)   │
        │ BREAKING CHANGE  → Major (1.0.0 → 2.0.0)   │
        │ chore/docs/etc   → No bump                  │
        └─────────────────────────────────────────────┘
        ↓
        Creates release.version file with new version

Step 3: Read release.version
        ↓
        File has content?
        ├─ YES → Use semantic version (e.g., 1.0.3)
        └─ NO  → Fallback to Git SHA (abc123def...)

Result: RELEASE_TAG = "1.0.3" or "abc123def456..."
```

### Semantic Versioning Examples

#### Example 1: Patch Release (Bug Fix)

```bash
# Last tag: v1.0.2
# Commit: "fix: resolve database timeout issue"

Semantic Release Output:
  - Detected: fix: → PATCH bump
  - New version: 1.0.3
  - Creates tag: v1.0.3
  - Image: cost-estimator-calc-service:1.0.3
```

#### Example 2: Minor Release (New Feature)

```bash
# Last tag: v1.0.3
# Commit: "feat: add support for multiple payment methods"

Semantic Release Output:
  - Detected: feat: → MINOR bump
  - New version: 1.1.0
  - Creates tag: v1.1.0
  - Image: cost-estimator-calc-service:1.1.0
```

#### Example 3: Major Release (Breaking Change)

```bash
# Last tag: v1.1.0
# Commit: "feat!: redesign API structure\n\nBREAKING CHANGE: endpoints now use /v2/"

Semantic Release Output:
  - Detected: BREAKING CHANGE → MAJOR bump
  - New version: 2.0.0
  - Creates tag: v2.0.0
  - Image: cost-estimator-calc-service:2.0.0
```

#### Example 4: No Version Bump (Non-semantic Commit)

```bash
# Last tag: v1.1.0
# Commit: "update README and add comments"

Semantic Release Output:
  - Detected: No semantic type
  - No version bump
  - No new tag created
  - Fallback to Git SHA
  - Image: cost-estimator-calc-service:abc123def456789...
```

### Commit Message Format

For proper semantic versioning, use this format:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**Types:**
- `feat:` - New feature (minor version bump)
- `fix:` - Bug fix (patch version bump)
- `docs:` - Documentation only
- `style:` - Code style changes
- `refactor:` - Code refactoring
- `perf:` - Performance improvements
- `test:` - Adding tests
- `chore:` - Maintenance tasks

**Examples:**
```bash
feat: add user authentication
fix: resolve memory leak in cache
docs: update API documentation
chore: upgrade dependencies
feat!: redesign database schema  # Breaking change (!)
```

### Version Outputs

**Semantic Version:**
```yaml
Image Tag: 1.0.3
Full Image: cvsh.jfrog.io/cvsdigital-docker/ghemu/cvs-health-source-code/cost-estimator-calc-service:1.0.3
Git SHA: abc123def456789012345678901234567890abcd
```

**Git SHA Fallback:**
```yaml
Image Tag: abc123def456789012345678901234567890abcd
Full Image: cvsh.jfrog.io/cvsdigital-docker/ghemu/.../cost-estimator-calc-service:abc123def456789012345678901234567890abcd
Git SHA: abc123def456789012345678901234567890abcd
```

---

## Environment Variable Injection

### Overview

The CD pipeline automatically injects **27 environment variables** from GitHub Environments into the Kubernetes deployment configuration.

### GitHub Environments

Two separate GitHub Environments store environment-specific secrets and variables:

1. **hcb-edmlprvdraaece-pdev** - Development environment
2. **hcb-edmlprvdraaece-prod** - Production environment

### Variables Injected (27 Total)

#### Core Environment Variables (4)
1. `ENVIRONMENT` - Environment name (pdev/prod)
2. `ENV_NAME` - Environment identifier (PDEV/PROD)
3. `CLIENT_ID` - OAuth client ID
4. `CLIENT_SECRET` - OAuth client secret (via K8s secretRef)

#### Database Configuration (6)
5. `SPANNER_PROJECT_ID` - GCP project ID
6. `SPANNER_INSTANCE_ID` - Cloud Spanner instance
7. `SPANNER_DATABASE_ID` - Database name
8. `BENEFITS_URL` - Benefits service API endpoint
9. `ACCUMULATOR_URL` - Accumulator service API endpoint
10. `TOKEN_URL` - Token service endpoint

#### OpenTelemetry - Exporters (6)
11. `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` - Metrics collector endpoint
12. `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` - Traces collector endpoint
13. `OTEL_METRICS_EXPORTER` - Metrics exporter type
14. `OTEL_TRACES_EXPORTER` - Traces exporter type
15. `OTEL_LOGS_EXPORTER` - Logs exporter type
16. `OTEL_EXPORTER_OTLP_PROTOCOL` - Protocol (grpc/http)

#### OpenTelemetry - Configuration (5)
17. `OTEL_TRACES_SAMPLER` - Trace sampling strategy
18. `OTEL_PROPAGATORS` - Context propagation format
19. `OTEL_SERVICE_NAME` - Service name for telemetry
20. `OTEL_RESOURCE_ATTRIBUTES` - Resource attributes
21. `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS` - Disabled instrumentations

#### OpenTelemetry - Instrumentation (6)
22. `OTEL_INSTRUMENTATION_HTTP_ENABLED` - HTTP instrumentation
23. `OTEL_INSTRUMENTATION_FASTAPI_ENABLED` - FastAPI instrumentation
24. `OTEL_INSTRUMENTATION_HTTPX_ENABLED` - HTTPX instrumentation
25. `OTEL_INSTRUMENTATION_REQUESTS_ENABLED` - Requests instrumentation
26. `OTEL_INSTRUMENTATION_AIOHTTP_CLIENT_ENABLED` - AIOHTTP instrumentation
27. `OTEL_INSTRUMENTATION_HTTP_SERVER_ENABLED` - HTTP server instrumentation

### Injection Process

#### Before Injection (v1.yaml)

```yaml
# deploy-configs/hcb-edmlprvdraaece-pdev/values/v1.yaml

envVars: |
  - name: TEST_KEY
    value: TEST_VALUE
  - name: ENABLE_TRACING
    value: 'true'
  - name: ENV
    value: pdev
```

#### After Injection (v1.yaml)

```yaml
# environment variables for app - injected from GitHub Environment
envVars: |
  - name: ENVIRONMENT
    value: "pdev"
  - name: SPANNER_PROJECT_ID
    value: "edp-pdev-hcb-gke"
  - name: ENV_NAME
    value: "PDEV"
  - name: SPANNER_INSTANCE_ID
    value: "pdev-spanner-instance"
  - name: SPANNER_DATABASE_ID
    value: "cost-estimator-db"
  - name: BENEFITS_URL
    value: "https://qaapi01.int.aetna.com/hcb/benefits"
  - name: ACCUMULATOR_URL
    value: "https://qaapi01.int.aetna.com/hcb/accumulator"
  - name: TOKEN_URL
    value: "https://auth-pdev.cvshealth.com/oauth/token"
  - name: CLIENT_ID
    value: "5ed6497d44e6a39631f45f7a9971d..."
  - name: CLIENT_SECRET
    valueFrom:
      secretKeyRef:
        name: anbc-hcb-cet-pdev
        key: CLIENT_SECRET
  - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
    value: "https://otelc-gmp-collector-grpc.hcb-telemetry.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
    value: "https://otelc-traces-collector-grpc.hcb-telemetry.svc.cluster.local:4317"
  # ... (all 27 variables)
```

### How to Update Environment Variables

#### Step 1: Update GitHub Environment

1. Go to: `https://github.com/YOUR_ORG/cost-estimator-calc-service/settings/environments`
2. Select: `hcb-edmlprvdraaece-pdev` or `hcb-edmlprvdraaece-prod`
3. Update variable values
4. Save changes

#### Step 2: Trigger CD Pipeline

**Option 1: Make a small change to v1.yaml**
```bash
git checkout pdev
# Edit any value in v1.yaml (e.g., update dockerImageTag)
git add deploy-configs/hcb-edmlprvdraaece-pdev/values/v1.yaml
git commit -m "chore: trigger env var injection"
git push origin pdev
```

**Option 2: Manual workflow dispatch**
- Go to Actions → App Deployment → Run workflow

#### Step 3: Verify Injection

Check the commit created by `github-actions[bot]`:
```
chore: inject environment variables from GitHub Environment for hcb-edmlprvdraaece-pdev
```

---

## Complete Workflow Examples

### Example 1: Full Development Workflow (Pdev)

#### Step 1: Developer Makes Code Change

```bash
# Developer working on pdev branch
git checkout pdev
git pull origin pdev

# Make code changes
vim app/services/calculation_service.py

# Commit with semantic message
git add .
git commit -m "fix: resolve division by zero error in premium calculation"
git push origin pdev
```

#### Step 2: CI Pipeline Executes

```bash
Workflow: ci_pdev.yaml
Branch: pdev
Commit: abc123def456...
Message: "fix: resolve division by zero error in premium calculation"

Job: setup
  ✓ Checkout code
  ✓ Install Python 3.10
  ✓ Install dependencies (pipenv)
  ✓ Cache dependencies

Job: test (parallel)
  ✓ Run pytest
  ✓ SonarQube scan
  Result: All tests passed

Job: sast-scan (parallel)
  ✓ Snyk code scan
  Result: No high vulnerabilities

Job: sca-scan (parallel)
  ✓ Snyk dependency scan
  Result: No high vulnerabilities

Job: build-scan-publish
  ✓ Semantic Release
    - Detected: fix: → PATCH bump
    - Last tag: v1.0.2
    - New tag: v1.0.3
    - Wrote: release.version = "1.0.3"
  
  ✓ Docker Build
    - Image: cvsh.jfrog.io/.../cost-estimator-calc-service:1.0.3
    - Image ID: sha256:9a8b7c6d5e4f...
  
  ✓ Container Scan
    - Snyk scan passed
  
  ✓ Docker Push
    - Pushed to Artifactory
    - Digest: sha256:7f5e2a1b9c8d...

Job: sbom
  ✓ Generate Software Bill of Materials
  ✓ Upload to Artifactory

✓ CI Pipeline Complete!
  Image: cost-estimator-calc-service:1.0.3
  Location: cvsh.jfrog.io/cvsdigital-docker/ghemu/.../cost-estimator-calc-service:1.0.3
```

#### Step 3: Update v1.yaml with New Image Tag

```bash
# Update deployment config
git checkout pdev
vim deploy-configs/hcb-edmlprvdraaece-pdev/values/v1.yaml

# Change:
- dockerImageTag: 1.0.2
+ dockerImageTag: 1.0.3

git add deploy-configs/hcb-edmlprvdraaece-pdev/values/v1.yaml
git commit -m "chore: update image tag to 1.0.3"
git push origin pdev
```

#### Step 4: CD Pipeline Executes

```bash
Workflow: cd.yaml
Trigger: v1.yaml changed
Branch: pdev

Job: detect-changes
  ✓ Detected: hcb-edmlprvdraaece-pdev/values/v1.yaml
  Output: {"ENV_NAME":"hcb-edmlprvdraaece-pdev","HELM_FILE_VERSION":"v1"}

Job: inject-github-env-vars
  ✓ Load GitHub Environment: hcb-edmlprvdraaece-pdev
  ✓ Read 27 environment variables
  ✓ Create injection template
  ✓ Replace envVars section in v1.yaml
  ✓ Commit updated v1.yaml
    Commit: "chore: inject environment variables from GitHub Environment for hcb-edmlprvdraaece-pdev"

Job: deploy
  ✓ Call helm-deploy reusable workflow
  ✓ Deploy to GKE cluster: pdev-hcb-gke
  ✓ Namespace: hcb-edmlprvdraaece-pdev
  ✓ Deployment successful

✓ CD Pipeline Complete!
  Environment: pdev
  Image: cost-estimator-calc-service:1.0.3
  Status: Running
```

#### Step 5: Verify Deployment

```bash
# Check pod is running
kubectl get pods -n hcb-edmlprvdraaece-pdev

NAME                                        READY   STATUS    RESTARTS   AGE
cost-estimator-calc-service-7d9f8b-xyz12   1/1     Running   0          2m

# Check environment variables in pod
kubectl exec -n hcb-edmlprvdraaece-pdev cost-estimator-calc-service-7d9f8b-xyz12 -- env | grep SPANNER

SPANNER_PROJECT_ID=edp-pdev-hcb-gke
SPANNER_INSTANCE_ID=pdev-spanner-instance
SPANNER_DATABASE_ID=cost-estimator-db

# Check image version
kubectl get deployment -n hcb-edmlprvdraaece-pdev cost-estimator-calc-service -o jsonpath='{.spec.template.spec.containers[0].image}'

cvsh.jfrog.io/.../cost-estimator-calc-service:1.0.3
```

### Example 2: No Semantic Commit (Git SHA Tag)

#### Developer Commit

```bash
git commit -m "update configuration and add comments"
git push origin pdev
```

#### CI Execution

```bash
Job: build-scan-publish
  ✓ Semantic Release
    - No semantic commit type found
    - No version bump
    - release.version = empty
  
  ✓ Assign Variables
    - release.version is empty
    - Fallback to Git SHA
    - RELEASE_TAG = "xyz789abc123456789012345678901234567890"
  
  ✓ Docker Build
    - Image: cost-estimator-calc-service:xyz789abc123456789012345678901234567890
  
  ✓ Docker Push
    - Pushed with SHA tag

Result:
  Image: cost-estimator-calc-service:xyz789abc123456789012345678901234567890
```

---

## Troubleshooting Guide

### Issue 1: CI Not Building Images

**Symptom:**
```
CI workflow completes but no new image in Artifactory
```

**Possible Causes:**
1. Build failed silently
2. Artifactory credentials expired
3. Network issues

**Solution:**
```bash
# Check workflow logs
1. Go to Actions → Latest CI run
2. Check "build-scan-publish" job
3. Look for "Docker Push" step
4. Verify credentials:
   - ARTIFACTORY_USERNAME
   - ARTIFACTORY_TOKEN
```

### Issue 2: Wrong Image Being Deployed

**Symptom:**
```yaml
# v1.yaml shows:
dockerImageTag: 1.0.3

# But Artifactory has:
- 1.0.4
- 1.0.5
- abc123def...

# Old version being deployed!
```

**Root Cause:**
v1.yaml not updated with latest image tag

**Solution:**
```bash
# Option 1: Manual update
1. Check Artifactory for latest tag
2. Update v1.yaml:
   dockerImageTag: 1.0.5
3. Commit and push

# Option 2: Enable auto-update in ci_pdev.yaml
Add to workflow:
  CD_DEPLOY_FILE: "deploy-configs/hcb-edmlprvdraaece-pdev/values/v1.yaml"
```

### Issue 3: Environment Variables Not Injected

**Symptom:**
```bash
kubectl exec pod -- env | grep SPANNER
# No output - variables missing!
```

**Possible Causes:**
1. CD pipeline didn't run
2. Injection job failed
3. GitHub Environment not configured

**Solution:**
```bash
# Check CD workflow logs
1. Go to Actions → App Deployment
2. Check "inject-github-env-vars" job
3. Verify:
   - GitHub Environment exists: hcb-edmlprvdraaece-pdev
   - All 27 variables are set
   - Injection step succeeded

# Trigger manually:
git checkout pdev
# Make small change to v1.yaml
git commit -m "chore: trigger injection"
git push origin pdev
```

### Issue 4: Semantic Versioning Not Working

**Symptom:**
```
All images tagged with Git SHA instead of versions
```

**Root Cause:**
Missing semantic-release configuration

**Solution:**
```bash
# Check for .releaserc file
ls -la | grep releaserc

# If missing, create .releaserc:
cat > .releaserc << 'EOF'
{
  "branches": ["master", "main", "pdev"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/github"
  ]
}
EOF

git add .releaserc
git commit -m "chore: add semantic-release configuration"
git push origin pdev
```

### Issue 5: CD Pipeline Validation Error

**Symptom:**
```
Invalid workflow file: .github/workflows/cd.yaml#L1
(Line: 158, Col: 5): Unexpected value 'environment'
```

**Root Cause:**
Cannot use `environment:` keyword in reusable workflow call

**Solution:**
```yaml
# WRONG:
deploy:
  uses: .../helm-deploy.yaml@v1
  environment: ${{ matrix.environment.ENV_NAME }}  # ❌ Not allowed!

# CORRECT:
deploy:
  uses: .../helm-deploy.yaml@v1
  with:
    environment: ${{ matrix.environment.ENV_NAME }}  # ✅ Pass as parameter
```

### Issue 6: Cannot Find Image in Artifactory

**Symptom:**
```
Error: Failed to pull image "cost-estimator-calc-service:1.0.3": not found
```

**Solution:**
```bash
# Check if image exists
1. Log into Artifactory: https://cvsh.jfrog.io
2. Navigate to: cvsdigital-docker/ghemu/cvs-health-source-code/cost-estimator-calc-service
3. Check available tags

# Verify image path in v1.yaml:
dockerHub: cvsh.jfrog.io
dockerImageGroupName: cvsdigital-docker/ghemu/cvs-health-source-code/cost-estimator-calc-service
dockerImageName: cost-estimator-calc-service
dockerImageTag: 1.0.3  # Must match tag in Artifactory
```

---

## Best Practices

### 1. Commit Message Convention

**Always use semantic commit messages:**

```bash
# Good commits:
git commit -m "feat: add payment retry logic"
git commit -m "fix: resolve null pointer in calculator"
git commit -m "docs: update API documentation"

# Bad commits:
git commit -m "update code"
git commit -m "fixes"
git commit -m "WIP"
```

### 2. Version Management

**For Production:**
- Always use semantic versions (not Git SHA)
- Tag releases properly: v1.0.0, v1.1.0, v2.0.0
- Keep versions in sync between v1.yaml and Artifactory

**For Development:**
- Git SHA tags are acceptable for testing
- Update v1.yaml after each CI build
- Consider enabling auto-update feature

### 3. Environment Variables

**DO:**
- Store all environment-specific values in GitHub Environments
- Keep secrets out of v1.yaml
- Document all variables and their purposes

**DON'T:**
- Hardcode secrets in code or config files
- Use same values for pdev and prod
- Commit sensitive data to repository

### 4. Deployment Safety

**Pre-Production (Pdev):**
- Test all changes in pdev first
- Verify environment variables are correct
- Check application logs after deployment

**Production (Prod):**
- Always deploy tested code from pdev
- Create RFC tickets for audit trail
- Have rollback plan ready
- Monitor application after deployment

### 5. Security

**Image Scanning:**
- Always run security scans before deployment
- Fix high/critical vulnerabilities immediately
- Review scan reports regularly

**Secrets Management:**
- Use Kubernetes secrets for sensitive data
- Rotate credentials regularly
- Never commit secrets to git

### 6. Monitoring

**After Deployment:**
```bash
# Check pod status
kubectl get pods -n <namespace>

# Check logs
kubectl logs -f <pod-name> -n <namespace>

# Check metrics
# Monitor application performance in Grafana/Prometheus

# Verify endpoints
curl https://your-app-endpoint/health
```

---

## Quick Reference

### Common Commands

```bash
# Check current branch
git branch --show-current

# Check git tags
git tag --list | tail -10

# Trigger CI (commit code changes)
git add .
git commit -m "feat: add new feature"
git push origin pdev

# Trigger CD (update v1.yaml)
git add deploy-configs/hcb-edmlprvdraaece-pdev/values/v1.yaml
git commit -m "chore: update config"
git push origin pdev

# Check deployed image
kubectl get deployment -n hcb-edmlprvdraaece-pdev cost-estimator-calc-service -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check environment variables
kubectl exec -n hcb-edmlprvdraaece-pdev <pod-name> -- env | grep -E "^(SPANNER|OTEL)"

# Check pods
kubectl get pods -n hcb-edmlprvdraaece-pdev -l app=cost-estimator-calc-service

# View logs
kubectl logs -f <pod-name> -n hcb-edmlprvdraaece-pdev
```

### File Locations

```
Repository Structure:
├── .github/workflows/
│   ├── ci.yaml              # CI for master branch
│   ├── ci_pdev.yaml         # CI for pdev branch
│   └── cd.yaml              # CD for deployments
├── deploy-configs/
│   ├── hcb-edmlprvdraaece-pdev/
│   │   └── values/v1.yaml   # Pdev deployment config
│   └── hcb-edmlprvdraaece-prod/
│       └── values/v1.yaml   # Prod deployment config
├── app/                     # Application code
├── tests/                   # Test files
├── Dockerfile              # Container definition
├── Pipfile                 # Python dependencies
└── .releaserc              # Semantic release config
```

### Important URLs

- **Artifactory:** https://cvsh.jfrog.io
- **GitHub Actions:** https://github.com/YOUR_ORG/cost-estimator-calc-service/actions
- **GitHub Environments:** https://github.com/YOUR_ORG/cost-estimator-calc-service/settings/environments

---

## Support

For issues or questions:
1. Check this documentation
2. Review workflow logs in GitHub Actions
3. Check Artifactory for image availability
4. Contact DevOps team
5. Open a GitHub issue

---

**Last Updated:** 2025-11-09
**Version:** 1.0.0
**Maintained By:** DevOps Team


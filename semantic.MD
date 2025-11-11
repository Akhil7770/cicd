# Branch Check Action - Detailed Documentation

## Table of Contents
- [Overview](#overview)
- [Purpose and Benefits](#purpose-and-benefits)
- [How It Works](#how-it-works)
- [Configuration Parameters](#configuration-parameters)
- [Validation Process](#validation-process)
- [Semantic Versioning Patterns](#semantic-versioning-patterns)
- [Real-World Examples](#real-world-examples)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

---

## Overview

The `branch_check` action is a GitHub Actions workflow component that implements **branch-based artifact publishing controls**. It determines whether the current branch is authorized to push/publish artifacts (Docker images, packages, binaries, etc.) to artifact repositories like Artifactory, JFrog, or container registries.

### Why This Matters

Without proper controls:
- ❌ Artifacts from feature branches could pollute production repositories
- ❌ Storage costs increase exponentially with uncontrolled publishing
- ❌ Version confusion when tracking production-ready releases
- ❌ Security risks from untrusted branch deployments

With `branch_check`:
- ✅ Only approved branches can publish artifacts
- ✅ Semantic versioning support for automated releases
- ✅ Cost control through branch limits
- ✅ Clear audit trail of what gets published

---

## Purpose and Benefits

### Key Benefits

1. **Governance & Security**
   - Enforce organizational policies on artifact publishing
   - Prevent unauthorized branches from deploying to production repositories
   - Maintain clear separation between development and production artifacts

2. **Cost Management**
   - Limit the number of branches that can publish artifacts
   - Reduce storage costs in artifact repositories
   - Prevent repository pollution from temporary branches

3. **Version Control**
   - Support semantic versioning patterns for automated releases
   - Enable hotfix and patch workflows
   - Maintain consistent versioning across projects

4. **Flexibility**
   - Support for wildcard patterns
   - Semantic version matching
   - Override via commit message keywords
   - Configurable branch count limits

---

## How It Works

### High-Level Flow

```
┌─────────────────────────────────────┐
│   Trigger: Git Push/Pull Request    │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Fetch Branch Configuration         │
│  - Allowed branches                 │
│  - Semantic patterns                │
│  - Branch count limit               │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Validation Checks (in order):      │
│  1. Branch count limit              │
│  2. Semantic version patterns       │
│  3. Exact branch name match         │
│  4. Wildcard pattern match          │
│  5. Default branch check            │
│  6. Commit message override         │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│  Output: BRANCH_PUSH_ENABLED        │
│  - true: Allow artifact push        │
│  - false: Block artifact push       │
└─────────────────────────────────────┘
```

### Validation Order

The action evaluates conditions in this specific order (stops at first match):

1. **Branch Count Validation** - Ensures not too many branches are configured
2. **Semantic Version Patterns** - Checks version-like branch names
3. **Exact Branch Match** - Direct match against allowed branches
4. **Wildcard Patterns** - Pattern matching with wildcards
5. **Default Branch Check** - Special handling for main/master
6. **Commit Message Override** - `[publish]` or `[push]` keywords

---

## Configuration Parameters

### Input Parameters

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `push_on_branches` | String | Yes | Comma-separated list of branches allowed to push | `pdev,prod,staging` |
| `default_branch` | String | Yes | The repository's default branch | `master` or `main` |
| `current_branch` | String | Yes | The branch triggering the workflow | `pdev` |
| `org` | String | Yes | GitHub organization name | `cvs-health-source-code` |
| `ghtoken` | String | Yes | GitHub token for API access | `${{ secrets.GITHUB_TOKEN }}` |
| `commit_message` | String | No | Commit message (for override detection) | `feat: add feature [publish]` |
| `branch_pattern` | String | No | Additional wildcard patterns | `feature/*,dev/*` |
| `tag_pattern` | String | No | Semantic version patterns | `v*,hotfix,release/*` |

### Environment Variables

The action uses these organization-level variables:

- **`ALLOWED_BRANCHES`**: Comma-separated list of allowed branches
- **`BRANCH_COUNT`**: Maximum number of branches allowed to push (default: 5)

### Output

- **`BRANCH_PUSH_ENABLED`**: Boolean (`true`/`false`) indicating if push is allowed

---

## Validation Process

### 1. Branch Count Limit Check

**Purpose**: Prevent repository pollution by limiting publishing branches.

```bash
# Configuration
ALLOWED_BRANCHES="pdev,prod,staging,hotfix,release"
BRANCH_COUNT=5  # Org-level setting

# Count check
branch_count = count(ALLOWED_BRANCHES.split(','))  # = 5

if branch_count > BRANCH_COUNT:
    ❌ FAIL: "Too many branches configured"
    PUSH_ENABLED = false
else:
    ✅ PASS: Continue validation
```

**Why This Matters**:
- Prevents teams from adding unlimited branches to the allowed list
- Forces intentional decisions about what can be published
- Reduces storage costs and version confusion

**Example Output**:
```
❌ ARTIFACTS CAN BE PUSHED FROM A MAXIMUM OF 5 BRANCHES, 
   BUT ALLOWED_BRANCHES IS SET TO 7 BRANCHES, WHICH ARE: 
   pdev,prod,staging,hotfix,release,beta,alpha

- Publishing from too many branches can lead to:
  - Artifact repository pollution
  - Version confusion
  - Increased storage costs
  - Difficulty tracking what's production-ready
```

### 2. Semantic Version Pattern Matching

**Purpose**: Support automated releases with semantic versioning.

The action checks if the branch name matches semantic versioning patterns:

```bash
TAG_PATTERN="v*,number,dot,beta,patch,snapshot,release,hotfix,hotfix/v*,pullrequest/*,patch/v*"

# For each pattern, test against current branch
for pattern in TAG_PATTERN:
    if branch_matches(current_branch, pattern):
        ✅ PUSH_ENABLED = true
        break
```

See [Semantic Versioning Patterns](#semantic-versioning-patterns) section for detailed pattern explanations.

### 3. Exact Branch Match

**Purpose**: Direct authorization for named branches.

```bash
ALLOWED_BRANCHES="pdev,prod"
CURRENT_BRANCH="pdev"

if CURRENT_BRANCH in ALLOWED_BRANCHES:
    ✅ Branch pdev is allowed for pushing (exact match)
    PUSH_ENABLED = true
```

**This is the most common match type** - simple and explicit.

### 4. Wildcard Pattern Matching

**Purpose**: Support pattern-based branch strategies.

```bash
BRANCH_PATTERN="feature/*,hotfix/*"
CURRENT_BRANCH="feature/user-auth"

# Convert to lowercase for case-insensitive matching
if current_branch matches any wildcard pattern:
    ✅ Pattern Branch feature/user-auth is allowed for pushing
    PUSH_ENABLED = true
```

**Example Patterns**:
- `feature/*` - All feature branches
- `release/*` - All release branches
- `hotfix/*` - All hotfix branches

### 5. Default Branch Check

**Purpose**: Always allow default branch (master/main).

```bash
DEFAULT_BRANCH="master"
CURRENT_BRANCH="master"

if CURRENT_BRANCH == DEFAULT_BRANCH:
    ✅ It's default branch master
    PUSH_ENABLED = true
```

### 6. Commit Message Override

**Purpose**: Allow manual override via commit message.

```bash
COMMIT_MESSAGE="feat: add feature [publish]"

if "[publish]" in COMMIT_MESSAGE or "[push]" in COMMIT_MESSAGE:
    ✅ Commit message contains build push pattern - enabling push
    PUSH_ENABLED = true
```

**Use Case**: Manually trigger artifact push from a branch not normally allowed.

**Example Commit Messages**:
- `"fix: critical bug [publish]"` ✅
- `"feat: new feature [push]"` ✅
- `"chore: update deps"` ❌ (no keyword)

---

## Semantic Versioning Patterns

### What is Semantic Versioning?

**Semantic Versioning (SemVer)** is a versioning scheme that uses a three-part version number: `MAJOR.MINOR.PATCH`

- **MAJOR**: Incompatible API changes
- **MINOR**: Backward-compatible functionality additions
- **PATCH**: Backward-compatible bug fixes

Example: `v2.3.1`
- Major version: 2
- Minor version: 3
- Patch version: 1

### Supported Patterns

The `branch_check` action supports these semantic patterns:

#### 1. **`v*` - Version Tags**

**Pattern**: Branch names starting with `v` followed by version numbers

**Matches**:
- ✅ `v1.0.0`
- ✅ `v2.1.3`
- ✅ `v8.1.1`
- ✅ `v1.0.0-alpha`
- ✅ `v2.0.0-rc.1`
- ✅ `v3.2`

**Does NOT Match**:
- ❌ `version-1.0.0` (doesn't start with 'v')
- ❌ `1.0.0` (no 'v' prefix)
- ❌ `pdev` (not a version)

**Use Case**: Git tags for releases
```bash
git tag v1.2.3
git push origin v1.2.3
# → Triggers build and publishes artifact
```

**Regex**: `^v[0-9]+(\.[0-9]+)*`

---

#### 2. **`number` - Pure Numeric Versions**

**Pattern**: Pure semantic version numbers (MAJOR.MINOR.PATCH)

**Matches**:
- ✅ `1.0.0`
- ✅ `2.1.3`
- ✅ `10.5.2`

**Does NOT Match**:
- ❌ `1.0` (missing patch version)
- ❌ `1.0.0-alpha` (has suffix)
- ❌ `v1.0.0` (has 'v' prefix)

**Use Case**: Maven/NPM style versioning
```bash
git checkout -b 2.1.0
# → Auto-deploys version 2.1.0
```

**Regex**: `^[0-9]+\.[0-9]+\.[0-9]+$`

---

#### 3. **`dot` - Dot-Separated Versions**

**Pattern**: Flexible dot-separated numbers (supports 2+ parts)

**Matches**:
- ✅ `1.0`
- ✅ `1.0.0`
- ✅ `1.0.0.0`
- ✅ `2023.11.1`

**Does NOT Match**:
- ❌ `1` (single number)
- ❌ `1.0-alpha` (has non-numeric suffix)

**Use Case**: Date-based versioning or flexible version schemes
```bash
git tag 2025.11.1
# → Calendar versioning
```

**Regex**: `^[0-9]+(\.[0-9]+)*$`

---

#### 4. **`beta` - Beta Releases**

**Pattern**: Branch names indicating beta/pre-release versions

**Matches**:
- ✅ `beta`
- ✅ `beta-1.0`
- ✅ `beta/v2.0.0`
- ✅ `feature-beta`
- ✅ `v1.0.0-beta`

**Does NOT Match**:
- ❌ `pdev` (not beta-related)
- ❌ `beta123` (no separator after 'beta')

**Use Case**: Pre-release testing
```bash
git checkout -b beta/v2.0.0
# → Publishes beta artifact for testing
```

**Regex**: `^beta([-/]|$)|-beta$`

---

#### 5. **`patch` - Patch Releases**

**Pattern**: Branch names for patch/bugfix releases

**Matches**:
- ✅ `patch`
- ✅ `patch-1.0.1`
- ✅ `patch/v1.0.1`
- ✅ `bugfix-patch`

**Does NOT Match**:
- ❌ `pdev`
- ❌ `patch123` (no separator)

**Use Case**: Emergency bug fixes
```bash
git checkout -b patch/v1.0.1
# → Quick patch deployment
```

**Regex**: `^patch([-/]|$)|-patch$`

---

#### 6. **`snapshot` - Snapshot Builds**

**Pattern**: Development snapshot versions

**Matches**:
- ✅ `snapshot`
- ✅ `snapshot-2025-11-11`
- ✅ `snapshot/latest`
- ✅ `dev-snapshot`

**Does NOT Match**:
- ❌ `pdev`
- ❌ `snapshot123` (no separator)

**Use Case**: Nightly builds, development snapshots
```bash
git checkout -b snapshot/nightly
# → Publishes SNAPSHOT artifact
```

**Regex**: `^snapshot([-/]|$)|-snapshot$`

---

#### 7. **`hotfix` - Hotfix Branches**

**Pattern**: Emergency production fixes

**Matches**:
- ✅ `hotfix`
- ✅ `hotfix-critical-bug`
- ✅ `hotfix/security-fix`
- ✅ `critical-hotfix`

**Does NOT Match**:
- ❌ `pdev`
- ❌ `hotfix123` (no separator)

**Use Case**: Critical production issues
```bash
git checkout -b hotfix/security-cve-2025
# → Emergency deployment to production
```

**Regex**: `^hotfix([-/]|$)|-hotfix$`

---

#### 8. **`release/*` - Release Branches**

**Pattern**: Formal release preparation branches

**Matches**:
- ✅ `release/1.0.0`
- ✅ `release/v2.1.0`
- ✅ `release/2025-q4`

**Does NOT Match**:
- ❌ `release` (no slash)
- ❌ `releases/1.0.0` (plural)

**Use Case**: Release candidate preparation
```bash
git checkout -b release/v2.0.0
# → Prepare and publish release candidate
```

**Regex**: `^release/`

---

#### 9. **`hotfix/v*` - Versioned Hotfixes**

**Pattern**: Hotfix branches with semantic version numbers

**Matches**:
- ✅ `hotfix/v1.0.1`
- ✅ `hotfix/v2.1.3`
- ✅ `hotfix/v1.0.0-patch1`

**Does NOT Match**:
- ❌ `hotfix/1.0.1` (no 'v')
- ❌ `hotfix/bug-fix` (no version)

**Use Case**: Versioned emergency fixes
```bash
git checkout -b hotfix/v1.0.1
git push origin hotfix/v1.0.1
# → Deploys hotfix version 1.0.1
```

**Regex**: `^hotfix/v[0-9]+\.[0-9]+\.[0-9]+.*$`

---

#### 10. **`patch/v*` - Versioned Patches**

**Pattern**: Patch branches with semantic version numbers

**Matches**:
- ✅ `patch/v1.0.1`
- ✅ `patch/v2.0.5`

**Does NOT Match**:
- ❌ `patch/1.0.1` (no 'v')
- ❌ `patch/fix` (no version)

**Use Case**: Versioned patch releases
```bash
git checkout -b patch/v1.0.2
# → Deploys patch version 1.0.2
```

**Regex**: `^patch/v[0-9]+\.[0-9]+\.[0-9]+.*$`

---

#### 11. **`pullrequest/*` - PR Merge Branches**

**Pattern**: GitHub pull request merge branches

**Matches**:
- ✅ `123/merge`
- ✅ `456/merge`

**Does NOT Match**:
- ❌ `pr/123`
- ❌ `pull-request-123`

**Use Case**: Automated PR builds
```bash
# GitHub automatically creates these during PR merge
refs/pull/123/merge
# → Can trigger automated deployment
```

**Regex**: `^[0-9]+/merge$`

---

### Semantic Pattern Matching Logic

Here's how the action evaluates semantic patterns:

```bash
# Pseudo-code
function check_semantic_version(branch, pattern):
    # Convert to lowercase for case-insensitive matching
    branch_lower = branch.toLowerCase()
    pattern_lower = pattern.toLowerCase()
    
    # Test specific pattern
    case pattern_lower:
        when "v*":
            return branch_lower matches ^v[0-9]+(\.[0-9]+)*
        when "number":
            return branch_lower matches ^[0-9]+\.[0-9]+\.[0-9]+$
        when "dot":
            return branch_lower matches ^[0-9]+(\.[0-9]+)*$
        # ... etc for all patterns
        
    return false

# Main validation
for pattern in TAG_PATTERN.split(','):
    if check_semantic_version(CURRENT_BRANCH, pattern):
        ✅ PUSH_ENABLED = true
        BREAK  # Stop at first match
```

### Combining Patterns

You can combine multiple patterns:

```yaml
tag_pattern: "v*,hotfix,release/*,beta"
```

The action will:
1. Test `v*` against branch name
2. If no match, test `hotfix`
3. If no match, test `release/*`
4. If no match, test `beta`
5. Stop at first match

---

## Real-World Examples

### Example 1: Your Workflow (from the output)

**Configuration**:
```yaml
push_on_branches: pdev,prod
default_branch: master
current_branch: pdev
tag_pattern: v*,number,dot,beta,patch,snapshot,release,hotfix,hotfix/v*,pullrequest/*,patch/v*
```

**Result**:
```
🔍 Checking semantic patterns... ❌ (no match for 'pdev')
✅ Exact branch match: pdev ∈ {pdev,prod}
→ BRANCH_PUSH_ENABLED=true
```

**Why It Worked**: `pdev` is explicitly listed in `push_on_branches`.

---

### Example 2: Semantic Version Tag

**Configuration**:
```yaml
push_on_branches: prod
current_branch: v1.2.3
tag_pattern: v*
```

**Result**:
```
🔍 Checking semantic patterns...
✅ Semantic pattern 'v*' matched for branch 'v1.2.3'
→ BRANCH_PUSH_ENABLED=true
```

**Use Case**: Git tag-based releases
```bash
git tag v1.2.3
git push origin v1.2.3
# GitHub Actions triggers on tag push
# → Builds and publishes version 1.2.3
```

---

### Example 3: Hotfix Workflow

**Configuration**:
```yaml
push_on_branches: prod
current_branch: hotfix/v1.0.1
tag_pattern: hotfix/v*
```

**Result**:
```
🔍 Testing semantic pattern: 'hotfix/v*'
✅ Semantic pattern 'hotfix/v*' matched
→ BRANCH_PUSH_ENABLED=true
```

**Workflow**:
```bash
# Production is at v1.0.0
# Critical bug discovered!

git checkout prod
git checkout -b hotfix/v1.0.1
# ... fix bug ...
git commit -m "fix: critical security issue"
git push origin hotfix/v1.0.1

# GitHub Actions automatically:
# 1. Runs tests
# 2. Builds artifact
# 3. Publishes v1.0.1 to production
# 4. Merges back to prod branch
```

---

### Example 4: Feature Branch (Blocked)

**Configuration**:
```yaml
push_on_branches: prod
current_branch: feature/user-authentication
tag_pattern: v*,hotfix
```

**Result**:
```
❌ Semantic pattern 'v*' did not match
❌ Semantic pattern 'hotfix' did not match
❌ Branch not in allowed list
❌ Not default branch
→ BRANCH_PUSH_ENABLED=false
```

**Outcome**: Feature branch can run tests but **cannot publish artifacts**.

---

### Example 5: Commit Message Override

**Configuration**:
```yaml
push_on_branches: prod
current_branch: feature/urgent-fix
commit_message: "fix: critical issue [publish]"
```

**Result**:
```
❌ Branch not in allowed list
✅ Commit message contains build push pattern
→ BRANCH_PUSH_ENABLED=true
```

**Use Case**: Emergency deployment from non-standard branch.

```bash
git checkout -b feature/urgent-fix
# ... fix critical issue ...
git commit -m "fix: XSS vulnerability [publish]"
git push origin feature/urgent-fix

# Despite being a feature branch, artifact is published
# due to [publish] keyword in commit message
```

---

### Example 6: Too Many Branches (Blocked)

**Configuration**:
```yaml
push_on_branches: pdev,prod,staging,beta,alpha,rc,snapshot
# 7 branches configured
BRANCH_COUNT: 5  # Org limit
```

**Result**:
```
❌ ARTIFACTS CAN BE PUSHED FROM A MAXIMUM OF 5 BRANCHES,
   BUT ALLOWED_BRANCHES IS SET TO 7 BRANCHES
→ BRANCH_PUSH_ENABLED=false
```

**Resolution**: Remove 2 branches from the allowed list.

---

## Best Practices

### 1. Minimize Allowed Branches

**Recommended**:
```yaml
push_on_branches: prod
# Only production pushes artifacts
```

**Why**: 
- Reduces storage costs
- Clear separation of environments
- Easier to track what's production-ready

### 2. Use Semantic Versioning for Releases

**Recommended**:
```yaml
push_on_branches: prod
tag_pattern: v*,hotfix/v*,patch/v*
```

**Workflow**:
```bash
# Release process
git tag v1.2.0
git push origin v1.2.0
# → Automated release deployment

# Hotfix process
git checkout -b hotfix/v1.2.1
git push origin hotfix/v1.2.1
# → Automated hotfix deployment
```

### 3. Separate Development and Production

**Recommended**:
```yaml
# Development workflow (no artifact push)
- Branch: feature/*
  Publish: ❌ (run tests only)

# Staging workflow (push to staging registry)
- Branch: staging
  Publish: ✅ → staging-registry

# Production workflow (push to production registry)
- Branch: prod
  Publish: ✅ → production-registry
```

### 4. Document Your Branch Strategy

Create a `BRANCHING.md` in your repo:

```markdown
# Branch Strategy

## Production Branches
- `prod` - Production releases (auto-deploys)
- `v*` tags - Versioned releases (auto-deploys)

## Development Branches
- `pdev` - Pre-production development (manual deploy)
- `feature/*` - Feature development (tests only)
- `hotfix/v*` - Emergency fixes (auto-deploys)

## Artifact Publishing Rules
Only these branches can publish artifacts:
- prod
- pdev
- Tags matching v*
- Hotfix branches matching hotfix/v*
```

### 5. Use Commit Message Override Sparingly

**Good Use**:
```bash
# Emergency fix that needs immediate deployment
git commit -m "fix: critical security vulnerability [publish]"
```

**Bad Use**:
```bash
# Regular feature (should go through normal process)
git commit -m "feat: add new button [publish]"
```

### 6. Set Appropriate Branch Count Limits

**Organization Sizes**:
- **Small teams**: `BRANCH_COUNT=3` (dev, staging, prod)
- **Medium teams**: `BRANCH_COUNT=5` (dev, staging, prod, hotfix, beta)
- **Large teams**: `BRANCH_COUNT=7` (multiple environments)

### 7. Monitor Artifact Storage

Regularly audit your artifact repositories:
```bash
# Check artifact sizes
jfrog rt search --limit 100 --sort-by created --sort-order desc

# Clean up old artifacts
jfrog rt delete --quiet "repo/artifact-*" --older-than 90d
```

---

## Troubleshooting

### Issue 1: Branch Should Be Allowed But Isn't

**Symptom**:
```
❌ Branch pdev is not allowed for pushing
```

**Diagnosis**:
```bash
# Check configuration
echo "Current branch: $CURRENT_BRANCH"
echo "Allowed branches: $ALLOWED_BRANCHES"
```

**Solutions**:

1. **Typo in branch name**:
   ```yaml
   # Wrong
   push_on_branches: "pdev,prod"
   current_branch: "Pdev"  # Capital P!
   
   # Fixed (case-insensitive matching)
   # Should work, but verify no extra spaces
   ```

2. **Extra spaces**:
   ```yaml
   # Wrong
   push_on_branches: "pdev , prod"  # Space before comma
   
   # Correct
   push_on_branches: "pdev,prod"
   ```

3. **Branch count exceeded**:
   ```
   ALLOWED_BRANCHES="pdev,prod,staging,beta,alpha,rc"  # 6 branches
   BRANCH_COUNT=5  # Limit
   
   # Solution: Remove a branch
   ALLOWED_BRANCHES="pdev,prod,staging,beta,alpha"  # 5 branches
   ```

---

### Issue 2: Semantic Pattern Not Matching

**Symptom**:
```
🔍 Testing semantic pattern: 'v*'
❌ Semantic pattern 'v*' did not match branch 'v1.0'
```

**Diagnosis**:
```bash
# Check branch name format
git branch --show-current
```

**Common Issues**:

1. **Missing version parts**:
   ```bash
   # Pattern: "number" expects MAJOR.MINOR.PATCH
   v1.0    # ❌ Missing patch version
   v1.0.0  # ✅ Correct
   ```

2. **Wrong prefix**:
   ```bash
   # Pattern: "v*" expects 'v' prefix
   1.0.0       # ❌ No prefix
   version1.0  # ❌ Wrong prefix
   v1.0.0      # ✅ Correct
   ```

3. **Extra characters**:
   ```bash
   # Pattern: "number" expects pure numbers
   1.0.0-beta  # ❌ Has suffix
   1.0.0       # ✅ Correct
   ```

**Solution**: Match your branch name to the pattern requirements.

---

### Issue 3: Too Many Branches Error

**Symptom**:
```
❌ ARTIFACTS CAN BE PUSHED FROM A MAXIMUM OF 5 BRANCHES,
   BUT ALLOWED_BRANCHES IS SET TO 7 BRANCHES
```

**Solution 1**: Reduce allowed branches
```yaml
# Before
push_on_branches: pdev,prod,staging,beta,alpha,rc,snapshot

# After (keep only essential)
push_on_branches: prod,staging
tag_pattern: v*,hotfix/v*  # Use semantic patterns instead
```

**Solution 2**: Request org-level increase
```bash
# Contact your DevOps team to increase BRANCH_COUNT
# Only if you have legitimate business need
```

---

### Issue 4: Commit Message Override Not Working

**Symptom**:
```
Commit: "fix: bug [publish]"
❌ Branch still not allowed
```

**Check**:
```bash
# Verify commit message is passed to action
echo "COMMIT_MESSAGE: $COMMIT_MESSAGE"
```

**Solutions**:

1. **Case sensitivity**:
   ```bash
   # These work:
   "fix: bug [publish]"  ✅
   "fix: bug [push]"     ✅
   "fix: bug [PUBLISH]"  ✅  # Case-insensitive
   
   # These don't:
   "fix: bug (publish)"  ❌  # Wrong brackets
   "fix: bug publish"    ❌  # No brackets
   ```

2. **Parameter not passed**:
   ```yaml
   # Workflow file must include:
   - uses: cvs-health-source-code/gha_workflow_actions/actions/branch_check@latest
     with:
       commit_message: ${{ github.event.head_commit.message }}  # Add this!
   ```

---

### Issue 5: Wildcard Pattern Not Matching

**Symptom**:
```
Branch: feature/user-auth
Pattern: feature/*
❌ Not matching
```

**Check Pattern Syntax**:
```yaml
# Correct
branch_pattern: "feature/*,hotfix/*"

# Wrong
branch_pattern: "feature*"     # Missing /
branch_pattern: "feature/**"   # Too many *
```

---

## Advanced Configuration Examples

### Example 1: Microservices Monorepo

```yaml
# Different services deploy from different branches
name: Build Service

on:
  push:
    branches:
      - prod
      - staging
      - service-a-dev
      - service-b-dev

jobs:
  branch-check:
    runs-on: ubuntu-latest
    steps:
      - uses: cvs-health-source-code/gha_workflow_actions/actions/branch_check@latest
        with:
          push_on_branches: prod,staging,service-a-dev,service-b-dev
          current_branch: ${{ github.ref_name }}
          tag_pattern: v*,hotfix/v*
```

### Example 2: GitFlow Workflow

```yaml
# GitFlow: master, develop, release/*, hotfix/*, feature/*
jobs:
  branch-check:
    steps:
      - uses: cvs-health-source-code/gha_workflow_actions/actions/branch_check@latest
        with:
          push_on_branches: master,develop
          default_branch: master
          current_branch: ${{ github.ref_name }}
          branch_pattern: release/*,hotfix/*
          tag_pattern: v*
```

### Example 3: Trunk-Based Development

```yaml
# Single main branch, all work in short-lived feature branches
jobs:
  branch-check:
    steps:
      - uses: cvs-health-source-code/gha_workflow_actions/actions/branch_check@latest
        with:
          push_on_branches: main
          default_branch: main
          current_branch: ${{ github.ref_name }}
          tag_pattern: v*  # Only tags push artifacts
          commit_message: ${{ github.event.head_commit.message }}
          # Feature branches blocked unless [publish] in commit
```

---

## Summary

The `branch_check` action provides:

✅ **Governance** - Control what branches can publish artifacts  
✅ **Cost Control** - Limit branches to reduce storage costs  
✅ **Semantic Versioning** - Support automated version-based releases  
✅ **Flexibility** - Multiple matching strategies (exact, pattern, semantic)  
✅ **Safety** - Prevent accidental deployments from feature branches  
✅ **Auditability** - Clear logs of what branches are publishing  

**Key Takeaways**:
1. Use semantic version patterns (`v*`) for automated releases
2. Keep allowed branches to a minimum (3-5)
3. Use commit message override (`[publish]`) only for emergencies
4. Document your branching strategy in your repository
5. Regularly audit artifact storage

---

## Additional Resources

- [Semantic Versioning Specification](https://semver.org/)
- [Git Branching Strategies](https://git-scm.com/book/en/v2/Git-Branching-Branching-Workflows)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Artifact Repository Best Practices](https://jfrog.com/whitepaper/best-practices-structuring-naming-artifactory-repositories/)

---

## Questions?

If you have questions about this action or need help configuring it:

1. Check the [Troubleshooting](#troubleshooting) section
2. Review the [Examples](#real-world-examples)
3. Contact your DevOps team
4. Open an issue in the action repository

---

**Last Updated**: November 11, 2025  
**Version**: 1.0.0


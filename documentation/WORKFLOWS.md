# GitHub Actions Workflows Documentation

This document provides detailed explanations of each workflow included in this repository.

## Table of Contents

1. [Continuous Integration (CI)](#continuous-integration-ci)
2. [Continuous Deployment (CD)](#continuous-deployment-cd)
3. [Security Scanning](#security-scanning)
4. [Python CI](#python-ci)
5. [Docker Build and Push](#docker-build-and-push)
6. [Kubernetes Deployment](#kubernetes-deployment)

## Continuous Integration (CI)

**File**: `.github/workflows/ci.yml`

### Purpose

The CI workflow ensures code quality and functionality before merging changes. It runs automatically on pull requests and feature branch pushes.

### Trigger Events

```yaml
on:
  pull_request:
    branches:
      - main
      - develop
  push:
    branches:
      - feature/**
```

### Jobs

#### 1. Lint Job

**Purpose**: Checks code style and quality

```yaml
lint:
  name: Lint Code
  runs-on: ubuntu-latest
```

**Steps**:
- Checkout code
- Setup Node.js with caching
- Install dependencies with `npm ci`
- Run ESLint

**Why it's important**: Catches style issues early and ensures consistent code formatting across the team.

#### 2. Test Job

**Purpose**: Runs test suite across multiple Node.js versions

```yaml
test:
  needs: lint
  strategy:
    matrix:
      node-version: [18, 20]
```

**Steps**:
- Checkout code
- Setup Node.js for each version in matrix
- Install dependencies
- Run tests with coverage
- Upload coverage to Codecov (for Node 20 only)

**Why it's important**: Ensures compatibility across Node.js versions and maintains test coverage.

**Matrix Strategy**: Tests run in parallel for each Node version, speeding up the overall pipeline.

#### 3. Build Job

**Purpose**: Verifies the application builds successfully

```yaml
build:
  needs: test
```

**Steps**:
- Checkout code
- Setup Node.js
- Install dependencies
- Run build command
- Archive build artifacts

**Why it's important**: Catches build errors before deployment and creates artifacts for deployment.

#### 4. Code Quality Job

**Purpose**: Enforces code coverage thresholds

**Steps**:
- Run tests with coverage
- Check coverage meets minimum thresholds (70%)

**Why it's important**: Maintains high code quality standards.

### Configuration Options

#### Customize Node Versions

```yaml
strategy:
  matrix:
    node-version: [16, 18, 20, 22]
```

#### Adjust Coverage Thresholds

```yaml
--coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80,"statements":80}}'
```

#### Cache Dependencies

Caching is automatically enabled:

```yaml
- uses: actions/setup-node@v4
  with:
    cache: 'npm'
    cache-dependency-path: app/package-lock.json
```

### Best Practices

1. **Fail Fast**: The lint job runs first and must pass before tests run
2. **Parallel Execution**: Tests run in parallel across Node versions
3. **Artifact Storage**: Build artifacts are kept for 7 days
4. **Coverage Upload**: Only one version uploads to avoid duplicates

## Continuous Deployment (CD)

**File**: `.github/workflows/cd.yml`

### Purpose

Automates deployment to staging and production environments after code is merged to the main branch.

### Trigger Events

```yaml
on:
  push:
    branches:
      - main
```

### Jobs

#### 1. Build and Test

**Purpose**: Final verification before deployment

**Steps**:
- Run full CI pipeline (lint, test, build)
- Create production artifacts

**Why it's important**: Ensures only tested code is deployed.

#### 2. Deploy to Staging

**Purpose**: Deploy to staging environment for final validation

```yaml
deploy-staging:
  needs: build
  environment:
    name: staging
    url: https://staging.example.com
```

**Steps**:
- Download build artifacts
- Deploy to staging server
- Run smoke tests

**Environment Protection**: Can require approvals or wait timers in GitHub settings.

#### 3. Deploy to Production

**Purpose**: Deploy to production after staging validation

```yaml
deploy-production:
  needs: deploy-staging
  environment:
    name: production
    url: https://example.com
```

**Steps**:
- Download build artifacts
- Create deployment record
- Deploy to production
- Update deployment status
- Run production smoke tests
- Send notifications

**Why it's important**: Provides a safe, automated path to production with proper tracking.

### Deployment Strategies

#### Rolling Deployment

Deploy new version gradually:

```yaml
- name: Deploy with rolling update
  run: |
    for server in server1 server2 server3; do
      ssh user@$server "cd /app && git pull && npm install && pm2 restart app"
      sleep 30  # Wait between servers
    done
```

#### Blue-Green Deployment

Deploy to inactive environment, then switch:

```yaml
- name: Deploy to green environment
  run: deploy.sh green

- name: Run health checks
  run: test_environment.sh green

- name: Switch traffic to green
  run: switch_traffic.sh green
```

#### Canary Deployment

Deploy to subset of servers first:

```yaml
- name: Deploy to 10% of servers
  run: deploy_canary.sh 0.1

- name: Monitor metrics
  run: monitor.sh --duration 300

- name: Full rollout
  run: deploy_full.sh
```

### Customization

#### Add Deployment Steps

```yaml
- name: Deploy to AWS
  run: |
    aws s3 sync ./build s3://my-bucket
    aws cloudfront create-invalidation --distribution-id XXX --paths "/*"
```

#### Add Slack Notifications

```yaml
- name: Notify Slack
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## Security Scanning

**File**: `.github/workflows/security-scan.yml`

### Purpose

Identifies security vulnerabilities in dependencies and code.

### Trigger Events

```yaml
on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

### Jobs

#### 1. Dependency Scan

**Purpose**: Find vulnerabilities in npm packages

**Tools**:
- `npm audit`: Built-in npm security auditing

```yaml
- name: Run npm audit
  run: npm audit --audit-level=moderate
```

**Severity Levels**:
- `low`: Informational
- `moderate`: Should be fixed
- `high`: Must be fixed
- `critical`: Critical security issue

#### 2. Snyk Scan

**Purpose**: Advanced vulnerability scanning with Snyk

**Configuration**:

```yaml
- uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

**Setup**:
1. Sign up at [snyk.io](https://snyk.io)
2. Get API token from account settings
3. Add as `SNYK_TOKEN` secret in GitHub

#### 3. Trivy Scan

**Purpose**: Filesystem vulnerability scanning

**Features**:
- Scans for OS vulnerabilities
- Checks for misconfigurations
- Finds secrets in code

```yaml
- uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    format: 'sarif'
    severity: 'CRITICAL,HIGH'
```

**Results**: Uploaded to GitHub Security tab

#### 4. CodeQL Analysis

**Purpose**: Semantic code analysis

**Supported Languages**:
- JavaScript/TypeScript
- Python
- Java
- C/C++
- C#
- Go
- Ruby

```yaml
- uses: github/codeql-action/init@v3
  with:
    languages: javascript
```

**Benefits**:
- Finds security vulnerabilities
- Detects code quality issues
- Identifies common bugs

#### 5. Dependency Review

**Purpose**: Reviews dependency changes in PRs

```yaml
- uses: actions/dependency-review-action@v4
  with:
    fail-on-severity: moderate
```

**Checks**:
- New vulnerabilities introduced
- License compliance
- Dependency health

#### 6. License Check

**Purpose**: Ensures license compliance

```yaml
- name: Check licenses
  run: |
    license-checker --onlyAllow 'MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;0BSD'
```

**Allowed Licenses**: Customize based on your requirements

### Security Best Practices

1. **Run Regularly**: Schedule weekly scans
2. **Block PRs**: Fail CI on security issues
3. **Monitor Results**: Check Security tab regularly
4. **Auto-fix**: Use `npm audit fix` for automatic fixes
5. **Keep Updated**: Update dependencies regularly

## Python CI

**File**: `examples/python-ci.yml`

### Purpose

CI/CD pipeline for Python applications using pytest, black, and other Python tools.

### Jobs

#### 1. Lint

**Tools**:
- **Black**: Code formatting
- **isort**: Import sorting
- **Flake8**: Style guide enforcement
- **Pylint**: Static analysis

```yaml
- run: black --check .
- run: isort --check-only .
- run: flake8 .
- run: pylint **/*.py
```

#### 2. Test

**Features**:
- Tests across Python versions (3.9, 3.10, 3.11, 3.12)
- Coverage reporting
- pytest fixtures and async support

```yaml
strategy:
  matrix:
    python-version: ['3.9', '3.10', '3.11', '3.12']
```

#### 3. Type Check

**Tool**: mypy for static type checking

```yaml
- run: mypy . --ignore-missing-imports
```

#### 4. Security

**Tools**:
- **Safety**: Dependency vulnerability checking
- **Bandit**: Security issue detection

```yaml
- run: safety check --json
- run: bandit -r . -f json
```

## Docker Build and Push

**File**: `examples/docker-build.yml`

### Purpose

Build, scan, and publish Docker images to registries.

### Features

#### Multi-platform Builds

```yaml
- uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64,linux/arm64
```

**Supported Platforms**:
- linux/amd64 (Intel/AMD)
- linux/arm64 (ARM64/Apple Silicon)
- linux/arm/v7 (ARM32)

#### Multiple Registries

**Docker Hub**:
```yaml
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```

**GitHub Container Registry**:
```yaml
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

#### Smart Tagging

```yaml
tags: |
  type=ref,event=branch
  type=ref,event=pr
  type=semver,pattern={{version}}
  type=semver,pattern={{major}}.{{minor}}
  type=sha,prefix={{branch}}-
  type=raw,value=latest,enable={{is_default_branch}}
```

**Example Tags**:
- `main` (branch name)
- `pr-123` (pull request)
- `1.2.3`, `1.2`, `1` (semantic version)
- `main-abc1234` (branch + commit SHA)
- `latest` (default branch only)

#### Build Cache

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

**Benefits**:
- Faster builds
- Reduced CI minutes
- Layer reuse

### Security Scanning

**Trivy**:
- Scans for OS vulnerabilities
- Checks for misconfigurations

**Snyk**:
- Advanced vulnerability detection
- License compliance

## Kubernetes Deployment

**File**: `examples/k8s-deploy.yml`

### Purpose

Automated deployment to Kubernetes clusters with canary and rollback support.

### Deployment Strategies

#### Rolling Update (Default)

```yaml
- name: Deploy to Kubernetes
  run: |
    kubectl set image deployment/app \
      app=$IMAGE:$TAG \
      -n production
```

**Behavior**:
- Gradual replacement of pods
- Zero downtime
- Automatic rollback on failure

#### Canary Deployment

```yaml
# Deploy to 10% of traffic
- name: Deploy canary
  run: |
    kubectl set image deployment/app-canary app=$IMAGE:$TAG

# Monitor metrics
- name: Monitor canary
  run: check_error_rate.sh

# Promote to 100%
- name: Promote canary
  run: |
    kubectl set image deployment/app app=$IMAGE:$TAG
```

**Benefits**:
- Test with real traffic
- Minimal risk
- Quick rollback

### Environments

#### Staging

```yaml
deploy-staging:
  environment:
    name: staging
    url: https://staging.example.com
```

**Purpose**: Testing before production

#### Production

```yaml
deploy-production:
  needs: deploy-staging
  environment:
    name: production
    url: https://example.com
```

**Protection**: Requires manual approval

### Verification Steps

1. **Cluster Connection**
   ```yaml
   - run: kubectl cluster-info
   ```

2. **Rollout Status**
   ```yaml
   - run: kubectl rollout status deployment/app -n production --timeout=5m
   ```

3. **Smoke Tests**
   ```yaml
   - run: curl -f $SERVICE_URL/health
   ```

### Rollback

Automatic rollback on failure:

```yaml
rollback:
  if: failure()
  needs: [deploy-production]
  steps:
    - run: kubectl rollout undo deployment/app -n production
```

## Common Patterns

### Caching Strategies

**npm**:
```yaml
- uses: actions/setup-node@v4
  with:
    cache: 'npm'
```

**pip**:
```yaml
- uses: actions/setup-python@v5
  with:
    cache: 'pip'
```

**Docker layers**:
```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

### Conditional Execution

**Run on specific branches**:
```yaml
if: github.ref == 'refs/heads/main'
```

**Run on file changes**:
```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
```

### Matrix Builds

**Multiple versions**:
```yaml
strategy:
  matrix:
    node-version: [18, 20]
    os: [ubuntu-latest, windows-latest]
```

### Artifacts

**Upload**:
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
```

**Download**:
```yaml
- uses: actions/download-artifact@v4
  with:
    name: build-output
```

## Troubleshooting

### Common Issues

#### Workflow Not Triggering

- Check branch names match
- Verify file is in `.github/workflows/`
- Ensure valid YAML syntax

#### Build Timeouts

```yaml
jobs:
  build:
    timeout-minutes: 30
```

#### Permission Errors

```yaml
permissions:
  contents: read
  packages: write
  security-events: write
```

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Actions Marketplace](https://github.com/marketplace?type=actions)

# CI/CD Best Practices

This guide covers best practices for implementing and maintaining CI/CD pipelines with GitHub Actions.

## Table of Contents

1. [Pipeline Design](#pipeline-design)
2. [Performance Optimization](#performance-optimization)
3. [Security Best Practices](#security-best-practices)
4. [Testing Strategies](#testing-strategies)
5. [Deployment Strategies](#deployment-strategies)
6. [Monitoring and Observability](#monitoring-and-observability)
7. [Cost Optimization](#cost-optimization)
8. [Team Collaboration](#team-collaboration)

## Pipeline Design

### Keep Pipelines Fast

Fast feedback is crucial for developer productivity.

**Target Times**:
- CI pipeline: < 10 minutes
- Full deployment: < 30 minutes

**Strategies**:
```yaml
# Run jobs in parallel
jobs:
  lint:
    runs-on: ubuntu-latest
  test:
    runs-on: ubuntu-latest
  security:
    runs-on: ubuntu-latest
```

### Fail Fast

Catch errors early to save time and resources.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest

  test:
    needs: lint  # Only run if lint passes
    runs-on: ubuntu-latest

  deploy:
    needs: test  # Only deploy if tests pass
    runs-on: ubuntu-latest
```

**Benefits**:
- Saves CI minutes
- Faster feedback
- Reduced noise

### Use Matrix Builds Wisely

Test multiple configurations in parallel.

```yaml
strategy:
  matrix:
    node-version: [18, 20]
    os: [ubuntu-latest, macos-latest]
  fail-fast: false  # Continue even if one combination fails
```

**When to Use**:
- Testing across multiple versions
- Supporting multiple platforms
- Running tests in different configurations

**When to Avoid**:
- Excessive combinations (2 × 3 × 4 = 24 jobs)
- Long-running tests

### Keep Workflows Modular

Break complex workflows into reusable components.

**Bad**:
```yaml
# One giant workflow with 50 steps
jobs:
  everything:
    steps:
      - lint
      - test
      - build
      - deploy
      - notify
      # ...50 more steps
```

**Good**:
```yaml
# Separate workflows
# .github/workflows/ci.yml
# .github/workflows/cd.yml
# .github/workflows/security.yml

# Or use reusable workflows
jobs:
  test:
    uses: ./.github/workflows/test.yml
```

## Performance Optimization

### Caching Dependencies

Cache dependencies to speed up builds.

**npm**:
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'
```

**pip**:
```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.11'
    cache: 'pip'
```

**Custom Caching**:
```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache
      node_modules
    key: ${{ runner.os }}-deps-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-deps-
```

**Benefits**:
- 2-10x faster installs
- Reduced network usage
- More reliable builds

### Use Sparse Checkouts

Only checkout necessary files.

```yaml
- uses: actions/checkout@v4
  with:
    sparse-checkout: |
      src/
      package.json
      package-lock.json
```

**Use Cases**:
- Large repositories
- Monorepos
- Documentation-only changes

### Optimize Docker Builds

**Multi-stage Builds**:
```dockerfile
# Build stage
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --production
CMD ["node", "dist/index.js"]
```

**Layer Caching**:
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Benefits**:
- Smaller images (10x reduction possible)
- Faster builds
- Better security

### Parallel Testing

Split tests across multiple runners.

```yaml
test:
  strategy:
    matrix:
      shard: [1, 2, 3, 4]
  steps:
    - name: Run tests
      run: npm test -- --shard=${{ matrix.shard }}/4
```

**When to Use**:
- Large test suites (> 10 minutes)
- Integration tests
- E2E tests

## Security Best Practices

### Never Commit Secrets

**Never Do This**:
```javascript
const API_KEY = "sk_live_abc123";  // WRONG!
```

**Always Do This**:
```yaml
- name: Deploy
  env:
    API_KEY: ${{ secrets.API_KEY }}
  run: deploy.sh
```

### Use GitHub Secrets

Store sensitive data in GitHub Secrets:
1. Settings > Secrets and variables > Actions
2. Click "New repository secret"
3. Add name and value

**Access in Workflows**:
```yaml
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_KEY: ${{ secrets.API_KEY }}
```

### Least Privilege Principle

Grant minimum necessary permissions.

```yaml
permissions:
  contents: read      # Read repository
  packages: write     # Push packages
  security-events: write  # Upload security results
```

**Default Permissions** (restrictive):
```yaml
permissions:
  contents: read
```

### Pin Action Versions

**Bad**:
```yaml
- uses: actions/checkout@v4  # Could break
```

**Better**:
```yaml
- uses: actions/checkout@v4.1.1  # Specific version
```

**Best**:
```yaml
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # SHA
```

**Benefits**:
- Predictable behavior
- Security (prevents supply chain attacks)
- Easier debugging

### Scan Dependencies Regularly

```yaml
on:
  schedule:
    - cron: '0 0 * * 1'  # Every Monday
  push:
    branches: [main]

jobs:
  security:
    steps:
      - run: npm audit
      - uses: snyk/actions/node@master
```

### Review Third-Party Actions

Before using an action:
1. Check the source code
2. Verify the maintainer
3. Check for recent updates
4. Read reviews and issues

**Trusted Sources**:
- GitHub official actions (`actions/*`)
- Verified creators
- Popular community actions with good reputation

### Protect Sensitive Branches

Configure branch protection for `main` and `production`:

1. Settings > Branches > Add rule
2. Enable:
   - Require pull request reviews
   - Require status checks to pass
   - Require branches to be up to date
   - Include administrators

## Testing Strategies

### Test Pyramid

**Structure**:
```
        /\
       /E2E\      10% - End-to-end tests
      /------\
     /  Integ \   20% - Integration tests
    /----------\
   /    Unit    \ 70% - Unit tests
  /--------------\
```

**Implementation**:
```yaml
jobs:
  unit-tests:
    name: Unit Tests (Fast)
    run: npm test -- --testPathPattern=unit

  integration-tests:
    name: Integration Tests (Medium)
    needs: unit-tests
    run: npm test -- --testPathPattern=integration

  e2e-tests:
    name: E2E Tests (Slow)
    needs: integration-tests
    run: npm run test:e2e
```

### Code Coverage

Set minimum thresholds:

```yaml
- name: Check coverage
  run: |
    npm test -- --coverage \
      --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80,"statements":80}}'
```

**Recommended Thresholds**:
- Greenfield projects: 80%+
- Legacy projects: Start at 60%, increase gradually
- Critical modules: 90%+

### Smoke Tests

Quick tests after deployment:

```yaml
- name: Smoke tests
  run: |
    curl -f https://api.example.com/health || exit 1
    curl -f https://api.example.com/version || exit 1

    # Check critical endpoints
    response=$(curl -s https://api.example.com/api/status)
    echo $response | jq -e '.status == "ok"' || exit 1
```

**What to Test**:
- Health endpoints
- Critical user flows
- Database connectivity
- External service integration

### Contract Testing

Test API contracts:

```yaml
- name: Contract tests
  run: |
    # Provider testing (API)
    npm run test:pact:provider

    # Consumer testing (Client)
    npm run test:pact:consumer
```

**Benefits**:
- Catch breaking changes
- Enable independent deployment
- Document API contracts

## Deployment Strategies

### Blue-Green Deployment

Minimize downtime with two identical environments.

```yaml
deploy:
  steps:
    - name: Deploy to green
      run: deploy.sh green

    - name: Test green environment
      run: test.sh green

    - name: Switch traffic to green
      run: switch.sh green

    - name: Keep blue as backup
      run: echo "Blue environment available for rollback"
```

**Pros**:
- Zero downtime
- Quick rollback
- Full testing before switch

**Cons**:
- Requires 2x resources
- Database migrations complex

### Canary Deployment

Gradual rollout to subset of users.

```yaml
deploy:
  steps:
    - name: Deploy to 5% of servers
      run: deploy_canary.sh 0.05

    - name: Monitor metrics for 10 minutes
      run: |
        for i in {1..10}; do
          check_error_rate.sh || exit 1
          sleep 60
        done

    - name: Deploy to 100%
      run: deploy_full.sh
```

**Monitoring Points**:
- Error rate
- Response time
- CPU/Memory usage
- Business metrics

### Feature Flags

Decouple deployment from release.

```javascript
if (featureFlags.isEnabled('new-checkout-flow', user)) {
  return newCheckoutFlow();
} else {
  return oldCheckoutFlow();
}
```

**Benefits**:
- Deploy anytime
- Test in production safely
- Quick rollback (no deployment)
- A/B testing

**Tools**:
- LaunchDarkly
- Unleash
- Split.io
- ConfigCat

### Rollback Strategy

Always have a rollback plan.

```yaml
deploy:
  steps:
    - name: Create deployment marker
      run: |
        echo "${{ github.sha }}" > /deployments/current
        echo "$PREVIOUS_SHA" > /deployments/previous

    - name: Deploy
      id: deploy
      run: deploy.sh

    - name: Rollback on failure
      if: failure()
      run: |
        PREVIOUS_SHA=$(cat /deployments/previous)
        git checkout $PREVIOUS_SHA
        deploy.sh
```

**Rollback Types**:
1. **Redeploy previous version** (Docker/K8s)
2. **Git revert** (Simple apps)
3. **Feature flag disable** (Feature flags)
4. **Traffic switch** (Blue-green)

## Monitoring and Observability

### Workflow Monitoring

Track workflow metrics:

```yaml
- name: Send metrics
  if: always()
  run: |
    curl -X POST https://metrics.example.com/ci \
      -d '{
        "workflow": "${{ github.workflow }}",
        "status": "${{ job.status }}",
        "duration": "${{ steps.test.outputs.duration }}",
        "commit": "${{ github.sha }}"
      }'
```

**Key Metrics**:
- Build success rate
- Average build time
- Deployment frequency
- Mean time to recovery (MTTR)

### Deployment Tracking

Track deployments:

```yaml
- uses: chrnorm/deployment-action@v2
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    environment: production
    description: 'Deploy ${{ github.sha }}'
```

**Benefits**:
- Deployment history
- Correlation with incidents
- Audit trail

### Notification Strategy

**On Failure Only**:
```yaml
- name: Notify on failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: failure
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

**Summary Reports**:
```yaml
- name: Daily summary
  if: github.event_name == 'schedule'
  run: |
    # Send daily CI/CD metrics
    generate_report.sh | send_to_slack.sh
```

**Who to Notify**:
- Failed builds: Team channel
- Production deployments: General channel
- Security issues: Security team
- Coverage drops: Code owners

### Logging Best Practices

**Structured Logging**:
```yaml
- name: Deploy
  run: |
    echo "::group::Deployment"
    echo "Environment: production"
    echo "Version: ${{ github.sha }}"
    echo "Timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
    deploy.sh
    echo "::endgroup::"
```

**Set Outputs**:
```yaml
- name: Build
  id: build
  run: |
    VERSION=$(npm run version)
    echo "version=$VERSION" >> $GITHUB_OUTPUT

- name: Use output
  run: echo "Built version ${{ steps.build.outputs.version }}"
```

## Cost Optimization

### Reduce CI Minutes

**Strategies**:
1. Use caching
2. Run tests in parallel
3. Skip unnecessary jobs
4. Use self-hosted runners for heavy workloads

**Conditional Execution**:
```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package.json'
  # Skip CI for documentation changes
```

### Self-Hosted Runners

For heavy workloads, use self-hosted runners:

```yaml
jobs:
  build:
    runs-on: self-hosted  # Your own hardware
```

**When to Use**:
- Large builds (> 30 min)
- Special hardware needs (GPU)
- Private network access
- Cost optimization (> 2000 min/month)

**Cost Comparison**:
- GitHub-hosted: $0.008/minute (Linux)
- Self-hosted: Server cost / minutes used

### Optimize Docker Builds

**Use Smaller Base Images**:
```dockerfile
# Heavy (1GB)
FROM node:20

# Light (180MB)
FROM node:20-alpine
```

**Multi-stage Builds**:
```dockerfile
# Build: 1GB
FROM node:20 AS builder
RUN npm ci && npm run build

# Production: 200MB
FROM node:20-alpine
COPY --from=builder /app/dist ./dist
```

## Team Collaboration

### Standardize Workflows

Create organization-wide templates:

```
.github/
  workflow-templates/
    node-ci.yml
    node-cd.yml
    python-ci.yml
    docker.yml
```

**Benefits**:
- Consistency across repos
- Easier maintenance
- Best practices built-in

### Documentation

Document your CI/CD:

```markdown
# CI/CD Documentation

## Required Secrets
- `API_KEY`: Production API key
- `DATABASE_URL`: Connection string

## Deployment Process
1. PR merged to main
2. CI runs tests
3. Builds Docker image
4. Deploys to staging
5. Manual approval
6. Deploys to production

## Rollback
Run: `kubectl rollout undo deployment/app`
```

### Branch Strategy

**Common Strategies**:

1. **GitHub Flow** (Simple):
   - `main` (production)
   - Feature branches

2. **Git Flow** (Complex):
   - `main` (production)
   - `develop` (staging)
   - `feature/*` (features)
   - `release/*` (releases)
   - `hotfix/*` (hotfixes)

**Workflow Triggers**:
```yaml
on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]
    tags: ['v*']
```

### Code Review Integration

Require CI to pass before merge:

1. Settings > Branches > Branch protection rules
2. Require status checks to pass:
   - CI / lint
   - CI / test
   - CI / build
   - Security Scan / dependency-scan

### Team Training

**Onboarding Checklist**:
- [ ] Review CI/CD documentation
- [ ] Understand workflow triggers
- [ ] Know how to read workflow logs
- [ ] Practice creating and reviewing PRs
- [ ] Learn rollback procedures
- [ ] Understand monitoring and alerts

## Checklist

Use this checklist for new projects:

### Pipeline Setup
- [ ] CI workflow for pull requests
- [ ] CD workflow for deployments
- [ ] Security scanning enabled
- [ ] Code coverage tracking
- [ ] Artifact storage configured

### Security
- [ ] No secrets in code
- [ ] GitHub Secrets configured
- [ ] Branch protection enabled
- [ ] Dependency scanning enabled
- [ ] Action versions pinned

### Performance
- [ ] Dependency caching enabled
- [ ] Jobs run in parallel where possible
- [ ] Pipeline completes in < 10 minutes
- [ ] Docker layer caching configured

### Deployment
- [ ] Staging environment configured
- [ ] Production requires approval
- [ ] Smoke tests after deployment
- [ ] Rollback procedure documented
- [ ] Deployment notifications configured

### Monitoring
- [ ] Workflow metrics tracked
- [ ] Deployment tracking enabled
- [ ] Failure notifications configured
- [ ] Regular security scans scheduled

### Documentation
- [ ] README with setup instructions
- [ ] Workflow documentation
- [ ] Required secrets documented
- [ ] Deployment process documented
- [ ] Rollback procedure documented

## Resources

### Official Documentation
- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Security Hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

### Tools
- [act](https://github.com/nektos/act) - Run GitHub Actions locally
- [actionlint](https://github.com/rhysd/actionlint) - Lint workflow files
- [GitHub Actions Toolkit](https://github.com/actions/toolkit) - Build custom actions

### Learning
- [GitHub Learning Lab](https://lab.github.com/)
- [GitHub Actions Workshop](https://github.com/skills/hello-github-actions)
- [Awesome Actions](https://github.com/sdras/awesome-actions)

### Communities
- [GitHub Community](https://github.com/community)
- [GitHub Actions Forum](https://github.community/c/code-to-cloud/github-actions)
- [DevOps Subreddit](https://reddit.com/r/devops)

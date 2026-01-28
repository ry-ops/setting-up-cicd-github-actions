# CI/CD Setup Guide

This guide will walk you through setting up GitHub Actions CI/CD for your project.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Configuring Workflows](#configuring-workflows)
4. [Setting Up Secrets](#setting-up-secrets)
5. [Testing Your Setup](#testing-your-setup)
6. [Troubleshooting](#troubleshooting)

## Prerequisites

Before you begin, ensure you have:

- A GitHub account with a repository
- Basic understanding of Git and GitHub
- Your application code ready to deploy
- Access to your deployment environment (if applicable)

## Initial Setup

### 1. Create Workflow Directory

Create the `.github/workflows` directory in your repository root:

```bash
mkdir -p .github/workflows
```

### 2. Choose Your Workflow Templates

Based on your technology stack, copy the appropriate workflow files:

#### For Node.js Projects

Copy the following workflows:
- `ci.yml` - Runs tests and linting on pull requests
- `cd.yml` - Deploys to staging and production
- `security-scan.yml` - Scans for security vulnerabilities

```bash
cp .github/workflows/ci.yml your-project/.github/workflows/
cp .github/workflows/cd.yml your-project/.github/workflows/
cp .github/workflows/security-scan.yml your-project/.github/workflows/
```

#### For Python Projects

```bash
cp examples/python-ci.yml your-project/.github/workflows/ci.yml
```

#### For Docker Projects

```bash
cp examples/docker-build.yml your-project/.github/workflows/docker.yml
```

#### For Kubernetes Deployments

```bash
cp examples/k8s-deploy.yml your-project/.github/workflows/deploy.yml
```

### 3. Customize Workflow Files

Edit the copied workflow files to match your project structure:

#### Update Node Versions

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'  # Change to your required version
```

#### Update Working Directory

If your application is in a subdirectory:

```yaml
- name: Install dependencies
  working-directory: ./your-app-directory
  run: npm ci
```

#### Update Branch Names

Modify the trigger branches to match your workflow:

```yaml
on:
  pull_request:
    branches:
      - main        # Your main branch
      - develop     # Your development branch
```

## Configuring Workflows

### CI Workflow Configuration

The CI workflow runs on pull requests and includes:

1. **Linting**: Checks code quality and style
2. **Testing**: Runs unit and integration tests
3. **Building**: Verifies the application builds successfully
4. **Code Coverage**: Measures test coverage

Key configuration options:

```yaml
jobs:
  test:
    strategy:
      matrix:
        node-version: [18, 20]  # Test multiple versions
```

### CD Workflow Configuration

The CD workflow handles deployments:

1. **Staging Deployment**: Automatic deployment to staging
2. **Production Deployment**: Manual or automatic deployment to production
3. **Smoke Tests**: Verification after deployment

Configure environments in GitHub:

1. Go to your repository settings
2. Navigate to "Environments"
3. Create "staging" and "production" environments
4. Add required reviewers for production

### Security Scan Configuration

The security workflow runs:

1. **On Schedule**: Weekly scans
2. **On Push**: When code is merged to main
3. **On Pull Request**: Before merging changes

## Setting Up Secrets

### Required Secrets

Configure these in your repository (Settings > Secrets and variables > Actions):

#### For Deployment

```
DEPLOY_TOKEN          # Your deployment authentication token
```

#### For Docker

```
DOCKER_USERNAME       # Docker Hub username
DOCKER_PASSWORD       # Docker Hub access token
```

#### For Kubernetes

```
KUBE_CONFIG_STAGING      # Staging cluster config (base64 encoded)
KUBE_CONFIG_PRODUCTION   # Production cluster config (base64 encoded)
```

#### For Security Scanning

```
SNYK_TOKEN            # Snyk API token (optional)
```

### Creating Base64 Encoded Secrets

For Kubernetes config:

```bash
cat ~/.kube/config | base64
```

Copy the output and add it as a secret in GitHub.

### Creating Docker Hub Access Token

1. Log in to Docker Hub
2. Go to Account Settings > Security
3. Click "New Access Token"
4. Copy the token and add it as `DOCKER_PASSWORD` secret

## Testing Your Setup

### 1. Test CI Workflow

Create a test pull request:

```bash
git checkout -b test-ci
# Make a small change
echo "# Test" >> README.md
git add README.md
git commit -m "test: verify CI workflow"
git push origin test-ci
```

Then create a pull request on GitHub and verify:
- All checks pass
- Linting runs successfully
- Tests execute
- Build completes

### 2. Test CD Workflow

After merging to main:

1. Check the Actions tab
2. Verify the CD workflow triggers
3. Monitor the deployment to staging
4. Approve production deployment if required
5. Verify smoke tests pass

### 3. Test Security Scan

Trigger manually:

1. Go to Actions tab
2. Select "Security Scanning" workflow
3. Click "Run workflow"
4. Monitor the scan results

## Troubleshooting

### Common Issues

#### Workflow Not Triggering

**Problem**: Workflow doesn't run on push or PR

**Solution**:
- Check branch names in workflow file match your branches
- Ensure workflow file is in `.github/workflows/` directory
- Verify YAML syntax is correct
- Check that Actions are enabled in repository settings

#### Build Failures

**Problem**: Build fails in CI

**Solution**:
- Test build locally first: `npm ci && npm run build`
- Check Node.js version matches between local and CI
- Verify all dependencies are in `package.json`
- Check for environment-specific issues

#### Authentication Errors

**Problem**: Deployment fails with authentication error

**Solution**:
- Verify secrets are set correctly in repository settings
- Check secret names match exactly in workflow file
- Ensure tokens haven't expired
- Verify credentials have necessary permissions

#### Timeout Issues

**Problem**: Jobs timeout

**Solution**:
- Increase timeout in workflow:
  ```yaml
  jobs:
    test:
      timeout-minutes: 30  # Increase as needed
  ```
- Optimize slow tests
- Use caching for dependencies

### Checking Workflow Logs

1. Go to the Actions tab in your repository
2. Click on the failed workflow run
3. Click on the failed job
4. Expand the failed step to see detailed logs

### Getting Help

If you encounter issues:

1. Check the [GitHub Actions documentation](https://docs.github.com/en/actions)
2. Review workflow syntax in the [workflow syntax reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
3. Search for similar issues in GitHub Actions community forums
4. Open an issue in this repository

## Next Steps

After successful setup:

1. Read [WORKFLOWS.md](WORKFLOWS.md) for detailed workflow explanations
2. Review [BEST-PRACTICES.md](BEST-PRACTICES.md) for optimization tips
3. Customize workflows for your specific needs
4. Set up notifications for workflow failures
5. Configure branch protection rules

## Advanced Configuration

### Matrix Builds

Test across multiple versions:

```yaml
strategy:
  matrix:
    node-version: [18, 20, 22]
    os: [ubuntu-latest, windows-latest, macos-latest]
```

### Conditional Execution

Run jobs only when specific files change:

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
```

### Reusable Workflows

Create reusable workflows for common tasks:

```yaml
jobs:
  call-workflow:
    uses: your-org/shared-workflows/.github/workflows/test.yml@main
```

### Custom Actions

Create custom actions for repeated tasks. See [GitHub Actions documentation](https://docs.github.com/en/actions/creating-actions) for details.

## Security Best Practices

1. **Never commit secrets** - Always use GitHub Secrets
2. **Limit secret access** - Use environment-specific secrets
3. **Rotate tokens regularly** - Update access tokens periodically
4. **Use least privilege** - Grant minimum necessary permissions
5. **Enable branch protection** - Require status checks before merging
6. **Review dependencies** - Keep dependencies updated and secure

## Monitoring and Maintenance

### Regular Tasks

- Review workflow runs weekly
- Update actions to latest versions monthly
- Review and update dependencies regularly
- Monitor security scan results
- Optimize slow workflows

### Metrics to Track

- Build success rate
- Average build time
- Deployment frequency
- Time to deploy
- Failed deployment rate

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Actions Marketplace](https://github.com/marketplace?type=actions)
- [Security Hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

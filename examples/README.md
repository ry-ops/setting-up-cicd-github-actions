# Workflow Examples

This directory contains example GitHub Actions workflows for different technology stacks and use cases.

## Available Examples

### python-ci.yml
Complete CI pipeline for Python applications including:
- Linting with Black, isort, Flake8, and Pylint
- Testing across multiple Python versions (3.9-3.12)
- Type checking with mypy
- Security scanning with Safety and Bandit
- Coverage reporting

**Use this for**: Flask, Django, FastAPI, or any Python application

### docker-build.yml
Docker image build and publish workflow with:
- Multi-platform builds (amd64, arm64)
- Push to Docker Hub and GitHub Container Registry
- Automated tagging strategies
- Security scanning with Trivy and Snyk
- Container testing
- Release automation

**Use this for**: Containerized applications, microservices

### k8s-deploy.yml
Kubernetes deployment workflow featuring:
- Build and push Docker images
- Deploy to staging and production
- Canary deployment strategy
- Automatic rollback on failure
- Health checks and smoke tests
- Manual approval for production

**Use this for**: Applications deployed to Kubernetes clusters

## How to Use These Examples

1. **Copy the example** to your repository's `.github/workflows/` directory
2. **Rename the file** to something meaningful (e.g., `ci.yml`)
3. **Customize the workflow** for your project:
   - Update environment variables
   - Modify job steps for your build process
   - Adjust trigger conditions
   - Configure secrets in GitHub repository settings
4. **Test the workflow** by creating a pull request or pushing to the trigger branch

## Customization Guide

### Python CI Example

```yaml
# Change Python versions
strategy:
  matrix:
    python-version: ['3.10', '3.11', '3.12']

# Customize linting rules
- run: black --check . --line-length 100

# Add your requirements file
- run: pip install -r requirements.txt
```

### Docker Build Example

```yaml
# Change registry
env:
  REGISTRY: your-registry.io
  IMAGE_NAME: your-org/your-app

# Add build arguments
build-args: |
  NODE_ENV=production
  API_URL=https://api.example.com
```

### Kubernetes Deploy Example

```yaml
# Change namespace
kubectl apply -f k8s/ -n your-namespace

# Update deployment name
kubectl set image deployment/your-app app=$IMAGE:$TAG

# Customize health checks
curl -f https://your-app.com/health || exit 1
```

## Required Secrets

Each workflow may require different secrets:

### Docker Workflows
- `DOCKER_USERNAME` - Docker Hub username
- `DOCKER_PASSWORD` - Docker Hub access token

### Kubernetes Workflows
- `KUBE_CONFIG_STAGING` - Staging cluster kubeconfig (base64)
- `KUBE_CONFIG_PRODUCTION` - Production cluster kubeconfig (base64)

### Security Scanning
- `SNYK_TOKEN` - Snyk API token (optional but recommended)

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Actions Marketplace](https://github.com/marketplace?type=actions)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)

## Contributing

If you have additional workflow examples that would benefit others, please submit a pull request!

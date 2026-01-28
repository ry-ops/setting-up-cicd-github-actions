# Setting Up CI/CD with GitHub Actions

A comprehensive guide and production-ready template repository for implementing CI/CD pipelines using GitHub Actions. This repository includes working examples, best practices, and reusable workflow templates for Node.js, Python, Docker, and Kubernetes deployments.

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/ry-ops/setting-up-cicd-github-actions.git
cd setting-up-cicd-github-actions
```

### 2. Install Dependencies (for Node.js sample app)

```bash
cd app
npm install
```

### 3. Run Tests Locally

```bash
npm test
```

### 4. Start the Application

```bash
npm start
```

The sample application will be available at `http://localhost:3000`.

### 5. Customize Workflows

Copy the workflow files from `.github/workflows/` to your own repository and customize them according to your needs. See [documentation/WORKFLOWS.md](documentation/WORKFLOWS.md) for detailed configuration options.

## Project Structure

```
setting-up-cicd-github-actions/
├── LICENSE                      # MIT License
├── README.md                    # This file
├── app/                         # Sample Node.js application
│   ├── package.json            # Node.js dependencies and scripts
│   ├── index.js                # Express.js application
│   └── tests/                  # Test files
│       └── app.test.js         # Jest test suite
├── .github/
│   └── workflows/              # GitHub Actions workflows
│       ├── ci.yml              # Continuous Integration
│       ├── cd.yml              # Continuous Deployment
│       └── security-scan.yml   # Security scanning
├── examples/                    # Workflow templates for different stacks
│   ├── python-ci.yml           # Python CI workflow
│   ├── docker-build.yml        # Docker build and push
│   └── k8s-deploy.yml          # Kubernetes deployment
└── documentation/              # Detailed documentation
    ├── SETUP.md                # Initial setup guide
    ├── WORKFLOWS.md            # Workflow explanations
    └── BEST-PRACTICES.md       # CI/CD best practices
```

## Features

### Production-Ready Workflows

- **Continuous Integration (CI)**: Automatically runs on pull requests
  - Code linting (ESLint)
  - Unit and integration tests (Jest)
  - Build verification
  - Code coverage reporting

- **Continuous Deployment (CD)**: Automatically deploys on merge to main
  - Build and test verification
  - Automated deployment to staging/production
  - Rollback capabilities

- **Security Scanning**: Regular security checks
  - Dependency vulnerability scanning
  - SAST (Static Application Security Testing)
  - License compliance

### Sample Application

A fully functional Node.js/Express.js application with:
- RESTful API endpoints
- Comprehensive test coverage
- Production-ready configuration
- Health check endpoints

### Examples for Multiple Stacks

Ready-to-use workflow templates for:
- **Python**: Flask/Django applications with pytest
- **Docker**: Multi-stage builds with image optimization
- **Kubernetes**: Automated deployments with rolling updates

## Examples

### Basic CI Workflow

The CI workflow automatically runs on every pull request:

```yaml
on:
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

### Deploying to Production

The CD workflow deploys to production when code is merged to main:

```yaml
on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and Deploy
        run: |
          npm ci
          npm run build
          # Deploy to your hosting platform
```

### Security Scanning

Automated security checks run on a schedule:

```yaml
on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly on Sunday
  push:
    branches: [ main ]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run security scan
        uses: aquasecurity/trivy-action@master
```

## Documentation

- [SETUP.md](documentation/SETUP.md) - Step-by-step setup instructions for new projects
- [WORKFLOWS.md](documentation/WORKFLOWS.md) - Detailed explanation of each workflow
- [BEST-PRACTICES.md](documentation/BEST-PRACTICES.md) - CI/CD best practices and optimization tips

## Getting Started with Your Project

1. Copy the `.github/workflows/` directory to your repository
2. Customize the workflows based on your stack (see `examples/` for templates)
3. Configure required secrets in your GitHub repository settings
4. Update workflow triggers and conditions as needed
5. Test your workflows by creating a pull request

## Required GitHub Secrets

Configure these secrets in your repository settings (Settings > Secrets and variables > Actions):

- `DEPLOY_TOKEN`: Authentication token for deployment
- `DOCKER_USERNAME`: Docker Hub username (if using Docker)
- `DOCKER_PASSWORD`: Docker Hub password or access token
- `KUBE_CONFIG`: Kubernetes config file (base64 encoded, if using K8s)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)

## Support

For questions and support, please open an issue in this repository.

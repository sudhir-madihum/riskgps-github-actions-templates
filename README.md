# RiskGPS GitHub Actions Centralized Pipeline Templates

> **Single Source of Truth for All CI/CD Pipelines**

## Overview

This repository contains **reusable GitHub Actions workflows** for all RiskGPS services. Instead of maintaining identical pipeline code in 4+ repositories, we maintain it once here and all services call these templates.

### 🎯 Key Benefits

| Benefit | Impact |
|---------|--------|
| **Single Source of Truth** | Update pipeline logic once, applies everywhere |
| **Reduced Maintenance** | No code duplication across 4+ repos |
| **Consistency** | All services follow identical patterns |
| **Faster Updates** | New feature/fix deployed to all services immediately |
| **Easier Debugging** | One place to fix issues across all pipelines |

---

## 📁 Repository Structure

```
riskgps-github-actions-templates/
├── .github/workflows/                 # Reusable workflow templates
│   ├── determine-environment.yml       # Environment detection (dev/prod/feature)
│   ├── build-scan-push-nodejs.yml      # Node.js build, test, scan, push
│   └── build-scan-push-python.yml      # Python build, test, scan, push
├── examples/                           # Example implementations per service
│   ├── release-frontend-example.yml    # How frontend uses the templates
│   ├── release-backend-app-example.yml # How backend-app uses the templates
│   └── release-backend-agent-example.yml # How backend-agent uses the templates
└── docs/                               # Documentation
    ├── SETUP.md                        # How to set up
    ├── USAGE.md                        # How to use
    └── TROUBLESHOOTING.md              # Common issues
```

---

## 🚀 Quick Start

### Step 1: Create This Template Repository

```bash
# Create new repo called riskgps-github-actions-templates
# Clone and add these files
git clone https://github.com/<your-org>/riskgps-github-actions-templates.git
cd riskgps-github-actions-templates
```

### Step 2: Update Service Repositories

For **riskgps-frontend**, **riskgps-backend-app**, **riskgps-backend-agent**, **riskgps-backend-connector**:

Replace `.github/workflows/release.yml` with the appropriate example file:

```yaml
# riskgps-frontend/.github/workflows/release.yml
name: Frontend Build, Scan, and Push to AWS ECR

on:
  push:
    branches: ['**']
  pull_request:
    branches: ['dev', 'prod']
  workflow_dispatch:

jobs:
  determine-environment:
    uses: <your-org>/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@main
    with:
      dev-branch: dev
      prod-branch: prod

  build-scan-push:
    needs: determine-environment
    uses: <your-org>/riskgps-github-actions-templates/.github/workflows/build-scan-push-nodejs.yml@main
    with:
      service-name: frontend
      dockerfile-path: ./Dockerfile

  # ... rest of workflow
```

---

## 📚 Available Templates

### 1. `determine-environment.yml`

**Purpose**: Determines environment (dev/prod/feature) and whether to push to ECR

**Inputs**:
```yaml
with:
  dev-branch: "dev"              # Development branch name
  prod-branch: "prod"            # Production branch name
  skip-tests-on-prod: true       # Skip tests on prod merges
```

**Outputs**:
```yaml
environment: "dev"              # dev, prod, or feature
should-push-ecr: "true"         # Push to ECR (true only on merge)
run-tests: "true"               # Run tests (false on prod)
should-build: "true"            # Build Docker image
```

**Usage**:
```yaml
jobs:
  determine:
    uses: <org>/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@main
    with:
      dev-branch: dev
      prod-branch: prod
```

---

### 2. `build-scan-push-nodejs.yml`

**Purpose**: Build, test, lint, and scan Node.js services

**Inputs**:
```yaml
with:
  service-name: "frontend"           # Service name for tagging
  dockerfile-path: "./Dockerfile"    # Path to Dockerfile
  docker-build-context: "."          # Docker build context
  node-version: "18"                 # Node.js version
  npm-cache: "npm"                   # npm, yarn, pnpm
  run-tests: true                    # Run npm test
  run-linter: true                   # Run npm lint
```

**Outputs**:
```yaml
image-tag: "v1.0.42"           # Generated tag (v1.0.{run_number})
image-name: "frontend:v1.0.42" # Full image name
```

**Usage**:
```yaml
jobs:
  build:
    needs: determine-environment
    uses: <org>/riskgps-github-actions-templates/.github/workflows/build-scan-push-nodejs.yml@main
    with:
      service-name: frontend
      run-tests: ${{ needs.determine-environment.outputs.run-tests == 'true' }}
```

---

### 3. `build-scan-push-python.yml`

**Purpose**: Build, test, lint, and scan Python services

**Inputs**:
```yaml
with:
  service-name: "backend-agent"      # Service name for tagging
  dockerfile-path: "./Dockerfile"    # Path to Dockerfile
  docker-build-context: "."          # Docker build context
  python-version: "3.11"             # Python version
  run-tests: true                    # Run pytest
  run-linter: true                   # Run flake8
```

**Outputs**:
```yaml
image-tag: "v1.0.42"                # Generated tag
image-name: "backend-agent:v1.0.42" # Full image name
```

**Usage**:
```yaml
jobs:
  build:
    needs: determine-environment
    uses: <org>/riskgps-github-actions-templates/.github/workflows/build-scan-push-python.yml@main
    with:
      service-name: backend-agent
      python-version: "3.11"
      run-tests: ${{ needs.determine-environment.outputs.run-tests == 'true' }}
```

---

## 🏗️ Workflow Architecture

### How It Works

```
Your Service Repo (.github/workflows/release.yml)
    │
    ├─→ Call: determine-environment.yml@main
    │        └─→ Outputs: environment, should-push-ecr, run-tests
    │
    ├─→ Call: build-scan-push-nodejs.yml@main (or python)
    │   (depends on determine-environment)
    │        └─→ Outputs: image-name, image-tag
    │
    └─→ Call: Push to ECR step
        (conditionally, based on should-push-ecr)
```

### Decision Flow

```
GitHub Event
    │
    ├─ PUSH to dev?      → should-push-ecr=true, environment=dev
    ├─ PUSH to prod?     → should-push-ecr=true, environment=prod
    ├─ PUSH to feature?  → should-push-ecr=false, run-tests=true
    │
    └─ PR to dev/prod?   → should-push-ecr=false, build anyway
```

---

## ⚙️ Configuration

### Global Variables (In Each Service Repo)

Every service repo should have these in `.github/workflows/release.yml`:

```yaml
env:
  AWS_REGION: us-east-1
  SERVICE_NAME: frontend              # Different per service
  DEV_BRANCH: dev                     # Same across all
  PROD_BRANCH: prod                   # Same across all
  ORGANIZATION: bluocean             # Same across all
  PROJECT: riskgps                    # Same across all
```

**Change Branch Names**: Update in ONE place (each service's `env:` section)

---

## 📋 Examples

### Example 1: Frontend (Node.js)

```yaml
# riskgps-frontend/.github/workflows/release.yml
name: Frontend Build, Scan, and Push

on:
  push:
    branches: ['**']
  pull_request:
    branches: ['dev', 'prod']
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

concurrency:
  group: ${{ github.ref }}
  cancel-in-progress: true

env:
  AWS_REGION: us-east-1
  SERVICE_NAME: frontend
  DEV_BRANCH: dev
  PROD_BRANCH: prod
  ORGANIZATION: bluocean
  PROJECT: riskgps

jobs:
  determine-environment:
    uses: bluocean/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@main
    with:
      dev-branch: dev
      prod-branch: prod

  build-scan-push:
    needs: determine-environment
    uses: bluocean/riskgps-github-actions-templates/.github/workflows/build-scan-push-nodejs.yml@main
    with:
      service-name: frontend
      dockerfile-path: ./Dockerfile
      node-version: "18"
      run-tests: ${{ needs.determine-environment.outputs.run-tests == 'true' }}

  push-to-ecr:
    name: Push to AWS ECR
    runs-on: ubuntu-latest
    needs: [build-scan-push, determine-environment]
    if: needs.determine-environment.outputs.should-push-ecr == 'true'
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::739962689681:role/GithubActionRole
          aws-region: us-east-1
      - run: |
          # Push image to ECR using determine-environment outputs
          echo "Pushing to ECR for ${{ needs.determine-environment.outputs.environment }}"
```

---

## 🔄 Maintenance & Updates

### Updating Pipeline Logic

**Old Way**: Edit 4+ repos independently
```
riskgps-frontend/.github/workflows/release.yml       ← Update
riskgps-backend-app/.github/workflows/release.yml    ← Update
riskgps-backend-agent/.github/workflows/release.yml  ← Update
riskgps-backend-connector/.github/workflows/release.yml ← Update
```

**New Way**: Edit templates once
```
riskgps-github-actions-templates/.github/workflows/
  ├── determine-environment.yml  ← Update here
  ├── build-scan-push-nodejs.yml ← Update here
  └── build-scan-push-python.yml ← Update here

All services automatically use updated templates on next run
```

### Release Tags

Use semantic versioning for templates:

```bash
git tag v1.0.0
git push origin v1.0.0
```

In service repos:
```yaml
uses: bluocean/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@v1.0.0
```

---

## ✅ Migration Checklist

- [ ] Create `riskgps-github-actions-templates` repository
- [ ] Push templates to templates repo
- [ ] Update `riskgps-frontend/.github/workflows/release.yml`
- [ ] Update `riskgps-backend-app/.github/workflows/release.yml`
- [ ] Update `riskgps-backend-agent/.github/workflows/release.yml`
- [ ] Update `riskgps-backend-connector/.github/workflows/release.yml`
- [ ] Test each service with a feature branch push
- [ ] Test with PR to dev
- [ ] Test with merge to dev
- [ ] Test with merge to prod
- [ ] Document branch/environment configuration

---

## 🔗 References

- [GitHub Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Workflow Inputs](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onworkflow_callinputs)
- [Workflow Outputs](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onworkflow_calloutputs)

---

## 📞 Support

For questions or issues:
1. Check [TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md)
2. Review [USAGE.md](./docs/USAGE.md)
3. Open an issue in this repository
#   r i s k g p s - g i t h u b - a c t i o n s - t e m p l a t e s 
 
 
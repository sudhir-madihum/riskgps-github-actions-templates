# Reusable Frontend Build, Scan, and Push to ECR Workflow

This is a centralized, reusable GitHub Actions workflow template for building, scanning, and pushing frontend applications to AWS ECR.

## Overview

The workflow is designed to:
- Automatically detect the environment (dev, prod, or feature)
- Run tests on feature branches and dev merges
- Scan dependencies and container images with Trivy
- Build Docker images
- Push to ECR only on merge to `dev` or `prod` branches
- Generate detailed workflow summaries

## Benefits

✅ **Single Source of Truth**: One template maintained centrally  
✅ **Consistent CI/CD**: All frontend services follow the same pipeline  
✅ **Easy to Update**: Changes propagate to all services using the template  
✅ **Reusable Parameters**: Customize for different services and configurations  
✅ **Same Logic**: Maintains the original workflow logic without modifications  

## Template Location

```
riskgps-github-actions-templates/.github/workflows/frontend-build-scan-push-template.yml
```

## How to Use

### 1. In Your Frontend Application Repository

Create or update `.github/workflows/release.yml`:

```yaml
name: Frontend Build, Scan, and Push to AWS ECR

on:
  push:
    branches:
      - '**'
    paths:
      - 'src/**'
      - 'public/**'
      - 'package.json'
      - 'Dockerfile'
      - 'tsconfig.json'
      - 'next.config.ts'
      - '.github/workflows/release.yml'
  pull_request:
    branches:
      - 'dev'
      - 'prod'
    paths:
      - 'src/**'
      - 'public/**'
      - 'package.json'
      - 'Dockerfile'
      - 'tsconfig.json'
      - 'next.config.ts'
      - '.github/workflows/release.yml'
  workflow_dispatch:

jobs:
  call-build-template:
    name: Call Frontend Build Template
    uses: bluocean/riskgps-github-actions-templates/.github/workflows/frontend-build-scan-push-template.yml@main
    with:
      service-name: frontend
      organization: bluocean
      project: riskgps
      aws-region: us-east-1
      dev-branch: dev
      prod-branch: prod
      dockerfile-path: Dockerfile
      build-context: .
      iam-role-arn: arn:aws:iam::739962689681:role/GithubActionRole
```

### 2. Update the `uses` Reference

Replace the repository reference with your actual template repository:

```yaml
uses: <organization>/<template-repo>/.github/workflows/frontend-build-scan-push-template.yml@<branch/tag>
```

Examples:
- `bluocean/riskgps-github-actions-templates/.github/workflows/frontend-build-scan-push-template.yml@main`
- `bluocean/riskgps-github-actions-templates/.github/workflows/frontend-build-scan-push-template.yml@v1.0`

## Available Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `service-name` | ✅ | - | Service name (e.g., `frontend`, `api`) |
| `organization` | ✅ | - | Organization name (e.g., `bluocean`) |
| `project` | ✅ | - | Project name (e.g., `riskgps`) |
| `aws-region` | ❌ | `us-east-1` | AWS region for ECR and SSM |
| `dev-branch` | ❌ | `dev` | Development branch name |
| `prod-branch` | ❌ | `prod` | Production branch name |
| `dockerfile-path` | ❌ | `Dockerfile` | Path to Dockerfile in repository |
| `build-context` | ❌ | `.` | Docker build context directory |
| `iam-role-arn` | ✅ | - | IAM Role ARN for AWS credentials |

## Environment Detection Logic

### Push Events
- **To `dev` branch**: Environment=`dev`, Run tests, Build, **Push to ECR**
- **To `prod` branch**: Environment=`prod`, Skip tests, Build, **Push to ECR**
- **To other branches**: Environment=`feature`, Run tests, Build, **Do NOT push to ECR**

### Pull Request Events
- **To `dev` branch**: Environment=`dev`, Run tests, Build, **Do NOT push to ECR**
- **To `prod` branch**: Environment=`prod`, Skip tests, Build, **Do NOT push to ECR**
- **To other branches**: Environment=`feature`, Run tests, Build, **Do NOT push to ECR**

## Workflow Steps

1. **Determine Environment**: Analyzes the branch and event type
2. **Run Tests**: (if applicable) Runs linter and unit tests
3. **Scan Files**: Scans dependencies for vulnerabilities
4. **Build, Scan, Push**:
   - Builds Docker image
   - Scans with Trivy
   - Pushes to ECR (only for dev/prod merges)
5. **Workflow Summary**: Generates detailed job summary

## Requirements

### In Your Application Repository

- **Node.js project**: `package.json`, `npm install`, `npm test`, `npm run lint`
- **Docker**: `Dockerfile` in repository root (or custom path)
- **GitHub Actions Access**: Repository must allow GitHub Actions

### In AWS

- **SSM Parameters**: Must have ECR credentials stored in SSM Parameter Store
  - Path: `/{organization}/{project}/{environment}/ec2/credentials`
  - Format: JSON with `ecr_registry_repository_urls` object
  ```json
  {
    "ecr_registry_repository_urls": {
      "frontend": "123456789.dkr.ecr.us-east-1.amazonaws.com/riskgps-frontend",
      "api": "123456789.dkr.ecr.us-east-1.amazonaws.com/riskgps-api"
    }
  }
  ```

- **IAM Role**: Must have permissions for:
  - SSM Parameter Store (read, decrypt)
  - ECR (login, push)

## Examples

### Example 1: RiskGPS Frontend

```yaml
uses: bluocean/riskgps-github-actions-templates/.github/workflows/frontend-build-scan-push-template.yml@main
with:
  service-name: frontend
  organization: bluocean
  project: riskgps
  iam-role-arn: arn:aws:iam::739962689681:role/GithubActionRole
```

### Example 2: Different Project

```yaml
uses: bluocean/riskgps-github-actions-templates/.github/workflows/frontend-build-scan-push-template.yml@main
with:
  service-name: ui
  organization: myorg
  project: myapp
  aws-region: us-west-2
  dev-branch: develop
  prod-branch: main
  dockerfile-path: ./docker/Dockerfile
  build-context: ./apps/frontend
  iam-role-arn: arn:aws:iam::123456789:role/GitHubActionsRole
```

## Versioning

To ensure stability, always reference a specific version:

```yaml
# Reference a branch (automatic updates)
uses: bluocean/riskgps-github-actions-templates/.github/workflows/frontend-build-scan-push-template.yml@main

# Reference a tag (recommended for production)
uses: bluocean/riskgps-github-actions-templates/.github/workflows/frontend-build-scan-push-template.yml@v1.0.0
```

## Troubleshooting

### Workflow Fails on SSM Parameter Fetch

**Error**: `Failed to fetch credentials from SSM: /{org}/{project}/{env}/ec2/credentials`

**Solution**: 
1. Verify SSM parameter exists in AWS Systems Manager
2. Check IAM role has `ssm:GetParameter` permission
3. Ensure parameter path matches exactly

### Image Not Pushed to ECR

**Check**:
1. Are you pushing to `dev` or `prod` branch?
2. Is `should-push-ecr` output showing `true`?
3. Verify IAM role has ECR push permissions

### Tests Failing

**For `prod` branch merges**: Tests are intentionally skipped. 

**For `dev` and feature branches**: Check test logs in workflow output.

## Customization

### Modify Test Commands

Edit the template's `run-tests` job if your project has different test scripts:

```yaml
- name: Run unit tests
  run: npm test -- --passWithNoTests --coverage
```

### Add More Inputs

To add new inputs, update the template's `on.workflow_call.inputs` section:

```yaml
on:
  workflow_call:
    inputs:
      new-input:
        description: "Description"
        required: false
        default: "value"
        type: string
```

### Extend the Pipeline

Add new jobs to the template that run before `build-scan-push`:

```yaml
jobs:
  new-job:
    runs-on: ubuntu-latest
    steps: [...]
  
  build-scan-push:
    needs: [new-job, scan-files, determine-environment, run-tests]
```

## Migration Guide

If you have existing workflows in each service:

1. Create the template in `riskgps-github-actions-templates`
2. Update each service's workflow to call the template
3. Test on a feature branch first
4. Merge to `dev` for production deployment

## Support

For issues or improvements:
1. Update the template in `riskgps-github-actions-templates`
2. Create a new release/tag
3. Notify teams to update their `uses` reference

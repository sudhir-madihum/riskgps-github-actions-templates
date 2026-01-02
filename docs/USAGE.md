# Central Pipeline Templates - Usage Guide

## Table of Contents

1. [Basic Usage](#basic-usage)
2. [Customization](#customization)
3. [Common Scenarios](#common-scenarios)
4. [Branch Strategies](#branch-strategies)
5. [Advanced Configuration](#advanced-configuration)

---

## Basic Usage

### Minimum Workflow (Using All Templates)

```yaml
# .github/workflows/release.yml

name: Service Build and Deploy

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
  SERVICE_NAME: my-service
  DEV_BRANCH: dev
  PROD_BRANCH: prod
  ORGANIZATION: bluocean
  PROJECT: riskgps

jobs:
  # Step 1: Determine environment
  determine-environment:
    uses: bluocean/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@main
    with:
      dev-branch: ${{ env.DEV_BRANCH }}
      prod-branch: ${{ env.PROD_BRANCH }}

  # Step 2: Build and scan (Node.js example)
  build-scan-push:
    needs: determine-environment
    uses: bluocean/riskgps-github-actions-templates/.github/workflows/build-scan-push-nodejs.yml@main
    with:
      service-name: ${{ env.SERVICE_NAME }}
      dockerfile-path: ./Dockerfile
      run-tests: ${{ needs.determine-environment.outputs.run-tests == 'true' }}

  # Step 3: Push to ECR (conditional)
  push-to-ecr:
    runs-on: ubuntu-latest
    needs: [build-scan-push, determine-environment]
    if: needs.determine-environment.outputs.should-push-ecr == 'true'
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::739962689681:role/GithubActionRole
          aws-region: ${{ env.AWS_REGION }}
      - run: |
          # Your ECR push logic here
          echo "Pushing to ECR..."
```

---

## Customization

### Node.js Service (Frontend/Backend-App)

Use `build-scan-push-nodejs.yml` template:

```yaml
build-scan-push:
  uses: bluocean/riskgps-github-actions-templates/.github/workflows/build-scan-push-nodejs.yml@main
  with:
    service-name: frontend
    dockerfile-path: ./Dockerfile
    docker-build-context: .
    node-version: "18"           # Customize Node version
    npm-cache: npm               # Use npm, yarn, or pnpm
    run-tests: true              # Run npm test
    run-linter: true             # Run npm lint
```

### Python Service (Backend-Agent/Connector)

Use `build-scan-push-python.yml` template:

```yaml
build-scan-push:
  uses: bluocean/riskgps-github-actions-templates/.github/workflows/build-scan-push-python.yml@main
  with:
    service-name: backend-agent
    dockerfile-path: ./Dockerfile
    docker-build-context: .
    python-version: "3.11"       # Customize Python version
    run-tests: true              # Run pytest
    run-linter: true             # Run flake8
```

### Skip Tests on Specific Branch

```yaml
determine-environment:
  uses: bluocean/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@main
  with:
    dev-branch: dev
    prod-branch: prod
    skip-tests-on-prod: true     # Don't run tests on prod merges
```

---

## Common Scenarios

### Scenario 1: Feature Branch Push

**What Happens**:
1. `determine-environment.yml` detects feature branch
2. Sets `should-push-ecr=false`, `run-tests=true`
3. Builds image but does NOT push to ECR
4. Developer can verify locally before PR

**Your Workflow Does**:
```bash
push to feature/my-feature
  ↓
determine-environment (should-push-ecr=false, run-tests=true)
  ↓
build-scan-push (runs tests, builds image)
  ↓
skip ECR push (because should-push-ecr=false)
```

---

### Scenario 2: PR to Development

**What Happens**:
1. Open PR from feature → dev
2. Template detects PR event + dev target
3. Runs tests, builds, but does NOT push
4. Code review happens
5. After approval, merge runs ECR push

**Your Workflow Does**:
```bash
PR: feature/my-feature → dev
  ↓
determine-environment (should-push-ecr=false, run-tests=true)
  ↓
build-scan-push (runs tests, builds)
  ↓
skip ECR push (PR, not merge)
  ↓
[Code Review]
  ↓
Merge PR
  ↓
determine-environment (should-push-ecr=true, run-tests=true)
  ↓
build-scan-push + PUSH TO ECR
```

---

### Scenario 3: Merge to Production

**What Happens**:
1. Merge dev → prod branch
2. Template detects prod branch push
3. Skips tests (faster deployment)
4. Scans and builds
5. Pushes to prod ECR

**Your Workflow Does**:
```bash
Merge: dev → prod
  ↓
determine-environment (should-push-ecr=true, run-tests=false)
  ↓
build-scan-push (skips tests, faster)
  ↓
PUSH TO PROD ECR
```

---

## Branch Strategies

### Strategy 1: GitFlow (dev + prod)

**Setup**:
```yaml
env:
  DEV_BRANCH: dev
  PROD_BRANCH: prod
```

**Workflow**:
```
feature/xxx ──→ PR to dev ──→ merge ──→ PUSH TO dev ECR
                  (build, test)    (build, test, push)

dev ──→ PR to prod ──→ merge ──→ PUSH TO prod ECR
           (build)        (scan, push)
```

### Strategy 2: Main Only

**Setup**:
```yaml
env:
  DEV_BRANCH: main
  PROD_BRANCH: main
```

**Workflow**:
```
feature/xxx ──→ PR to main ──→ merge ──→ PUSH TO ECR
                 (build, test)    (build, test, push)
```

### Strategy 3: Trunk-Based Development

**Setup**:
```yaml
env:
  DEV_BRANCH: main
  PROD_BRANCH: release
```

**Workflow**:
```
feature/xxx ──→ merge to main ──→ PUSH TO main ECR
               (build, test, push)

main ──→ tag release ──→ merge to release ──→ PUSH TO release ECR
```

---

## Advanced Configuration

### Custom Docker Build Options

In your service's `release.yml`:

```yaml
build-scan-push:
  needs: determine-environment
  uses: bluocean/riskgps-github-actions-templates/.github/workflows/build-scan-push-nodejs.yml@main
  with:
    service-name: frontend
    dockerfile-path: ./docker/Dockerfile.prod  # Custom path
    docker-build-context: ./app                # Custom context
    node-version: "20"                         # Latest Node
```

### Conditional Test Execution

Skip tests for specific branches:

```yaml
build-scan-push:
  needs: determine-environment
  uses: bluocean/riskgps-github-actions-templates/.github/workflows/build-scan-push-nodejs.yml@main
  with:
    service-name: frontend
    run-tests: ${{ needs.determine-environment.outputs.run-tests == 'true' }}
    # Only runs tests if determine-environment allows it
```

### Using Specific Template Versions

```yaml
determine-environment:
  uses: bluocean/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@v1.0.0
  # Lock to specific version for stability

build-scan-push:
  uses: bluocean/riskgps-github-actions-templates/.github/workflows/build-scan-push-nodejs.yml@main
  # Use latest for flexibility
```

### Custom ECR Push Logic

If default ECR push in service workflow doesn't match your needs:

```yaml
push-to-custom-ecr:
  needs: [build-scan-push, determine-environment]
  if: needs.determine-environment.outputs.should-push-ecr == 'true'
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::739962689681:role/GithubActionRole
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Custom ECR Push
      env:
        SERVICE: ${{ env.SERVICE_NAME }}
        TAG: ${{ needs.build-scan-push.outputs.image-tag }}
        IMAGE: ${{ needs.build-scan-push.outputs.image-name }}
      run: |
        # Your custom logic here
        ECR_REPO="<your-ecr-url>/$SERVICE"
        docker tag $IMAGE $ECR_REPO:$TAG
        docker push $ECR_REPO:$TAG
```

---

## Template Outputs Reference

### `determine-environment.yml` Outputs

```yaml
environment      # "dev", "prod", or "feature"
should-push-ecr  # "true" or "false"
run-tests        # "true" or "false"
should-build     # "true" (always)
```

### `build-scan-push-nodejs.yml` Outputs

```yaml
image-tag        # e.g., "v1.0.42"
image-name       # e.g., "frontend:v1.0.42"
```

### `build-scan-push-python.yml` Outputs

```yaml
image-tag        # e.g., "v1.0.42"
image-name       # e.g., "backend-agent:v1.0.42"
```

---

## Environment Variables Reference

In your service's `.github/workflows/release.yml`:

```yaml
env:
  AWS_REGION: us-east-1                   # AWS region for ECR
  SERVICE_NAME: frontend                  # Service identifier
  DEV_BRANCH: dev                         # Dev branch name
  PROD_BRANCH: prod                       # Prod branch name
  ORGANIZATION: bluocean                  # GitHub org
  PROJECT: riskgps                        # Project name
```

These are used by:
- `determine-environment.yml`: Branch comparison
- SSM parameter path: `/{ORGANIZATION}/{PROJECT}/{environment}/ec2/credentials`
- Image tagging: `{SERVICE_NAME}:v1.0.{run_number}`

---

## Best Practices

1. **Lock Template Versions in Production**
   ```yaml
   uses: org/templates/.github/workflows/xxx.yml@v1.0.0
   ```

2. **Use Global Variables for Branch Names**
   ```yaml
   env:
     DEV_BRANCH: dev   # Change once, applies everywhere
   ```

3. **Test Template Changes on Staging First**
   - Update template
   - Run tests on templates repo
   - Then use in service repos

4. **Document Custom Configurations**
   - Add comments in service's release.yml
   - Explain why defaults were changed

5. **Maintain Examples**
   - Keep examples/ directory updated
   - Use as reference for new services

---

## FAQ

**Q: How often do templates update?**
A: Whenever pipeline logic needs change. Use tags for stability.

**Q: Can I override template behavior?**
A: Yes, add custom steps before/after template calls.

**Q: What if a service needs unique pipeline logic?**
A: Create a custom template for that service in templates repo.

**Q: How do I test template changes?**
A: Push to templates repo, test with feature branch in service repo.

**Q: What if templates repo goes down?**
A: Workflows fail. Ensure templates repo high availability.

---

## Next Steps

- Review [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for common issues
- Check [examples/](../examples/) for reference implementations
- Set up templates repo and migrate services

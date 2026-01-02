# Central Pipeline Templates - Setup Guide

## Prerequisites

- GitHub organization account
- Read/Write access to create repositories
- All service repositories (frontend, backend-app, backend-agent, backend-connector)

---

## Step 1: Create Template Repository

```bash
# Create new repository
# Name: riskgps-github-actions-templates
# Visibility: Internal or Private (depending on org policy)

git clone https://github.com/<org>/riskgps-github-actions-templates.git
cd riskgps-github-actions-templates
```

---

## Step 2: Copy Template Files

Copy all files from this directory structure:

```
.github/workflows/
├── determine-environment.yml
├── build-scan-push-nodejs.yml
└── build-scan-push-python.yml

docs/
├── SETUP.md (this file)
├── USAGE.md
└── TROUBLESHOOTING.md

examples/
├── release-frontend-example.yml
├── release-backend-app-example.yml
└── release-backend-agent-example.yml

README.md
```

---

## Step 3: Push to Templates Repository

```bash
git add .
git commit -m "Initial: Central GitHub Actions templates"
git push origin main
```

---

## Step 4: Update Each Service Repository

### For `riskgps-frontend`:

1. Open `.github/workflows/release.yml`
2. Replace entire content with `examples/release-frontend-example.yml`
3. Update `<your-org>` placeholder:

```yaml
uses: <your-org>/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@main
```

Replace with:

```yaml
uses: bluocean/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@main
```

4. Commit and push:

```bash
git add .github/workflows/release.yml
git commit -m "refactor: use central pipeline templates"
git push
```

### For `riskgps-backend-app`:

Follow same steps as frontend, but use `release-backend-app-example.yml`

### For `riskgps-backend-agent`:

Follow same steps, but use `release-backend-agent-example.yml`

### For `riskgps-backend-connector`:

Follow same steps, but use `release-backend-agent-example.yml` (same as agent)

---

## Step 5: Test Templates

### Test 1: Feature Branch Push

```bash
cd riskgps-frontend
git checkout -b test-pipeline
echo "// test" >> src/app.tsx
git add src/app.tsx
git commit -m "test: pipeline"
git push origin test-pipeline
```

**Expected**: Pipeline runs, builds image, does NOT push to ECR

### Test 2: PR to Dev

```bash
# On test-pipeline branch
git push origin test-pipeline

# Create PR to dev on GitHub UI
```

**Expected**: Pipeline runs, builds image, does NOT push to ECR

### Test 3: Merge to Dev

```bash
# On GitHub: Merge PR to dev
```

**Expected**: Pipeline runs, builds image, PUSHES to dev ECR

### Test 4: Push to Prod

```bash
git checkout prod
git merge dev
git push origin prod
```

**Expected**: Pipeline runs, builds image, PUSHES to prod ECR (tests skipped)

---

## Step 6: Configure Workflow Permissions

Each service repo needs these permissions:

```yaml
permissions:
  id-token: write    # For AWS OIDC authentication
  contents: read     # For checkout
```

✅ Already included in example workflows

---

## Step 7: Verify AWS Configuration

Ensure each repo has these environment variables:

```yaml
env:
  AWS_REGION: us-east-1
  SERVICE_NAME: <service-name>  # frontend, backend-app, etc.
  DEV_BRANCH: dev
  PROD_BRANCH: prod
  ORGANIZATION: bluocean
  PROJECT: riskgps
```

✅ Already included in example workflows

---

## Step 8: Verify SSM Parameters

Templates expect SSM parameters at:

```
/bluocean/riskgps/dev/ec2/credentials
/bluocean/riskgps/prod/ec2/credentials
```

Each should contain JSON:

```json
{
  "ecr_registry_repository_urls": {
    "frontend": "739962689681.dkr.ecr.us-east-1.amazonaws.com/riskgps/frontend",
    "backend-app": "739962689681.dkr.ecr.us-east-1.amazonaws.com/riskgps/backend-app",
    "backend-agent": "739962689681.dkr.ecr.us-east-1.amazonaws.com/riskgps/backend-agent",
    "backend-connector": "739962689681.dkr.ecr.us-east-1.amazonaws.com/riskgps/backend-connector"
  }
}
```

---

## Troubleshooting Setup

### Issue: "Reusable workflow not found"

**Solution**: Ensure template repository is public or accessible to calling repos

```bash
# Make templates repo accessible
# Settings → Visibility → Change to "Public" or grant org access
```

### Issue: "AWS credentials failed"

**Solution**: Verify IAM role has SSM access

```bash
# AWS Console → IAM → Roles → GithubActionRole
# Add policy: AmazonSSMReadOnlyAccess
```

### Issue: "ECR push failed"

**Solution**: Verify ECR repositories exist

```bash
aws ecr describe-repositories --region us-east-1
# Should show all 4 service repos
```

---

## Validation Checklist

- [ ] Templates repository created and accessible
- [ ] All 4 services updated with new release.yml
- [ ] Test pipeline runs on feature branch (no ECR push)
- [ ] Test pipeline runs on PR to dev (no ECR push)
- [ ] Test pipeline runs on merge to dev (ECR push ✅)
- [ ] Test pipeline runs on merge to prod (ECR push ✅)
- [ ] AWS credentials working
- [ ] SSM parameters accessible
- [ ] All service repos green/passing

---

## Next Steps

1. See [USAGE.md](./USAGE.md) for detailed usage examples
2. See [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for common issues
3. Document your organization's specific branch strategy
4. Train team on new pipeline architecture

---

## Support

Open issue in `riskgps-github-actions-templates` repo with:
- Error message
- Service name
- Branch name
- Run URL

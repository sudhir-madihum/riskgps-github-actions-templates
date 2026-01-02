# Central Pipeline Templates - Troubleshooting Guide

## Common Issues & Solutions

---

## 1. "Reusable workflow not found"

### Error Message
```
Error: The referenced workflow file for 'bluocean/riskgps-github-actions-templates/.github/workflows/determine-environment.yml' could not be retrieved (HTTP 404).
```

### Causes & Solutions

**Cause 1: Template repository is private**
```bash
# Solution: Make repo public
# GitHub UI: Settings → Visibility → Change to Public
```

**Cause 2: Wrong repository name**
```yaml
# ❌ Wrong
uses: bluocean/riskgps-github-templates/.github/workflows/determine-environment.yml@main

# ✅ Correct
uses: bluocean/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@main
```

**Cause 3: Wrong workflow file path**
```yaml
# ❌ Wrong
uses: bluocean/riskgps-github-actions-templates/workflows/determine-environment.yml@main

# ✅ Correct (note .github/workflows/)
uses: bluocean/riskgps-github-actions-templates/.github/workflows/determine-environment.yml@main
```

**Cause 4: Branch/Tag doesn't exist**
```bash
# Make sure templates repo has main/v1.0.0 branch/tag
git branch -a
git tag
```

**Cause 5: Service repo doesn't have pull_request permission**
```yaml
# Add to calling workflow
permissions:
  id-token: write
  contents: read
```

---

## 2. "Step failed: AWS credentials"

### Error Message
```
Error: Couldn't retrieve credentials from /bluocean/riskgps/dev/ec2/credentials
AccessDenied: User: arn:aws:iam::739962689681:assumed-role/GithubActionRole/... 
is not authorized to perform: ssm:GetParameter
```

### Causes & Solutions

**Cause 1: SSM parameter doesn't exist**
```bash
# Check if parameter exists
aws ssm get-parameter \
  --name "/bluocean/riskgps/dev/ec2/credentials" \
  --region us-east-1 \
  --with-decryption

# If not, create it
aws ssm put-parameter \
  --name "/bluocean/riskgps/dev/ec2/credentials" \
  --type "SecureString" \
  --value '{"ecr_registry_repository_urls":{"frontend":"REPO_URL",...}}' \
  --region us-east-1
```

**Cause 2: IAM role lacks SSM permissions**
```bash
# AWS Console: IAM → Roles → GithubActionRole
# Add policy: AmazonSSMReadOnlyAccess
# Or create custom policy:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters"
      ],
      "Resource": "arn:aws:ssm:us-east-1:739962689681:parameter/bluocean/riskgps/*"
    }
  ]
}
```

**Cause 3: Wrong AWS region**
```yaml
# Check env variable
env:
  AWS_REGION: us-east-1  # Must match where SSM parameter exists
```

**Cause 4: OIDC token issue**
```bash
# Verify GitHub OIDC provider exists in AWS
aws iam list-open-id-connect-providers

# If not, create it:
# AWS Console → IAM → Identity providers → Add provider
# GitHub OIDC endpoint: https://token.actions.githubusercontent.com
```

---

## 3. "Build failed: Docker image not found"

### Error Message
```
Error: failed to get previous image from image reference "frontend:v1.0.42": image not found
```

### Causes & Solutions

**Cause 1: Dockerfile path incorrect**
```yaml
# Check your service repo structure
ls -la Dockerfile
ls -la ./docker/Dockerfile.dev  # If custom path

# Update template input
with:
  dockerfile-path: ./Dockerfile  # Default
  # OR
  dockerfile-path: ./docker/Dockerfile.prod  # Custom
```

**Cause 2: Build context missing files**
```yaml
# If dockerfile references ./app/src, ensure context includes it
with:
  docker-build-context: .  # Root of service repo
  # OR
  docker-build-context: ./app  # Specific subdirectory
```

**Cause 3: Docker build fails silently**
```bash
# Check detailed logs in GitHub Actions
# Step: "Build Docker image" should show full error
# Look for:
#   - Missing base image
#   - Failed RUN commands
#   - Invalid syntax
```

**Solution**: Test build locally
```bash
docker build -t test:latest -f Dockerfile .
```

---

## 4. "Tests failed, but workflow continued"

### Issue
Tests fail but image still builds and pushes.

### Cause
Template catches test failures but continues:
```yaml
run: |
  npm test || echo "⚠️ Tests had issues but continuing"
  # This allows build to proceed even if tests fail
```

### Solutions

**Option 1: Fail on test failure**
```yaml
# Edit your service's release.yml
# Add step before build-scan-push:
- name: Strict test check
  if: needs.determine-environment.outputs.run-tests == 'true'
  run: npm test  # No || echo
```

**Option 2: Prevent ECR push on test failure**
```yaml
push-to-ecr:
  if: needs.build-scan-push.result == 'success' && needs.determine-environment.outputs.should-push-ecr == 'true'
  # This requires build-scan-push to fully succeed
```

---

## 5. "ECR push always fails"

### Error Message
```
error: denied: User ... is not authorized to perform: ecr:GetDownloadUrlForLayer
```

### Causes & Solutions

**Cause 1: IAM role lacks ECR permissions**
```bash
# AWS Console: IAM → Roles → GithubActionRole
# Add policy: AmazonEC2ContainerRegistryPowerUser
# Or create custom:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:us-east-1:739962689681:repository/riskgps/*"
    }
  ]
}
```

**Cause 2: ECR repository doesn't exist**
```bash
# Check if repo exists
aws ecr describe-repositories \
  --repository-names riskgps/frontend \
  --region us-east-1

# If not, create it
aws ecr create-repository \
  --repository-name riskgps/frontend \
  --region us-east-1
```

**Cause 3: ECR image URL wrong in SSM**
```bash
# Check SSM parameter format
aws ssm get-parameter \
  --name "/bluocean/riskgps/dev/ec2/credentials" \
  --with-decryption \
  --query 'Parameter.Value'

# Should output JSON like:
# {"ecr_registry_repository_urls":{"frontend":"739962689681.dkr.ecr.us-east-1.amazonaws.com/riskgps/frontend",...}}
```

**Cause 4: should-push-ecr is false**
```yaml
# If workflow doesn't push even though it should:
# Check determine-environment logic

# You can test by checking job output:
# GitHub Actions → Run details → Jobs → determine-environment → Output

# Should show:
# should-push-ecr=true
```

---

## 6. "Template outputs are empty"

### Issue
`needs.build-scan-push.outputs.image-tag` is empty

### Cause
Build job failed or was skipped

### Solutions

**Check job dependencies**:
```yaml
build-scan-push:
  needs: [scan-files, determine-environment, run-tests]
  if: |
    always() &&
    needs.determine-environment.outputs.should-build == 'true' &&
    (needs.scan-files.result == 'success' || needs.scan-files.result == 'skipped') &&
    (needs.run-tests.result == 'success' || needs.run-tests.result == 'skipped')
```

**If job is skipped**:
- Check `if:` condition
- Verify parent jobs succeeded
- See parent job logs for errors

---

## 7. "Concurrency: Workflow cancelled unexpectedly"

### Issue
Workflow cancelled immediately after starting

### Cause
Concurrency control cancelled it:
```yaml
concurrency:
  group: ${{ github.ref }}
  cancel-in-progress: true
```

### Solutions

**This is expected behavior**:
- Only latest run on a branch executes
- Previous runs are cancelled

**If unwanted**:
```yaml
# Remove or modify concurrency
concurrency:
  group: ${{ github.ref }}
  cancel-in-progress: false  # Keep all runs
```

---

## 8. "Service branch names not respected"

### Issue
Branch strategy not working:
- Merge to dev doesn't push to ECR
- Feature branch push does push to ECR

### Cause
Environment variable mismatch

### Solution
```yaml
# Verify env variables in EVERY service
env:
  DEV_BRANCH: dev      # ✅ Correct
  PROD_BRANCH: prod    # ✅ Correct

# ❌ Wrong - if you use 'develop' but env says 'dev'
git checkout develop
git push origin develop  # Won't match DEV_BRANCH: dev
```

**Check workflow logs**:
- GitHub Actions → Run → determine-environment step
- Look for: "Source Branch: <actual>" and "Dev Branch: dev"
- Should match

---

## 9. "Node.js/Python versions incompatible"

### Error
```
npm ERR! node v18.0.0 incompatible with npm 10.0.0
```

### Solution
```yaml
# Update template input in your service's release.yml
build-scan-push:
  with:
    node-version: "20"  # Update to compatible version
    # OR
    python-version: "3.12"
```

---

## 10. "Changes to templates not reflecting"

### Issue
Updated template but workflows still use old version

### Cause
Template version locked to specific tag/commit

### Solution

**If using `@main`**:
```yaml
uses: org/templates/.github/workflows/xxx.yml@main
# Automatically gets latest from main branch
# Wait ~5-10 minutes for cache to clear
```

**If using `@v1.0.0`**:
```yaml
# ❌ Locked to old version
uses: org/templates/.github/workflows/xxx.yml@v1.0.0

# ✅ Switch to main (or create new tag)
uses: org/templates/.github/workflows/xxx.yml@main
# Then create new release tag when stable
git tag v1.0.1
git push origin v1.0.1
```

**Clear GitHub Actions cache**:
```bash
# GitHub doesn't provide direct cache clear
# Workaround: Commit dummy change to main branch
echo "# Cache buster" >> README.md
git add README.md
git commit -m "cache: clear"
git push origin main
```

---

## Debug Checklist

When a workflow fails, check in order:

- [ ] Template repository exists and is accessible
- [ ] Workflow file path is correct (`.github/workflows/xxx.yml`)
- [ ] Service repo has required permissions (id-token, contents)
- [ ] Global env variables are set (DEV_BRANCH, PROD_BRANCH, etc.)
- [ ] AWS IAM role has required permissions (SSM, ECR)
- [ ] SSM parameters exist with correct format/path
- [ ] ECR repositories exist in AWS
- [ ] Dockerfile path and build context are correct
- [ ] Template branch/tag exists
- [ ] Dependencies between jobs are correct

---

## Getting Help

1. **Check workflow logs**:
   - GitHub repo → Actions → Failed run
   - Expand failed step for full error

2. **Test locally**:
   ```bash
   docker build -f Dockerfile .
   npm test
   pytest
   ```

3. **Open issue in templates repo**:
   - Include error message
   - Include workflow run URL
   - Include service name
   - Include branch name

4. **Reference docs**:
   - [SETUP.md](./SETUP.md)
   - [USAGE.md](./USAGE.md)
   - [GitHub Actions docs](https://docs.github.com/en/actions)
   - [GitHub Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

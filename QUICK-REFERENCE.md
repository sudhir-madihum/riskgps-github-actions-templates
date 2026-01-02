# RiskGPS Pipeline Templates - Quick Reference

## 📁 Directory Structure

```
riskgps-github-actions-templates/
├── .github/workflows/                 # TEMPLATES (What to keep in sync)
│   ├── determine-environment.yml      # Determine dev/prod/feature, should-push-ecr
│   ├── build-scan-push-nodejs.yml     # Node.js: test, lint, build, scan
│   └── build-scan-push-python.yml     # Python: test, lint, build, scan
│
├── examples/                          # EXAMPLES (Reference implementations)
│   ├── release-frontend-example.yml   # How frontend uses templates
│   ├── release-backend-app-example.yml # How backend-app uses templates
│   └── release-backend-agent-example.yml # How backend-agent/connector use templates
│
├── docs/                              # DOCUMENTATION
│   ├── SETUP.md                       # Step-by-step setup guide
│   ├── USAGE.md                       # How to use and customize
│   ├── TROUBLESHOOTING.md             # Common issues & fixes
│   └── QUICK-REFERENCE.md             # (this file)
│
└── README.md                          # Overview & architecture
```

---

## 🚀 5-Minute Setup

### 1. Create Template Repository
```bash
# Create: riskgps-github-actions-templates
# Push all files from this directory
```

### 2. Copy Files Structure
```
.github/workflows/
  - determine-environment.yml
  - build-scan-push-nodejs.yml
  - build-scan-push-python.yml

docs/
  - SETUP.md, USAGE.md, TROUBLESHOOTING.md

examples/
  - release-frontend-example.yml
  - release-backend-app-example.yml
  - release-backend-agent-example.yml

README.md
```

### 3. Update Each Service (repeat 4 times)
```bash
# For riskgps-frontend:
1. Open .github/workflows/release.yml
2. Copy content from examples/release-frontend-example.yml
3. Replace <your-org> with your organization name
4. git commit "refactor: use central pipeline templates"
5. git push

# For riskgps-backend-app: use release-backend-app-example.yml
# For riskgps-backend-agent: use release-backend-agent-example.yml
# For riskgps-backend-connector: use release-backend-agent-example.yml
```

### 4. Test
```bash
# Push to feature branch → should build, NOT push ECR
# PR to dev → should build, NOT push ECR
# Merge to dev → should build AND push ECR
# Merge to prod → should build AND push ECR
```

---

## 🔧 Template Reference

### determine-environment.yml
```yaml
# Input
with:
  dev-branch: "dev"
  prod-branch: "prod"
  skip-tests-on-prod: true

# Output
outputs:
  environment: "dev" | "prod" | "feature"
  should-push-ecr: "true" | "false"     # KEY FLAG
  run-tests: "true" | "false"
  should-build: "true"
```

### build-scan-push-nodejs.yml
```yaml
# Input
with:
  service-name: "frontend"
  dockerfile-path: "./Dockerfile"
  docker-build-context: "."
  node-version: "18"
  npm-cache: "npm"
  run-tests: true
  run-linter: true

# Output
outputs:
  image-tag: "v1.0.42"
  image-name: "frontend:v1.0.42"
```

### build-scan-push-python.yml
```yaml
# Input
with:
  service-name: "backend-agent"
  dockerfile-path: "./Dockerfile"
  docker-build-context: "."
  python-version: "3.11"
  run-tests: true
  run-linter: true

# Output
outputs:
  image-tag: "v1.0.42"
  image-name: "backend-agent:v1.0.42"
```

---

## 📋 Environment Variables (per service)

```yaml
env:
  AWS_REGION: us-east-1           # AWS region
  SERVICE_NAME: frontend          # Different per service
  DEV_BRANCH: dev                 # Same across all
  PROD_BRANCH: prod               # Same across all
  ORGANIZATION: bluocean          # Same across all
  PROJECT: riskgps                # Same across all
```

**Change branch names once** → All services adapt automatically

---

## 🎯 How It Works

```
GitHub Event
    ↓
Call determine-environment.yml
    ↓ Outputs: should-push-ecr, environment, run-tests
    ↓
Call build-scan-push-nodejs/python.yml
    ↓ Outputs: image-name, image-tag
    ↓
If should-push-ecr == true: Push to ECR
Else: Skip ECR (feature/PR)
```

---

## 📊 Decision Matrix

| Event | Branch | run-tests | should-push-ecr | Notes |
|-------|--------|-----------|-----------------|-------|
| push | feature/xxx | true | false | Build & test, no push |
| push | dev | true | **true** | Build & push to dev ECR |
| push | prod | false | **true** | Build & push to prod ECR (no tests) |
| PR | → dev | true | false | Build & test, no push |
| PR | → prod | false | false | Build only, no push |

---

## ✅ Common Tasks

### Change Branch Names
```yaml
# OLD: DEV_BRANCH: dev
# NEW: DEV_BRANCH: develop

# Change in ONE place per service
# All template logic adapts automatically
```

### Use Different Node Version
```yaml
build-scan-push:
  with:
    node-version: "20"  # Update here
```

### Use Different Python Version
```yaml
build-scan-push:
  with:
    python-version: "3.12"  # Update here
```

### Skip Tests for Feature Branches
```yaml
# Already built-in:
# determine-environment outputs run-tests
# Template checks it automatically
```

### Push to Custom ECR
```yaml
# Add custom push-to-ecr job after build-scan-push
# Reference outputs:
#   needs.build-scan-push.outputs.image-name
#   needs.build-scan-push.outputs.image-tag
#   needs.determine-environment.outputs.should-push-ecr
```

---

## 🔄 Update Templates

### When to Update
- Add new feature to all services
- Fix bug in build process
- Change test command
- Upgrade tool versions

### How to Update
1. Edit templates in `riskgps-github-actions-templates`
2. Create tag: `git tag v1.0.1`
3. Push: `git push origin v1.0.1`
4. Services automatically use new version (if using `@main`)
5. Or manually update services to `@v1.0.1`

### Testing Changes
```bash
# In templates repo
1. Make change
2. git commit && git push origin feature/test
3. In service repo:
   uses: org/templates/.github/workflows/xxx.yml@feature/test
4. Test workflow
5. If good: merge PR in templates repo
6. Create tag for release
```

---

## 🛠️ Troubleshooting Quick Links

| Issue | Solution |
|-------|----------|
| "Reusable workflow not found" | See [TROUBLESHOOTING.md#1](./docs/TROUBLESHOOTING.md#1-reusable-workflow-not-found) |
| "AWS credentials failed" | See [TROUBLESHOOTING.md#2](./docs/TROUBLESHOOTING.md#2-step-failed-aws-credentials) |
| "Docker build failed" | See [TROUBLESHOOTING.md#3](./docs/TROUBLESHOOTING.md#3-build-failed-docker-image-not-found) |
| "Tests failed but push continued" | See [TROUBLESHOOTING.md#4](./docs/TROUBLESHOOTING.md#4-tests-failed-but-workflow-continued) |
| "ECR push always fails" | See [TROUBLESHOOTING.md#5](./docs/TROUBLESHOOTING.md#5-ecr-push-always-fails) |
| "Template outputs empty" | See [TROUBLESHOOTING.md#6](./docs/TROUBLESHOOTING.md#6-template-outputs-are-empty) |

Full troubleshooting guide: [TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md)

---

## 📚 Documentation Files

| File | Purpose | Read Time |
|------|---------|-----------|
| [README.md](./README.md) | Overview & benefits | 5 min |
| [docs/SETUP.md](./docs/SETUP.md) | How to set up | 10 min |
| [docs/USAGE.md](./docs/USAGE.md) | How to use & customize | 15 min |
| [docs/TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md) | Common issues | As needed |

---

## 🔑 Key Concepts

### 1. Single Source of Truth
- Templates defined once in `riskgps-github-actions-templates`
- All services reference same templates
- Update happens in one place

### 2. Reusable Workflows
- GitHub Actions feature for workflow reuse
- Callable from any repository
- Supports inputs & outputs

### 3. Smart Environment Detection
- `determine-environment.yml` inspects GitHub event
- Decides: dev? prod? feature?
- Decides: should push to ECR?

### 4. Conditional ECR Push
- All services always build & test
- Only pushes to ECR when `should-push-ecr=true`
- Prevents feature branch pollution

### 5. Global Configuration
- Branch names in `env:` section
- Change once per service
- All templates use same values

---

## 🎓 Learning Path

### For Developers
1. Read [README.md](./README.md) - Understand what this is
2. Check [examples/](./examples/) - See how your service uses it
3. Read [docs/USAGE.md](./docs/USAGE.md) - Learn customization

### For DevOps
1. Read [README.md](./README.md) - Understand architecture
2. Read [docs/SETUP.md](./docs/SETUP.md) - Set up templates
3. Read [docs/USAGE.md](./docs/USAGE.md) - Customization options
4. Bookmark [docs/TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md) - Common issues

### For Maintainers
1. Read all documentation
2. Understand template inputs/outputs
3. Know when/how to update templates
4. Monitor template usage in services

---

## 💡 Pro Tips

1. **Lock versions in production**
   ```yaml
   uses: org/templates/.github/workflows/xxx.yml@v1.0.0
   ```

2. **Use global variables**
   ```yaml
   env:
     DEV_BRANCH: dev  # Change once, everywhere adapts
   ```

3. **Test locally first**
   ```bash
   docker build -f Dockerfile .
   npm test
   ```

4. **Check GitHub Actions logs**
   - Repository → Actions → Failed run → Job logs
   - Expand failed steps for details

5. **Monitor SSM parameters**
   ```bash
   aws ssm get-parameter \
     --name "/bluocean/riskgps/dev/ec2/credentials" \
     --with-decryption
   ```

---

## 📞 Support Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Reusable Workflows Guide](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Troubleshooting Guide](./docs/TROUBLESHOOTING.md)
- Open issue in `riskgps-github-actions-templates` repo

---

## ✨ Next Steps

1. [ ] Create `riskgps-github-actions-templates` repository
2. [ ] Push all template files
3. [ ] Update `riskgps-frontend` to use templates
4. [ ] Update `riskgps-backend-app` to use templates
5. [ ] Update `riskgps-backend-agent` to use templates
6. [ ] Update `riskgps-backend-connector` to use templates
7. [ ] Test all 4 services (feature → dev → prod flow)
8. [ ] Team training & documentation

---

**Version**: 1.0.0  
**Last Updated**: 2026-01-02  
**Templates**: Ready for Production ✅

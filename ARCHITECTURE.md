# Central Pipeline Templates - Visual Architecture

## 📚 Document Map

```
START HERE: README.md (this repo overview)
    ↓
SETUP: docs/SETUP.md (step-by-step installation)
    ↓
USAGE: docs/USAGE.md (how to use & customize)
    ↓
EXAMPLES: examples/*.yml (reference implementations)
    ↓
REFERENCE: QUICK-REFERENCE.md (quick lookup)
    ↓
HELP: docs/TROUBLESHOOTING.md (when things break)
```

---

## 🏗️ System Architecture

### Before: Monolithic Approach

```
Each Service Repo
├── riskgps-frontend
│   └── .github/workflows/release.yml          ← 500 lines (ALL logic)
├── riskgps-backend-app
│   └── .github/workflows/release.yml          ← 500 lines (SAME logic)
├── riskgps-backend-agent
│   └── .github/workflows/release.yml          ← 500 lines (SAME logic)
└── riskgps-backend-connector
    └── .github/workflows/release.yml          ← 500 lines (SAME logic)

Problem: 2000 lines of duplicated logic
         Update needed? Fix all 4 places
         Consistency issues? Hard to track
```

### After: Template Approach

```
Central Template Repository
└── riskgps-github-actions-templates/
    ├── .github/workflows/
    │   ├── determine-environment.yml          ← Shared by ALL
    │   ├── build-scan-push-nodejs.yml         ← Shared by frontend, backend-app
    │   └── build-scan-push-python.yml         ← Shared by backend-agent, connector
    └── docs/ + examples/ + QUICK-REFERENCE.md

Each Service (NOW SIMPLE)
├── riskgps-frontend
│   └── .github/workflows/release.yml          ← 100 lines (calls templates)
├── riskgps-backend-app
│   └── .github/workflows/release.yml          ← 100 lines (calls templates)
├── riskgps-backend-agent
│   └── .github/workflows/release.yml          ← 100 lines (calls templates)
└── riskgps-backend-connector
    └── .github/workflows/release.yml          ← 100 lines (calls templates)

Benefit: Update logic once, applies everywhere
```

---

## 🔄 Workflow Execution Flow

### Simplified View

```
Service Repo (calls templates)
    ↓
    ├─ Step 1: determine-environment.yml
    │  └─ Output: should-push-ecr, environment, run-tests
    │
    ├─ Step 2: build-scan-push-nodejs/python.yml
    │  └─ Output: image-name, image-tag
    │
    └─ Step 3: Push to ECR (if should-push-ecr=true)
```

### Detailed View

```
1. GitHub Event (push / PR)
        ↓
2. determine-environment.yml
   ├─ INPUT: github event + branch
   ├─ LOGIC:
   │  ├─ Is it push to dev?         → should-push-ecr=true
   │  ├─ Is it push to prod?        → should-push-ecr=true
   │  ├─ Is it push to feature?     → should-push-ecr=false
   │  └─ Is it PR to any branch?    → should-push-ecr=false
   │
   └─ OUTPUT: should-push-ecr, environment, run-tests
        ↓
3. build-scan-push-nodejs/python.yml
   ├─ INPUT: service-name, dockerfile-path, run-tests, etc.
   ├─ LOGIC:
   │  ├─ Install dependencies
   │  ├─ Run tests (if run-tests=true)
   │  ├─ Run linter (if configured)
   │  ├─ Scan source files (Trivy)
   │  ├─ Build Docker image
   │  └─ Scan Docker image (Trivy)
   │
   └─ OUTPUT: image-name, image-tag
        ↓
4. Push to ECR (conditional)
   ├─ IF should-push-ecr=true:
   │  ├─ Get AWS credentials
   │  ├─ Fetch ECR config from SSM
   │  ├─ Login to ECR
   │  ├─ Tag image
   │  └─ Push to ECR ✅
   │
   └─ ELSE (feature/PR):
      └─ Log "NOT pushing" ⏭️
```

---

## 📊 When ECR Push Happens

```
PUSH to feature/my-feature
    ↓ determine-environment detects: feature branch
    ↓ Sets: should-push-ecr=false
    ↓ Builds image, does NOT push to ECR
    ↓ Developer verifies locally

PUSH to dev (after merge)
    ↓ determine-environment detects: dev branch
    ↓ Sets: should-push-ecr=true
    ↓ Builds image AND PUSHES to dev ECR ✅

PUSH to prod (after merge)
    ↓ determine-environment detects: prod branch
    ↓ Sets: should-push-ecr=true
    ↓ Builds image AND PUSHES to prod ECR ✅

PR to dev
    ↓ determine-environment detects: PR event
    ↓ Sets: should-push-ecr=false
    ↓ Builds image, does NOT push to ECR
    ↓ Code review + approval
    ↓ (Then merge happens above ↑)
```

---

## 🎯 Template Hierarchy

```
determine-environment.yml
├─ Purpose: Decide environment & ECR push flag
├─ Language: Bash
├─ Input: dev-branch, prod-branch, skip-tests-on-prod
├─ Output: should-push-ecr, environment, run-tests, should-build
└─ Used by: ALL 4 services

build-scan-push-nodejs.yml
├─ Purpose: Build Node.js service
├─ Language: Node.js + npm
├─ Input: service-name, dockerfile, node-version, run-tests, run-linter
├─ Output: image-name, image-tag
└─ Used by: riskgps-frontend, riskgps-backend-app

build-scan-push-python.yml
├─ Purpose: Build Python service
├─ Language: Python + pytest + flake8
├─ Input: service-name, dockerfile, python-version, run-tests, run-linter
├─ Output: image-name, image-tag
└─ Used by: riskgps-backend-agent, riskgps-backend-connector
```

---

## 🔧 Configuration Points

### Per Service (in service repo's release.yml)

```yaml
env:
  SERVICE_NAME: frontend              # Different per service
  DEV_BRANCH: dev                     # Same across all
  PROD_BRANCH: prod                   # Same across all
  ORGANIZATION: bluocean              # Same across all
  PROJECT: riskgps                    # Same across all
  AWS_REGION: us-east-1               # Same across all
```

### Per Build (inputs to templates)

```yaml
# For Node.js services
build-scan-push:
  with:
    service-name: frontend
    dockerfile-path: ./Dockerfile
    node-version: "18"
    npm-cache: npm
    run-tests: true
    run-linter: true

# For Python services
build-scan-push:
  with:
    service-name: backend-agent
    dockerfile-path: ./Dockerfile
    python-version: "3.11"
    run-tests: true
    run-linter: true
```

### Per Organization (AWS/SSM)

```bash
# SSM Parameter (set once in AWS)
/bluocean/riskgps/dev/ec2/credentials
{
  "ecr_registry_repository_urls": {
    "frontend": "739962689681.dkr.ecr.us-east-1.amazonaws.com/riskgps/frontend",
    "backend-app": "739962689681.dkr.ecr.us-east-1.amazonaws.com/riskgps/backend-app",
    ...
  }
}
```

---

## 📈 Implementation Timeline

```
Phase 1: Setup (30 min)
├─ Create riskgps-github-actions-templates repo
├─ Copy template files
├─ Push to GitHub
└─ ✅ Complete

Phase 2: Frontend Migration (15 min)
├─ Update riskgps-frontend/.github/workflows/release.yml
├─ Test: push feature branch (no ECR)
├─ Test: merge to dev (ECR push)
└─ ✅ Complete

Phase 3: Backend-App Migration (15 min)
├─ Update riskgps-backend-app/.github/workflows/release.yml
├─ Test workflows
└─ ✅ Complete

Phase 4: Backend-Agent Migration (15 min)
├─ Update riskgps-backend-agent/.github/workflows/release.yml
├─ Test workflows
└─ ✅ Complete

Phase 5: Backend-Connector Migration (15 min)
├─ Update riskgps-backend-connector/.github/workflows/release.yml
├─ Test workflows
└─ ✅ Complete

Total Time: ~1.5 hours for full rollout
```

---

## ✅ Checklist Before Using

- [ ] Create `riskgps-github-actions-templates` repository
- [ ] Repository is public or accessible to all service repos
- [ ] All template files copied to `.github/workflows/`
- [ ] All documentation copied to `docs/`
- [ ] All examples copied to `examples/`
- [ ] AWS IAM role configured (SSM + ECR permissions)
- [ ] SSM parameters created with ECR URLs
- [ ] ECR repositories created for all 4 services
- [ ] Each service repo has correct env variables
- [ ] Team trained on how to use templates

---

## 🎓 For Each Role

### DevOps Engineer
**Responsible for**: Template creation, AWS setup, deployment
- [ ] Create templates repository
- [ ] Configure AWS IAM & SSM
- [ ] Set up ECR repositories
- [ ] Verify all 4 services working
- [ ] Monitor template usage

### Developer
**Responsible for**: Using templates, pushing code, merging PRs
- [ ] Read [docs/USAGE.md](./docs/USAGE.md)
- [ ] Understand branch strategy
- [ ] Know when ECR push happens
- [ ] Test feature branches locally
- [ ] Create PRs to dev/prod for reviews

### Tech Lead
**Responsible for**: Strategy, updates, troubleshooting
- [ ] Understand full architecture
- [ ] Maintain templates when logic changes
- [ ] Review template updates before release
- [ ] Document org-specific configuration
- [ ] Troubleshoot complex issues

---

## 💾 File Size Reduction

```
Before Central Templates:
├─ riskgps-frontend/.github/workflows/release.yml          500 lines
├─ riskgps-backend-app/.github/workflows/release.yml       500 lines
├─ riskgps-backend-agent/.github/workflows/release.yml     500 lines
└─ riskgps-backend-connector/.github/workflows/release.yml 500 lines
                                                           ─────────
                                                           2000 lines

After Central Templates:
├─ riskgps-github-actions-templates/.github/workflows/
│  ├─ determine-environment.yml                              80 lines
│  ├─ build-scan-push-nodejs.yml                            120 lines
│  └─ build-scan-push-python.yml                            120 lines
│
├─ riskgps-frontend/.github/workflows/release.yml           100 lines (calls templates)
├─ riskgps-backend-app/.github/workflows/release.yml        100 lines (calls templates)
├─ riskgps-backend-agent/.github/workflows/release.yml      100 lines (calls templates)
└─ riskgps-backend-connector/.github/workflows/release.yml  100 lines (calls templates)
                                                            ─────────
                                                             820 lines

REDUCTION: 2000 → 820 lines (-59%)
DUPLICATION: 1180 lines of code eliminated
```

---

## 🚀 Key Advantages

| Aspect | Before | After |
|--------|--------|-------|
| **Maintenance** | Update 4 repos | Update 1 template |
| **Consistency** | Manual sync | Automatic sync |
| **Time to Fix** | ~1 hour | ~5 minutes |
| **New Service** | Copy 500 lines | Write 50 lines |
| **Bug Fix Scope** | All 4 services | 1 place |
| **Learning Curve** | Understand full pipelines | Understand 3 templates |
| **Versioning** | Hard to track | Easy (git tags) |

---

## 🔗 Cross-References

- [README.md](./README.md) - Overview
- [docs/SETUP.md](./docs/SETUP.md) - Step-by-step setup
- [docs/USAGE.md](./docs/USAGE.md) - Usage guide
- [docs/TROUBLESHOOTING.md](./docs/TROUBLESHOOTING.md) - Troubleshooting
- [QUICK-REFERENCE.md](./QUICK-REFERENCE.md) - Quick lookup
- [examples/](./examples/) - Implementation examples

---

## 📞 Quick Support

**Q: Where do I make changes?**
A: In `riskgps-github-actions-templates` repo, not in service repos

**Q: How do changes apply to all services?**
A: Service repos call the templates, they automatically get updates

**Q: What if a service needs different logic?**
A: Create a separate template for that service's special needs

**Q: How do I test template changes?**
A: Update template → test with a feature branch in a service repo

**Q: How long does migration take?**
A: ~1.5 hours total for all 4 services

---

**Next Step**: Go to [docs/SETUP.md](./docs/SETUP.md) to get started! 🚀

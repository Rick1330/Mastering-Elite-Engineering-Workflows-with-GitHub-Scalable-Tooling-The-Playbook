# Repository Structure: Organizing for Scale

<Tip>
**TL;DR**: Choose monorepo for tightly-coupled services with shared dependencies, polyrepo for independent microservices. Use standardized templates, enforce structure via automation, and maintain a GitHub Service Catalog for discoverability.
</Tip>

## Introduction

Repository organization is a foundational decision that impacts developer productivity, CI/CD complexity, and long-term maintainability. For organizations with 50+ engineers and dozens (or hundreds) of repositories, consistency and clarity become critical.

This guide covers:
- **Monorepo vs. Polyrepo**: Decision frameworks based on team topology
- **Standardized Templates**: Enforcing structure at scale
- **Service Catalog**: Making repositories discoverable
- **Build System Optimization**: Nx, Bazel, and Turborepo patterns

---

## Monorepo vs. Polyrepo: Decision Framework

### When to Choose Monorepo

| Factor | Monorepo Advantage |
|--------|-------------------|
| **Team Size** | 20+ engineers working on related services |
| **Dependencies** | Many shared libraries between projects |
| **Release Cadence** | Coordinated releases across services |
| **Code Sharing** | Frequent cross-project refactoring |
| **Atomic Changes** | Need to change multiple services together |

### When to Choose Polyrepo

| Factor | Polyrepo Advantage |
|--------|-------------------|
| **Team Autonomy** | Independent teams with clear ownership |
| **Deployment** | Services deploy independently |
| **Access Control** | Different security/access requirements |
| **Technology Diversity** | Different languages/frameworks per service |
| **CI/CD Speed** | Faster builds for isolated changes |

### Hybrid Approach: The Best of Both

```yaml
# Example: Hybrid structure for mid-size organization
organization/
├── monorepos/
│   ├── platform-core/        # Shared infrastructure
│   │   ├── packages/
│   │   │   ├── auth/
│   │   │   ├── logging/
│   │   │   └── utils/
│   │   └── apps/
│   │       ├── admin-portal/
│   │       └── api-gateway/
│   └── customer-experience/  # Customer-facing products
│       ├── packages/
│       │   └── ui-components/
│       └── apps/
│           ├── web-app/
│           └── mobile-app/
└── polyrepos/
    ├── billing-service/       # Independent service
    ├── data-pipeline/         # Specialized workload
    └── ml-training/           # Different tech stack
```

---

## Standardized Repository Templates

### Template Repository Structure

```
template-service/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   ├── workflows/
│   │   ├── ci.yml              # Reusable CI workflow
│   │   ├── cd.yml              # Reusable CD workflow
│   │   └── security.yml        # Security scanning
│   ├── CODEOWNERS
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── settings.yml            # Repository settings
├── src/
│   ├── index.ts
│   └── ...
├── tests/
│   ├── unit/
│   └── integration/
├── docs/
│   ├── api.md
│   └── architecture.md
├── scripts/
│   ├── setup.sh
│   └── deploy.sh
├── .gitignore
├── .eslintrc.js
├── .prettierrc
├── docker-compose.yml
├── Dockerfile
├── package.json
├── tsconfig.json
├── README.md                   # With standard sections
├── CONTRIBUTING.md
├── CHANGELOG.md
├── LICENSE
└── SECURITY.md
```

### Automated Template Enforcement

```yaml
# .github/workflows/enforce-template.yml
name: Enforce Repository Standards

on:
  push:
    branches: [main]

jobs:
  check-required-files:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Check Required Files
        run: |
          REQUIRED_FILES=(
            "README.md"
            "CONTRIBUTING.md"
            "LICENSE"
            "SECURITY.md"
            ".github/CODEOWNERS"
            ".github/PULL_REQUEST_TEMPLATE.md"
          )
          
          MISSING=()
          for FILE in "${REQUIRED_FILES[@]}"; do
            if [ ! -f "$FILE" ]; then
              MISSING+=("$FILE")
            fi
          done
          
          if [ ${#MISSING[@]} -gt 0 ]; then
            echo "❌ Missing required files:"
            printf '%s\n' "${MISSING[@]}"
            exit 1
          fi
          
          echo "✅ All required files present"

  check-readme-structure:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate README Sections
        run: |
          REQUIRED_SECTIONS=(
            "Description"
            "Quickstart"
            "Installation"
            "Usage"
            "API Reference"
            "Contributing"
          )
          
          for SECTION in "${REQUIRED_SECTIONS[@]}"; do
            if ! grep -q "## $SECTION" README.md; then
              echo "⚠️ Missing recommended section: $SECTION"
            fi
          done
```

---

## GitHub Service Catalog

### Service Catalog Implementation

```yaml
# .github/service-catalog.yml
# Central registry of all services and their metadata

services:
  - name: user-authentication
    repository: org/user-auth
    type: service
    team: platform-team
    tier: tier-1  # Critical path
    language: typescript
    framework: express
    database: postgresql
    owner: "@platform-lead"
    slack_channel: "#alerts-auth"
    pagerduty_service: "auth-prod"
    runbook_url: "https://wiki.company.com/runbooks/auth"
    dependencies:
      - user-database
      - session-cache
    dependents:
      - web-app
      - mobile-api
    sla:
      availability: 99.99%
      latency_p99: 100ms
    
  - name: payment-processing
    repository: org/payment-service
    type: service
    team: payments-team
    tier: tier-1
    language: java
    framework: spring-boot
    database: postgresql
    owner: "@payments-lead"
    compliance:
      - pci-dss
      - soc2
    sla:
      availability: 99.99%
      
  - name: analytics-pipeline
    repository: org/analytics
    type: pipeline
    team: data-team
    tier: tier-2
    language: python
    framework: airflow
    schedule: "0 */6 * * *"
    
  - name: design-system
    repository: org/design-system
    type: library
    team: frontend-platform
    tier: tier-2
    language: typescript
    published_package: "@company/ui-components"
```

### Auto-Discovery Workflow

```yaml
# .github/workflows/update-service-catalog.yml
name: Update Service Catalog

on:
  push:
    paths: ['.github/service-catalog.yml']
    branches: [main]

jobs:
  validate-and-publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate Service Catalog
        run: |
          # Validate YAML syntax
          yamllint .github/service-catalog.yml
          
          # Validate required fields
          node scripts/validate-catalog.js
          
      - name: Generate Service Map
        run: |
          node scripts/generate-service-map.js > site/service-map.html
          
      - name: Deploy to Internal Portal
        run: |
          # Deploy to internal developer portal
          ./scripts/deploy-catalog.sh
          
      - name: Notify Teams
        run: |
          curl -X POST "$SLACK_WEBHOOK" \
            -H 'Content-Type: application/json' \
            -d '{
              "text": "📋 Service Catalog updated\nChanges: ${{ github.event.head_commit.message }}"
            }'
```

---

## Build System Optimization

### Nx Monorepo Configuration

```json
// nx.json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint"],
        "parallel": 4
      }
    }
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"]
    },
    "test": {
      "inputs": ["default", "^production"]
    }
  },
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.ts",
      "!{projectRoot}/tsconfig.spec.json"
    ],
    "sharedGlobals": ["{workspaceRoot}/package-lock.json"]
  }
}
```

### Affected Commands for CI

```yaml
# .github/workflows/nx-ci.yml
name: Nx Monorepo CI

on:
  pull_request:
    branches: [main]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      affected-projects: ${{ steps.affected.outputs.projects }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Detect Affected Projects
        id: affected
        run: |
          AFFECTED=$(npx nx affected --target=build --plain)
          echo "projects=$AFFECTED" >> $GITHUB_OUTPUT

  build:
    needs: detect-changes
    runs-on: ubuntu-latest
    if: needs.detect-changes.outputs.affected-projects != ''
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Build Affected Projects
        run: npx nx affected --target=build --parallel=4
      
      - name: Test Affected Projects
        run: npx nx affected --target=test --parallel=4

  # Always run these for critical projects
  lint-all:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx nx run-many --target=lint --all
```

### Bazel Remote Caching

```python
# BUILD.bazel
load("@npm//:package_json.bzl", "npm_bin")

npm_bin(
    name = "jest",
    entry_point = "jest",
)

js_test(
    name = "unit_tests",
    srcs = glob(["**/*.spec.ts"]),
    deps = [
        "//packages/core",
        "//packages/utils",
        "@npm//jest",
    ],
)
```

```yaml
# .bazelrc
build --remote_cache=https://cache.company.com
build --remote_instance_address=grpc://bazel-builder:9092
build --disk_cache=/tmp/bazel-cache
build --jobs=8
test --test_output=errors
```

---

## Repository Governance

### Automated Repository Creation

```yaml
# .github/workflows/create-repository.yml
name: Create New Repository

on:
  workflow_dispatch:
    inputs:
      name:
        description: 'Repository name'
        required: true
      template:
        description: 'Template to use (service, library, app)'
        required: true
        type: choice
        options:
          - service
          - library
          - app
      team:
        description: 'Owning team'
        required: true

jobs:
  create-repo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Create Repository from Template
        run: |
          gh repo create "$NAME" \
            --template "org/template-$TEMPLATE" \
            --private \
            --description "Created by $TEAM team"
        env:
          NAME: ${{ github.event.inputs.name }}
          TEMPLATE: ${{ github.event.inputs.template }}
          TEAM: ${{ github.event.inputs.team }}
      
      - name: Apply Team Permissions
        run: |
          gh api \
            --method PUT \
            /repos/org/$NAME/teams/$TEAM \
            --input - <<EOF
          {
            "permission": "admin"
          }
          EOF
      
      - name: Add to Service Catalog
        run: |
          node scripts/add-to-catalog.js \
            --name="$NAME" \
            --team="$TEAM" \
            --template="$TEMPLATE"
```

---

## Implementation Checklist

### Week 1-2: Foundation
- [ ] Audit existing repository structures
- [ ] Create 3 template repositories (service, library, app)
- [ ] Document repository naming conventions
- [ ] Set up automated template enforcement

### Week 3-4: Service Catalog
- [ ] Inventory all existing repositories
- [ ] Create initial service-catalog.yml
- [ ] Build simple discovery UI
- [ ] Integrate with team onboarding

### Month 2: Build Optimization
- [ ] Evaluate monorepo vs. polyrepo for each domain
- [ ] Implement Nx or Bazel where appropriate
- [ ] Set up remote caching infrastructure
- [ ] Measure and document build time improvements

### Ongoing: Continuous Improvement
- [ ] Quarterly repository structure reviews
- [ ] Monthly template updates based on feedback
- [ ] Service catalog maintenance
- [ ] Build performance monitoring

---

<Success>
**Elite Practice**: Treat your repository templates as products. Gather feedback from developers, iterate based on pain points, and measure adoption rates.
</Success>

---

## Related Resources

- [Nx Monorepo Tools](https://nx.dev/)
- [Bazel Build System](https://bazel.build/)
- [Turborepo Documentation](https://turbo.build/repo)
- [GitHub Template Repositories](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-template-repository)

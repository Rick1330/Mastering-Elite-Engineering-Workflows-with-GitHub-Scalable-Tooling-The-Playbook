# GitHub Actions & Automation: Elite CI/CD Patterns

<Tip>
**TL;DR**: Elite teams use composite actions for reusability, implement test sharding for speed, leverage OIDC for secure deployments, and employ change detection to run only affected tests. This guide provides production-ready "Golden Path" workflows.
</Tip>

## Introduction

GitHub Actions has evolved from a simple CI/CD tool to a comprehensive automation platform. For teams of 50+ engineers, the difference between basic and elite automation can mean the difference between shipping daily and shipping weekly, between confident deployments and rollback nightmares.

This guide covers:
- **Composite Actions**: Modular, reusable workflow components
- **Test Sharding**: Parallel test execution for faster feedback
- **Change Detection**: Running only tests affected by changes
- **OIDC Authentication**: Secretless, secure deployments
- **Environment-Aware CD**: Granular deployment controls

---

## Composite Actions Architecture

### Why Composite Actions?

| Benefit | Description |
|---------|-------------|
| **Reusability** | Write once, use across all repositories |
| **Consistency** | Ensure every team follows the same patterns |
| **Maintenance** | Update in one place, propagate everywhere |
| **Abstraction** | Hide complexity behind simple interfaces |

### Validation Composite Action

```yaml
# .github/actions/validate/action.yml
name: 'Validate'
description: 'Run all validation checks (lint, type-check, security)'
author: 'Elite Engineering Team'

inputs:
  node-version:
    description: 'Node.js version to use'
    required: false
    default: '20'
  fail-on-security-issue:
    description: 'Whether to fail on security issues'
    required: false
    default: 'true'
  working-directory:
    description: 'Working directory for commands'
    required: false
    default: '.'

outputs:
  lint-result:
    description: 'Result of linting'
    value: ${{ steps.lint.outputs.result }}
  type-check-result:
    description: 'Result of type checking'
    value: ${{ steps.type-check.outputs.result }}

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
        cache-dependency-path: '${{ inputs.working-directory }}/package-lock.json'

    - name: Install dependencies
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: npm ci --ignore-scripts

    - name: Lint
      id: lint
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        echo "Running ESLint..."
        npm run lint || exit 1
        echo "result=success" >> $GITHUB_OUTPUT

    - name: Type Check
      id: type-check
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        echo "Running TypeScript compiler..."
        npm run type-check || exit 1
        echo "result=success" >> $GITHUB_OUTPUT

    - name: Security Scan with Semgrep
      if: inputs.fail-on-security-issue == 'true'
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        npx semgrep --config auto --error || exit 1

    - name: Dependency Audit
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        npm audit --audit-level=moderate --json > audit-results.json || true
```

### Test Composite Action with Sharding

```yaml
# .github/actions/test/action.yml
name: 'Test'
description: 'Run tests with sharding support'
author: 'Elite Engineering Team'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
  shard-count:
    description: 'Number of shards for parallelization'
    required: false
    default: '4'
  shard-index:
    description: 'Current shard index (0-based)'
    required: false
    default: '0'
  coverage-enabled:
    description: 'Enable code coverage'
    required: false
    default: 'false'
  flaky-detection:
    description: 'Enable flaky test detection'
    required: false
    default: 'true'

outputs:
  coverage-percentage:
    description: 'Code coverage percentage'
    value: ${{ steps.coverage.outputs.percentage }}
  test-duration:
    description: 'Test execution duration in seconds'
    value: ${{ steps.test.outputs.duration }}

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      shell: bash
      run: npm ci

    - name: Run Tests (Sharded)
      id: test
      shell: bash
      run: |
        START_TIME=$(date +%s)
        
        if [ "${{ inputs.shard-count }}" -gt 1 ]; then
          echo "Running shard ${{ inputs.shard-index }} of ${{ inputs.shard-count }}"
          npm run test:shard -- --shards=${{ inputs.shard-index }}/${{ inputs.shard-count }}
        else
          npm test
        fi
        
        END_TIME=$(date +%s)
        DURATION=$((END_TIME - START_TIME))
        echo "duration=$DURATION" >> $GITHUB_OUTPUT

    - name: Detect Flaky Tests
      if: inputs.flaky-detection == 'true'
      shell: bash
      continue-on-error: true
      run: |
        echo "Running flaky test detection..."
        npm run test:flaky || echo "Flaky tests detected - see report"

    - name: Generate Coverage Report
      if: inputs.coverage-enabled == 'true'
      id: coverage
      shell: bash
      run: |
        npm run coverage
        COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
        echo "percentage=$COVERAGE" >> $GITHUB_OUTPUT
        echo "Coverage: ${COVERAGE}%"

    - name: Upload Coverage to Codecov
      if: inputs.coverage-enabled == 'true'
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage/lcov.info
        fail_ci_if_error: false
```

### Build Composite Action

```yaml
# .github/actions/build/action.yml
name: 'Build'
description: 'Build application with optimization and artifact management'
author: 'Elite Engineering Team'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
  build-type:
    description: 'Build type (development, production)'
    required: false
    default: 'production'
  upload-artifacts:
    description: 'Upload build artifacts'
    required: false
    default: 'true'
  bundle-analysis:
    description: 'Run bundle size analysis'
    required: false
    default: 'false'

outputs:
  build-duration:
    description: 'Build duration in seconds'
    value: ${{ steps.build.outputs.duration }}
  artifact-name:
    description: 'Name of uploaded artifact'
    value: ${{ steps.upload.outputs.artifact-name }}

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install dependencies
      shell: bash
      run: npm ci

    - name: Build Application
      id: build
      shell: bash
      run: |
        START_TIME=$(date +%s)
        
        if [ "${{ inputs.build-type }}" == "production" ]; then
          npm run build:prod
        else
          npm run build:dev
        fi
        
        END_TIME=$(date +%s)
        DURATION=$((END_TIME - START_TIME))
        echo "duration=$DURATION" >> $GITHUB_OUTPUT

    - name: Bundle Analysis
      if: inputs.bundle-analysis == 'true'
      shell: bash
      run: |
        npm run bundle-analysis
        # Upload bundle stats as artifact
        mkdir -p bundle-stats
        cp dist/stats.json bundle-stats/ || true

    - name: Upload Build Artifacts
      id: upload
      if: inputs.upload-artifacts == 'true'
      uses: actions/upload-artifact@v4
      with:
        name: build-${{ github.sha }}
        path: |
          dist/
          bundle-stats/
        retention-days: 7
      shell: bash
      run: echo "artifact-name=build-${{ github.sha }}" >> $GITHUB_OUTPUT
```

---

## Golden Path Workflow

### Complete CI/CD Pipeline

```yaml
# .github/workflows/golden-path.yml
name: Golden Path CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  release:
    types: [created]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true

permissions:
  contents: read
  id-token: write
  packages: write
  security-events: write

jobs:
  # ===========================================
  # VALIDATION STAGE
  # ===========================================
  validate:
    name: Validate Code Quality
    runs-on: ubuntu-latest
    timeout-minutes: 15
    outputs:
      lint-result: ${{ steps.validate.outputs.lint-result }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Validation
        id: validate
        uses: ./.github/actions/validate
        with:
          node-version: ${{ env.NODE_VERSION }}
          fail-on-security-issue: true

  # ===========================================
  # TEST STAGE (SHARDED)
  # ===========================================
  test:
    name: Test Shard ${{ matrix.shard }}
    needs: validate
    runs-on: ubuntu-latest
    timeout-minutes: 30
    strategy:
      fail-fast: false
      matrix:
        shard: [0, 1, 2, 3]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Tests
        id: test
        uses: ./.github/actions/test
        with:
          node-version: ${{ env.NODE_VERSION }}
          shard-count: 4
          shard-index: ${{ matrix.shard }}
          coverage-enabled: ${{ matrix.shard == 0 }}
          flaky-detection: true

  # Aggregate test results
  test-aggregate:
    name: Aggregate Test Results
    needs: test
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Check test results
        run: |
          if [[ "${{ needs.test.result }}" == "failure" ]]; then
            echo "❌ Some test shards failed"
            exit 1
          fi
          echo "✅ All test shards passed"

  # ===========================================
  # BUILD STAGE
  # ===========================================
  build:
    name: Build Application
    needs: [validate, test-aggregate]
    runs-on: ubuntu-latest
    timeout-minutes: 20
    outputs:
      artifact-name: ${{ steps.build.outputs.artifact-name }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Build
        id: build
        uses: ./.github/actions/build
        with:
          node-version: ${{ env.NODE_VERSION }}
          build-type: production
          upload-artifacts: true
          bundle-analysis: true

  # ===========================================
  # SECURITY SCAN STAGE
  # ===========================================
  security-scan:
    name: Security Scanning
    needs: build
    runs-on: ubuntu-latest
    timeout-minutes: 20
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript
          queries: security-extended
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
      
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/security-audit
          generateSarif: "1"
      
      - name: Upload Semgrep Results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif

  # ===========================================
  # DEPLOY TO STAGING
  # ===========================================
  deploy-staging:
    name: Deploy to Staging
    needs: [build, security-scan]
    runs-on: ubuntu-latest
    environment: staging
    timeout-minutes: 15
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
          path: dist/
      
      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-staging
          aws-region: us-east-1
      
      - name: Deploy to Staging
        run: |
          aws s3 sync dist/ s3://your-staging-bucket/
          aws cloudfront create-invalidation --distribution-id YOUR_CF_ID --paths "/*"
      
      - name: Run Smoke Tests
        run: |
          npm run test:smoke -- --environment=staging
      
      - name: Notify Deployment
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Staging deployment successful\n*Commit:* ${{ github.sha }}\n*Environment:* staging"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_STAGING }}

  # ===========================================
  # DEPLOY TO PRODUCTION (Manual Approval)
  # ===========================================
  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: 
      name: production
      url: https://your-app.com
    timeout-minutes: 20
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
          path: dist/
      
      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-production
          aws-region: us-east-1
      
      - name: Deploy to Production
        run: |
          aws s3 sync dist/ s3://your-production-bucket/
          aws cloudfront create-invalidation --distribution-id YOUR_CF_ID --paths "/*"
      
      - name: Run Health Checks
        run: |
          for i in {1..5}; do
            curl -f https://your-app.com/health && break || sleep 30
          done
      
      - name: Create Deployment Record
        run: |
          cat << EOF > deployment-record.json
          {
            "version": "${{ github.sha }}",
            "deployed_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "deployed_by": "${{ github.actor }}",
            "environment": "production"
          }
          EOF
      
      - name: Notify Production Deployment
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚀 Production deployment successful\n*Version:* ${{ github.sha }}\n*Deployed by:* ${{ github.actor }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_PRODUCTION }}
```

---

## Change Detection for Optimized Testing

### Detect Changed Files

```yaml
# .github/workflows/smart-test.yml
name: Smart Test Runner

on:
  pull_request:
    branches: [main]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.changes.outputs.frontend }}
      backend: ${{ steps.changes.outputs.backend }}
      shared: ${{ steps.changes.outputs.shared }}
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Detect Changes
        id: changes
        uses: dorny/paths-filter@v3
        with:
          filters: |
            frontend:
              - 'src/frontend/**'
              - 'packages/ui/**'
            backend:
              - 'src/backend/**'
              - 'packages/api/**'
            shared:
              - 'src/shared/**'
              - 'packages/utils/**'

  test-frontend:
    needs: detect-changes
    if: needs.detect-changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Frontend Tests
        run: npm run test:frontend

  test-backend:
    needs: detect-changes
    if: needs.detect-changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Backend Tests
        run: npm run test:backend

  test-shared:
    needs: detect-changes
    if: needs.detect-changes.outputs.shared == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Shared Tests
        run: npm run test:shared

  aggregate:
    needs: [test-frontend, test-backend, test-shared]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Check results
        run: |
          if [[ "${{ needs.test-frontend.result }}" == "failure" ]] || \
             [[ "${{ needs.test-backend.result }}" == "failure" ]] || \
             [[ "${{ needs.test-shared.result }}" == "failure" ]]; then
            exit 1
          fi
```

---

## Canary Deployments

```yaml
# .github/workflows/canary-deploy.yml
name: Canary Deployment

on:
  workflow_dispatch:
    inputs:
      percentage:
        description: 'Traffic percentage for canary (1-100)'
        required: true
        default: '10'

jobs:
  canary:
    runs-on: ubuntu-latest
    environment: production-canary
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy Canary
        run: |
          # Deploy new version to canary instances
          kubectl set image deployment/app app=your-registry/app:${{ github.sha }} \
            -n production-canary
      
      - name: Scale Canary Instances
        run: |
          kubectl scale deployment/app-canary --replicas=2 -n production-canary
      
      - name: Configure Traffic Split
        run: |
          # Route {{ github.event.inputs.percentage }}% traffic to canary
          istio-traffic-split --canary-percentage=${{ github.event.inputs.percentage }}
      
      - name: Monitor Canary Metrics
        run: |
          # Wait and collect metrics
          sleep 300
          
          ERROR_RATE=$(curl -s https://metrics.your-app.com/error-rate?version=${{ github.sha }})
          LATENCY_P99=$(curl -s https://metrics.your-app.com/latency-p99?version=${{ github.sha }})
          
          echo "Error Rate: ${ERROR_RATE}%"
          echo "P99 Latency: ${LATENCY_P99}ms"
          
          if (( $(echo "$ERROR_RATE > 1" | bc -l) )); then
            echo "❌ Error rate too high, triggering rollback"
            exit 1
          fi
      
      - name: Promote or Rollback
        run: |
          if [ $? -eq 0 ]; then
            echo "✅ Canary successful, promoting to full deployment"
            # Trigger full deployment workflow
          else
            echo "❌ Canary failed, rolling back"
            kubectl rollout undo deployment/app -n production-canary
          fi
```

---

## Environment Configuration

### GitHub Environments Setup

Configure these environments in your repository settings:

```yaml
# Environment: staging
name: staging
url: https://staging.your-app.com
deployment_branch_policy:
  protected_branches: false
  custom_branches:
    - develop
    - main
rules:
  - type: wait_timer
    wait_timer: 0
  - type: required_reviewers
    reviewers:
      - type: team
        id: 123456  # DevOps team

# Environment: production
name: production
url: https://your-app.com
deployment_branch_policy:
  protected_branches: true
  custom_branches:
    - main
rules:
  - type: wait_timer
    wait_timer: 300  # 5 minute wait
  - type: required_reviewers
    reviewers:
      - type: team
        id: 123456  # DevOps team
      - type: team
        id: 789012  # On-call team
  - type: prevent_self_review
    prevent_self_review: true
```

---

## Implementation Checklist

### Week 1: Foundation
- [ ] Create composite actions for validate, test, build
- [ ] Migrate existing workflows to use composite actions
- [ ] Set up GitHub Environments (staging, production)
- [ ] Configure OIDC for cloud provider authentication

### Week 2-3: Optimization
- [ ] Implement test sharding (4+ shards)
- [ ] Add change detection for selective testing
- [ ] Configure bundle analysis and performance budgets
- [ ] Set up deployment notifications (Slack, Teams)

### Month 2: Advanced Patterns
- [ ] Implement canary deployment strategy
- [ ] Add automated rollback on health check failures
- [ ] Integrate observability (logging, metrics, tracing)
- [ ] Create deployment dashboards

### Ongoing: Continuous Improvement
- [ ] Monitor workflow execution times
- [ ] Optimize slowest jobs quarterly
- [ ] Review and update composite actions monthly
- [ ] Conduct post-incident workflow reviews

---

<Warning>
**Security Critical**: Always use OIDC instead of long-lived credentials. Pin all action versions to full SHA hashes in production workflows.
</Warning>

<Success>
**Elite Practice**: Your CI/CD should be so reliable that developers trust it implicitly. If a test fails, they should immediately know it's their code, not the pipeline.
</Success>

---

## Related Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Composite Actions Guide](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#runs-for-composite-actions)
- [OIDC with GitHub Actions](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Test Sharding Best Practices](https://docs.cypress.io/guides/guides/parallelization)

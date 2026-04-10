# Governance & Security: Enterprise-Grade Policy Enforcement

<Tip>
**TL;DR**: Modern governance uses GitHub Repository Rulesets instead of branch protection, implements Policy as Code for automated enforcement, and integrates security scanning throughout the CI/CD pipeline. This guide provides production-ready configurations for teams of 50+ engineers.
</Tip>

## Introduction

As engineering organizations scale beyond 50 engineers, maintaining security, compliance, and consistency across hundreds of repositories becomes a critical challenge. Elite engineering teams shift from manual governance to **automated enforcement** through Policy as Code, ensuring standards are applied consistently without creating bottlenecks.

This guide covers:
- **GitHub Repository Rulesets**: The modern replacement for branch protection rules
- **Policy as Code**: Automated policy enforcement using Probot and Safe-Settings
- **Supply Chain Security**: SLSA, SBOMs, and artifact signing
- **Security Automation**: Integrated scanning and vulnerability management

---

## GitHub Repository Rulesets: Modern Governance

### Why Rulesets Over Branch Protection?

| Feature | Branch Protection | Repository Rulesets |
|---------|------------------|---------------------|
| Scope | Per-branch | Organization-wide or repository-level |
| Conditions | Limited | Rich conditions (actor, repository, branch patterns) |
| Enforcement | Manual configuration | API-driven, version-controlled |
| Scalability | Poor (per-repo config) | Excellent (centralized policies) |
| Audit Trail | Limited | Comprehensive via audit logs |

### Enterprise Ruleset Configuration

```json
{
  "name": "enterprise-main-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main", "refs/heads/master"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 2,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "require_last_push_approval": true,
        "allowed_merge_methods": ["squash", "rebase"],
        "automatic_copilot_code_review_enabled": true
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          { "context": "validate" },
          { "context": "test" },
          { "context": "security-scan" },
          { "context": "codeql" }
        ]
      }
    },
    {
      "type": "required_signatures",
      "parameters": {}
    },
    {
      "type": "commit_message_pattern",
      "parameters": {
        "pattern": "^(feat|fix|docs|style|refactor|perf|test|chore|ci|build|revert): .+",
        "name": "Conventional Commits"
      }
    },
    {
      "type": "commit_author_email",
      "parameters": {
        "operator": "ends_with",
        "pattern": "@yourcompany.com"
      }
    },
    {
      "type": "max_commits_per_push",
      "parameters": {
        "max_count": 20
      }
    }
  ]
}
```

### Ruleset for Release Branches

```json
{
  "name": "release-branch-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/release/*"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 2,
        "require_code_owner_review": true,
        "allowed_merge_methods": ["merge"]
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          { "context": "validate" },
          { "context": "test" },
          { "context": "integration-tests" },
          { "context": "performance-tests" }
        ]
      }
    },
    {
      "type": "required_deployment_branch_policy",
      "parameters": {
        "custom_branches": ["main", "develop"]
      }
    }
  ]
}
```

### Tag Protection Ruleset

```json
{
  "name": "tag-protection",
  "target": "tag",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/tags/v*", "refs/tags/release-*"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "required_deployments",
      "parameters": {
        "required_deployment_environments": ["staging"]
      }
    },
    {
      "type": "non_fast_forward",
      "parameters": {}
    }
  ]
}
```

---

## Policy as Code: Automated Enforcement

### Implementing with Probot

Probot allows you to build GitHub Apps that automate governance policies:

```javascript
// probot-app/index.js
module.exports = (app) => {
  // Enforce CODEOWNERS on every PR
  app.on('pull_request.opened', async (context) => {
    const { owner, repo, number } = context.issue();
    
    // Check if CODEOWNERS file exists
    try {
      await context.octokit.repos.getContent({
        owner,
        repo,
        path: '.github/CODEOWNERS'
      });
    } catch (error) {
      await context.octokit.issues.createComment({
        owner,
        repo,
        issue_number: number,
        body: '⚠️ This repository is missing a CODEOWNERS file. Please add one to ensure proper review assignment.'
      });
    }
  });

  // Enforce PR size limits
  app.on('pull_request.synchronize', async (context) => {
    const { owner, repo, number } = context.issue();
    const pr = await context.octokit.pulls.get({ owner, repo, pull_number: number });
    
    if (pr.data.changed_files > 500) {
      await context.octokit.issues.addLabels({
        owner,
        repo,
        issue_number: number,
        labels: ['needs-triage', 'large-pr']
      });
      
      await context.octokit.issues.createComment({
        owner,
        repo,
        issue_number: number,
        body: '🚨 This PR changes more than 500 files. Consider breaking it into smaller, reviewable chunks.'
      });
    }
  });

  // Enforce security scan completion
  app.on('pull_request.opened', async (context) => {
    const { owner, repo } = context.repo();
    
    // Check if security workflow is configured
    const workflows = await context.octokit.actions.listRepoWorkflows({
      owner,
      repo
    });
    
    const hasSecurityScan = workflows.data.workflows.some(
      w => w.name.includes('security') || w.name.includes('codeql')
    );
    
    if (!hasSecurityScan) {
      await context.octokit.issues.createComment({
        owner,
        repo,
        issue_number: context.issue().number,
        body: '⚠️ This repository does not have automated security scanning enabled. Please add CodeQL or similar security checks.'
      });
    }
  });
};
```

### Safe-Settings Configuration

Safe-Settings allows you to manage repository settings as code:

```yaml
# .github/settings.yml
repository:
  name: my-repository
  description: "Production service for user authentication"
  homepage: "https://auth.yourcompany.com"
  topics: ["authentication", "security", "backend"]
  private: true
  has_issues: true
  has_projects: false
  has_wiki: false
  default_branch: main
  allow_squash_merge: true
  allow_merge_commit: false
  allow_rebase_merge: true
  delete_branch_on_merge: true
  enable_automated_security_fixes: true
  enable_vulnerability_alerts: true

branches:
  - name: main
    protection:
      required_pull_request_reviews:
        required_approving_review_count: 2
        dismiss_stale_reviews: true
        require_code_owner_reviews: true
        require_last_push_approval: true
      required_status_checks:
        strict: true
        contexts:
          - validate
          - test
          - security-scan
      enforce_admins: true
      restrictions: null
      allow_force_pushes: false
      allow_deletions: false
      block_creations: false
      required_linear_history: true
      required_conversation_resolution: true

teams:
  - name: backend-team
    permission: push
  - name: security-team
    permission: admin
  - name: devops-team
    permission: maintain

labels:
  - name: security
    color: d73a4a
    description: "Security-related issues"
  - name: compliance
    color: 0e8a16
    description: "Compliance requirements"
  - name: needs-review
    color: fbca04
    description: "Awaiting code review"
```

---

## Supply Chain Security: Beyond Dependency Scanning

### SLSA Compliance Framework

The Supply-chain Levels for Software Artifacts (SLSA) provides a framework for securing your supply chain:

```yaml
# .github/workflows/slsa-build.yml
name: SLSA Compliant Build

on:
  push:
    branches: [main]
  release:
    types: [created]

permissions: read-all

jobs:
  build:
    permissions:
      contents: write
      id-token: write
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be80849d7340846
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Build
        run: npm run build
        
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          format: spdx-json
          output-file: sbom.spdx.json
          
      - name: Sign artifacts with Sigstore
        uses: sigstore/cosign-installer@v3
        with:
          cosign-release: 'v2.2.2'
          
      - name: Sign container image
        run: |
          cosign sign --yes ${{ env.IMAGE_NAME }}:${{ github.sha }}
        env:
          IMAGE_NAME: ghcr.io/${{ github.repository }}
          
      - name: Upload attestation
        uses: actions/upload-artifact@v4
        with:
          name: attestation
          path: ./*.sig
```

### SBOM Generation Workflow

```yaml
# .github/workflows/sbom.yml
name: SBOM Generation

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1' # Weekly

permissions:
  contents: read
  security-events: write

jobs:
  generate-sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate SBOM with Syft
        uses: anchore/sbom-action@v0
        with:
          format: cyclonedx-json
          output-file: sbom.cyclonedx.json
          
      - name: Upload SBOM as artifact
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.cyclonedx.json
          
      - name: Submit SBOM to Dependency Graph
        uses: anchore/sbom-action/publish-sbom@v0
        with:
          sbom_artifact: sbom
```

### Artifact Signing and Verification

```bash
#!/bin/bash
# scripts/sign-release.sh

set -euo pipefail

ARTIFACT_PATH=$1
VERSION=$2

# Sign the artifact
cosign sign-blob --output-signature "${ARTIFACT_PATH}.sig" "${ARTIFACT_PATH}"

# Generate provenance attestation
cosign attest --predicate predicate.json "${ARTIFACT_PATH}"

# Verify signature
cosign verify-blob --signature "${ARTIFACT_PATH}.sig" "${ARTIFACT_PATH}"

echo "✅ Artifact signed and verified: ${ARTIFACT_PATH}"
```

---

## Security Automation in CI/CD

### Integrated Security Scanning Pipeline

```yaml
# .github/workflows/security-pipeline.yml
name: Security Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * *' # Daily at 6 AM

permissions:
  contents: read
  security-events: write
  actions: read

concurrency:
  group: security-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

jobs:
  codeql-analysis:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript,python
          queries: security-extended,security-and-quality
          
      - name: Autobuild
        uses: github/codeql-action/autobuild@v3
        
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:javascript"

  semgrep-scan:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
          generateSarif: "1"
          
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif

  dependency-audit:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          
      - name: Run npm audit
        run: npm audit --audit-level=moderate
        
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  secret-scanning:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: TruffleHog Secret Scan
        uses: trufflesecurity/trufflehog@v3.63.0
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          extra_args: --only-verified

  container-security:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      
      - name: Build container image
        run: docker build -t app:${{ github.sha }} .
        
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'app:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  security-gate:
    needs: [codeql-analysis, semgrep-scan, dependency-audit, secret-scanning, container-security]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Check security job results
        run: |
          if [[ "${{ needs.codeql-analysis.result }}" == "failure" ]] || \
             [[ "${{ needs.semgrep-scan.result }}" == "failure" ]] || \
             [[ "${{ needs.dependency-audit.result }}" == "failure" ]]; then
            echo "❌ Security checks failed"
            exit 1
          fi
          echo "✅ All security checks passed"
```

---

## Advanced CODEOWNERS Patterns

### Enterprise CODEOWNERS Configuration

```CODEOWNERS
# Global fallback owners
* @engineering-leads @devops-team

# Executive oversight for critical changes
/SECURITY.md @ciso @security-team
/LICENSE @legal-team @cto

# Frontend ownership
/src/components/ @frontend-team @ux-lead
/src/styles/ @frontend-team @design-system-team
/public/ @frontend-team

# Backend ownership
/src/api/ @backend-team @api-architect
/src/services/ @backend-team
/src/models/ @backend-team @database-team

# Infrastructure and DevOps
/.github/workflows/ @devops-team @platform-team
/docker/ @devops-team
/kubernetes/ @devops-team @sre-team
/terraform/ @devops-team @infrastructure-team

# Security-critical paths
/**/auth* @security-team @backend-team
/**/crypto* @security-team
/**/secrets* @security-team
/.env* @security-team @devops-team

# Documentation
/docs/ @docs-team @technical-writers
/*.md @docs-team

# Configuration files
/package.json @frontend-team @backend-team
/tsconfig.json @frontend-team
/pyproject.toml @backend-team
/docker-compose.yml @devops-team

# Test ownership
/tests/e2e/ @qa-team @sre-team
/tests/integration/ @backend-team @qa-team
/tests/unit/ @backend-team @frontend-team

# Performance-critical code
/**/performance* @performance-team
/**/cache* @backend-team @performance-team
/**/optimization* @performance-team

# Compliance and audit trails
/audit/ @compliance-team @security-team
/logs/ @sre-team @compliance-team
```

---

## Implementation Roadmap

### Week 1-2: Foundation
- [ ] Audit existing branch protection rules
- [ ] Create organization-wide ruleset templates
- [ ] Implement basic CODEOWNERS structure
- [ ] Enable Dependabot for all repositories

### Week 3-4: Automation
- [ ] Deploy Probot app for policy enforcement
- [ ] Configure Safe-Settings for repository management
- [ ] Implement security scanning pipeline
- [ ] Set up SBOM generation

### Month 2: Advanced Security
- [ ] Achieve SLSA Level 2 compliance
- [ ] Implement artifact signing
- [ ] Deploy secret scanning across all repos
- [ ] Establish security metrics dashboard

### Month 3: Optimization
- [ ] Refine rulesets based on team feedback
- [ ] Automate policy exception handling
- [ ] Integrate with compliance reporting tools
- [ ] Conduct security governance training

---

<Warning>
**Critical**: Never commit secrets or credentials, even in governance configurations. Use GitHub Secrets, OIDC, or external secret managers (HashiCorp Vault, AWS Secrets Manager) for all sensitive data.
</Warning>

<Success>
**Elite Practice**: Governance should be invisible to developers when things are working correctly. Focus on automation, clear error messages, and self-service remediation paths.
</Success>

---

## Related Resources

- [GitHub Repository Rulesets Documentation](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [Probot Framework](https://probot.github.io/)
- [Safe-Settings App](https://github.com/apps/settings)
- [SLSA Specification](https://slsa.dev/)
- [Sigstore Project](https://www.sigstore.dev/)

# GitHub Actions and Automation

GitHub Actions can automate checks, releases, and deployments. A useful workflow gives fast, trustworthy feedback while keeping permissions and secrets limited to the jobs that need them. Examples below use current major action versions; review upstream release notes and pin actions to full commit SHAs when your supply-chain policy requires immutable references.

## A practical CI workflow

This example assumes a Node.js project with a committed `package-lock.json` and the `lint`, `test`, and `build` scripts in `package.json`. Change the scripts and supported Node.js versions to match the project.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node-version: [22, 24]
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

Each job runs on a fresh runner. If checks are split into separate jobs, each job must check out the source and install its own dependencies. Jobs do not share a filesystem unless artifacts or another explicit storage mechanism is used. Add a concurrency group when newer commits should cancel superseded pull-request runs.

`setup-node` can cache package-manager data without a separate cache step. It does not cache `node_modules`. For repositories with elevated workflow permissions or sensitive data, disable automatic package-manager caching when it is not needed, and review cache access across branches and pull requests.[1]

For a Python project, use an explicit cache key and the correctly hyphenated `restore-keys` input:

```yaml
- name: Cache Python downloads
  uses: actions/cache@v6
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

A cache is an optimization, not a source of truth. Always install from a lockfile or pinned requirements and ensure a cache miss still produces a correct build.[2]

## Build and publish artifacts

Artifacts let one job pass build output to another job without rebuilding. Give each artifact a useful name and a deliberate retention period. Do not put credentials or other secrets in an artifact.

```yaml
- name: Upload build output
  uses: actions/upload-artifact@v7
  with:
    name: web-dist
    path: dist/
    if-no-files-found: error
    retention-days: 7
```

Use artifacts to transfer immutable build output between jobs, then deploy the exact artifact that passed the required checks. Do not assume a new runner contains output from a previous job.

## Deployment environments and approvals

Define environments such as `staging` and `production` in repository settings. Configure allowed branches, required reviewers, wait timers, and environment secrets there. The workflow's `environment` key selects an environment; it does not by itself create an approval rule. A job cannot access environment secrets until configured protection rules pass.[3]

Required reviewer availability and other environment features vary by repository visibility and plan. Check current GitHub plan limits before making an approval rule part of a release guarantee.[3]

## AWS deployment with OIDC

Prefer short-lived credentials through OpenID Connect (OIDC) over long-lived access keys. This example assumes the AWS identity provider, IAM role, trust policy, bucket, and GitHub environment have already been configured. The IAM trust policy should constrain access to the intended repository and branch or environment. Match the policy to the claims your GitHub organization actually emits; do not copy a generic subject claim without checking it.[4]

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy staging

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: 24
          cache: npm
      - run: npm ci
      - run: npm run build
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ vars.AWS_STAGING_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - name: Deploy
        env:
          BUCKET: ${{ vars.AWS_STAGING_BUCKET }}
        run: aws s3 sync dist/ "s3://${BUCKET}" --delete
```

The AWS credentials action requires `aws-region`. The role ARN is a configuration value rather than a password, so it can be stored as an environment variable. Keep the IAM role limited to the resources and operations the deployment needs. For production, use a separate role and environment, and configure production protection in repository settings.[3] [4]

When an OIDC trust policy uses an environment, its subject claim identifies that environment. Branch restrictions may therefore need to be enforced on the GitHub environment as well as in AWS. Review current GitHub OIDC claim formats and AWS trust conditions when creating or changing the policy.[4]

## Rollback and release safety

Prefer versioned, immutable deployment artifacts. Deploy a known build, verify health after rollout, and keep a tested way to restore the previous version. Database changes need a compatibility and recovery plan; reverting application code alone may not reverse a schema change.

For higher-risk releases, use a staged rollout, health checks, and an explicit decision point before increasing traffic. Record who owns the rollback decision and how operators will trigger it. Avoid automatic rollback rules that can repeatedly flip a service without a human-visible incident signal.

## Code scanning with CodeQL

This example is for a JavaScript or TypeScript repository. Use CodeQL's supported language identifier and change the triggers and permissions to fit the repository.

```yaml
# .github/workflows/codeql.yml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '23 4 * * 1'

permissions:
  actions: read
  contents: read
  security-events: write

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: github/codeql-action/init@v4
        with:
          languages: javascript-typescript
      - uses: github/codeql-action/autobuild@v4
      - uses: github/codeql-action/analyze@v4
```

Check the CodeQL documentation and action release notes for supported language identifiers, build requirements, and permissions before copying this workflow to another language or repository.[5]

## Reusable workflows

Put a reusable workflow in `.github/workflows/` and declare its inputs and secrets under `workflow_call`. Keep permissions narrow in both the caller and reusable workflow. A called workflow cannot elevate the permissions granted by its caller.

```yaml
# .github/workflows/reusable-node-checks.yml
name: Reusable Node checks

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: '24'

permissions:
  contents: read

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - run: npm ci
      - run: npm test
```

Call it from another workflow with a job-level `uses` entry:

```yaml
jobs:
  test:
    uses: ./.github/workflows/reusable-node-checks.yml
    with:
      node-version: '24'
```

Use reusable workflows for shared job-level procedures. Use a composite action when the reusable unit is a sequence of steps inside a job. Keep inputs explicit and version shared workflows when they are consumed across repositories.

## Workflow security

### Set minimum permissions

Start with read-only repository contents at workflow level. Grant additional scopes only to the job that needs them. OIDC deployment needs `id-token: write`; code scanning may need `security-events: write`. Avoid granting `contents: write` or broad organization permissions to routine test jobs.[1]

### Handle untrusted pull-request code safely

Treat pull-request titles, branch names, issue text, and source code as untrusted input. Do not interpolate attacker-controlled expressions directly into shell scripts. Pass values through environment variables and quote them, or use a safe action interface.

Workflows triggered by `pull_request_target` run with elevated access to the base repository. Do not check out and execute untrusted pull-request code in that privileged context. Prefer `pull_request` for ordinary validation, and review GitHub's secure-use guidance before using `pull_request_target` or `workflow_run`.[1]

### Protect secrets and credentials

- Store credentials in GitHub secrets or a supported identity federation flow. Never commit them or print them to logs.
- Give each credential only the resource access required for its task. Rotate credentials if they are exposed.
- Prefer environment-scoped deployment credentials and gate access to them with environment protection rules.
- Do not run untrusted code in a job that can read deployment secrets.
- Review third-party action source and permissions before adoption.

### Pin action dependencies

For a stricter supply-chain policy, pin each action to a full commit SHA from the action's own repository. A version comment helps maintainers update the pin deliberately. This checkout SHA is from the verified upstream v7.0.1 release:[6]

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

A major-version tag such as `@v7` is easier to update but can move as new releases are published. If using tags, review updates promptly and apply an automated dependency update process. GitHub's security guidance recommends full-length SHA pins where immutable action references are required.[1]

## Self-hosted runners

Self-hosted runners can provide specialized hardware or access to private networks. They also retain more state and are not automatically isolated between jobs in the same way as fresh hosted runners. Do not use a persistent runner for untrusted pull-request code if it can access secrets, deployment credentials, or sensitive network resources. Keep runner images patched, restrict runner groups, and remove sensitive state after each job.[1]

## Troubleshooting and operations

Use the Actions run view to inspect logs and job summaries. The GitHub CLI can list and inspect runs:

```bash
gh run list
gh run view <run-id>
```

Track duration, failure rate, queue time, and flaky checks at workflow level. Fix recurring failures rather than normalizing retries. Keep logs useful, but do not print secrets or unnecessary personal data.

## Implementation checklist

- [ ] Start with read-only token permissions and expand per job only as required.
- [ ] Check out code and install dependencies independently in every job that needs them.
- [ ] Commit lockfiles and verify that cache misses do not change build correctness.
- [ ] Transfer build output explicitly with artifacts when jobs are separated.
- [ ] Configure environment approvals and branch restrictions in repository settings.
- [ ] Prefer OIDC or short-lived credentials over stored cloud access keys.
- [ ] Keep untrusted pull-request code away from privileged workflows and persistent runners.
- [ ] Pin actions to verified full commit SHAs when policy requires immutable dependencies.
- [ ] Test deployment, health checks, and rollback procedures before relying on them.

## References

[1]: https://docs.github.com/en/actions/reference/security/secure-use "GitHub Actions secure use reference"
[2]: https://github.com/actions/cache "actions/cache documentation"
[3]: https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments "GitHub Actions deployments and environments"
[4]: https://github.com/aws-actions/configure-aws-credentials "AWS credentials action and OIDC guidance"
[5]: https://github.com/github/codeql-action/releases "CodeQL action releases"
[6]: https://github.com/actions/checkout/releases/tag/v7.0.1 "actions/checkout v7.0.1 release"

*Last reviewed: 2026-11-24*

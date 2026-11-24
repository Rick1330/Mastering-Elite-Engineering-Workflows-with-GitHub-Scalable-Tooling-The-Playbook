# Repository Structure: Organizing for Scale

## Introduction

Repository boundaries shape code ownership, access control, build and release workflows, and the cost of cross-team changes. There is no universally best monorepo or polyrepo model. Choose the simplest structure that fits product boundaries, team responsibilities, security needs, and release cadence, then revisit the decision as those conditions change.

This guide covers the choice between monorepos and polyrepos, repository layout, dependency practices, visibility, and discovery. It is a decision and implementation guide, not a prescription based on headcount alone.

## Choose repository boundaries

### Monorepo

A monorepo stores multiple related applications, libraries, or services in one Git repository. It can make atomic cross-project changes and shared tooling easier. It also increases the need for clear ownership, selective builds, and access-control design.

A monorepo is a good candidate when projects frequently change together, share internal packages, or benefit from common build and test tooling. It is a poor fit when projects require sharply different access boundaries, have unrelated release and ownership models, or cannot be isolated by the available tooling.

### Polyrepo

A polyrepo model gives each service, product, or library its own repository. This supports independent permissions, ownership, and releases. It adds work for cross-repository changes, shared tooling, dependency upgrades, and organization-wide discovery.

Polyrepo can work well when components have independent lifecycles, different security requirements, or distinct maintainer communities. It is not automatically faster: build time depends on the project and CI design, not only on repository count.

### Hybrid model

A hybrid model uses a monorepo for tightly coupled applications and shared libraries while keeping independently governed products or infrastructure in separate repositories. Define the boundary explicitly. Avoid splitting a codebase solely because one team or one tool prefers it.

### Decision questions

Before choosing or changing a boundary, answer:

1. Which components routinely need to change and deploy together?
2. Who owns each component, and can ownership be represented clearly in the chosen structure?
3. Do teams need separate read/write access or different compliance controls?
4. Can the build system identify affected projects and reuse build outputs safely?
5. How will dependencies be versioned, upgraded, and supported?
6. How will engineers discover a repository, its owners, deployment path, dependencies, and support channel?
7. What migration cost and operational burden will the new boundary create?

Record the decision, assumptions, and signals that would trigger a review. Re-evaluate using actual change and operational data rather than a generic team-size threshold.

## Recommended layouts

### Application repository

```text
service-name/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   ├── workflows/
│   └── PULL_REQUEST_TEMPLATE.md
├── docs/
├── src/
├── tests/
├── scripts/
├── config/
├── .gitignore
├── CONTRIBUTING.md
├── LICENSE
├── README.md
├── SECURITY.md
└── package.json or pyproject.toml or go.mod
```

Treat `dist/`, `build/`, `target/`, coverage output, and other generated files as build outputs. Commit them only when the publishing or runtime model specifically requires it; otherwise ignore them and publish versioned artifacts from CI.

### Monorepo

```text
product/
├── .github/
│   └── workflows/
├── apps/
│   ├── web/
│   └── admin/
├── services/
│   ├── billing/
│   └── identity/
├── packages/
│   └── shared-ui/
├── infra/
├── tools/
├── docs/
├── CODEOWNERS
├── package.json
└── workspace-config.json
```

Group code by deployable or owned unit. Give each unit a clear owner and an explicit dependency direction. Keep shared libraries small and intentional. Use a workspace/build tool that supports affected-project detection, caching, and dependency graphs when repository size makes full builds too expensive.

### Library repository

```text
library-name/
├── .github/
│   └── workflows/
├── docs/
├── examples/
├── src/
├── tests/
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE
├── README.md
└── package.json or pyproject.toml or go.mod
```

Document the supported runtime and compatibility policy. Test published packages from the built artifact, not only from the source checkout.

## Monorepo implementation

Use a workspace or build system that matches the ecosystem and scale. Nx, Turborepo, Bazel, Gradle, and language-native workspaces solve overlapping but not identical problems. Compare setup cost, incremental correctness, remote-cache trust, supported languages, local developer workflow, and maintenance ownership before adopting one.

A practical rollout is:

1. Map packages, deployable units, dependencies, and owners.
2. Establish a consistent package and test layout before moving code.
3. Introduce workspace tooling and make local commands work without CI-specific assumptions.
4. Add CI that tests affected projects and a scheduled or protected full-suite run.
5. Cache only reproducible inputs and outputs; include toolchain and lockfile state in cache keys.
6. Publish artifacts with immutable versions and explicit provenance.
7. Track cross-project build time, cache hit rate, flaky tests, and ownership gaps.

Do not use path filters as the only correctness guard if changes can affect shared tooling or downstream projects. Include shared configuration in the affected-project graph or run broader checks when impact cannot be determined.

## Polyrepo implementation

Standardize the minimum needed to make repositories predictable while allowing teams to choose their own implementation details. A shared template can provide repository metadata, contribution and security files, baseline CI, branch rules, and dependency automation.

For shared packages, publish immutable versions to an appropriate registry and use each consumer's lockfile or dependency lock mechanism. Define compatibility and deprecation policies. Avoid cross-repository coordination based on unversioned branches or undocumented manual steps.

Where coordinated releases are required, use an explicit release plan, compatible version ranges, and an integration environment. For automation between repositories, scope credentials narrowly, make operations idempotent, and define what happens when one step fails.

## Dependency and supply-chain practices

- Commit the ecosystem's lockfile where applicable and keep it in sync with dependency manifests.
- Use supported version ranges or exact versions according to the package manager and release policy; do not apply a single pinning rule to every ecosystem.
- Automate update proposals, then test and review them. Separate urgent security fixes from routine upgrades.
- Scan the relevant artifact types: source-code analysis, dependency vulnerabilities, container images, and infrastructure configuration are different checks.
- Generate an SBOM when a product, customer, or regulatory requirement justifies it. State the format, scope, and release to which it applies.
- Protect build credentials, registries, caches, and release artifacts. Define who can publish and how compromised credentials are revoked.

## Visibility and discoverability

Choose public, internal, or private visibility according to data classification, open-source intent, customer commitments, and access needs. Visibility does not replace permissions for sensitive actions or secrets. Review visibility when ownership or product scope changes.

Every repository should make these items easy to find:

- Purpose, lifecycle status, and supported versions.
- Owning team and a support or escalation path.
- Setup, test, build, and release commands.
- Deployment target and operational dashboard or runbook.
- Dependencies, APIs, and consumers where material.
- Contribution and security-reporting instructions.

For larger estates, maintain a service catalog or repository index with ownership, criticality, dependencies, deployment status, and documentation links. Assign an owner to keep that metadata current. Innersource works when contribution expectations, review ownership, and support commitments are clear; visibility alone does not create collaboration.

## Common failure patterns

- Repositories grouped by current team structure even though ownership boundaries change frequently.
- A monorepo without affected-project checks, module ownership, or a reliable local workflow.
- Polyrepos that duplicate workflow policy while allowing security controls to drift.
- Shared libraries published without compatibility guarantees or a maintainer.
- Generated build outputs committed accidentally or used as the only release artifact.
- A service catalog with no owner or update process.
- Repository migrations performed without a rollback plan, redirect strategy, or dependency inventory.

## Implementation checklist

- [ ] Write down the boundary decision and the conditions that would change it.
- [ ] Identify owners for each repository, service, package, and sensitive path.
- [ ] Standardize the minimum repository metadata and onboarding commands.
- [ ] Ensure dependencies and build outputs are reproducible and distinguishable.
- [ ] Add CI checks appropriate to the repository's risk and release model.
- [ ] Document release, rollback, and support paths.
- [ ] Add repositories and services to a maintained discovery index.
- [ ] Review the structure after significant changes to teams, products, access needs, or deployment cadence.

*Last reviewed: 2026-11-24*

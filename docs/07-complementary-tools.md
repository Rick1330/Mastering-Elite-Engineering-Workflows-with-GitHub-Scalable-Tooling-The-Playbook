# Complementary Tools & Integrations

## Purpose

GitHub can be the source repository and automation hub while other services support planning, code quality, observability, or developer workflows. Add an integration only when it solves a defined problem that GitHub or the current toolchain does not already solve well. More integrations also mean more credentials, data copies, failure modes, and maintenance.

## Choose tools by capability

- **Planning and issue tracking:** GitHub Issues and Projects, Linear, or Jira. Decide which system owns status, priority, and identifiers, and how cross-repository work is linked.
- **Documentation and knowledge:** repository Markdown, a documentation platform, or a knowledge base. Decide which content must be versioned with code and who maintains external links and access.
- **Code quality and coverage:** CodeQL, Codecov, SonarQube, SonarCloud, or Codacy. Confirm language coverage, the required quality gate, and ownership of false positives.
- **Error reporting and product signals:** Sentry, LogRocket, or platform-native observability. Review collected data, access, retention, privacy, and incident ownership.
- **Source search and discovery:** GitHub search, Sourcegraph, or a service catalog. Check repository coverage, index freshness, and protection of private results.
- **Release and package distribution:** GitHub Releases, a package registry, or an artifact repository. Define who can publish and how versions, provenance, retention, and rollback are managed.

These are examples, not endorsements. Product features, plans, and integration behavior change. Validate the specific capability, permissions, data handling, and support commitment before standardizing a tool.

## Integration design principles

### Preserve a clear source of truth

Choose one system to own each important record. If an issue is mirrored between GitHub and another tracker, define which side is authoritative for description, status, and closure. Document synchronization direction, field mapping, conflict behavior, and expected delay. Avoid a two-way sync unless conflicts are explicitly handled.

### Scope credentials narrowly

Use the integration’s narrowest supported permissions. Store credentials in the appropriate secret store, rotate them, and make ownership explicit. Prefer short-lived identity federation when supported over long-lived broad-scope tokens. Never expose credentials to untrusted pull-request code or publish them in logs.

### Make automation safe to retry

Webhook delivery and CI jobs can be delayed, duplicated, or retried. Use stable event identifiers, idempotent updates, bounded retries, and visible failure handling. Do not silently discard a failed sync or deployment notification; route it to an owner and provide a recovery path.

### Keep sensitive data in scope

Before connecting a tool, review what repository content, code, issue metadata, telemetry, or user data it can access. Determine where that data is stored, who can view it, and how it can be deleted. Apply retention and privacy requirements to logs, crash reports, and AI services as well as the Git repository.

### Avoid gates that no one owns

A status check should have a named owner, a documented purpose, and an agreed failure response. Review flaky or redundant checks. Separate advisory signals from checks that block merging so teams understand which requirements are mandatory.

## Common integration patterns

- **Issue tracker link:** Include the external issue key in branch or pull-request metadata, then link records in both systems. Decide whether issue closure follows merge or release.
- **Coverage and analysis:** Run checks on pull requests, publish a readable report, and make only defined thresholds blocking. Establish ownership for baseline updates and false positives.
- **Error tracking:** Include release or commit identifiers in deployments so an incident can be connected to the code and rollout. Restrict access to customer data and redact secrets or personal information.
- **Repository mirroring:** Treat a mirror as a backup or distribution channel with a documented one-way source of truth. Define credential rotation and the response to divergence.
- **CLI automation:** Use the GitHub CLI for repeatable developer actions, but avoid embedding personal tokens in scripts. Provide tested commands, expected permissions, and an error path.

## Build an integration inventory

For each integration, record:

1. Business or engineering purpose and accountable owner.
2. Systems, repositories, and data it can access.
3. Authentication method and permission scope.
4. Source-of-truth and synchronization behavior.
5. Availability, failure, retry, and manual recovery expectations.
6. Cost, retention, privacy, and vendor exit considerations.
7. Review date and decommissioning procedure.

Review this inventory when an integration changes scope, credentials, owner, or data classification. Remove unused applications and revoke their access.

## Selection checklist

- [ ] The need is specific and not already covered by an existing supported tool.
- [ ] The integration has an owner and a maintained support path.
- [ ] The data flow and permissions are understood and documented.
- [ ] Automation is safe under duplicate events, retries, and partial failure.
- [ ] Blocking checks have actionable failure messages and explicit ownership.
- [ ] The team can export or recover important records if the service is unavailable.
- [ ] Costs and security/privacy requirements have been reviewed.

*Last reviewed: 2026-11-24*

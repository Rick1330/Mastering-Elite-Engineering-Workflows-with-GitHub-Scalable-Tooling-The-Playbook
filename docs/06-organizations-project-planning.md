# GitHub Organizations & Project Planning

## Purpose

Organization structure, access rules, and project tracking should make ownership and delivery visible without creating needless hierarchy. This guide provides a practical starting point for GitHub organizations, teams, permissions, Projects, and cross-repository planning. Adapt the examples to the organization’s identity, compliance, and release requirements.

## Organization and team structure

Use an organization boundary for a meaningful security, ownership, or administration boundary. Within an organization, create teams around durable ownership domains such as a product area, platform capability, or service group. Avoid a deep team hierarchy that merely mirrors reporting lines.

A team should have a named maintainer, a clear purpose, and a documented set of repositories or responsibilities. Use parent/child teams only when they improve discovery or permission management. For cross-functional work, add collaborators to a project or issue rather than creating a permanent team with no continuing ownership.

Before choosing one or several organizations, consider:

- Whether repositories need separate visibility, policies, billing, or administrators.
- Whether teams can share repositories and enterprise policy while retaining clear ownership.
- How identity-provider synchronization, offboarding, and audit requirements will work.
- Whether cross-organization collaboration will create duplicate permissions or workflow complexity.

## Access-control model

Grant the least access needed for a person or team to perform its work. Prefer team-based repository permissions over individually managed access. Review access when people change roles and on a regular schedule. Use a time-bounded process for exceptional elevated access and record who approved it and why.

- **Organization owner:** reserve for a small number of trusted administrators. Confirm that each owner needs organization-wide configuration access.
- **Repository administrator:** use for repository settings and access management. Delegate narrower responsibilities when practical.
- **Maintainer or write access:** use for normal contribution and review. Confirm that the team needs write access to each repository in the grant.
- **Triage access:** use for issue and pull-request management when direct code changes are not required.
- **Read access:** use for consumers, auditors, or contributors who do not need to write. Check that repository visibility fits the intended audience.

The names and capabilities of roles can differ by plan and product settings. Validate permissions in the organization before applying a template. Keep emergency access separate from normal day-to-day access and review its use afterward.

## Repository ownership and CODEOWNERS

Keep repository owners, CODEOWNERS entries, and team memberships aligned. A CODEOWNERS file requests reviews from owners; it does not itself block a merge. If owner approval is required, enable the corresponding required-code-owner-review rule in branch protection or a repository ruleset. Owners and teams need appropriate repository access, and the CODEOWNERS file must be present on the pull request’s base branch.

Use valid GitHub usernames or organization-qualified team names. For example:

```text
# .github/CODEOWNERS
*                       @example-org/repo-maintainers
/docs/                  @example-org/documentation
/services/billing/      @example-org/billing-team
/.github/workflows/     @example-org/platform-security
```

Review the file when teams, paths, or ownership change. Use branch rules to protect sensitive ownership and workflow files themselves.

## GitHub Projects as a planning layer

Use a Project when a team needs a shared view across issues, pull requests, repositories, or iterations. Start with the minimum fields needed to answer the team’s planning questions. For example:

- **Status:** Backlog, Ready, In progress, In review, Done.
- **Priority:** A small, agreed scale that identifies sequencing rather than individual performance.
- **Iteration:** A timebox only when the team actually plans work in iterations.
- **Owner:** The person or team coordinating the next action.
- **Target date or milestone:** A commitment with a clear decision-maker and review cadence.

A Kanban team can use a board with explicit work-in-progress limits. A team using Scrum can connect iteration planning to a defined goal, backlog, and review. A roadmap can show outcomes, dependencies, and confidence ranges without treating estimated dates as guarantees. Choose one process model that fits the team; avoid keeping duplicate status fields in both the project and the issue tracker unless ownership is explicit.

## Issue and pull-request templates

A good issue template captures the information needed to route and assess work without forcing every request into a long form. A feature request might ask for the problem, intended outcome, constraints, and acceptance criteria. A bug report should request reproduction steps, expected and observed behavior, affected version, and relevant logs with sensitive values removed.

Pull-request templates should prompt authors to explain the change, link related work, describe tests, identify risk, and note rollout or migration requirements where relevant. Do not make checkboxes a substitute for review; remove questions that do not influence a decision.

## Cross-repository initiatives

For work spanning multiple repositories, designate one parent issue or project item as the coordination record. Link implementation issues in each repository and state the dependency order, owner, integration criteria, and release sequence. Keep local work actionable in its own repository. Update the parent when scope, owner, or dependency dates change.

Use epics, milestones, and roadmaps to answer different questions:

- An epic groups related implementation work toward an outcome.
- A milestone or iteration groups work for a release or timebox.
- A roadmap communicates intended outcomes, sequencing, dependencies, and confidence.

Avoid copying the same long description into many issues. Link to the source of truth and keep a concise summary in each work item.

## Operating cadence and reporting

Set a planning cadence suited to the work. At each review, confirm ownership, priority, dependencies, and whether the work still supports the intended outcome. Close or re-scope stale items. Use cycle time and work-in-progress data to identify flow problems, not to rank individuals. Keep project status useful to contributors by limiting manual updates and automating only reliable transitions.

## Adoption checklist

- [ ] Define organization boundaries, administrators, and team ownership.
- [ ] Apply repository access through durable teams where practical.
- [ ] Document how access is requested, approved, reviewed, and removed.
- [ ] Use branch rules and CODEOWNERS together where owner approval is required.
- [ ] Choose a single source of truth for each planning field.
- [ ] Make cross-repository dependencies and integration ownership explicit.
- [ ] Review stale issues, permissions, teams, and project fields periodically.

*Last reviewed: 2026-11-24*

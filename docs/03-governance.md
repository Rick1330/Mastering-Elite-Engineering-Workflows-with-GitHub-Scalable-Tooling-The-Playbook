# Governance and Security

Repository governance should make safe work the normal path without adding controls that do not reduce a clear risk. Set a small organizational baseline, document exceptions, and review the controls when ownership or system risk changes.

## Repository baseline

A service or library may use a structure like this, adapted to its build system:

```text
.github/
  ISSUE_TEMPLATE/
  workflows/
  PULL_REQUEST_TEMPLATE.md
  CODEOWNERS
docs/
src/
tests/
scripts/
CHANGELOG.md
CODE_OF_CONDUCT.md
CONTRIBUTING.md
LICENSE
README.md
SECURITY.md
```

Use repository templates or an internal starter repository to make the baseline easy to adopt. Keep templates maintained, and avoid requiring files that do not fit the project.

## Branch protection and repository rules

Protect important branches such as `main` with pull-request review and required checks. Choose the number and type of approvals based on change risk and team size. Require checks that actually run for that branch, and periodically remove obsolete check names so the merge queue cannot become stuck.

Consider requiring resolved conversations, linear history, signed commits, or restricted push access when they solve a defined need. Document who can bypass a rule and how emergency changes are reviewed afterward. Rulesets and branch protection offer overlapping controls; verify the exact feature availability and scope in repository settings before standardizing a policy across an organization.

### Signed commits

A signed-commit requirement checks that commits have a signature GitHub can verify. A verified signature is useful evidence about the signing key and integrity of the commit. It does not by itself prove that the named author personally created the change or that the change was reviewed. Treat commit signing as one control, not a replacement for review, access control, or audit.

## CODEOWNERS

A `CODEOWNERS` file routes review requests for paths. Put it in `.github/CODEOWNERS`, the repository root, or `docs/CODEOWNERS`; GitHub uses the first one it finds in that order. Owners must have write access, and teams must be visible and have explicit write access to the repository.[1]

```text
# .github/CODEOWNERS
# Default owner for files not matched below
* @example-org/maintainers

# More specific rules override the earlier matching rule
/docs/ @example-org/docs-team
/src/frontend/ @example-org/frontend-team
/src/backend/ @example-org/backend-team @example-org/security-team
/.github/CODEOWNERS @example-org/maintainers
```

The last matching pattern takes precedence. Patterns follow most, but not all, `.gitignore` behavior. In particular, do not use `^` to make an owner optional or `!` to negate a pattern. CODEOWNERS paths are case-sensitive. If multiple owners appear on one matching line, any one of them can satisfy a required code-owner approval; the rule does not require every listed owner to approve.[1]

A CODEOWNERS entry alone does not prevent merging. Enable required code-owner review in branch protection or an applicable ruleset when owner approval must be a merge requirement. Keep the CODEOWNERS file itself owned so a change cannot silently remove review routing.[1]

## Access management

Grant each person and team the least repository role needed for their work. Prefer team-based access over repeated individual grants in an organization. Remove access when responsibilities end, and review outside collaborators, deploy keys, GitHub Apps, machine users, and personal access tokens on a regular schedule.

Use a consistent process for requesting and approving access. Separate routine contribution from administration: only a small set of trusted maintainers should manage repository settings, rulesets, secrets, and runner groups. Document break-glass access and review its use afterward.

Before rolling out a role model, check the exact permissions attached to the role in the current GitHub settings. Role names and available controls can differ between organization and personal repositories and between GitHub plans.

## Security reporting and maintenance

Add a `SECURITY.md` file that explains supported versions, the private vulnerability-reporting path, and the expected response process. Use private vulnerability reporting or a GitHub Security Advisory where available. Do not ask reporters to publish exploit details in a public issue.

Enable Dependabot alerts to identify vulnerable dependencies. Security updates can open pull requests for vulnerable dependencies when enabled, while version updates keep dependencies current through `dependabot.yml`. Confirm the dependency graph, alerts, security-update setting, and version-update configuration separately. Review proposed updates before merging them.[2] [3]

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: weekly
  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: weekly
```

Adapt package ecosystems and directories to the repository. Use pull-request review, CI, and a plan for handling breaking updates instead of auto-merging every dependency change without qualification.

## Secrets and Actions policy

Store secrets at the narrowest practical scope. Prefer environment-scoped secrets for deployments and short-lived credentials through OIDC for cloud access. Limit `GITHUB_TOKEN` permissions in workflow files and avoid running untrusted pull-request code in privileged contexts. Review third-party actions before use and require verified full-length commit SHA pins if the organization's supply-chain policy calls for immutable action references.[3]

For self-hosted runners, restrict which repositories and workflows can use each runner group. Keep runners patched, prevent untrusted code from reaching persistent state, and remove credentials and workspace data after a job.

## Auditing and exceptions

Maintain an inventory of repositories, owners, criticality, and the controls that apply. Review membership and access grants periodically. Use organization audit logs where available to investigate settings changes, access changes, and sensitive administrative actions.

Record exceptions with an owner, reason, compensating control, and review date. Remove exceptions when the underlying need ends. When a control blocks routine work, confirm whether the rule is misconfigured or too broad before weakening a security requirement.

## Implementation checklist

- [ ] Protect production branches with appropriate pull-request reviews and required checks.
- [ ] Give teams and users only the access they need, and review grants regularly.
- [ ] Use CODEOWNERS for routing and repository rules for required approvals.
- [ ] Protect the CODEOWNERS file and workflow definitions from unreviewed changes.
- [ ] Publish a private security reporting path and define supported versions.
- [ ] Enable and configure Dependabot alerts and version updates separately.
- [ ] Limit workflow token permissions and use scoped, short-lived deployment credentials.
- [ ] Document rule bypasses, exceptions, and emergency review procedures.

## References

[1]: https://docs.github.com/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners "GitHub CODEOWNERS documentation"
[2]: https://docs.github.com/en/code-security/dependabot/dependabot-alerts/about-dependabot-alerts "About Dependabot alerts"
[3]: https://docs.github.com/en/code-security/dependabot/dependabot-security-updates/about-dependabot-security-updates "About Dependabot security updates"
[4]: https://docs.github.com/en/actions/reference/security/secure-use "GitHub Actions secure use reference"

*Last reviewed: 2026-11-24*

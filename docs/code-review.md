# Code Review Checklist and Expectations

Guidelines for reviewing and submitting code at Entur.

## For Authors

### Before Opening a PR

- [ ] Code compiles and all tests pass locally
- [ ] New code has appropriate test coverage
- [ ] PR title follows [conventional commits](https://www.conventionalcommits.org/en/v1.0.0/) format (validated by CI)
- [ ] PR description explains **what** changed and **why**
- [ ] Large changes are broken into smaller, reviewable PRs
- [ ] No unrelated changes bundled into the PR
- [ ] Secrets and credentials are not committed

### PR Description Template

```markdown
## What
Brief description of the change.

## Why
Motivation and context. Link to issue if applicable.

## How
Approach taken and key decisions.

## Testing
How was this tested? Any manual verification steps?
```

## For Reviewers

### What to Look For

#### Correctness

- Does the code do what it claims to do?
- Are edge cases handled (null values, empty collections, boundary conditions)?
- Are errors handled appropriately (not swallowed, logged with context)?
- Are race conditions possible in concurrent code?

#### Design

- Does the change follow the existing architecture patterns in the project?
- Is the code at the right level of abstraction?
- Are responsibilities clearly separated (controller / service / repository)?
- Are new dependencies justified?

#### Security

- No hardcoded secrets or credentials
- Input is validated at boundaries
- Error responses don't leak internal details
- IAM roles from the [approved list](terraform/iam-roles.md) only
- SQL queries use parameterized statements (no string concatenation)

#### Entur Platform Compliance

- Uses Entur shared Terraform modules (not raw GCP resources)
- Uses Entur common Helm chart
- Uses Entur reusable GitHub Actions workflows
- Follows golden path conventions (naming, structure, configuration)
- Dependencies are pinned
- Dependabot is configured (`.github/dependabot.yml`)
- Code health checked with SonarCloud (if enabled for the repository)

#### Testing

- New functionality has tests
- Tests are readable and test behavior, not implementation
- Mocks are used at boundaries, not for internal classes
- No flaky or non-deterministic tests
- Integration tests use testcontainers (not external services)

#### Observability

- Appropriate logging with structured context
- Health checks are configured
- Prometheus metrics for key operations
- No sensitive data in logs

### Review Etiquette

- Be constructive: suggest improvements, don't just point out problems
- Distinguish between blocking issues and nitpicks
- Use prefixes: `nit:` for style suggestions, `question:` for clarifications, `blocker:` for issues that must be fixed
- Approve with minor comments when the overall approach is sound
- Review within one business day when possible

## CI Checks

Every PR must pass these automated checks before merge:

| Check | Workflow | Description |
|-------|----------|-------------|
| Lint | Various | Code formatting and style |
| Unit tests | Project-specific | All unit tests pass |
| Code scan | `codeql.yml` | CodeQL security analysis |
| Docker scan | CI pipeline | Grype vulnerability scan |
| Docker lint | CI pipeline | Hadolint Dockerfile lint |
| Helm lint | CI pipeline | Helm chart validation |
| PR title | `verify-pr.yml` | Conventional commit format |

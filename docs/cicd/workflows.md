# Entur Reusable GitHub Actions Workflows

Entur provides reusable GitHub Actions workflows for CI/CD. Always use these instead of writing custom pipeline steps.

## Available Workflows

| Repository | Purpose | Version |
|-----------|---------|---------|
| [gha-docker](https://github.com/entur/gha-docker) | Docker lint, build, push | `@v1` |
| [gha-helm](https://github.com/entur/gha-helm) | Helm lint, unit test, deploy | `@v1` |
| [gha-terraform](https://github.com/entur/gha-terraform) | Terraform lint, plan, apply | `@v2` |
| [gha-security](https://github.com/entur/gha-security) | Code scan (CodeQL), Docker scan (Grype) | `@v2` |
| [gha-meta](https://github.com/entur/gha-meta) | Releases, PR verification, auth actions | `@v1` |
| [gha-firebase](https://github.com/entur/gha-firebase) | Firebase Hosting preview and deploy | `@v1` |
| [gha-docs](https://github.com/entur/gha-docs) | Documentation publishing | `@v1` |
| [gha-slack](https://github.com/entur/gha-slack) | Slack notifications | `@v2` |
| [gha-artifactory](https://github.com/entur/gha-artifactory) | Artifactory publishing (Maven/Gradle) | `@v1` |

## CI Pipeline (`.github/workflows/ci.yml`)

The CI pipeline runs on pull requests and pushes to main. It lints, tests, builds, and scans.

### Standard CI Pipeline (Spring Boot)

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  # ---- Lint ----
  docker-lint:
    uses: entur/gha-docker/.github/workflows/lint.yml@v1

  helm-lint:
    uses: entur/gha-helm/.github/workflows/lint.yml@v1
    with:
      environment: dev

  terraform-lint:
    uses: entur/gha-terraform/.github/workflows/lint.yml@v2

  # ---- Test ----
  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew test

  # ---- Build ----
  docker-build:
    needs: [test]
    uses: entur/gha-docker/.github/workflows/build.yml@v1

  # ---- Scan ----
  docker-scan:
    needs: [docker-build]
    uses: entur/gha-security/.github/workflows/docker-scan.yml@v2
    secrets: inherit
    with:
      image_artifact: ${{ needs.docker-build.outputs.image_artifact }}

  # ---- Push (main branch only) ----
  docker-push:
    if: github.event_name != 'pull_request'
    needs: [docker-build, docker-scan]
    uses: entur/gha-docker/.github/workflows/push.yml@v1

  # ---- Terraform Plan ----
  terraform-plan-dev:
    if: github.event_name != 'pull_request'
    needs: [terraform-lint]
    uses: entur/gha-terraform/.github/workflows/plan.yml@v2
    with:
      environment: dev
```

### Standard CI Pipeline (Go)

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  docker-lint:
    uses: entur/gha-docker/.github/workflows/lint.yml@v1

  helm-lint:
    uses: entur/gha-helm/.github/workflows/lint.yml@v1
    with:
      environment: dev

  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
      - run: go test ./...

  docker-build:
    needs: [test]
    uses: entur/gha-docker/.github/workflows/build.yml@v1

  docker-scan:
    needs: [docker-build]
    uses: entur/gha-security/.github/workflows/docker-scan.yml@v2
    secrets: inherit
    with:
      image_artifact: ${{ needs.docker-build.outputs.image_artifact }}

  docker-push:
    if: github.event_name != 'pull_request'
    needs: [docker-build, docker-scan]
    uses: entur/gha-docker/.github/workflows/push.yml@v1
```

## CD Pipeline (`.github/workflows/cd.yml`)

The CD pipeline deploys to environments in order: dev -> tst -> prd.

### Standard CD Pipeline

```yaml
name: CD

on:
  push:
    branches: [main]

jobs:
  # ---- Docker ----
  docker-build:
    uses: entur/gha-docker/.github/workflows/build.yml@v1
  docker-push:
    needs: [docker-build]
    uses: entur/gha-docker/.github/workflows/push.yml@v1

  # ---- Terraform ----
  terraform-plan-dev:
    uses: entur/gha-terraform/.github/workflows/plan.yml@v2
    with:
      environment: dev

  terraform-apply-dev:
    needs: [terraform-plan-dev]
    uses: entur/gha-terraform/.github/workflows/apply.yml@v2
    with:
      environment: dev
      has_changes: ${{ needs.terraform-plan-dev.outputs.has_changes }}

  # ---- Deploy Dev ----
  deploy-dev:
    needs: [docker-push, terraform-apply-dev]
    if: ${{ always() && !cancelled() && !contains(needs.*.result, 'failure') }}
    uses: entur/gha-helm/.github/workflows/deploy.yml@v1
    with:
      environment: dev
      image: ${{ needs.docker-push.outputs.image_and_tag }}

  # ---- Terraform + Deploy Tst ----
  terraform-plan-tst:
    needs: [deploy-dev]
    uses: entur/gha-terraform/.github/workflows/plan.yml@v2
    with:
      environment: tst

  terraform-apply-tst:
    needs: [terraform-plan-tst]
    uses: entur/gha-terraform/.github/workflows/apply.yml@v2
    with:
      environment: tst
      has_changes: ${{ needs.terraform-plan-tst.outputs.has_changes }}

  deploy-tst:
    needs: [docker-push, terraform-apply-tst]
    if: ${{ always() && !cancelled() && !contains(needs.*.result, 'failure') }}
    uses: entur/gha-helm/.github/workflows/deploy.yml@v1
    with:
      environment: tst
      image: ${{ needs.docker-push.outputs.image_and_tag }}

  # ---- Terraform + Deploy Prd ----
  terraform-plan-prd:
    needs: [deploy-tst]
    uses: entur/gha-terraform/.github/workflows/plan.yml@v2
    with:
      environment: prd

  terraform-apply-prd:
    needs: [terraform-plan-prd]
    uses: entur/gha-terraform/.github/workflows/apply.yml@v2
    with:
      environment: prd
      has_changes: ${{ needs.terraform-plan-prd.outputs.has_changes }}

  deploy-prd:
    needs: [docker-push, terraform-apply-prd]
    if: ${{ always() && !cancelled() && !contains(needs.*.result, 'failure') }}
    uses: entur/gha-helm/.github/workflows/deploy.yml@v1
    with:
      environment: prd
      image: ${{ needs.docker-push.outputs.image_and_tag }}
```

### Important: `has_changes` and Conditional Jobs

When Terraform plan reports no changes, the apply job is skipped. Downstream jobs that depend on apply must use this condition to continue:

```yaml
if: ${{ always() && !cancelled() && !contains(needs.*.result, 'failure') }}
```

This allows the pipeline to continue when apply is skipped (no changes) but still fails if apply actually fails.

## Security Scanning (`.github/workflows/codeql.yml`)

This workflow **must** be named `codeql.yml`:

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'

jobs:
  code-scan:
    uses: entur/gha-security/.github/workflows/code-scan.yml@v2
    secrets: inherit
```

### Configuration Options

```yaml
jobs:
  code-scan:
    uses: entur/gha-security/.github/workflows/code-scan.yml@v2
    secrets: inherit
    with:
      java_version: "21"             # Default: 21
      java_distribution: "temurin"   # Default: temurin
      use_setup_java: true           # Set to true for Java projects
      codeql_queries: "security-extended"  # Default
```

## PR Verification

Validates that PR titles follow conventional commits:

```yaml
name: Verify PR

on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

jobs:
  verify:
    uses: entur/gha-meta/.github/workflows/verify-pr.yml@v1
```

## Releases (Semantic Versioning)

Automated releases via release-please and conventional commits:

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    uses: entur/gha-meta/.github/workflows/release.yml@v1
    with:
      release_type: simple      # simple | terraform-module | helm | maven | manifest
    permissions:
      contents: write
      pull-requests: write
      issues: write
```

Release types:

- `simple` -- standard application (default)
- `terraform-module` -- Terraform module
- `helm` -- Helm chart
- `maven` -- Maven/Gradle library
- `manifest` -- Multi-component releases (uses `release-please-config.json`)

## Firebase Hosting

### Preview on PR

```yaml
jobs:
  firebase-preview:
    uses: entur/gha-firebase/.github/workflows/firebase-hosting-preview.yml@v1
    with:
      gcp_project_id: my-project-dev
      environment: dev
      build_artifact_name: build-output
      build_artifact_path: build
```

### Deploy to Live

```yaml
jobs:
  firebase-deploy:
    uses: entur/gha-firebase/.github/workflows/firebase-hosting-deploy.yml@v1
    with:
      gcp_project_id: my-project-dev
      environment: dev
      build_artifact_name: build-output
      build_artifact_path: build
```

### Full Firebase Flow with Approval

```yaml
jobs:
  firebase-preview:
    uses: entur/gha-firebase/.github/workflows/firebase-hosting-preview.yml@v1
    with:
      gcp_project_id: my-project-dev
      environment: dev

  approve:
    needs: [firebase-preview]
    runs-on: ubuntu-latest
    environment: apr          # GitHub Environment with protection rules
    steps:
      - run: echo "Approved"

  firebase-deploy:
    needs: [approve]
    uses: entur/gha-firebase/.github/workflows/firebase-hosting-deploy.yml@v1
    with:
      gcp_project_id: my-project-dev
      environment: dev
```

## Documentation Publishing

```yaml
jobs:
  publish-docs:
    uses: entur/gha-docs/.github/workflows/publish.yml@v1
    secrets: inherit
    with:
      project: my-application     # Default: repo name
      directory: docs             # Default: docs
```

## Slack Notifications

### Prerequisites

In your Slack channel, run: `/invite @GitHub Actions Slack send`

```yaml
jobs:
  notify:
    uses: entur/gha-slack/.github/workflows/post.yml@v2
    with:
      channel_id: "C01ABCDEFGH"
      message: "Deployment to prd completed successfully"
    secrets: inherit
```

## Artifactory Publishing (Maven/Gradle)

For publishing shared libraries:

```yaml
jobs:
  update-version:
    permissions:
      contents: write
    uses: entur/gha-artifactory/.github/actions/update-version@v1

  maven-publish:
    needs: [update-version]
    uses: entur/gha-artifactory/.github/actions/maven-publish@v1
```

Requires `gradle.properties` at repo root with a `version` field in semver format.

## Docker Workflow Details

### Lint

```yaml
docker-lint:
  uses: entur/gha-docker/.github/workflows/lint.yml@v1
  with:
    ignore: "DL3008,DL3015"    # Optional: comma-separated hadolint ignores
```

### Build

```yaml
docker-build:
  uses: entur/gha-docker/.github/workflows/build.yml@v1
  with:
    dockerfile: Dockerfile      # Default
    context: "."                # Default
```

Outputs: `image_artifact` (used by docker-scan and docker-push)

### Push

```yaml
docker-push:
  uses: entur/gha-docker/.github/workflows/push.yml@v1
```

Outputs: `image_name`, `image_tag`, `image_and_tag`

Default image tag format: `branch_name.date-SHA`

## Helm Workflow Details

### Helm Lint

```yaml
helm-lint:
  uses: entur/gha-helm/.github/workflows/lint.yml@v1
  with:
    environment: dev
```

### Unit Test

```yaml
helm-unittest:
  uses: entur/gha-helm/.github/workflows/unittest.yml@v1
```

### Deploy

```yaml
helm-deploy:
  uses: entur/gha-helm/.github/workflows/deploy.yml@v1
  with:
    environment: dev
    image: ${{ needs.docker-push.outputs.image_and_tag }}
```

## Terraform Workflow Details

### Terraform Lint

```yaml
terraform-lint:
  uses: entur/gha-terraform/.github/workflows/lint.yml@v2
```

### Plan

```yaml
terraform-plan:
  uses: entur/gha-terraform/.github/workflows/plan.yml@v2
  with:
    environment: dev
```

Outputs: `has_changes` (true/false), `plan_summary`

### Apply

```yaml
terraform-apply:
  uses: entur/gha-terraform/.github/workflows/apply.yml@v2
  with:
    environment: dev
    has_changes: ${{ needs.terraform-plan.outputs.has_changes }}
```

## Environments

| Environment | Description |
|-------------|-------------|
| `dev` | Development |
| `tst` | Testing / staging |
| `prd` | Production |

All workflows accept the `environment` input.

## Preferred CI/CD Structure

The preferred pattern splits CI/CD into multiple focused workflow files instead of a single monolithic pipeline:

| File | Trigger | Purpose |
|------|---------|---------|
| `ci.yml` | `workflow_call` | Reusable CI build (lint, build, test, scan, push) |
| `ci-pr.yml` | `pull_request` to main | PR title lint + CI build |
| `ci-feature.yml` | Push to non-main branches | CI build if no open PR exists |
| `deploy.yml` | Push to main | Build + deploy dev -> tst -> prd |
| `codeql.yml` | PR, push to main, schedule | Security code scanning |
| `lint-api.yml` | PR changes to `specs/` | API spec linting |
| `lint-helm.yml` | PR changes to `helm/` | Helm chart linting per environment |

### ci.yml (Reusable Build Workflow)

```yaml
name: ci
on:
  workflow_call:
    outputs:
      image_and_tag:
        description: Fully qualified image reference
        value: ${{ jobs.docker-push.outputs.image_and_tag }}

jobs:
  docker-lint:
    uses: entur/gha-docker/.github/workflows/lint.yml@v1
    with:
      ignore: DL3059

  docker-build:
    needs: [docker-lint]
    uses: entur/gha-docker/.github/workflows/build.yml@v1
    secrets:
      BUILD_SECRETS: |
        "ARTIFACTORY_AUTH_USER=${{ secrets.ARTIFACTORY_AUTH_USER }}"
        "ARTIFACTORY_AUTH_TOKEN=${{ secrets.ARTIFACTORY_AUTH_TOKEN }}"

  test:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: liberica
          java-version: 25
      - uses: gradle/actions/setup-gradle@v5
      - run: ./gradlew test --no-daemon
        env:
          ARTIFACTORY_AUTH_USER: ${{ secrets.ARTIFACTORY_AUTH_USER }}
          ARTIFACTORY_AUTH_TOKEN: ${{ secrets.ARTIFACTORY_AUTH_TOKEN }}
      - uses: dorny/test-reporter@v2
        if: always()
        with:
          name: test-results
          path: build/test-results/test/*.xml
          reporter: java-junit

  docker-scan:
    if: github.event_name == 'pull_request' || github.ref == 'refs/heads/main'
    needs: [docker-build, test]
    uses: entur/gha-security/.github/workflows/docker-scan.yml@v2
    with:
      image_artifact: ${{ needs.docker-build.outputs.image_artifact }}
    secrets: inherit

  docker-push:
    if: github.ref == 'refs/heads/main'
    needs: [docker-scan]
    uses: entur/gha-docker/.github/workflows/push.yml@v1
    with:
      extra_image_tags: "latest"
    secrets: inherit
```

### ci-pr.yml (PR Verification)

Validates PR titles follow conventional commits with JIRA ticket scopes:

```yaml
name: ci-pr
on:
  pull_request:
    branches: [main]
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  pr-lint:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: read
    steps:
      - uses: amannn/action-semantic-pull-request@v6
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          scopes: ETU-\d+                    # Require JIRA ticket in scope
          ignoreLabels: dependencies          # Skip for Dependabot PRs
          validateSingleCommit: true

  build:
    needs: [pr-lint]
    uses: ./.github/workflows/ci.yml
    secrets: inherit
```

### ci-feature.yml (Feature Branch Build)

Skips CI if there's already an open PR (ci-pr.yml handles it):

```yaml
name: ci-feature
on:
  push:
    branches-ignore: [main]
    paths:
      - 'src/**'
      - 'Dockerfile'
      - '.dockerignore'
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  pr-check:
    runs-on: ubuntu-24.04
    steps:
      - id: pr-check
        run: |
          if gh pr list --state open --head "${{ github.ref_name }}" --json number --jq 'length > 0'; then
            echo "skip_build=true" >> $GITHUB_OUTPUT
          else
            echo "skip_build=false" >> $GITHUB_OUTPUT
          fi

  build:
    needs: [pr-check]
    if: needs.pr-check.outputs.skip_build == 'false'
    uses: ./.github/workflows/ci.yml
    secrets: inherit
```

### deploy.yml (Multi-Environment Matrix Deploy)

Use a **matrix strategy** for deploying to multiple environments or namespaces:

```yaml
name: deploy
on:
  push:
    branches: [main]
    paths-ignore:
      - '**/*.md'
      - '**/*.adoc'

jobs:
  build:
    uses: ./.github/workflows/ci.yml
    secrets: inherit

  helm-deploy-dev:
    needs: [build]
    strategy:
      fail-fast: false
      matrix:
        include:
          - environment: dev
            namespace: products
            release_name: products-api
            values: values-kub-ent-dev.yaml
          - environment: dev
            namespace: products-ep
            release_name: products-api-ep
            values: values-kub-ent-dev-ep.yaml
    uses: entur/gha-helm/.github/workflows/deploy.yml@v1
    with:
      environment: ${{ matrix.environment }}
      chart: helm/products-api
      namespace: ${{ matrix.namespace }}
      release_name: ${{ matrix.release_name }}
      image: ${{ needs.build.outputs.image_and_tag }}
      values: ${{ matrix.values }}
    secrets: inherit

  helm-deploy-tst:
    needs: [helm-deploy-dev, build]
    if: ${{ success() }}
    # ... same matrix pattern for tst environments

  helm-deploy-prd:
    needs: [helm-deploy-tst, build]
    if: ${{ success() }}
    # ... same matrix pattern for prd environments
```

### lint-api.yml (API Spec Linting)

```yaml
name: lint-api
on:
  pull_request:
    paths: ['specs/**']
jobs:
  api-lint:
    uses: entur/gha-api/.github/workflows/lint.yml@v5
    if: github.actor != 'dependabot[bot]'
    secrets: inherit
    with:
      spec: specs/*.yaml
```

### lint-helm.yml (Helm Linting per Environment)

```yaml
name: lint-helm
on:
  pull_request:
    paths: ['helm/**']
jobs:
  helm-lint:
    uses: entur/gha-helm/.github/workflows/lint.yml@v1
    strategy:
      matrix:
        environment: [dev, dev-ep, tst, tst-ep, prd]
    with:
      chart: helm/my-app
      environment: ${{ matrix.environment }}
      values: values-kub-ent-${{ matrix.environment }}.yaml
```

### Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "docker"
    directories: ["/"]
    schedule:
      interval: "weekly"
    ignore:
      - dependency-name: "bellsoft/liberica-runtime-container"
        versions: [">=26.0.0"]
  - package-ecosystem: "gradle"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      flyway:
        applies-to: version-updates
        patterns: ["org.flywaydb*"]
```

## Best Practices

1. **Always use `secrets: inherit`** for security scanning and docs workflows
2. **Pin workflow versions** to major tags: `@v1`, `@v2`
3. **Use `has_changes` output** from terraform-plan to skip unnecessary applies
4. **Use the conditional if clause** for jobs after skippable terraform-apply
5. **Run scans on every PR** -- not just on main branch pushes
6. **Name the CodeQL workflow `codeql.yml`** -- the security tooling depends on this name
7. **Use GitHub Environments** with protection rules for deployment approvals
8. **Split workflows into focused files** -- separate CI, deploy, linting, and security scanning
9. **Use matrix strategy** for deploying to multiple namespaces or environments from a single Helm chart
10. **Use concurrency groups** with `cancel-in-progress: true` on PR and feature branch workflows
11. **Pass build secrets via `BUILD_SECRETS`** when using multi-stage Docker builds with Artifactory
12. **Use `paths` filters** on PR workflows to run linting only when relevant files change
13. **Upload test reports** using `dorny/test-reporter` for visibility in GitHub PR checks

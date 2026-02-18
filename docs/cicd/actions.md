# Entur Composite Actions

Reference: [entur/gha-meta](https://github.com/entur/gha-meta)

In addition to reusable workflows, Entur provides composite actions for authentication and common tasks. These are used internally by the reusable workflows but can also be used directly in custom workflow steps.

## Available Actions

| Action | Purpose |
|--------|---------|
| `entur/gha-meta/.github/actions/cloud-auth` | Authenticate with GCP |
| `entur/gha-meta/.github/actions/k8s-auth` | Authenticate with GKE |
| `entur/gha-meta/.github/actions/docker-auth` | Authenticate with Google Artifact Registry |

## Cloud Authentication

Authenticate with Google Cloud Platform:

```yaml
steps:
  - uses: entur/gha-meta/.github/actions/cloud-auth@v1
    with:
      environment: dev
```

This sets up Workload Identity Federation for secure, keyless authentication.

## Kubernetes Authentication

Authenticate with GKE:

```yaml
steps:
  - uses: entur/gha-meta/.github/actions/k8s-auth@v1
    with:
      environment: dev
```

## Docker Registry Authentication

Authenticate with Google Artifact Registry:

```yaml
steps:
  - uses: entur/gha-meta/.github/actions/docker-auth@v1
```

## When to Use Composite Actions Directly

Prefer the reusable workflows (`gha-docker`, `gha-helm`, `gha-terraform`) for standard operations. Use the composite actions directly only when:

- You need custom workflow steps that aren't covered by the reusable workflows
- You need to combine authentication with custom commands in a single job
- You're building a new reusable workflow

Example: custom deployment with kubectl:

```yaml
jobs:
  custom-deploy:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: entur/gha-meta/.github/actions/cloud-auth@v1
        with:
          environment: dev
      - uses: entur/gha-meta/.github/actions/k8s-auth@v1
        with:
          environment: dev
      - run: kubectl apply -f k8s/custom-resource.yaml
```

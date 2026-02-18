# Self-Service Platform Provisioning

Entur's developer platform provides self-service capabilities for provisioning GCP projects, CI/CD pipelines, and application environments. You define YAML manifests in a `.entur/` directory in your repository and apply them through a GitOps-based pull request workflow.

## How It Works

The self-service system uses a **Platform Orchestrator** that watches for manifest changes in pull requests:

1. Create or modify YAML manifests in the `.entur/` folder of your repository.
2. Open a pull request. The orchestrator validates the manifests and presents a **plan** (similar to `terraform plan`) showing proposed changes.
3. Review the plan. If it looks correct, comment `entur apply` on the PR.
4. Wait for the apply to succeed, then merge the PR.

### Aborting or Rolling Back

- **Not yet applied**: Simply close the PR.
- **Already applied**: Revert the manifest change, run `entur apply` again, then merge or close the PR.

### GitHub Repository Requirements

By default the Platform Orchestrator has read/write access to all repositories in the Entur organization. If you use Repository Rulesets (restrict pushes, etc.), you must add a bypass for the **Platform Orchestrator** GitHub application on each rule.

## Manifest Kinds

There are two categories of manifests. The **GitHub manifest** must be applied before any **Application manifest**.

| Kind | apiVersion | Purpose |
|------|-----------|---------|
| `GitHubActions` | `orchestrator.entur.io/github/v1` | Configures GCP Workload Identity and GitHub environments for CI/CD |
| `GoogleCloudApplication` | `orchestrator.entur.io/apps/v1` | Provisions GCP projects for containerized K8s applications |
| `GoogleCloudFirebaseApplication` | `orchestrator.entur.io/apps/v1` | Provisions GCP projects for Firebase applications |
| `GoogleCloudDataProject` | `orchestrator.entur.io/apps/v1` | Provisions GCP projects for data workloads |

> **Note:** The `metadata.id` field has different constraints per kind. For `GitHubActions`, it must match the GitHub repository name (1--63 chars, alphanumeric plus `-`, `_`, `.`). For Application kinds, it is the "App ID" (3--10 lowercase alphanumeric chars only). See the reference sections below for full validation rules.

### File Conventions

- Manifests live in `.entur/` at the repository root
- One YAML document per file (no multi-doc `---`)
- Default naming: `.entur/<metadata.id>.yaml`

## Getting Started

This section walks through the most common setup: a containerized application on Kubernetes in Google Cloud.

### Prerequisites

- Read the [DevOps Handbook](https://enturnett.atlassian.net/wiki/spaces/ESP/overview) Plan section
- Onboard to GitHub and create a repository for your application
- Build a basic web application that listens on TCP port `8080` and returns HTTP `200 OK` on:
  - `GET /actuator/health/liveness`
  - `GET /actuator/health/readiness`

### Step 1: Create the GitHub Manifest

Create `.entur/cicd.yaml`:

```yaml
apiVersion: orchestrator.entur.io/github/v1
kind: GitHubActions
metadata:
  id: myuniquerepo  # must match your repository name exactly
spec:
  environments: [dev, tst, prd]
```

### Step 2: Create the Application Manifest

Create `.entur/application.yaml`:

```yaml
apiVersion: orchestrator.entur.io/apps/v1
kind: GoogleCloudApplication
metadata:
  id: myappid  # 3-10 lowercase alphanumeric chars, unique in Entur org ("App ID")
  displayName: My Application
  name: my-unique-app  # 3-30 chars, lowercase alphanumeric + hyphens, becomes your K8s namespace
  owner: team-excellence
  trigger: 1747398600  # current unix timestamp, see https://unixtime.org/
spec:
  environments: [dev, tst, prd]
  repositories: [myuniquerepo]  # repos that can deploy to this application
```

> **Important:** Remember `metadata.id` -- you need it for Helm configuration.

### Step 3: Apply via PR

1. Commit both files to a new branch (for example `application-init`).
2. Push and open a pull request targeting your main branch.
3. Wait for the plan output on both manifests. Review the proposed changes.
4. Comment `entur apply` on the PR.
5. Wait for successful apply status, then merge.

You can now build and deploy using GitHub Actions across `dev`, `tst`, and `prd` environments.

### Next Steps

1. **Document your API** -- See [API design](api-design.md)
2. **Create a container image** -- See [Docker guide](docker.md)
3. **Set up CI/CD pipelines** -- See [CI/CD workflows](cicd/workflows.md)
4. **Configure Helm deployment** -- See [Helm guide](helm.md)

---

## GitHubActions Manifest Reference

The `GitHubActions` kind configures GCP Workload Identity and GitHub environments for CI/CD access.

### GitHubActions Fields

#### `apiVersion` (required)

Must be `orchestrator.entur.io/github/v1`.

#### `kind` (required)

Must be `GitHubActions`.

#### `metadata.id` (required, immutable)

- Type: `string`
- Length: 1--63
- Pattern: `^[A-Za-z0-9_.-]+$`
- **Must match the GitHub repository name exactly**

If the repository is renamed, delete this manifest and create a new one with the updated name.

#### `metadata.trigger` (optional)

- Type: `integer` (Unix timestamp in seconds)
- Default: current timestamp
- Range: `1`--`9999999999`

Change this value to force the orchestrator to re-apply infrastructure without other manifest changes. Useful when you accidentally merge without running `entur apply`.

#### `spec.environments` (optional)

- Type: `array`
- Valid values: `dev`, `tst`, `prd`
- Default: `["dev", "tst", "prd"]`
- Values must be unique

GitHub environments and GCP Workload Identity credentials are provisioned for each listed environment. If set, must match environments in the linked Application manifest.

### GitHubActions Validation Rules

- `metadata.id` must be 1--63 characters, alphanumeric plus `-`, `_`, `.`
- Environment names must be `dev`, `tst`, or `prd`
- Environment list must contain unique values

### GitHubActions Examples

Minimal (uses defaults -- all three environments):

```yaml
apiVersion: orchestrator.entur.io/github/v1
kind: GitHubActions
metadata:
  id: my-repo
```

Dev only:

```yaml
apiVersion: orchestrator.entur.io/github/v1
kind: GitHubActions
metadata:
  id: my-repo
spec:
  environments:
    - dev
```

Full with trigger:

```yaml
apiVersion: orchestrator.entur.io/github/v1
kind: GitHubActions
metadata:
  id: my-repo
  trigger: 1654089480
spec:
  environments:
    - dev
    - tst
    - prd
```

---

## Application Manifest Reference

Application manifests provision GCP projects and related resources. Three kinds are supported: `GoogleCloudApplication`, `GoogleCloudFirebaseApplication`, and `GoogleCloudDataProject`.

### Application Fields

- **`apiVersion`** (required): Must be `orchestrator.entur.io/apps/v1`
- **`kind`** (required): One of `GoogleCloudApplication`, `GoogleCloudFirebaseApplication`, `GoogleCloudDataProject`

#### `metadata` (required)

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `id` | string | yes | 3--10 chars, `^[a-z0-9]+$`, must NOT end with `sbx\|dev\|tst\|test\|prd\|prod`, must NOT start with `ent-`. **Immutable -- changing this is destructive.** |
| `displayName` | string | yes | Human-friendly name |
| `name` | string | yes | 3--30 chars, `^[a-z0-9-]+$`. Becomes the Kubernetes namespace. Changing is disruptive (namespace rename, pod restarts). |
| `owner` | string | yes | Must start with `team-` |
| `trigger` | integer | no | Unix timestamp to force re-apply |
| `domain` | string | no | Deprecated -- avoid unless explicitly needed |

#### `spec.environments` (required)

One or more of: `dev`, `tst`, `prd`.

#### `spec.repositories` (recommended)

List of GitHub repository names that can deploy to this application. Required for GitHub Actions to have CI/CD permissions to the application's GCP projects.

### Optional spec Blocks

These are the schema-defined properties available under `spec`:

- `kubernetes.enabled`, `kubernetes.clusterGroup` (`entur` or `journeyPlanner`), `kubernetes.securityPolicy.level`, `kubernetes.networkPolicies.enabled|denyInternal|denyPublic|denyEgress|ingress.allowedNamespaces`
- `terraform.createBackend`
- `auth0.internal.enabled`, `auth0.internal.type` (only `m2m`)
- `appLogBucket.enabled|retentionDays|disableSink|logAnalyticsEnabled`
- `defaultLogBucket.logAnalyticsEnabled|location`
- `appEngine.enabled|databaseType`
- `secretManager.enabled|serviceAccount`
- `serviceAccounts[]` with `id`, `additionalRoles`, `kubernetesEnabled`, `displayName`, `description`, `roles`
- `quotas.enabled`, `quotas.bigQuery.dailyQuotaPerUser`, `quotas.bigQuery.dailyQuota`
- `network.sharedVpcEnabled`

For `GoogleCloudFirebaseApplication` only:

- `firebase.region` with enum: `europe-west`, `europe-west1`, `europe-west2`, `europe-west3`, `europe-west4`, `global`

For `GoogleCloudDataProject` only:

- `dataAccess.external` (boolean)

Both `metadata` and `spec` have `additionalProperties: false` -- do not introduce fields not listed above.

### Application Validation Rules

- `metadata.id`: 3--10 lowercase alphanumeric, no environment suffixes, no `ent-` prefix
- `metadata.name`: 3--30 lowercase alphanumeric plus hyphens
- `metadata.owner`: must start with `team-`
- `spec.environments`: at least one of `dev`, `tst`, `prd`
- Changing `metadata.id` is **destructive** (deletes and recreates GCP projects)
- Changing `metadata.name` is **disruptive** (namespace rename, pod restarts)

### YAML Conventions

- 2-space indentation, no tabs
- Properly indent lists (especially `repositories:` and `environments:`)
- Use explicit booleans (`true`/`false`)

### Application Minimal Templates

GoogleCloudApplication:

```yaml
apiVersion: orchestrator.entur.io/apps/v1
kind: GoogleCloudApplication
metadata:
  id: myappid
  displayName: "My Application"
  name: my-application
  owner: team-myteam
spec:
  environments:
    - dev
```

GoogleCloudFirebaseApplication:

```yaml
apiVersion: orchestrator.entur.io/apps/v1
kind: GoogleCloudFirebaseApplication
metadata:
  id: mywebapp
  displayName: "My Firebase Web App"
  name: my-firebase-app
  owner: team-myteam
spec:
  environments:
    - dev
```

GoogleCloudDataProject:

```yaml
apiVersion: orchestrator.entur.io/apps/v1
kind: GoogleCloudDataProject
metadata:
  id: mydataprj
  displayName: "My Data Project"
  name: my-data-project
  owner: team-myteam
spec:
  dataAccess:
    external: true
  organization: entur
  environments:
    - dev
```

### Application Full Example

```yaml
apiVersion: orchestrator.entur.io/apps/v1
kind: GoogleCloudApplication
metadata:
  id: exampleapp
  displayName: "This is an example app"
  name: my-example-app
  owner: team-myteam
  trigger: 1654089480
spec:
  kubernetes:
    enabled: true
    networkPolicies:
      enabled: true
      denyInternal: true
      denyPublic: true
      denyEgress: true
      ingress:
        allowedNamespaces:
          - up-k8s
          - helloworld
  network:
    sharedVpcEnabled: true
  terraform:
    createBackend: true
  auth0:
    internal:
      enabled: false
      type: m2m
  appLogBucket:
    enabled: true
    retentionDays: 30
    disableSink: false
    logAnalyticsEnabled: true
  defaultLogBucket:
    logAnalyticsEnabled: true
  appEngine:
    enabled: true
    databaseType: firestore
  secretManager:
    enabled: true
    serviceAccount: application
  serviceAccounts:
    - id: application
      additionalRoles:
        - roles/storage.objectCreator
    - id: "my-custom-account"
      kubernetesEnabled: true
      displayName: "MyCustomAccount"
      description: "A custom account for my app"
      roles:
        - roles/bigquery.admin
  organization: entur
  repositories:
    - my-github-repository
  environments:
    - dev
  quotas:
    bigQuery:
      dailyQuotaPerUser: 1.5
      dailyQuota: 20
```

---

## Testing with Mock Manifests

A mock manifest kind (`orchestrator.entur.io/mock/v1`, kind `MockItem`) exists for safely testing the self-service workflow without affecting real resources. The flow is identical to production manifests: create/modify/delete a `.entur/*.yaml` file, open a PR, review the plan, comment `entur apply`, and merge. Use this to verify the workflow before provisioning real GCP projects.

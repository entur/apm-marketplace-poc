# Entur Terraform Modules

Entur provides shared Terraform modules for provisioning GCP infrastructure. Always use these modules instead of raw `google_*` resources for managed services.

## Modules

| Module | Purpose | Source |
|--------|---------|--------|
| [terraform-google-init](https://github.com/entur/terraform-google-init) | Platform and app discovery (required by other modules) | `github.com/entur/terraform-google-init//modules/init?ref=v1` |
| [terraform-google-sql-db](https://github.com/entur/terraform-google-sql-db) | Cloud SQL (PostgreSQL) | `github.com/entur/terraform-google-sql-db//modules/postgresql?ref=v1` |
| [terraform-google-memorystore](https://github.com/entur/terraform-google-memorystore) | Memorystore (Redis) | `github.com/entur/terraform-google-memorystore//modules/redis?ref=v2` |

## Directory Structure

```text
terraform/
  main.tf             # Module declarations
  variables.tf        # Input variables
  outputs.tf          # Output values
  providers.tf        # Provider configuration
  env/
    dev.tfvars         # Dev environment values
    tst.tfvars         # Test environment values
    prd.tfvars         # Production environment values
```

## Init Module (Required)

The `terraform-google-init` module is a **data-only module** that discovers GCP platform and application attributes. All other Entur modules depend on it.

```hcl
module "init" {
  source      = "github.com/entur/terraform-google-init//modules/init?ref=v1"
  app_id      = var.app_id
  environment = var.environment
}
```

### Variables

```hcl
# variables.tf
variable "app_id" {
  description = "Application ID"
  type        = string
}

variable "environment" {
  description = "Environment: dev, tst, or prd"
  type        = string
}
```

```hcl
# env/dev.tfvars
app_id      = "my-application"
environment = "dev"
```

### Outputs

The init module provides:

| Output | Description |
|--------|-------------|
| `module.init.app.id` | Application ID |
| `module.init.app.name` | Application name |
| `module.init.app.project_id` | GCP project ID |
| `module.init.app.project_number` | GCP project number |
| `module.init.app.owner` | Application owner |
| `module.init.environment` | Environment descriptor |
| `module.init.is_production` | Boolean: is this production? |
| `module.init.kubernetes.project_id` | GKE project ID |
| `module.init.kubernetes.namespace` | Kubernetes namespace |
| `module.init.labels` | Standard labels for all resources |
| `module.init.networks.project_id` | Network project ID |
| `module.init.networks.vpc_name` | VPC name |
| `module.init.networks.vpc_id` | VPC ID |
| `module.init.service_accounts` | Application and project service accounts |

## Cloud SQL (PostgreSQL)

Provisions a Cloud SQL PostgreSQL instance with automatic secret and Kubernetes resource creation.

```hcl
module "postgresql" {
  source    = "github.com/entur/terraform-google-sql-db//modules/postgresql?ref=v1"
  init      = module.init
  databases = ["mydb"]
}
```

### Key Configuration

```hcl
module "postgresql" {
  source    = "github.com/entur/terraform-google-sql-db//modules/postgresql?ref=v1"
  init      = module.init
  databases = ["mydb"]

  # Machine sizing (optional, has sensible defaults per environment)
  machine_size = {
    cpu    = 1
    memory = 3840
  }

  database_version = "POSTGRES_16"    # Default: POSTGRES_14
  region           = "europe-west1"   # Default

  # Backup (defaults shown)
  enable_backup                    = true
  backup_start_time                = "00:00"
  point_in_time_recovery_enabled   = true

  # Storage
  disk_size             = 10          # Initial size in GB
  disk_autoresize       = true
  disk_autoresize_limit = null        # Auto: 500 (prd), 50 (non-prd)

  # Additional users (optional)
  additional_users = {
    readonly = {}
  }

  # Query performance (optional)
  query_insights_enabled = true
}
```

### Default Machine Sizing

| Environment | vCPU | Memory | High Availability |
|-------------|------|--------|-------------------|
| Non-production (dev, tst) | Shared | 600 MB | No (ZONAL) |
| Production (prd) | 1 dedicated | 3840 MB | Yes (REGIONAL) |

### What Gets Created

- Cloud SQL PostgreSQL instance
- Database(s)
- Application user with password
- Google Secret Manager secrets: `PG_USER`, `PG_PASSWORD`
- Kubernetes ConfigMap with connection info
- Kubernetes Secret with credentials

### Application Configuration (Spring Boot)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/${DB_NAME}
    username: ${PG_USER}
    password: ${PG_PASSWORD}
```

The Cloud SQL proxy sidecar (enabled via `postgres.enabled: true` in Helm) handles connectivity. The application connects to `localhost:5432`.

## Memorystore (Redis)

Provisions a Memorystore Redis instance with automatic secret and Kubernetes resource creation.

```hcl
module "redis" {
  source = "github.com/entur/terraform-google-memorystore//modules/redis?ref=v2"
  init   = module.init
}
```

### Redis Configuration

```hcl
module "redis" {
  source = "github.com/entur/terraform-google-memorystore//modules/redis?ref=v2"
  init   = module.init

  memory_size_gb    = 1             # Default: 1
  redis_version     = "REDIS_7_0"  # Default
  region            = "europe-west1"

  # High availability
  availability_type = "REGIONAL"    # Default (or ZONAL)
  enable_replicas   = false         # Default
  replica_count     = 1             # 1-5 if replicas enabled

  # Redis configuration
  redis_configs = {
    activedefrag     = "yes"
    maxmemory-policy = "allkeys-lfu"
  }

  maintenance_window = {
    day  = "TUESDAY"
    hour = 0
  }
}
```

### What Gets Created (Redis)

- Memorystore Redis instance
- Google Secret Manager secrets: `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`
- Kubernetes ConfigMap and Secret with connection info

### Redis Application Configuration (Spring Boot)

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT}
      password: ${REDIS_PASSWORD}
```

### Helm Configuration for Redis Secrets

```yaml
# helm/<app>/values.yaml
common:
  secrets:
    redis-credentials:
      - REDIS_HOST
      - REDIS_PORT
      - REDIS_PASSWORD
```

## Complete Example

```hcl
# main.tf

terraform {
  required_version = ">= 1.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  backend "gcs" {}
}

module "init" {
  source      = "github.com/entur/terraform-google-init//modules/init?ref=v1"
  app_id      = var.app_id
  environment = var.environment
}

module "postgresql" {
  source    = "github.com/entur/terraform-google-sql-db//modules/postgresql?ref=v1"
  init      = module.init
  databases = ["routedb"]
}

module "redis" {
  source = "github.com/entur/terraform-google-memorystore//modules/redis?ref=v2"
  init   = module.init
}

# Custom resources (use google_* only for resources not covered by Entur modules)
resource "google_pubsub_topic" "route_events" {
  name    = "${module.init.app.id}-route-events"
  project = module.init.app.project_id
  labels  = module.init.labels
}

resource "google_pubsub_subscription" "route_events_sub" {
  name    = "${module.init.app.id}-route-events-sub"
  topic   = google_pubsub_topic.route_events.name
  project = module.init.app.project_id
  labels  = module.init.labels

  ack_deadline_seconds = 20
  message_retention_duration = "604800s"  # 7 days

  retry_policy {
    minimum_backoff = "10s"
    maximum_backoff = "600s"
  }

  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.route_events_dlq.id
    max_delivery_attempts = 10
  }
}
```

```hcl
# variables.tf
variable "app_id" {
  description = "Application ID"
  type        = string
}

variable "environment" {
  description = "Environment"
  type        = string
  validation {
    condition     = contains(["dev", "tst", "prd"], var.environment)
    error_message = "Environment must be dev, tst, or prd."
  }
}
```

```hcl
# env/dev.tfvars
app_id      = "route-service"
environment = "dev"
```

## Workspaces

Entur uses Terraform workspaces to manage multiple environments with a single configuration. Each workspace has its own state file.

```bash
terraform init
terraform workspace new dev
terraform apply -var-file=env/dev.tfvars

# Switch environment:
terraform workspace select tst
terraform apply -var-file=env/tst.tfvars
```

Workspace-specific logic (e.g. larger instances in production):

```hcl
module "postgresql" {
  source       = "github.com/entur/terraform-google-sql-db//modules/postgresql?ref=v1"
  init         = module.init
  databases    = ["my-database"]
  machine_size = {
    tier = terraform.workspace == "prd" ? "db-custom-1-3840" : "db-f1-micro"
  }
}
```

## Backend State

State is stored in a GCS bucket auto-created by the platform (naming: `ent-gcs-tfa-<appId>`):

```hcl
terraform {
  backend "gcs" {
    bucket = "ent-gcs-tfa-<appId>"
  }
}
```

## Cloud Storage

Use the [terraform-google-cloud-storage](https://github.com/entur/terraform-google-cloud-storage) module for storage buckets:

```hcl
module "cloud-storage" {
  source = "github.com/entur/terraform-google-cloud-storage//modules/bucket?ref=v0.2.2"
  init   = module.init
}

resource "google_storage_bucket_iam_member" "reader" {
  bucket = module.cloud-storage.cloud_storage_bucket.name
  role   = "roles/storage.objectViewer"
  member = "user:example-user@entur.org"
}
```

Buckets are private by default. Avoid making them public unless absolutely necessary.

## PostgreSQL IAM Authentication

Enable IAM authentication for Cloud SQL to avoid password-based access:

```hcl
module "postgresql" {
  source    = "github.com/entur/terraform-google-sql-db//modules/postgresql?ref=v1"
  init      = module.init
  databases = ["my-database"]
  database_flags = {
    iam_authentication = { name = "cloudsql.iam_authentication", value = "on" }
  }
}

# IAM user:
resource "google_sql_user" "iam_user" {
  name     = "example-user@entur.org"
  instance = module.postgresql.instance.name
  type     = "CLOUD_IAM_USER"
}

# IAM group:
resource "google_sql_user" "iam_group" {
  name     = "sg-dig-team-name@entur.org"
  instance = module.postgresql.instance.name
  type     = "CLOUD_IAM_GROUP"
}
```

## BigQuery

```hcl
resource "google_bigquery_dataset" "example" {
  dataset_id = "example_dataset"
  project    = module.init.app.project_id
  location   = "EU"
  labels     = module.init.labels
}

resource "google_bigquery_dataset_iam_member" "viewer" {
  dataset_id = google_bigquery_dataset.example.dataset_id
  role       = "roles/bigquery.dataViewer"
  member     = "user:example-user@entur.org"
}
```

Grant IAM at the group level when possible. Be cautious with project-level `roles/bigquery.user` -- it allows creating new datasets.

## Troubleshooting

### State locked to newer version

Contact `#talk-utviklerplattform`. The GCS bucket has versioning enabled and the state file can be restored.

### State locked but no deployments running

Caused by a cancelled `terraform apply`. Use `terraform force-unlock`. As a last resort, manually delete the lock file from the GCS bucket.

### Resource not created properly or manually deleted

Use `terraform state list` to find the resource, `terraform state rm <id>` to remove it from state, then re-run to recreate. For resources created externally, use `terraform import`.

### Cannot find resource in cloud project

Ensure tfvars files are in `./terraform/env/`. Verify you are connected to the correct cluster and workspace:

```bash
gcloud container clusters get-credentials <cluster-name> --region europe-west1 --project <gcp-project-id>
terraform workspace select dev
```

## Best Practices

1. **Always pin module versions** with `?ref=TAG` -- never use unversioned references
2. **Always use the init module** as the base for all other modules and resources
3. **Use `module.init.labels`** on all resources for consistent labeling
4. **Use `module.init.app.project_id`** for the `project` field on all resources
5. **Use environment-specific tfvars** in `terraform/env/` for per-environment configuration
6. **Use GCS backend** for remote state storage (configured by the platform)
7. **Only use IAM roles from the [approved list](iam-roles.md)**
8. **Use Entur modules** for Cloud SQL and Memorystore -- do not create raw `google_sql_*` or `google_redis_*` resources
9. **Run `terraform plan` in CI** and `terraform apply` in CD using the `gha-terraform` reusable workflows

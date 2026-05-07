# Entur APM Marketplace

Curated APM marketplace for Entur AI agent plugins and skills.

This repository publishes granular marketplace packages so teams can install the broad Entur toolbox or discover the exact convention area they need.

APM and Claude-compatible marketplace artifacts use a top-level `plugins` array. In this repository, those plugin entries package **skills**: each local plugin directory contains metadata plus a `skills/` link to the underlying skill source. There are no prompt, hook, MCP server, or agent packages yet because this repository does not currently contain those primitives.

## Install the marketplace

```shell
apm marketplace add entur/apm-marketplace-poc
```

The marketplace declares `name: entur`, so APM registers it under the `entur` alias unless you override it:

```shell
apm marketplace add entur/apm-marketplace-poc --name entur
```

Browse everything:

```shell
apm marketplace browse entur
```

Search within this marketplace:

```shell
apm search "terraform@entur"
apm search "kotlin@entur"
apm search "cicd@entur"
```

## AI primitives

| Primitive | Marketplace status |
|-----------|--------------------|
| Skills | Published as installable APM packages. |
| Prompts | Not present yet. Add prompt files under a plugin package before listing them. |
| Hooks | Not present yet. Add hook resources under a plugin package before listing them. |
| MCP servers | Not present yet. Add MCP configuration/resources under a plugin package before listing them. |
| Agents | Not present yet. Add agent definitions under a plugin package before listing them. |

## Core skill packages

| Package | Primitive | Purpose |
|---------|-----------|---------|
| `bootstrap` | Skill | Bootstrap a new Entur app with self-service manifests, Helm, Terraform, Docker, and CI/CD wired together with consistent identifiers. |
| `cicd-workflows` | Skill | Generate Entur-standard CI/CD GitHub Actions workflows for Kotlin/Java, Go, or Python projects. |
| `scr` | Skill | Structure problems and decisions in Situation, Complication, Resolution format. |
| `conventions` | Skill | Full Entur platform convention router for AI agents. |

Install with:

```shell
apm install bootstrap@entur
apm install cicd-workflows@entur
apm install scr@entur
apm install conventions@entur
```

## Granular convention skill packages

The convention packages below all resolve to the shared `plugins/guides` router, but they are listed separately so marketplace browsing, search, and dependency provenance are topic-specific.

| Package | Primitive | Focus |
|---------|-----------|-------|
| `conventions-api-design` | Skill | REST and gRPC API contracts |
| `conventions-architecture` | Skill | Service boundaries, GCP project structure, resilience, production hardening |
| `conventions-authorization` | Skill | Permission Store, Permission Client, OAuth/OIDC, Spring Security |
| `conventions-cicd-actions` | Skill | Shared and composite GitHub Actions |
| `conventions-cicd-workflows` | Skill | Entur reusable CI/CD workflows |
| `conventions-code-review` | Skill | Code review checklist and review conventions |
| `conventions-docker` | Skill | Dockerfiles, multi-stage builds, distroless images |
| `conventions-documentation` | Skill | Documentation and user-facing prose |
| `conventions-go` | Skill | Go services |
| `conventions-helm` | Skill | Helm, Kubernetes, GKE, and the Entur common chart |
| `conventions-iam-roles` | Skill | Approved Google Cloud IAM roles |
| `conventions-java` | Skill | Java and Spring Boot services |
| `conventions-kafka` | Skill | Kafka producers, consumers, schemas, and Aiven Kafka |
| `conventions-kotlin` | Skill | Kotlin and Spring Boot services |
| `conventions-logging` | Skill | Structured logging |
| `conventions-markdown` | Skill | Markdown and markdownlint conventions |
| `conventions-observability` | Skill | Health checks, Prometheus metrics, logging, tracing |
| `conventions-security` | Skill | Secrets, security scanning, allowlists, headers |
| `conventions-self-service` | Skill | Self-service manifests and GCP project naming |
| `conventions-terraform` | Skill | Entur Terraform modules and GCP resources |

Example installs:

```shell
apm install conventions-kotlin@entur
apm install conventions-terraform@entur
apm install conventions-security@entur
```

## Repository layout

```text
apm.yml                           # Hand-edited APM marketplace source of truth
.claude-plugin/marketplace.json   # Generated marketplace artifact from `apm pack`
plugins/                          # Local packages exposed by the marketplace
skills/                           # Source skills used by plugin packages
guides/                           # Entur platform conventions used by the skills
tests/                            # Documentation comprehension tests
```

APM uses a single source-of-truth model:

1. Edit `apm.yml`.
2. Run `apm pack`.
3. Commit both `apm.yml` and `.claude-plugin/marketplace.json`.

The generated `.claude-plugin/marketplace.json` is the marketplace artifact consumed by Claude Code, Copilot CLI, and APM. Do not edit it by hand.

## Maintaining packages

Add or update packages in `marketplace.packages` in `apm.yml`. Marketplace entries should be granular enough to be searchable, and their `tags` should include the primitive type (`skill`, `prompt`, `hook`, `mcp`, or `agent`):

```yaml
marketplace:
  packages:
    - name: conventions-example
      source: ./plugins/guides
      version: 0.1.0
      description: Entur example conventions.
      tags: [entur, conventions, example]
```

Then regenerate the marketplace:

```shell
apm pack
```

Preview without writing:

```shell
apm pack --dry-run
```

`apm marketplace check` is useful for remote package entries. This marketplace currently uses local-path entries, which are validated by `apm pack`.

## Development checks

Run the lightweight test pass before opening a pull request:

```shell
cd tests
go run . --dry-run
```

If you change guides or skills, run the relevant comprehension tests described in `tests/README.md`.

## License

[EUPL v1.2](https://eupl.eu/1.2/en/) - Entur AS

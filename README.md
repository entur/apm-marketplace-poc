# Entur APM Marketplace

Curated APM marketplace for Entur AI agent plugins and skills.

This repository publishes a marketplace index that lets developers install Entur platform tooling through APM while keeping plugin sources versioned, reviewable, and reproducible.

## Install the marketplace

```shell
apm marketplace add entur/apm-marketplace-poc
```

The marketplace declares `name: entur`, so APM registers it under the `entur` alias unless you override it:

```shell
apm marketplace add entur/apm-marketplace-poc --name entur
```

Browse the available plugins:

```shell
apm marketplace browse entur
```

Search within this marketplace:

```shell
apm search "terraform@entur"
```

## Install plugins

Install a plugin with the `NAME@MARKETPLACE` syntax:

```shell
apm install bootstrap@entur
apm install cicd-workflows@entur
apm install guides@entur
apm install scr@entur
```

APM resolves marketplace entries to the local plugin packages in this repository and records the resolved source in the consumer project's `apm.yml` and `apm.lock.yaml`.

## Available plugins

| Plugin | Purpose |
|--------|---------|
| `bootstrap` | Bootstrap a new Entur app with self-service manifests, Helm, Terraform, Docker, and CI/CD wired together with consistent identifiers. |
| `cicd-workflows` | Generate Entur-standard CI/CD GitHub Actions workflows for Kotlin/Java, Go, or Python projects. |
| `guides` | Route Entur platform convention questions to the matching guide for AI agents. |
| `scr` | Structure problems and decisions in Situation, Complication, Resolution format. |

## Repository layout

```text
apm.yml                           # Hand-edited APM marketplace source of truth
.claude-plugin/marketplace.json   # Generated marketplace artifact from `apm pack`
plugins/                          # Local plugin packages exposed by the marketplace
skills/                           # Source skills used by plugin packages
guides/                           # Entur platform conventions used by the skills
tests/                            # Documentation comprehension tests
```

APM uses a single source-of-truth model:

1. Edit `apm.yml`.
2. Run `apm pack`.
3. Commit both `apm.yml` and `.claude-plugin/marketplace.json`.

The generated `.claude-plugin/marketplace.json` is the marketplace artifact consumed by Claude Code, Copilot CLI, and APM. Do not edit it by hand.

## Maintaining the marketplace

Add a local plugin package by editing the `marketplace.packages` list in `apm.yml`:

```yaml
marketplace:
  packages:
    - name: example
      source: ./plugins/example
      version: 0.1.0
      description: Example Entur plugin
      tags:
        - entur
```

Then regenerate the marketplace:

```shell
apm pack
```

Preview without writing:

```shell
apm pack --dry-run
```

Check the marketplace configuration before publishing changes:

```shell
apm pack --dry-run
```

`apm marketplace check` is useful for remote marketplace entries. This marketplace currently ships local-path entries, which are validated by `apm pack`.

For remote plugin packages, prefer immutable refs or semver ranges that resolve to pinned commits when packed.

## Development checks

Run the lightweight test pass before opening a pull request:

```shell
cd tests
go run . --dry-run
```

If you change guides or skills, run the relevant comprehension tests described in `tests/README.md`.

## License

[EUPL v1.2](https://eupl.eu/1.2/en/) - Entur AS

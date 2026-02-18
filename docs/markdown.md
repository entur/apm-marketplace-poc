# Markdown Standards

All documentation in Entur repositories must pass [markdownlint](https://github.com/DavidAnson/markdownlint) with the configuration defined in this project.

## Linting

### Configuration

Place a `.markdownlint-cli2.jsonc` file at the repository root:

```json
{
  "config": {
    "default": true,
    "MD013": false
  }
}
```

MD013 (line length) is disabled because tables, code blocks, and long URLs frequently exceed 80 characters.

### Running Locally

```bash
npm install -g markdownlint-cli2
markdownlint-cli2 "**/*.md"
```

### Running in CI

Add markdownlint to your CI pipeline or use a pre-commit hook. All markdown files must pass with zero violations before merge.

## Rules Summary

The following rules are enforced. Full reference: [markdownlint rules v0.40.0](https://github.com/DavidAnson/markdownlint/tree/v0.40.0/doc).

### Headings

- **MD001**: Heading levels must increment by one (no skipping from `#` to `###`)
- **MD003**: Use ATX-style headings (`# Heading`, not underlines)
- **MD018/MD019**: Exactly one space after `#` in headings
- **MD022**: Headings must be surrounded by blank lines (above and below)
- **MD023**: Headings must start at column 1
- **MD024**: No duplicate heading text at the same level (use `siblings_only` if needed)
- **MD025**: Only one top-level `#` heading per file
- **MD026**: No trailing punctuation in headings
- **MD041**: First line must be a top-level heading

### Lists

- **MD004**: Use consistent list markers (dashes `-` preferred)
- **MD005**: Consistent indentation for same-level list items
- **MD007**: Indent nested lists by 2 spaces
- **MD029**: Ordered lists use `1.` prefix (or sequential)
- **MD030**: Exactly one space after list markers
- **MD032**: Lists must be surrounded by blank lines

### Code Blocks

- **MD031**: Fenced code blocks must be surrounded by blank lines
- **MD040**: Fenced code blocks must specify a language (use `text` for plain text)
- **MD046**: Use fenced (not indented) code blocks

### Whitespace

- **MD009**: No trailing spaces
- **MD010**: No hard tabs
- **MD012**: No multiple consecutive blank lines

### Links and Images

- **MD011**: No reversed link syntax
- **MD034**: No bare URLs (use `<url>` or `[text](url)`)
- **MD039**: No spaces inside link text
- **MD042**: No empty links
- **MD045**: Images must have alt text

### Inline Formatting

- **MD037**: No spaces inside emphasis markers
- **MD038**: No spaces inside code span backticks

## Writing Guidelines

### Structure

- Start every file with a single `#` heading
- Use heading hierarchy without skipping levels
- Separate all block elements (headings, lists, code blocks, tables) with blank lines
- End every file with a single newline

### Code Block Language Tags

Always specify the language for syntax highlighting:

````markdown
```java
public class Example {}
```
````

Use `text` for blocks with no specific language:

````markdown
```text
Plain text output
```
````

Use `yaml`, `json`, `hcl`, `kotlin`, `go`, `python`, `bash`, `dockerfile`, `xml`, `sql` as appropriate.

### List Formatting

Use `-` for unordered lists. Use `1.` for ordered lists:

```markdown
- First item
- Second item
  - Nested item

1. Step one
2. Step two
```

### Tables

Align table columns for readability:

```markdown
| Name   | Description     |
|--------|-----------------|
| `foo`  | Does foo things |
| `bar`  | Does bar things |
```

### Links

Use relative paths for internal links:

```markdown
[CONVENTIONS.md](../CONVENTIONS.md)
[java.md](java.md)
[terraform modules](terraform/modules.md)
```

Use full URLs for external links:

```markdown
[entur/helm-charts](https://github.com/entur/helm-charts)
```

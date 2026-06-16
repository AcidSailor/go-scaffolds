# go-scaffolds

Copier templates for Go projects plus reusable GitHub Actions workflows.
Generated projects are licensed AGPL-3.0-or-later.

## Templates

| `project_type` | Produces | Notable extras |
|----------------|----------|----------------|
| `library`      | otelhttp-instrumented Go client library | doc.go, testify |
| `cli`          | CLI binary (cobra or kong) | GoReleaser + ghcr docker, dependabot |
| `mcp`          | MCP server on `mcpkit` (all transports) | otel, mcpb bundle, GoReleaser + ghcr docker |

## Usage

```bash
uvx copier copy gh:acidsailor/go-scaffolds <dest>
# non-interactive examples:
uvx copier copy --data project_type=library --data project_name=foo gh:acidsailor/go-scaffolds <dest>
uvx copier copy --data project_type=cli --data project_name=foo --data cli_framework=kong gh:acidsailor/go-scaffolds <dest>
uvx copier copy --data project_type=mcp --data project_name=foomcp gh:acidsailor/go-scaffolds <dest>
```

Update a generated project later: `uvx copier update` (uses the saved
`.copier-answers.yml`).

## Reusable workflows

Generated repos call these, pinned `@v1`:

- `go-ci.yml` — checkout, setup Go (from go.mod) + Task + golangci-lint, `task ci`, `task test`.
- `go-release.yml` — checkout, setup Go + Task + GoReleaser, ghcr login, `task release`.

Each project's `taskfile` owns the concrete `ci`/`test`/`release` commands, so the
workflows stay generic and CI cannot drift across repos.

## mcp prerequisite

The `mcp` template depends on the published, tagged `github.com/acidsailor/mcpkit`
module. Tag mcpkit before generating/retrofitting MCP servers.

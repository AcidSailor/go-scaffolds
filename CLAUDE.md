# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`go-scaffolds` is **not a Go project** — it is a [Copier](https://copier.readthedocs.io/) template collection plus reusable GitHub Actions workflows. There is no Go module, no build, and no test suite at the repo root. The "code" here is Jinja-templated source that gets rendered into *generated* downstream Go projects. Generated projects are licensed AGPL-3.0-or-later.

## Layout

- `copier.yml` — the single questionnaire. Its `_subdirectory: templates/go-{{ project_type }}` picks exactly one of the three template trees based on the answer to `project_type` (`cli` / `library` / `mcp`).
- `templates/go-{cli,library,mcp}/` — one self-contained template tree per project type.
- `.github/workflows/go-ci.yml` and `go-release.yml` — **reusable** workflows (`workflow_call`). Generated repos call these pinned `@v1`; they are not this repo's own CI.

## How templating works (read before editing templates)

- **Only `.jinja` files are rendered** (`_templates_suffix: ".jinja"`); the suffix is stripped on output. Every other file is copied verbatim. So `taskfile.yml` (no suffix) is literal, `taskfile.yaml.jinja` is rendered.
- **Filenames are templated too.** Directory names like `templates/go-cli/{% if cli_framework == 'cobra' %}cmd{% endif %}/` and `{{ project_name }}.go.jinja` are evaluated at render time — a falsy condition collapses the path segment.
- **Seed-once via `_exclude`** (`copier.yml`): `*.go`, `go.mod`, and `README.md` are excluded *only when `_copier_operation == 'update'`*. This means business logic, the module file, and the README are rendered on a fresh `copier copy` but **never re-rendered or 3-way-merged on `copier update`**. When editing templates, remember that changes to `*.go`/`go.mod`/`README.md.jinja` only reach *new* projects, while changes to tooling (taskfiles, `.golangci.yaml`, workflow stubs, `.goreleaser.yaml`) propagate to existing adopted repos on their next `task update`.
- Copier's default `_exclude` list is replaced when you set your own, so the defaults (`.git`, `.DS_Store`, `~*`, etc.) are re-listed explicitly — keep them when adding entries.

## CI / task split (intentional indirection)

The reusable workflows are deliberately generic: `go-ci.yml` runs `task ci` then `task test`; `go-release.yml` runs `task release`. Each template's taskfile owns the *concrete* commands, so CI logic lives in the generated repo and cannot drift across the workflow. When changing what CI does, edit the **template taskfile**, not the workflow, unless you're changing the runner setup (Go/Task/golangci-lint/GoReleaser install steps).

Standard task targets across all templates (`task <name>`):
- `lint` — `golangci-lint fmt` + `run --fix` (mutates files).
- `ci` — read-only `golangci-lint fmt --diff` + `run` (fail-fast, used by CI).
- `test` — `go test -race ./...`.
- `update` — `uvx copier update` (pull latest template tooling; source untouched).
- cli/mcp also have `build` (`goreleaser release --snapshot --clean`) and `release`; mcp adds `pack` / `_pack-mcpb` for the `.mcpb` bundle.

## Testing template changes

There is no test command. Validate by rendering into a throwaway directory and inspecting/building the output:

```bash
# render from the local working tree (not the published gh: ref)
uvx copier copy --data project_type=cli --data project_name=foo . /tmp/foo
uvx copier copy --data project_type=mcp --data project_name=foomcp . /tmp/foomcp
uvx copier copy --data project_type=library --data project_name=foolib . /tmp/foolib
# then in the rendered dir: go build ./... && task test
```

CLI template takes `--data cli_framework=cobra|kong`; MCP takes `--data mcp_title=...`.

## Template specifics

- **mcp** depends on the published, **tagged** module `github.com/acidsailor/mcpkit`. Tag mcpkit before generating or retrofitting MCP servers, or `go build` in the rendered project fails. The MCP server (`internal/server/server.go.jinja`) wires an otelhttp-instrumented `http.Client` and registers tools onto an `mcp.Server` wrapped by `mcpkit/server` (stdio/http/both transport selected by config).
- **cli** supports two framework skeletons (`cobra` → `cmd/` tree, `kong` → `main.go`), GoReleaser + ghcr docker, and dependabot.
- **library** is an otelhttp-instrumented client library skeleton with `doc.go` and testify.

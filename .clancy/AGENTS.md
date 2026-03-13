# AGENTS.md — wood-jp.github.io

<!-- Project-specific build, test, and lint commands. -->
<!-- Org-level defaults are in clancy/ralph/AGENTS.md. -->
<!-- Override or extend them here as needed for this project. -->

## Build

```bash
go build ./...
```

## Test

```bash
go test -race ./...
```

## Lint

```bash
go vet ./...
golangci-lint run
```

## Notes

<!-- Any project-specific instructions for Ralph:
- Files or directories to avoid modifying
- External dependencies or services required
- Known test fragility (e.g. line-number-sensitive tests)
-->

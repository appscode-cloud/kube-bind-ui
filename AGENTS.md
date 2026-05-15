# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

A small Go HTTP server that renders an HTML page used by the `kube-bind` consent / binding flow. Effectively the consent page the user lands on when running `kubectl bind`.

Module path is `github.com/Superm4n97/html-render` (personal namespace from the original author); the filesystem location under `go.bytebuilders.dev/kube-bind-ui` doesn't match the module path. Imports use the module path.

## Architecture

- `main.go` — entry point; calls `http.StartServer()`.
- `pkg/http/` — HTTP server implementation.
- `process.js` — supporting JS asset.
- `package.json` — npm deps if any JS preprocessing is needed.
- `vendor/` — checked-in Go deps.

## Common commands

```
go build .
go run .
```

No Makefile, no tests configured.

## Conventions

- Tiny by design; resist scope creep.
- Module path is `github.com/Superm4n97/html-render` (personal). If you turn this into a maintained library, move it to `go.bytebuilders.dev/...` first.
- License: see `LICENSE` if present.

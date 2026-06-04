# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

A small Go HTTP server that renders the HTML "consent / binding" page used by the [`kube-bind`](https://kube-bind.io/) flow. When a user runs `kubectl bind`, this server presents a page listing the CRDs (resources) available on a provider cluster and lets them choose which ones to bind to their consumer cluster. After a successful bind the user is redirected to a success page that links back to the provider.

The HTML is rendered server-side from Go templates (`html/template` + [Masterminds/sprig](https://github.com/Masterminds/sprig) helpers); styling reuses the AppsCode design system (Bulma + `@appscode/design-system`) compiled into a single static `main.css`.

## Tech stack

| Layer | Library / Tool |
|---|---|
| Language | Go (module `go 1.21`, `toolchain go1.21.2`) |
| HTTP | Standard library `net/http` |
| Templating | `html/template` + `github.com/Masterminds/sprig/v3` |
| Form decoding | `github.com/gorilla/schema` |
| Logging | `k8s.io/klog/v2` |
| Static asset embedding | `embed.FS` (`//go:embed`) |
| Styling source | SCSS (`pkg/template/static/assets/styles/main.scss`) importing `@appscode/design-system` and Bulma from `node_modules/` |
| CSS post-processing | PostCSS + `@fullhuman/postcss-purgecss` (`process.js`) — purges unused selectors from the generated `main.css` |
| Vendoring | Go modules with checked-in `vendor/` directory |
| Dependabot | Monitors both `gomod` and `npm` ecosystems (PR limit set to 0; tracking only) |

> **Module path quirk**: `go.mod` declares the module as `github.com/Superm4n97/html-render` (the original author's personal namespace), even though the working copy lives at `go.bytebuilders.dev/kube-bind-ui`. All internal imports use the `github.com/Superm4n97/html-render/...` prefix — do not "fix" them in passing without renaming the module everywhere (including `vendor/modules.txt` resolution and any deployment that references it).

## Folder structure

```
kube-bind-ui/
├── main.go                              # Entry point: calls http.StartServer()
├── go.mod / go.sum                      # Module: github.com/Superm4n97/html-render (go 1.21)
├── vendor/                              # Vendored Go dependencies (checked in)
├── package.json                         # npm deps for SCSS sources + PurgeCSS pipeline
├── package-lock.json
├── process.js                           # PostCSS + PurgeCSS script to slim down main.css
│                                        # NOTE: hard-codes /Users/mohin/... path — see "Known issues"
├── pkg/
│   ├── http/
│   │   └── handler.go                   # HTTP server, route registration, request handlers
│   │                                    # Exports: StartServer()
│   │                                    # Active routes (see addHandlers): /resource, /main.css
│   │                                    # Commented-out routes: /, /index, /student, /top-student,
│   │                                    #                       /crd-test, /bind, /success
│   ├── template/
│   │   ├── files.go                     # //go:embed * — embeds the whole template/ tree as embed.FS
│   │   ├── resources.gohtml             # Main page: navbar + grid/table of CRDs with bind form
│   │   ├── success.gohtml               # Post-bind success page with "Go to Provider" CTA
│   │   ├── index.html                   # Trivial placeholder HTML (used by httpFileHandler stub)
│   │   ├── student.html / success.html  # Legacy / experimental templates (not served)
│   │   └── static/assets/
│   │       ├── icons/checked.png        # Embedded icon (currently unused by served pages)
│   │       └── styles/
│   │           ├── main.scss            # SCSS source; imports design-system + Bulma from node_modules
│   │           ├── main.css             # Compiled CSS served at /main.css via embed.FS
│   │           └── test.go              # Package `styles`: embeds main.css as embed.FS (Files)
│   └── student/
│       └── student.go                   # Experimental / legacy handlers (TemplatedHandlerBasic,
│                                        # TemplatedHandlerFile, CrdTest). Not wired into the
│                                        # current router; kept for reference.
├── .github/
│   └── dependabot.yml                   # Daily refresh for gomod + npm (open-pull-requests-limit: 0)
├── .gitignore                           # Ignores .idea, node_modules, *.css.map
├── AGENTS.md                            # This file
├── CLAUDE.md                            # Single-line stub: @AGENTS.md
└── README.md                            # One-liner placeholder
```

## HTTP server

`pkg/http/handler.go` defines:

- `type handler struct { port string }` — hard-coded to `:8080` in `newHandler()`.
- `StartServer()` (exported) — constructs the handler, registers routes on the default `http.ServeMux`, and calls `http.ListenAndServe(":8080", nil)`.
- `addHandlers()` registers only two live routes today:

  | Route | Handler | Purpose |
  |---|---|---|
  | `GET /resource` | `handleResources` | Renders `resources.gohtml` with a hard-coded sample `CRDsInfo` (4 KubeDB CRDs: MongoDB, MySQL, Postgres, Kafka). |
  | `GET /main.css` | `handleCSS` | Renders the embedded `main.css` (treated as a template so sprig funcs are available, though it currently has no template directives). Content-Type: `text/css`. |

  The remaining handlers (`simpleIndexHandler`, `httpFileHandler`, `handleBind`, `handleSuccess`, and `student.*`) are present in source but their `http.HandleFunc` registrations are commented out. Uncomment them in `addHandlers()` to bring them back online.

### Cache headers

All dynamic page responses go through `prepareNoCache(w)`, which sets:

```
Expires:         Thu, 01 Jan 1970 00:00:00 UTC
Cache-Control:   no-cache, no-store, must-revalidate, max-age=0
X-Accel-Expires: 0
```

This prevents browser and Nginx-proxy caching of the consent UI.

### Bind form decoding

`/bind` (when enabled) expects a form POST with fields:

```go
type BindForm struct {
    SessionID string   `schema:"sessionID"`
    GRs       []string `schema:"crd"`     // one entry per checked checkbox
}
```

decoded with a package-level `gs.NewDecoder()` from `gorilla/schema`. The current `handleBind` simply echoes the decoded form back to the client (placeholder logic).

### Template helpers

Templates are parsed with the sprig HTML func map:

```go
htmltemplate.New(name).Funcs(sprig.HtmlFuncMap()).ParseFS(files, name)
```

`resources.gohtml` uses sprig's `list`, `compact`, and `join` (e.g. `{{ list $crd.Resource $crd.Group | compact | join "." }}` to build the `GroupResource` value posted back).

## Templates

| File | Purpose |
|---|---|
| `pkg/template/resources.gohtml` | Main consent page. Top half is the AppsCode navbar (app-drawer, profile menu, switch-account list — all currently rendered with static placeholder data). Body shows a `<form action="/bind" method="post">` with either a grid of CRD cards (when `len .CRDs <= 10`) or a table view (otherwise). Each CRD becomes a checkbox; bound CRDs are rendered disabled. |
| `pkg/template/success.gohtml` | Standalone success page styled inline (Bulma via CDN + a small `<style>` block). Shows a green checkmark, a "Success!" headline, and a "Go to Provider" button pointing at `{{.RedirectURL}}`. |
| `pkg/template/index.html` | Minimal `<h1>Main Application</h1>` stub used by the (currently commented-out) `httpFileHandler`. |
| `pkg/template/student.html`, `success.html` | Legacy fixtures, not served. |

The page expects `main.css` to be served at the **same path level** as the page itself (the template uses `<link rel="stylesheet" href="main.css">`). The current routes (`/resource` + `/main.css`) satisfy this when accessed at the root.

## Styling pipeline

The CSS that ships in the app (`pkg/template/static/assets/styles/main.css`, ~1.7k lines) is the **already-compiled, already-purged** output. The SCSS source and PurgeCSS script are kept for reproducibility:

1. `pkg/template/static/assets/styles/main.scss` imports:
   - `@appscode/design-system/vue-components/styles/base/utilities/...`
   - `bulma/bulma.sass`
   - selected design-system components (button, navbar, sidebar, header, table)

   Imports use relative paths into `../../../../../node_modules/...`, so the SCSS compiler must be invoked from the styles directory with `node_modules/` populated at the repo root.

2. `process.js` (root) runs PostCSS + `@fullhuman/postcss-purgecss` against the compiled CSS, scanning `./pkg/**/*.gohtml` for used selectors. It safelists `active`, `is-visible`, `is-right-0`, `fserv-field`, `select2-*`, `placeholder`, `fserv-button-submit`, anything matching `^fserv-`, and all `@keyframes`.

3. The purged CSS is written back to the same file and embedded into the Go binary via `pkg/template/static/assets/styles/test.go` (`//go:embed main.css`).

> **Known issue**: `process.js` hard-codes the absolute path `/Users/mohin/go/src/github.com/html-render/pkg/template/static/assets/styles/main.css`. Update this `cssFilePath` constant to a repo-relative path (or pass it in as `process.argv[2]`) before running locally.

## Common commands

| Command | Purpose |
|---|---|
| `go build .` | Compile the binary. |
| `go run .` | Run the server (listens on `:8080`). |
| `go mod vendor` | Refresh the vendored deps after editing `go.mod`. |
| `npm install` | Install SCSS / PostCSS / PurgeCSS toolchain. |
| `node process.js` | Re-purge `main.css` (after fixing the hard-coded path; requires an already-compiled `main.css` next to `main.scss`). |

There is **no Makefile, no test suite, no CI workflow, no Dockerfile, and no `go vet`/`golangci-lint` config** in the repo. `npm test` is the default "Error: no test specified" placeholder. The only automation in `.github/` is `dependabot.yml`.

## Conventions & best practices

- **Keep it tiny.** This binary serves two routes from a single embedded template tree; resist scope creep. Any new feature that wants its own datastore, auth, or backend integration almost certainly belongs in a different service.
- **Embed, don't ship file paths.** All templates and CSS are loaded via `embed.FS`. Do not introduce code that reads from `./pkg/template/...` at runtime (the `httpFileHandler` stub in `handler.go` is an anti-example kept only for reference — it would break in any deployment that doesn't carry the source tree alongside the binary).
- **Module path stays as-is.** The `github.com/Superm4n97/html-render` module path is load-bearing for every internal import. Renaming requires a coordinated change across `go.mod`, every `import` statement, and `vendor/modules.txt`. Do not change it in isolation.
- **Vendor is the source of truth.** Always commit `vendor/` changes alongside `go.mod`/`go.sum` (`go mod tidy && go mod vendor`).
- **Routes live in `addHandlers()`.** When enabling or adding a route, register it there. Don't sprinkle `http.HandleFunc` calls elsewhere — the package relies on the default `http.DefaultServeMux`.
- **Cache headers.** Any new page handler that returns dynamic HTML should call `prepareNoCache(w)` first. Static assets (e.g. CSS) currently do not — and arguably should not — set no-cache.
- **Templates go in `pkg/template/`.** Add new `*.gohtml` files there, register them via a `Template<Name>` constant in `files.go`, and load them with `getTemplate(template.Files, template.TemplateXxx)` so sprig functions are available.
- **Form parsing.** Use `gorilla/schema` with `schema:"..."` struct tags for any new POST handler; mirror the `BindForm` pattern. Always call `r.ParseForm()` before `decoder.Decode(&dst, r.PostForm)`.
- **Logging.** Use `k8s.io/klog/v2` consistently — prefer `klog.FromContext(r.Context()).WithValues(...)` inside handlers so log lines carry request context. Avoid `log` from the standard library.
- **Styling changes.** Edit `main.scss`, recompile, re-run `process.js`, then commit the regenerated `main.css`. Do not hand-edit the compiled CSS.
- **Dependabot PRs.** `open-pull-requests-limit: 0` means Dependabot currently only checks for updates; it will not open PRs automatically. Flip this number if you want automated bumps.
- **License.** No top-level `LICENSE` file is present in the repo; add one before any public distribution.

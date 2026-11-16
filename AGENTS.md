# Repository Guidelines

## Build, Test, and Development Commands
- `make` builds server + client using vendored modules and runs `go-assets.sh`.
- `make client` builds a CGO-free client; `make debug` keeps symbols.
- `make pb` regenerates protobuf stubs from `protobuf/`.
- `./go-tests.sh` runs the Go test suite, this is the only way you should run unit tests.
- Docs site: `cd docs/sliver-docs && npm install && npm run dev` for local dev; `npm run build` for a static build.

## Coding Style & Naming Conventions
- Run `gofmt` on all Go code; keep package names lowercase and idiomatic.
- Avoid `CGO` and empty interfaces; prefer explicit types and interfaces.
- Do not import `server` packages from `client`.

## Testing Guidelines
- Prefer `docker build --target test --build-arg GO_TESTS_FLAGS=--skip-generate .` 
- Add or update tests when behavior changes; some server packages run longer tests.

## CodeGraph

A code intelligence index is maintained at `.codegraph/`. It provides sub-millisecond symbol lookup, call-path tracing (including dynamic dispatch), and blast-radius analysis across the entire Go codebase.

### Usage

```
# MCP tool (when available in session)
codegraph_explore  query="<symbol names or question>"  projectPath="D:\vcs\testlab-ael\sliver"

# CLI fallback (always works)
codegraph explore "<symbol names or question>"
```

- One call returns verbatim line-numbered source grouped by file, call paths between named symbols, and a blast-radius summary — treat it as a Read you already performed.
- Prefer codegraph over grep/read loops for locating code, understanding flows, or checking what depends on a symbol before editing.
- The index auto-updates via file watcher (~1s lag after writes).

### Examples

```
# Understand implant startup flow
codegraph explore "Main sessionStartup beaconStartup"

# Find all callers of a handler
codegraph explore "WrapperHandler"

# Trace transport negotiation
codegraph explore "StartConnectionLoop httpConnect dnsConnect"
```

---

## Knowledge Graph (understand-anything)

A knowledge graph of this codebase is maintained at `.ua/knowledge-graph.json`.

- **8,041 nodes** (functions, files, classes, schemas) · **29,376 edges** · **1,801 files** analyzed
- **8 layers:** `layer:protocol` → `layer:implant` → `layer:server` → `layer:client` → `layer:utility` → `layer:test` → `layer:documentation` → `layer:infrastructure`
- **15-step guided tour** starting from the protobuf contract through to operator CLI

### Useful commands

```powershell
# Launch dashboard
$env:GRAPH_DIR = "d:\vcs\testlab-ael\sliver"
cd "C:\Users\minht\.claude\plugins\cache\understand-anything\understand-anything\2.9.8\packages\dashboard"
npx vite --host 127.0.0.1
# Token appears in output: 🔑 Dashboard URL: http://127.0.0.1:5173/?token=...

# Kill dashboard
netstat -ano | findstr ":5173"      # get PID
taskkill /PID <PID> /F
```

```
# Chat with codebase, ask questions about architecture/code
/understand-chat d:/vcs/testlab-ael/sliver

# Update graph after code changes (incremental — only re-analyzes changed files)
/understand d:/vcs/testlab-ael/sliver
```

### Key node IDs for fork work

| Area | Node IDs |
|---|---|
| Shellcode execution | `file:implant/sliver/taskrunner/task_windows.go` |
| Reflective loader | `file:implant/sliver/loaders/memmod/memmod_windows.go` |
| Evasion (RefreshPE) | `file:implant/sliver/evasion/evasion_windows.go` |
| HTTP transport | `file:implant/sliver/transports/httpclient/httpclient.go` |
| Handler dispatch | `file:implant/sliver/handlers/handlers_windows.go` |
| AMSI/ETW patch | `file:implant/sliver/taskrunner/task_windows.go` |
| Implant limits | `file:implant/sliver/limits/limits_windows.go` |


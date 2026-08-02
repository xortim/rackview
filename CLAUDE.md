# CLAUDE.md — rackview

Operating instructions for Claude Code in this repository. Read this before touching anything.

## What this is

`rackview` is a full-screen TUI dashboard for an always-on rack console. It runs as an unprivileged systemd service bound to `tty1`. It aggregates host telemetry from Prometheus, active alerts from Alertmanager, and curated entity states from Home Assistant.

**Design invariant:** `rackview` never talks to rack hosts directly. Prometheus is the aggregation point. Adding a host to the dashboard means adding a scrape target, not adding code.

### In scope

- TUI rendering, layout, and theming
- Read-only collection from Prometheus, Alertmanager, and Home Assistant
- Config loading, CLI surface, and deployment artifacts (systemd units, console setup)

### Out of scope

- Writing to Home Assistant (no service calls, no state mutation)
- Alerting, notification, or paging — Alertmanager and ntfy already own that
- Any privileged operation. The service runs as a system user with an empty capability bounding set
- Direct SSH or agent connections to rack hosts

---

## Git workflow

**Never commit to `main`.** No exceptions, no "just this once," no small fixes.

Before every commit, verify the current branch:

```bash
git branch --show-current
```

If it returns `main`, stop and create a branch first.

### Branch naming

`<type>/<short-slug>` where `<type>` matches the Conventional Commits type.

```bash
git switch -c feat/ha-websocket-subscribe
git switch -c fix/sparkline-negative-index
git switch -c refactor/collect-interface
```

One logical change per branch. If a branch grows a second concern, split it.

### Merge policy

- Merge to `main` via PR only
- Squash-merge feature branches; the PR title becomes the commit subject and must itself be a valid Conventional Commit
- `main` stays green: every commit on `main` must build and pass `make check`

---

## Conventional Commits

Follow the [Conventional Commits v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) spec.

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

| Type | Use for |
|---|---|
| `feat` | New user-visible capability |
| `fix` | Bug fix |
| `refactor` | Restructuring with no behavior change |
| `perf` | Performance improvement |
| `docs` | Documentation only |
| `test` | Test additions or corrections |
| `build` | `go.mod`, Makefile, build tooling |
| `ci` | Pipeline configuration |
| `chore` | Everything else (dependency bumps, housekeeping) |
| `revert` | Reverts a prior commit |

### Scopes

`cmd`, `config`, `collect`, `ui`, `deploy`, `docs`. Omit the scope only when a change is genuinely repo-wide.

### Rules

- Subject in imperative mood, lowercase, no trailing period, under 72 characters
- Body explains *why*, not *what* — the diff already says what
- Breaking changes get a `!` after the scope **and** a `BREAKING CHANGE:` footer

### Examples

```
feat(collect): subscribe to Home Assistant state via websocket

REST polling issued one request per entity per interval. The websocket
subscribe_entities API pushes diffs, cutting steady-state request volume
to zero and dropping observed state latency from ~10s to sub-second.

Closes #14
```

```
fix(ui): clamp sparkline index when a sample exceeds 100

node_cpu_seconds_total can briefly yield >100% on counter reset,
which indexed past the end of sparkRunes and panicked the render loop.
```

```
refactor(config)!: move all settings under viper

BREAKING CHANGE: RACKVIEW_* environment variables are still honored but
the canonical source is now /etc/rackview/config.yaml. Deployments that
set only environment variables continue to work unchanged.
```

---

## Go conventions

Follow [Effective Go](https://go.dev/doc/effective_go) and the [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments). Where they disagree with anything below, they win.

### Non-negotiables

- `gofmt -s` clean. Formatting is not a matter of taste and is not up for discussion in review
- `go vet ./...` clean before every commit
- All non-`main` code lives under `internal/`. Nothing in this repo is a public API
- No package name stutter: `config.Config`, not `config.RackviewConfig`
- Package names are lowercase, singular, no underscores
- `context.Context` is the first parameter of every function that does I/O. Never store it in a struct
- Wrap errors with `%w` and enough context to locate the call site: `fmt.Errorf("query %q: %w", q, err)`
- Error strings are lowercase and unpunctuated
- No `panic` outside `main`. No `log.Fatal` outside `main`
- Accept interfaces, return structs
- Table-driven tests. `testdata/` for fixtures
- Toolchain version is pinned in `go.mod`. Do not bump it inside a feature branch

### Dependencies

| Package | Role |
|---|---|
| `github.com/spf13/cobra` | CLI structure |
| `github.com/spf13/viper` | Config resolution |
| `github.com/charmbracelet/bubbletea` | TUI event loop |
| `github.com/charmbracelet/lipgloss` | Layout and styling |
| `github.com/prometheus/client_golang` | Prometheus HTTP API client |

Adding a dependency is a decision, not a convenience. If the standard library covers it, use the standard library. Alertmanager and Home Assistant are plain JSON over HTTP — they use `net/http` and `encoding/json`, not a vendored client.

**Never introduce Python** for tooling, scripts, or CI helpers. Go or POSIX shell.

---

## Layout

```
rackview/
├── main.go                    # thin; calls cmd.Execute()
├── cmd/
│   ├── root.go                # cobra root, persistent flags, viper binding
│   ├── serve.go               # default command: run the TUI
│   └── check.go               # probe all endpoints, exit non-zero on failure
├── internal/
│   ├── config/                # viper -> typed Config
│   ├── collect/               # prom.go, alertmanager.go, hass.go
│   └── ui/                    # model.go, view.go, panels.go, theme.go, spark.go
├── configs/rackview.example.yaml
└── deploy/                    # rackview.service, console setup
```

### Cobra

- Build commands with constructor functions (`newRootCmd()`, `newServeCmd(cfg *config.Config)`). Do not register commands from `init()` — it makes ordering implicit and testing painful
- Use `RunE`, never `Run`. Return errors; let `Execute()` handle exit codes
- Set `SilenceUsage: true` and `SilenceErrors: true` on root, then print the error once in `Execute()`. Otherwise a runtime failure dumps the full help text over the console

### Viper

Precedence, highest first: **flag → environment → config file → default.**

```go
v.SetEnvPrefix("RACKVIEW")
v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
v.AutomaticEnv()
v.SetConfigName("config")
v.AddConfigPath("/etc/rackview")
```

- Unmarshal once into a typed `config.Config` at startup. Do not call `viper.GetString()` from business logic — it turns config errors into runtime surprises and makes the code untestable
- Validate after unmarshal. Fail fast with a specific message naming the offending key
- **Secrets never go in the viper config file.** The Home Assistant token is delivered via systemd `LoadCredential=` and read from `$CREDENTIALS_DIRECTORY`. Environment fallback exists for local development only

### Bubble Tea

- The model is a value type. `Update` returns a modified copy, never mutates through a pointer
- **No I/O in `Update` or `View`.** Ever. Return a `tea.Cmd` and handle the result as a message
- `View` is pure and fast. It runs on every message
- Goroutines never write to the model. Message passing is the only channel
- One message type per collection source so a partial failure degrades one panel instead of the whole frame

### Rendering behavior

- Stale data must look stale. The header renders data age and turns red past three collection intervals. A dashboard that silently shows old numbers is worse than one that shows nothing
- Threshold colors are fixed, never auto-scaled. Auto-scaling alarm points is how you stop noticing alarms
- On collection failure, keep the last good payload and surface the error in the footer. Do not zero the panel

---

## Verification loop

Run before every commit:

```bash
make check
```

Which is:

```bash
gofmt -l -s .          # must output nothing
go vet ./...
go build ./...
go test ./... -race
```

Claude must run this and confirm it passes before proposing a commit. Do not report a change as done on the strength of a successful build alone.

`make check` does not validate rendering. TUI changes need a human on a real console.

---

## Gotchas

- **The Linux VT is 16-color and its fonts have no Braille glyphs.** Anything relying on truecolor or Braille only renders correctly under the `cage` + `foot` path. Lip Gloss degrades automatically, but verify both paths before claiming a theme change works
- **`tea.WithAltScreen()` is a no-op on `TERM=linux`.** The `linux` terminfo entry has no `smcup`. Do not build layout logic that assumes a clean screen restore
- **`PrivateDevices=yes` in the systemd unit breaks DRM and TTY access.** It looks like an obvious hardening win. It is not
- **Home Assistant long-lived tokens are not read-scoped.** A non-admin HA user narrows the blast radius but does not eliminate write capability. Treat the token as a write credential regardless of how `rackview` uses it
- **Never log the token**, including in error paths. Redact before wrapping

---

## Ask before doing

- Changing the `internal/` package boundaries
- Adding any dependency not listed above
- Anything that writes to Home Assistant or Prometheus
- Modifying `deploy/rackview.service` — a bad unit file locks the console
- Rewriting the PromQL in `collect/prom.go`. The queries are load-bearing and were tuned against the live scrape config

---

## Roadmap

- [ ] `refactor(config)`: migrate env-var loading to cobra + viper
- [ ] `refactor(collect)`: extract a `Collector` interface and fan out concurrently
- [ ] `feat(collect)`: Home Assistant websocket `subscribe_entities` replacing REST polling
- [ ] `feat(ui)`: ZFS panel — pool state, capacity, scrub age (needs a textfile collector for scrub age)
- [ ] `feat(ui)`: Proxmox guest panel via `prometheus-pve-exporter`
- [ ] `feat(collect)`: backfill sparkline history from `query_range` on startup
- [ ] `feat(cmd)`: `rackview check` as a systemd `ExecStartPre` gate
- [ ] `test`: golden-file tests for `View` output at fixed terminal dimensions
- [ ] `ci`: `make check` on PR

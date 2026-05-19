# golang-boilerplate

Production-ready Go boilerplate for a **Windows service** that exposes an HTTP
API and ships with an **out-of-process auto-updater** plus **CI-driven
releases** to GitHub Releases.

Fork this when you need a long-running Go agent that runs on customer machines
and updates itself without manual intervention.

## What you get

- **`server.exe`** — the Windows service. HTTP API, API-key auth, gzip,
  CORS, structured `/health`.
- **`updater.exe`** — sibling binary that owns the actual update apply: stops
  the service, swaps binaries on disk, restarts, polls `/health` for the new
  version, and rolls back to `.old` files on failure.
- **`internal/applier/`** — pure state machine for the swap/rollback flow.
- **`internal/updater/`** — manifest polling, integrity-checked download
  staging, nightly window enforcement, on-demand `POST /update` handler.
- **GitHub Actions release workflow** — pushes to `main` trigger
  [`go-semantic-release`](https://github.com/go-semantic-release/action), which
  computes the next version, tags it, builds both binaries, generates
  `manifest.json`, and uploads everything as release assets.
- **Cross-platform builds** — the service wrapper falls back to foreground
  mode on macOS/Linux so you can iterate locally without a real SCM.

## Why this shape

Out-of-process updates are the reliable pattern for self-updating Windows
services. A service cannot `sc stop` itself and then swap its own running
binary, so an external `updater.exe` does the dance. The split keeps
`server.exe` simple (poll + download + verify + handoff) and isolates the
risky filesystem/SCM steps in a helper that can be replaced independently.

## Project structure

```
.
├── cmd/
│   ├── server/main.go        # service entry point
│   └── updater/main.go       # update helper entry point
├── internal/
│   ├── applier/              # stop/swap/start/poll/rollback state machine
│   ├── config/               # config.json loader (next to the binary)
│   ├── server/               # HTTP server, middleware, handlers
│   ├── service/              # kardianos/service wrapper, SCM recovery
│   └── updater/              # manifest fetch, download staging, handoff
├── .github/workflows/
│   └── release.yml           # semantic-release + asset upload
├── config.example.json
└── README.md
```

## Quick start

### Build

```bash
# Cross-compile for the production target
GOOS=windows GOARCH=amd64 go build -o server.exe  ./cmd/server
GOOS=windows GOARCH=amd64 go build -o updater.exe ./cmd/updater

# Local sanity check
go build ./...
```

### Run locally (macOS / Linux)

`go run` puts the compiled binary in a temp dir, so the config lookup that
expects `config.json` next to the binary won't find it. Point
`BOILERPLATE_CONFIG` at the file explicitly:

```bash
cp config.example.json config.json
BOILERPLATE_CONFIG=$(pwd)/config.json go run ./cmd/server -action run
curl localhost:7000/health
```

The service wrapper falls back to foreground mode outside Windows; `Ctrl+C`
stops it. Logs go to stdout.

### Install on Windows (production)

1. Build both binaries with an ldflag version stamp (CI does this for you):
   ```bash
   go build -ldflags "-X main.version=v0.1.0" -o server.exe  ./cmd/server
   go build -ldflags "-X main.version=v0.1.0" -o updater.exe ./cmd/updater
   ```
2. Drop `server.exe`, `updater.exe`, and `config.json` next to each other
   (e.g. `C:\golang-boilerplate\`).
3. Register and start:
   ```cmd
   server.exe -action install
   server.exe -action start
   ```
4. Other actions: `stop`, `uninstall`, `run` (foreground).

`server.exe -action install` also configures Windows Service Recovery actions
to restart the service three times with a 10-second delay between attempts.

## Configuration

`config.json` lives next to the binary on Windows. On dev machines set
`BOILERPLATE_CONFIG` to point at it.

```json
{
  "port": 7000,
  "api_key": "change-me",
  "auto_update_enabled": true
}
```

| Field                 | Default      | Notes                                                          |
| --------------------- | ------------ | -------------------------------------------------------------- |
| `port`                | `7000`       | HTTP listen port.                                              |
| `api_key`             | `change-me`  | Required in `X-API-Key` header (or `?api_key=` query) on auth-gated routes. |
| `auto_update_enabled` | `true`       | Kill switch for the updater poll loop. Absent means default-on. |

## HTTP endpoints

| Method | Path      | Auth     | Description                                                                 |
| ------ | --------- | -------- | --------------------------------------------------------------------------- |
| `GET`  | `/health` | none     | `{ version, auto_update_enabled, started_at, uptime, last_poll_at, last_update }`. Used by the updater to confirm the new binary booted. |
| `GET`  | `/hello`  | API key  | Example handler returning `{ message, version }`.                            |
| `POST` | `/update` | API key  | Forces an immediate poll + apply (skips the nightly window).                 |
| `POST` | `/config` | API key  | Replaces `config.json` and reloads (port change requires a service restart). |

## Auto-update

`server.exe` polls the configured manifest URL hourly. When a newer version
is available, it downloads `server.exe` + `updater.exe` (and their sha256
sidecars) into `<exeDir>/.update/<version>/`, verifies integrity, and waits
for the nightly window **02:00-05:00 local time** before handing off to
`updater.exe`. The kill switch (`auto_update_enabled: false`) pauses polling
without redeploying. `POST /update` runs the same flow immediately.

### Flow

1. Hourly: `server.exe` does `GET <manifest-url>` (default points at this
   repo's latest GitHub release).
2. If the embedded `version` (set via `-X main.version=...` at build time) is
   older, the new `server.exe` + `updater.exe` + sha256 files land in
   `<exeDir>/.update/<version>/` and get checksum-verified.
3. Inside the window (or on `POST /update`), `server.exe` spawns the staged
   `updater.exe` detached and lets the SCM stop it.
4. `updater.exe`:
   - `sc stop <service>` (idempotent),
   - renames the live binaries to `*.old` and copies the new ones in,
   - writes a sentinel file with TTL,
   - `sc start <service>`,
   - polls `GET /health` for up to 60 s waiting for
     `version == toVersion`,
   - on success: cleans up `.old`, writes `status.json: ok`;
   - on failure: restores `.old` to live names, renames the bad binary to
     `*.failed-<version>`, restarts, writes `status.json: rolled_back`.

### Configuring the manifest source

The default manifest URL is:

```
https://github.com/tqrcisio/golang-boilerplate/releases/latest/download/manifest.json
```

In a fork, override either at build time or via env var:

```bash
# Build-time (best — bakes the value into the binary)
go build -ldflags \
  "-X main.version=v0.1.0 \
   -X github.com/tqrcisio/golang-boilerplate/internal/updater.DefaultReleaseRepo=youruser/yourrepo" \
  -o server.exe ./cmd/server

# Runtime override (handy for staging)
BOILERPLATE_RELEASE_REPO=youruser/yourrepo server.exe -action run

# Or point at any other manifest entirely
BOILERPLATE_MANIFEST_URL=https://example.com/manifest.json server.exe -action run
```

`manifest.json` shape (CI generates this):

```json
{
  "version": "v0.1.0",
  "released_at": "2026-05-18T00:00:00Z",
  "download_url":         "https://github.com/owner/repo/releases/download/v0.1.0/server.exe",
  "sha256_url":           "https://github.com/owner/repo/releases/download/v0.1.0/server.exe.sha256",
  "updater_download_url": "https://github.com/owner/repo/releases/download/v0.1.0/updater.exe",
  "updater_sha256_url":   "https://github.com/owner/repo/releases/download/v0.1.0/updater.exe.sha256"
}
```

### Dev builds skip auto-update

Builds without the ldflag default to `version = "dev"` and never poll. Set the
ldflag whenever you want to exercise the update path locally.

### First-time installs without `updater.exe`

If a host is running an old `server.exe` that predates this pattern, it will
download a matching `updater.exe` for its own version from the releases
endpoint on first boot. Auto-update stays paused until that succeeds.

## Releasing

Pushes to `main` trigger the release workflow:

1. `go-semantic-release` reads conventional commits since the last tag,
   computes the next version, creates the tag, and writes release notes.
2. The workflow builds `server.exe` + `updater.exe` with the version embedded
   via `-ldflags "-X main.version=v$VERSION"`.
3. SHA256 sidecars and `manifest.json` are generated.
4. All five files (`server.exe`, `server.exe.sha256`, `updater.exe`,
   `updater.exe.sha256`, `manifest.json`) plus `config.example.json` are
   uploaded as release assets.
5. The release body is rewritten with stable download links.

Because GitHub redirects `releases/latest/download/<asset>` to the actual
latest release, the manifest URL stays stable across releases — no extra
hosting needed.

### Commit conventions

```
feat:     new user-facing feature  → minor bump
fix:      bug fix                  → patch bump
feat!:    breaking change          → major bump
chore:    no release
docs:     no release
refactor: no release
```

### Cutting the first release

Push at least one `feat:` or `fix:` commit and merge to `main`. The workflow
creates `v0.1.0` (or whatever semrel computes) and uploads the assets.

## Dependencies

- [`github.com/kardianos/service`](https://github.com/kardianos/service) — cross-platform service wrapper (SCM on Windows, foreground elsewhere).
- [`github.com/kardianos/osext`](https://github.com/kardianos/osext) — resolves the directory of the running executable.

Both are pure Go and CGO-free. Builds happily on `ubuntu-latest` runners.

## License

MIT. See [LICENSE](./LICENSE).

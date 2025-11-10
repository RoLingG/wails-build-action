# Setup Go, Deno, Wails (sub-action)

Purpose
- Sets up toolchains needed for Wails builds:
  - Go (with optional caching)
  - Optional Garble (when obfuscating)
  - Optional Deno (ENV-first configuration)
  - Wails CLI
  - Installs `gon` on macOS for signing

Inputs
- `go-version` (default `1.23`)
- `build-cache` (default `true`) — enable Go modules cache
- `build-obfuscate` (default `false`) — installs Garble when true
- `deno-build` (default `''`) — build command; example: `deno task build`
- `deno-version` (default `v1.20.x`)
- `deno-working-directory` (default `.`)
- `wails-version` (default `latest`)
- `wails-dev-build` (default `false`) — when `true`, skips installing Go and Wails CLI

Deno via environment variables (ENV-first)
- Precedence: environment variables > inputs > defaults.
- Supported envs:
  - `DENO_ENABLE` — `true`/`1`/`yes`/`on` explicitly enables Deno
  - `DENO_BUILD` — full command to run (e.g., `deno task build`)
  - `DENO_VERSION` — e.g., `v1.44.x`
  - `DENO_WORKDIR` — working directory for Deno (default `.`)
  - Pass-through: `DENO_AUTH_TOKEN`, `DENO_DIR`, proxies, etc.

Behavior
- Resolves Deno configuration and exposes internal outputs: `ENABLED`, `BUILD`, `VERSION`, `WORKDIR`.
- Sets up Deno only when `ENABLED == '1'`.
- Runs the Deno command when `BUILD` is not empty.

Usage
```yaml
- name: Setup toolchains
  uses: snider/build-action/actions/setup@v3
  with:
    go-version: '1.23'
    build-cache: 'true'
    build-obfuscate: 'false'
    wails-version: 'latest'
  env:
    # Optional Deno via env
    DENO_ENABLE: 'true'
    DENO_VERSION: 'v1.44.x'
    DENO_WORKDIR: 'frontend'
    DENO_BUILD: 'deno task build'
```

Notes
- On macOS, `gon` is installed for later signing steps. No-op on other OSes.
- If you do not need Deno, leave envs and inputs empty — it will be skipped.
- If `wails-dev-build` is `true`, ensure `wails` is already available on PATH.

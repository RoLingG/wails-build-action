# Wails v2 (full pipeline)

Purpose
- Run the entire Wails v2 build pipeline in one step: Linux discovery, build option computation, toolchain setup (Go/Deno/Wails), build, optional signing, and packaging.

Usage (recommended)
```yaml
- name: Build Wails v2
  uses: snider/build-action/actions/wails2@v3
  with:
    build-name: wailsApp
    build-platform: linux/amd64 # or windows/amd64, darwin/universal
    # Optional features
    build-obfuscate: 'false'
    nsis: 'false'
    sign: 'false'
    package: 'true'
    wails-version: 'latest'
```

Notes
- Deno is optional and can be configured via environment variables (ENV-first): `DENO_ENABLE`, `DENO_BUILD`, `DENO_VERSION`, `DENO_WORKDIR`.
- On Linux, the action detects Ubuntu 20.04/22.04/24.04 and installs matching WebKitGTK packages; Ubuntu 24.04 implies `-tags webkit2_41` when appropriate.
- macOS signing and notarization only occur on tag builds when certs/passwords are provided.
- This sub-action is a convenience wrapper that delegates to the underlying sub-actions in this repository.

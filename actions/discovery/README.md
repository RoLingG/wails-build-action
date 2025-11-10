# Discovery (sub-action)

Purpose
- Detects OS and CPU architecture across all runners.
- On Linux, detects Ubuntu distro and installs required GTK/WebKit packages for Wails builds.
- Exposes useful repository/ref metadata (REF, TAG, IS_TAG, SHA, SHORT_SHA, REPO, OWNER) for later steps such as packaging.

Outputs
- `OS` — `Linux`, `macOS`, or `Windows`.
- `ARCH` — CPU architecture (e.g., `x86_64`/`amd64`/`arm64`).
- `DISTRO` — Ubuntu version string like `20.04`, `22.04`, or `24.04` (Linux only, empty otherwise).
- `REF` — Full Git ref (e.g., `refs/heads/main`, `refs/tags/v1.2.3`).
- `TAG` — Tag name when on a tag ref; empty otherwise.
- `IS_TAG` — `1` if ref is a tag, else `0`.
- `SHA` — Full commit SHA.
- `SHORT_SHA` — First 7 characters of the commit SHA.
- `REPO` — `owner/repo`.
- `OWNER` — Repository owner.

Usage
```yaml
- name: Discovery
  id: disc
  uses: snider/build-action/actions/discovery@v3
# Later examples:
#   ${{ steps.disc.outputs.OS }}
#   ${{ steps.disc.outputs.ARCH }}
#   ${{ steps.disc.outputs.DISTRO }}
#   ${{ steps.disc.outputs.TAG }}
```

Notes
- Runs on all OSes; only installs packages on Linux.
- Linux support currently covers Ubuntu 20.04, 22.04, and 24.04 (24.04 uses `libwebkit2gtk-4.1-dev`).
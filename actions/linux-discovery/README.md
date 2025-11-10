# Linux Discovery (sub-action)

Purpose
- Detects Ubuntu version and installs required GTK/WebKit packages for Wails builds.
- Outputs the detected distro version so other steps can adapt (e.g., add `webkit2_41` on 24.04).

Outputs
- `DISTRO` — Ubuntu version string like `20.04`, `22.04`, or `24.04`.

Usage
```yaml
- name: Linux Discovery
  id: linux
  uses: snider/build-action/actions/linux-discovery@v3
# Later: ${{ steps.linux.outputs.DISTRO }}
```

Notes
- Runs only on Linux runners. Will `apt-get install` necessary GTK/WebKit packages.
- Currently supports Ubuntu 20.04, 22.04, and 24.04. Other versions will fail fast with a clear message.

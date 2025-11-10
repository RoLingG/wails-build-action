# Package & Release (sub-action)

Purpose
- Uploads build artifacts from standard Wails output directories and publishes a GitHub Release on tag builds.

What it does
- Uses `actions/upload-artifact@v4` to upload files from `*/bin/` (Linux/macOS) and `*\bin\*` (Windows).
- If the workflow is running on a tag (`refs/tags/*`) and packaging is enabled, it attaches all `*/bin/*` files to a GitHub Release using `softprops/action-gh-release@v1`.

Inputs
- `package` (default `true`) — enable/disable uploads and release publishing.
- `build-name` (required) — used for the artifact name only (does not filter files).

Usage
```yaml
- name: Package & Release
  uses: snider/build-action/actions/package@v3
  with:
    package: 'true'
    build-name: 'wailsApp'
```

Notes
- To publish a release, push a tag (e.g., `v1.2.3`). On non-tag builds, only artifacts are uploaded.
- Adjust paths in your own workflow if your build layout differs from `*/bin/*`.

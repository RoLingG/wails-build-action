# Sign & Notarize macOS (sub-action)

Purpose
- Imports macOS signing certificates, signs your `.app` using `gon`, zips the `.app`, optionally builds a signed or unsigned `.pkg`, and notarizes on tag builds.

When it runs
- Only meaningful on `macOS` runners.
- Most signing steps run only when `sign != 'false'` AND the ref is a tag (i.e., `startsWith(github.ref, 'refs/tags/')`).

Inputs
- `sign` (default `false`) — enable/disable macOS signing and notarization.
- `app-working-directory` (default `.`) — Wails project dir.
- `build-name` (required) — the app bundle name (e.g., `wailsApp`).
- `sign-macos-apple-password` — Apple ID app-specific password (used by `gon`).
- `sign-macos-app-id` — Developer ID Application subject (used by `gon`).
- `sign-macos-app-cert` — Base64-encoded `.p12` (Developer ID Application).
- `sign-macos-app-cert-password` — Password for the above `.p12`.
- `sign-macos-installer-id` — Developer ID Installer subject (optional).
- `sign-macos-installer-cert` — Base64-encoded `.p12` (Installer).
- `sign-macos-installer-cert-password` — Password for the installer cert.

Required project files
- `build/darwin/gon-sign.json` — configuration for signing the `.app`.
- `build/darwin/gon-notarize.json` — configuration for notarizing `.pkg` and `.app.zip`.

Usage
```yaml
- name: Sign & Notarize (macOS)
  uses: snider/build-action/actions/sign-macos@v3
  with:
    sign: 'true'
    app-working-directory: '.'
    build-name: 'wailsApp'
    sign-macos-apple-password: ${{ secrets.APPLE_PASSWORD }}
    sign-macos-app-id: ${{ secrets.MACOS_DEVELOPER_CERT_ID }}
    sign-macos-app-cert: ${{ secrets.MACOS_DEVELOPER_CERT }}
    sign-macos-app-cert-password: ${{ secrets.MACOS_DEVELOPER_CERT_PASSWORD }}
    sign-macos-installer-id: ${{ secrets.MACOS_INSTALLER_CERT_ID }}
    sign-macos-installer-cert: ${{ secrets.MACOS_INSTALLER_CERT }}
    sign-macos-installer-cert-password: ${{ secrets.MACOS_INSTALLER_CERT_PASSWORD }}
```

Notes
- This action uses `Apple-Actions/import-codesign-certs@v1` to import P12s.
- `ditto` is used to zip the `.app` regardless of signing (useful for distribution and notarization).
- Installer creation: signed when `sign == 'true'` and an installer ID is provided; otherwise builds an unsigned `.pkg` on tags.
- Ensure `gon` is installed (the `setup` sub-action installs it automatically on macOS).

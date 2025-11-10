# Sign Windows binaries (sub-action)

Purpose
- Signs the built Windows `.exe` and the NSIS installer using your provided code-signing certificate (PFX, base64-encoded) and password.

When it runs
- Only meaningful on `Windows` runners.
- Runs when `sign != 'false'` AND a certificate is provided via `sign-windows-cert`.

Inputs
- `sign` (default `false`) — enable/disable Windows signing.
- `app-working-directory` (default `.`) — Wails project dir.
- `build-name` (required) — base name of the built artifacts.
- `sign-windows-cert` — Base64-encoded PFX contents.
- `sign-windows-cert-password` — Password for the PFX.

What gets signed
- `build/bin/<build-name>.exe`
- `build/bin/<build-name>-amd64-installer.exe`

Usage
```yaml
- name: Sign Windows
  uses: snider/build-action/actions/sign-windows@v3
  with:
    sign: 'true'
    app-working-directory: '.'
    build-name: 'wailsApp'
    sign-windows-cert: ${{ secrets.WIN_CERT_PFX_BASE64 }}
    sign-windows-cert-password: ${{ secrets.WIN_CERT_PASSWORD }}
```

Notes
- Uses `signtool.exe` from the Windows 10 SDK on the runner.
- Ensure your workflow has built the artifacts before this step.
- If you also build 32-bit or alternative installer names, adjust the paths or extend the action as needed.

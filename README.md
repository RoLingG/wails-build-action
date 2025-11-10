20/02/2025 - Wails Version 2.10.0 is problematic, please use `wails-version: "v2.9.0"` & report the bug if you get issues, tyvm <3

07/02/2025 - Repo renamed: please use `snider/build-action@v3` and consider starring the repo to get updates; the readme refers to v3.

# snider/build-action@v3
GitHub action to build Wails.io: the action will install GoLang, optionally Deno, and run a build. It now uses a modern, modular structure split into reusable sub-actions and an optional reusable workflow.
This will be used on a [Wails.io](https://wails.io) v2 project.

By default, the action will build and upload the results to Git Hub; on a tagged build, it will also upload to the release.

# Default build
```yaml
- uses: snider/build-action@v3
  with:
    build-name: wailsApp
    build-platform: linux/amd64
```

## Build with No uploading

```yaml
- uses: snider/build-action@v3
  with:
    build-name: wailsApp
    build-platform: linux/amd64
    package: false
```
## GitHub Action Options

| Name                                 | Default              | Description                                        |
|--------------------------------------|----------------------|----------------------------------------------------|
| `build-name`                         | none, required input | The name of the binary                             |
| `build-obfuscate`                    | `false`              | Obfuscate the binary                               |
| `build`                              | `true`               | Runs `wails build` on your source                  |
| `nsis`                               | `true`               | Runs `wails build` with or without -nsis           |
| `sign`                               | `false`              | After build, signs and creates signed installers   |
| `package`                            | `true`               | Upload workflow artifacts & publish release on tag |
| `build-platform`                     | `darwin/universal`   | Platform to build for                              |
| `build-tags`                         | ''                   | Build tags to pass to Go compiler. Must be quoted. |
| `wails-version`                      | `latest`             | Wails version to use                               |
| `wails-build-webview2`               | `download`           | Webview2 installing [download,embed,browser,error] |
| `go-version`                         | `1.18`               | Version of Go to use                               |
| `node-version`                       | `16.x`               | Node js version                                    |
| `deno-build`                         | ''                   | Deno compile command                               |
| `deno-working-directory`             | `.`                  | Working directory of your [Deno](https://deno.land/) server|
| `deno-version`                       | `v1.20.x`            | Deno version to use                                |
| `sign-macos-app-id`                  | ''                   | ID of the app signing cert                         |
| `sign-macos-apple-password`          | ''                   | MacOS Apple password                               |
| `sign-macos-app-cert`                | ''                   | MacOS Application Certificate                      |
| `sign-macos-app-cert-password`       | ''                   | MacOS Application Certificate Password             |
| `sign-macos-installer-id`            | ''                   | MacOS Installer Certificate id                     |
| `sign-macos-installer-cert`          | ''                   | MacOS Installer Certificate                        |
| `sign-macos-installer-cert-password` | ''                   | MacOS Installer Certificate Password               |
| `sign-windows-cert`                  | ''                   | Windows Signing Certificate                        |
| `sign-windows-cert-password`         | ''                   | Windows Signing Certificate Password               |



## Example Build

```yaml
name: Wails build

on: [push, pull_request]

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        build: [
          {name: wailsTest, platform: linux/amd64, os: ubuntu-latest},
          {name: wailsTest, platform: windows/amd64, os: windows-latest},
          {name: wailsTest, platform: darwin/universal, os: macos-latest}
        ]
    runs-on: ${{ matrix.build.os }}
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: recursive
      - uses: snider/build-action@v3
        with:
          build-name: ${{ matrix.build.name }}
          build-platform: ${{ matrix.build.platform }}
          build-obfuscate: true
```

## MacOS Code Signing

You need to make two gon configuration files, this is because we need to sign and notarize the .app before making an installer with it.

```yaml
  - uses: snider/wails-build-action@v3
    with:
      build-name: wailsApp
      sign: true
      build-platform: darwin/universal
      sign-macos-apple-password: ${{ secrets.APPLE_PASSWORD }}
      sign-macos-app-id: ${{ secrets.MACOS_DEVELOPER_CERT_ID }}
      sign-macos-app-cert: ${{ secrets.MACOS_DEVELOPER_CERT }}
      sign-macos-app-cert-password: ${{ secrets.MACOS_DEVELOPER_CERT_PASSWORD }}
      sign-macos-installer-id: ${{ secrets.MACOS_INSTALLER_CERT_ID }}
      sign-macos-installer-cert: ${{ secrets.MACOS_INSTALLER_CERT }}
      sign-macos-installer-cert-password: ${{ secrets.MACOS_INSTALLER_CERT_PASSWORD }}
```

`build/darwin/gon-sign.json`
```json
{
  "source" : ["./build/bin/wailsApp.app"],
  "bundle_id" : "com.wails.app",
  "apple_id": {
    "username": "username",
    "password": "@env:APPLE_PASSWORD"
  },
  "sign" :{
    "application_identity" : "Developer ID Application: XXXXXXXX (XXXXXX)",
    "entitlements_file": "./build/darwin/entitlements.plist"
  },
  "dmg" :{
    "output_path":  "./build/bin/wailsApp.dmg",
    "volume_name":  "Lethean"
  }
}
```
`build/darwin/gon-notarize.json`
```json
{
  "notarize": [{
    "path": "./build/bin/wailsApp.pkg",
    "bundle_id": "com.wails.app",
    "staple": true
  },{
    "path": "./build/bin/wailsApp.app.zip",
    "bundle_id": "com.wails.app",
    "staple": false
  }],
  "apple_id": {
    "username": "USER name",
    "password": "@env:APPLE_PASSWORD"
  }
}
```
`build/darwin/entitlements.plist`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.security.app-sandbox</key>
  <true/>
  <key>com.apple.security.network.client</key>
  <true/>
  <key>com.apple.security.network.server</key>
  <true/>
  <key>com.apple.security.files.user-selected.read-write</key>
  <true/>
</dict>
</plist>
```


## Configure Deno via environment variables (optional)

Deno is not required. If you want to run a Deno build/asset step before Wails, you can configure it entirely via env vars — no `deno-*` inputs are needed.

Precedence used by the action (inside `actions/setup`):
- Environment variables > action inputs > defaults.
- If nothing is provided, Deno is skipped.

Supported variables:
- `DENO_ENABLE` — `true`/`1`/`yes`/`on` explicitly enables Deno even without a build command.
- `DENO_BUILD` — full command to run (e.g., `deno task build`, `deno run -A build.ts`).
- `DENO_VERSION` — e.g., `v1.44.x`.
- `DENO_WORKDIR` — working directory for the Deno command (default `.`).
- Pass-throughs (used by Deno if present): `DENO_AUTH_TOKEN`, `DENO_DIR`, `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`, etc.

Example (job-level env):
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DENO_ENABLE: 'true'
      DENO_VERSION: 'v1.44.x'
      DENO_WORKDIR: 'frontend'
      DENO_BUILD: 'deno task build'
    steps:
      - uses: actions/checkout@v4
      - uses: snider/build-action@v3
        with:
          build-name: wailsApp
          build-platform: linux/amd64
```

Using `$GITHUB_ENV` in a prior step:
```yaml
- name: Configure Deno via $GITHUB_ENV
  run: |
    echo "DENO_ENABLE=true" >> "$GITHUB_ENV"
    echo "DENO_VERSION=v1.44.x" >> "$GITHUB_ENV"
    echo "DENO_WORKDIR=frontend" >> "$GITHUB_ENV"
    echo "DENO_BUILD=deno task build" >> "$GITHUB_ENV"
- uses: snider/build-action@v3
  with:
    build-name: wailsApp
    build-platform: linux/amd64
```

Secrets example (private modules):
```yaml
env:
  DENO_AUTH_TOKEN: ${{ secrets.DENO_AUTH_TOKEN }}
```

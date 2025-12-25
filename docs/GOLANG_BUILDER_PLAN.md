# GoLang Binary Builder Plan

## Overview

This document outlines a plan to refactor the build action to use **GoTask (go-task/task)** as the build orchestrator instead of inline shell scripts in GitHub Actions. This approach provides:

- **Cross-platform consistency**: Taskfiles work identically on Linux, macOS, and Windows
- **Local development parity**: Developers can run the same tasks locally that run in CI
- **Cleaner separation**: GitHub Actions handle CI concerns (checkout, caching, artifacts); GoTask handles build logic
- **Extensibility**: Easy to add new stacks (Wails3, pure Go CLI, Docker, etc.)

## Primary Goals

1. **Wails 2 builds** - Full support (current functionality preserved)
2. **Wails 3 builds** - New support for Wails v3 architecture

## Stretch Goals

3. Pure Go CLI/binary builds (no Wails)
4. Docker image builds
5. C++ builds (via Conan/CMake)
6. PHP builds (Composer-based)

---

## Architecture

### Current v3 Architecture (Shell-based)

```
action.yml (root)
├── actions/discovery/action.yml     # Detect OS/ARCH/project-type
├── actions/action.yml               # Directory orchestrator (stack router)
│   └── actions/build/wails2/action.yml  # Wails2 full pipeline
│       ├── actions/options/action.yml   # Compute build flags
│       ├── actions/setup/action.yml     # Install toolchains
│       ├── actions/build/wails2/build/  # Run wails build
│       └── actions/sign/action.yml      # Code signing
└── actions/package/action.yml       # Artifacts & releases
```

### Proposed GoTask Architecture

```
action.yml (root)
├── actions/discovery/action.yml     # Keep: Detect OS/ARCH/project-type
├── actions/setup/gotask/action.yml  # NEW: Install go-task
├── actions/action.yml               # Orchestrator runs: task build:$STACK
│   └── Taskfile.yml                 # Master taskfile includes stack taskfiles
│       ├── tasks/wails2.yml         # Wails 2 build tasks
│       ├── tasks/wails3.yml         # Wails 3 build tasks
│       ├── tasks/go-cli.yml         # Pure Go CLI builds
│       └── tasks/common.yml         # Shared tasks (setup, sign, package)
└── actions/package/action.yml       # Keep: Artifacts & releases
```

---

## Taskfile Structure

### Root Taskfile.yml

```yaml
version: '3'

includes:
  common: ./tasks/common.yml
  wails2: ./tasks/wails2.yml
  wails3: ./tasks/wails3.yml
  go: ./tasks/go-cli.yml

vars:
  BUILD_NAME: '{{.BUILD_NAME | default "app"}}'
  BUILD_PLATFORM: '{{.BUILD_PLATFORM | default "linux/amd64"}}'
  BUILD_OUTPUT: '{{.BUILD_OUTPUT | default "build/bin"}}'

tasks:
  default:
    desc: "Show available tasks"
    cmds:
      - task --list

  build:
    desc: "Auto-detect stack and build"
    cmds:
      - task: "build:{{.STACK | default \"wails2\"}}"
    vars:
      STACK:
        sh: |
          if [ -f "wails.json" ]; then
            if grep -q '"wailsVersion": "v3' wails.json 2>/dev/null; then
              echo "wails3"
            else
              echo "wails2"
            fi
          elif [ -f "go.mod" ] && [ -f "main.go" ]; then
            echo "go"
          else
            echo "wails2"
          fi

  build:wails2:
    desc: "Build Wails 2 application"
    cmds:
      - task: wails2:build

  build:wails3:
    desc: "Build Wails 3 application"
    cmds:
      - task: wails3:build

  build:go:
    desc: "Build Go CLI/binary"
    cmds:
      - task: go:build
```

### tasks/wails2.yml

```yaml
version: '3'

vars:
  WAILS_VERSION: '{{.WAILS_VERSION | default "latest"}}'
  WEBVIEW2: '{{.WEBVIEW2 | default "download"}}'
  BUILD_TAGS: '{{.BUILD_TAGS | default ""}}'
  OBFUSCATE: '{{.OBFUSCATE | default "false"}}'
  NSIS: '{{.NSIS | default "false"}}'

tasks:
  setup:
    desc: "Install Wails v2 CLI"
    cmds:
      - go install github.com/wailsapp/wails/v2/cmd/wails@{{.WAILS_VERSION}}
    status:
      - which wails

  setup:garble:
    desc: "Install Garble for obfuscation"
    cmds:
      - go install mvdan.cc/garble@latest
    status:
      - which garble

  build:
    desc: "Build Wails 2 application"
    deps: [setup]
    cmds:
      - |
        BUILD_OPTS=""
        {{if eq .OBFUSCATE "true"}}BUILD_OPTS="$BUILD_OPTS -obfuscated"{{end}}
        {{if .BUILD_TAGS}}BUILD_OPTS="$BUILD_OPTS -tags {{.BUILD_TAGS}}"{{end}}
        {{if eq .NSIS "true"}}BUILD_OPTS="$BUILD_OPTS -nsis"{{end}}
        wails build --platform {{.BUILD_PLATFORM}} -webview2 {{.WEBVIEW2}} -o {{.BUILD_NAME}} $BUILD_OPTS
    generates:
      - "{{.BUILD_OUTPUT}}/*"

  build:all:
    desc: "Build for all platforms"
    cmds:
      - task: build
        vars: { BUILD_PLATFORM: "linux/amd64" }
      - task: build
        vars: { BUILD_PLATFORM: "windows/amd64" }
      - task: build
        vars: { BUILD_PLATFORM: "darwin/universal" }
```

### tasks/wails3.yml

```yaml
version: '3'

vars:
  WAILS3_VERSION: '{{.WAILS3_VERSION | default "latest"}}'

tasks:
  setup:
    desc: "Install Wails v3 CLI"
    cmds:
      - go install github.com/wailsapp/wails/v3/cmd/wails3@{{.WAILS3_VERSION}}
    status:
      - which wails3

  build:
    desc: "Build Wails 3 application"
    deps: [setup]
    cmds:
      - |
        # Wails 3 uses a different build command structure
        # Platform is specified differently in v3
        GOOS={{.GOOS}} GOARCH={{.GOARCH}} wails3 build -o {{.BUILD_NAME}}
    vars:
      GOOS:
        sh: echo "{{.BUILD_PLATFORM}}" | cut -d'/' -f1 | sed 's/darwin/darwin/;s/linux/linux/;s/windows/windows/'
      GOARCH:
        sh: echo "{{.BUILD_PLATFORM}}" | cut -d'/' -f2
    generates:
      - "{{.BUILD_OUTPUT}}/*"

  dev:
    desc: "Run Wails 3 in dev mode"
    deps: [setup]
    cmds:
      - wails3 dev
```

### tasks/common.yml

```yaml
version: '3'

tasks:
  setup:go:
    desc: "Verify Go installation"
    cmds:
      - go version
    preconditions:
      - sh: which go
        msg: "Go is not installed"

  setup:node:
    desc: "Install frontend dependencies"
    dir: "{{.FRONTEND_DIR | default \"frontend\"}}"
    cmds:
      - npm ci --prefer-offline
    sources:
      - package-lock.json
    generates:
      - node_modules/**/*
    status:
      - test -d node_modules

  clean:
    desc: "Clean build artifacts"
    cmds:
      - rm -rf build/bin/*
      - rm -rf dist/*
    ignore_error: true

  lint:
    desc: "Run linters"
    cmds:
      - go vet ./...
      - |
        if which golangci-lint >/dev/null 2>&1; then
          golangci-lint run
        fi

  test:
    desc: "Run tests"
    cmds:
      - go test -v ./...

  sign:macos:
    desc: "Sign macOS application"
    platforms: [darwin]
    cmds:
      - |
        if [ -n "$SIGN_MACOS_APP_ID" ]; then
          codesign --deep --force --options runtime \
            --sign "$SIGN_MACOS_APP_ID" \
            "{{.BUILD_OUTPUT}}/{{.BUILD_NAME}}.app"
        else
          echo "Skipping signing: SIGN_MACOS_APP_ID not set"
        fi

  sign:windows:
    desc: "Sign Windows executable"
    platforms: [windows]
    cmds:
      - |
        if [ -n "$SIGN_WINDOWS_CERT" ]; then
          # Windows signing via signtool
          echo "Windows signing configured"
        else
          echo "Skipping signing: SIGN_WINDOWS_CERT not set"
        fi

  package:
    desc: "Create distributable package"
    cmds:
      - |
        OS=$(echo "{{.BUILD_PLATFORM}}" | cut -d'/' -f1)
        ARCH=$(echo "{{.BUILD_PLATFORM}}" | cut -d'/' -f2)
        ARTIFACT_NAME="{{.BUILD_NAME}}_${OS}_${ARCH}"

        mkdir -p dist
        case $OS in
          darwin)
            ditto -c -k --keepParent "{{.BUILD_OUTPUT}}/{{.BUILD_NAME}}.app" "dist/${ARTIFACT_NAME}.zip"
            ;;
          windows)
            zip -r "dist/${ARTIFACT_NAME}.zip" "{{.BUILD_OUTPUT}}/{{.BUILD_NAME}}.exe"
            ;;
          linux)
            tar -czvf "dist/${ARTIFACT_NAME}.tar.gz" -C "{{.BUILD_OUTPUT}}" "{{.BUILD_NAME}}"
            ;;
        esac
```

---

## GitHub Action Integration

### actions/setup/gotask/action.yml

```yaml
name: "Setup GoTask"
description: "Install go-task/task CLI"
inputs:
  version:
    description: "Task version to install"
    required: false
    default: "latest"
runs:
  using: "composite"
  steps:
    - name: Install Task (Linux/macOS)
      if: runner.os != 'Windows'
      shell: bash
      run: |
        sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b ~/.local/bin
        echo "$HOME/.local/bin" >> $GITHUB_PATH

    - name: Install Task (Windows)
      if: runner.os == 'Windows'
      shell: powershell
      run: |
        choco install go-task -y

    - name: Verify Task installation
      shell: bash
      run: task --version
```

### Modified actions/action.yml (Orchestrator)

```yaml
name: "Directory Orchestrator"
description: "Runs GoTask with detected stack"
inputs:
  build:
    required: false
    default: "true"
  build-name:
    required: true
  build-platform:
    required: false
    default: "darwin/universal"
  app-working-directory:
    required: false
    default: "."
  STACK:
    required: false
    default: ""

runs:
  using: "composite"
  steps:
    - name: Setup GoTask
      uses: ./actions/setup/gotask

    - name: Determine stack
      id: stack
      shell: bash
      working-directory: ${{ inputs.app-working-directory }}
      run: |
        if [ -n "${{ inputs.STACK }}" ]; then
          echo "STACK=${{ inputs.STACK }}" >> "$GITHUB_OUTPUT"
        elif [ -f "wails.json" ]; then
          if grep -q '"wailsVersion": "v3' wails.json 2>/dev/null; then
            echo "STACK=wails3" >> "$GITHUB_OUTPUT"
          else
            echo "STACK=wails2" >> "$GITHUB_OUTPUT"
          fi
        elif [ -f "go.mod" ]; then
          echo "STACK=go" >> "$GITHUB_OUTPUT"
        else
          echo "STACK=wails2" >> "$GITHUB_OUTPUT"
        fi

    - name: Run build task
      if: inputs.build == 'true'
      shell: bash
      working-directory: ${{ inputs.app-working-directory }}
      env:
        BUILD_NAME: ${{ inputs.build-name }}
        BUILD_PLATFORM: ${{ inputs.build-platform }}
      run: |
        task build:${{ steps.stack.outputs.STACK }}

outputs:
  SELECTED_STACK:
    description: "Stack that was built"
    value: ${{ steps.stack.outputs.STACK }}
```

---

## Wails 2 vs Wails 3 Detection

The system will auto-detect Wails version by examining `wails.json`:

| Indicator | Wails 2 | Wails 3 |
|-----------|---------|---------|
| `wails.json` schema | Has `name`, `frontend:install` | Has `wailsVersion: "v3..."` |
| CLI command | `wails build` | `wails3 build` |
| Output structure | `build/bin/{name}.app` | Different structure |
| Go module | `github.com/wailsapp/wails/v2` | `github.com/wailsapp/wails/v3` |

### Detection Logic

```bash
# In Taskfile or action
if [ -f "wails.json" ]; then
  if grep -q '"wailsVersion":\s*"v3' wails.json; then
    STACK="wails3"
  else
    STACK="wails2"
  fi
elif grep -q 'github.com/wailsapp/wails/v3' go.mod 2>/dev/null; then
  STACK="wails3"
elif grep -q 'github.com/wailsapp/wails/v2' go.mod 2>/dev/null; then
  STACK="wails2"
else
  STACK="go"  # fallback to pure Go
fi
```

---

## Implementation Phases

### Phase 1: Foundation (Primary Goal)

1. **Add GoTask setup action** (`actions/setup/gotask/action.yml`)
2. **Create root Taskfile.yml** with basic structure
3. **Migrate Wails 2 build logic** to `tasks/wails2.yml`
4. **Update orchestrator** to call `task build:wails2`
5. **Preserve all existing inputs/outputs** for backward compatibility

### Phase 2: Wails 3 Support (Primary Goal)

1. **Create `tasks/wails3.yml`** with Wails 3 build commands
2. **Add Wails 3 detection** to discovery and Taskfile
3. **Create TDD fixture** for Wails 3 (`tdd/wails3-root/`)
4. **Add CI tests** for Wails 3 builds

### Phase 3: Refactor Remaining Shell Logic

1. **Move signing logic** to `tasks/common.yml` (sign:macos, sign:windows)
2. **Move npm setup** to common tasks
3. **Simplify GitHub Actions** to thin wrappers around `task` commands

### Phase 4: Stretch Goals

1. **Pure Go CLI** (`tasks/go-cli.yml`)
2. **Docker builds** (`tasks/docker.yml`)
3. **C++ builds** (`tasks/cpp.yml`)
4. **PHP builds** (`tasks/php.yml`)

---

## File Changes Summary

### New Files

| File | Description |
|------|-------------|
| `Taskfile.yml` | Root taskfile with includes |
| `tasks/common.yml` | Shared tasks (setup, sign, package) |
| `tasks/wails2.yml` | Wails 2 build tasks |
| `tasks/wails3.yml` | Wails 3 build tasks |
| `tasks/go-cli.yml` | Pure Go CLI build tasks |
| `actions/setup/gotask/action.yml` | GoTask installation action |
| `tdd/wails3-root/` | Wails 3 test fixture |

### Modified Files

| File | Changes |
|------|---------|
| `actions/action.yml` | Add GoTask setup, call `task build:$STACK` |
| `actions/discovery/action.yml` | Add Wails 3 detection |
| `.github/workflows/ci.yml` | Add Wails 3 build tests |

### Deprecated (Phase 3)

| File | Status |
|------|--------|
| `actions/build/wails2/build/action.yml` | Replaced by `task wails2:build` |
| `actions/options/action.yml` | Logic moves to Taskfile vars |

---

## Environment Variables

All existing `WAILS_*` environment variables remain supported:

| Variable | Taskfile Var | Default |
|----------|--------------|---------|
| `WAILS_VERSION` | `WAILS_VERSION` | `latest` |
| `WAILS_GO_VERSION` | N/A (Go setup) | `1.23` |
| `WAILS_BUILD_TAGS` | `BUILD_TAGS` | `""` |
| `WAILS_OBFUSCATE` | `OBFUSCATE` | `false` |
| `WAILS_NSIS` | `NSIS` | `false` |
| `WAILS_WEBVIEW2` | `WEBVIEW2` | `download` |

New for Wails 3:

| Variable | Taskfile Var | Default |
|----------|--------------|---------|
| `WAILS3_VERSION` | `WAILS3_VERSION` | `latest` |

---

## Local Development Usage

With GoTask, developers can run the same builds locally:

```bash
# Install task (one-time)
brew install go-task  # macOS
# or: sh -c "$(curl --location https://taskfile.dev/install.sh)"

# Run builds
task build              # Auto-detect stack
task build:wails2       # Explicit Wails 2
task build:wails3       # Explicit Wails 3
task wails2:build:all   # Build all platforms

# Other tasks
task clean              # Clean build artifacts
task lint               # Run linters
task test               # Run tests
```

---

## Backward Compatibility

The action maintains full backward compatibility:

1. **Same inputs** - All existing action inputs work identically
2. **Same outputs** - Same artifacts, same naming conventions
3. **Same ENV mapping** - `WAILS_*` variables work as before
4. **Gradual migration** - Shell scripts coexist with Taskfiles during transition

---

## Next Steps

1. [ ] Create `actions/setup/gotask/action.yml`
2. [ ] Create `Taskfile.yml` and `tasks/` directory
3. [ ] Implement `tasks/wails2.yml` matching current behavior
4. [ ] Update orchestrator to use GoTask
5. [ ] Add Wails 3 detection and `tasks/wails3.yml`
6. [ ] Create Wails 3 TDD fixture
7. [ ] Update CI workflow with new tests
8. [ ] Document new local development workflow

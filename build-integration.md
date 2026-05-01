# MemScope Integration Guide

This guide covers practical integration for:

- CMake post-build and custom targets
- Make/Ninja invocation
- Vendor IDE external tools
- Headless CI usage

MemScope is CLI-first and non-interactive, so these patterns work locally and in CI.

## 1. CMake Integration

### 1.1 Add the helper module

```cmake
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
include(FirmwareFootprint)
```

### 1.2 Generate a stable `.map` file

For GNU-style toolchains (`gcc`, `clang` with GNU linker flags), add a deterministic map path:

```cmake
target_link_options(app PRIVATE
  "-Wl,-Map,$<TARGET_FILE_DIR:app>/$<TARGET_FILE_BASE_NAME:app>.map"
  "-Wl,--cref"
)
```

This keeps the map location predictable for post-build analysis.

### 1.3 Post-build report generation (`POST_BUILD`)

```cmake
set(app_map "$<TARGET_FILE_DIR:app>/$<TARGET_FILE_BASE_NAME:app>.map")

firmware_footprint_report(
  TARGET app
  ELF $<TARGET_FILE:app>
  MAP ${app_map}
  LINKER ${CMAKE_SOURCE_DIR}/linker.ld
  CONFIG ${CMAKE_SOURCE_DIR}/memscope.toml
  HTML ${CMAKE_BINARY_DIR}/reports/app.html
  JSON ${CMAKE_BINARY_DIR}/reports/app.json
  CSV  ${CMAKE_BINARY_DIR}/reports/app.csv
  STRICT
)
```

If MemScope exits non-zero, the build step fails, which is usually desired for CI gating.

### 1.4 Custom target workflow

Use this when you do not want every build to run analysis automatically:

```cmake
firmware_footprint_custom_target(
  NAME app_footprint
  ELF $<TARGET_FILE:app>
  MAP $<TARGET_FILE_DIR:app>/$<TARGET_FILE_BASE_NAME:app>.map
  LINKER ${CMAKE_SOURCE_DIR}/linker.ld
  JSON ${CMAKE_BINARY_DIR}/reports/app.json
  HTML ${CMAKE_BINARY_DIR}/reports/app.html
)
add_dependencies(app_footprint app)
```

Run manually:

```bash
cmake --build build --target app_footprint
```

### 1.5 Optional diff target

```cmake
firmware_footprint_diff_target(
  NAME app_footprint_diff
  CURRENT_JSON ${CMAKE_BINARY_DIR}/reports/app.json
  BASELINE_JSON ${CMAKE_SOURCE_DIR}/ci/baseline/app.json
  OUTPUT_JSON ${CMAKE_BINARY_DIR}/reports/app_diff.json
  OUTPUT_HTML ${CMAKE_BINARY_DIR}/reports/app_diff.html
)
```

## 2. Make / Ninja Integration

### 2.1 Direct command invocation

```bash
memscope --strict analyze \
  --elf build/app.elf \
  --map build/app.map \
  --linker linker.ld \
  --json build/reports/app.json \
  --html build/reports/app.html
```

### 2.2 Helper scripts

Scripts included in this repository:

- `scripts/memscope_post_build.sh`
- `scripts/memscope_post_build.ps1`

Example (`bash`):

```bash
scripts/memscope_post_build.sh \
  --elf build/app.elf \
  --map build/app.map \
  --linker linker.ld \
  --out-dir build/reports \
  --strict
```

Example (`PowerShell`):

```powershell
.\scripts\memscope_post_build.ps1 `
  -Elf build\app.elf `
  -Map build\app.map `
  -Linker linker.ld `
  -OutDir build\reports `
  -Strict
```

## 3. Vendor IDE External Tool Integration

Use your IDE "External Tools" or "Post-build step" feature to run MemScope.

### 3.1 Integration pattern

1. Configure linker to emit a `.map` file at a known path.
2. Add an external tool/post-build command that calls MemScope.
3. Pass artifact paths via IDE variables/macros.
4. Save reports under the build output folder.

### 3.2 Generic external-tool command

```text
memscope --strict analyze --elf <ELF_PATH> --map <MAP_PATH> --linker <LINKER_PATH> --json <JSON_OUT> --html <HTML_OUT>
```

Notes:

- Variable names differ by IDE; map your IDE macros to the placeholders above.
- Keep command non-interactive (no prompts) for reproducible builds.
- Start with `validate` during setup if paths are uncertain:
  `memscope validate --elf <ELF_PATH> --map <MAP_PATH> --linker <LINKER_PATH>`.

## 4. CI / Headless Behavior

MemScope is suitable for headless CI runners:

- Non-interactive command surface
- Local artifact analysis (no server dependency)
- Stable process exit codes

### 4.1 Exit codes

- `0`: success
- `1`: policy failure
- `2`: input/parsing/config failure
- `3`: license failure

### 4.2 CI command example

```bash
memscope --strict analyze \
  --config memscope.toml \
  --baseline-json ci/baseline/app.json \
  --json build/reports/app.json \
  --html build/reports/app.html
```

This is enough to gate pipeline execution and publish JSON/HTML as CI artifacts.

## 5. VS Code Wrapper Extension

A thin VS Code wrapper is available at:

- `packaging/vscode-extension/`

It calls the local CLI engine only (`memscope`), detects config in workspace root, and runs:

- `MemScope: Analyze Workspace`
- `MemScope: Validate Workspace`
- `MemScope: Open Last HTML Report`

Quick start:

```bash
cd packaging/vscode-extension
npm install
npm run compile
```

If your environment uses module invocation instead of a direct executable, configure:

- `memscope.cliExecutable = "python"`
- `memscope.cliBaseArgs = ["-m", "memscope"]`

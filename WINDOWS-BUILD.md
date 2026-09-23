# Windows desktop and execution plugin

Run in PowerShell with Bun 1.4.2 installed. From your checkout:

```powershell
git switch v2
git pull --ff-only origin v2
bun install
```

## Desktop (Windows x64, local production build)

Build the bundled CLI from this checkout, then build the desktop application and installer.

The CLI must use the `latest` channel. Do not build the CLI with `OPENCODE_CHANNEL=prod`.

In OpenCode v2.0.11, `prod` is treated as a custom CLI channel. This causes the background service to use a channel-specific registration file such as `service-prod.json`, while the production Desktop application expects the normal `service.json`. Using `latest` produces the correct production CLI behavior.

```powershell
$env:OPENCODE_CHANNEL = "latest"
$env:OPENCODE_VERSION = (Get-Content packages/cli/package.json | ConvertFrom-Json).version

bun run --cwd packages/cli script/build.ts --target=opencode-windows-x64-baseline

$env:OPENCODE_CLI_DIST = (Resolve-Path packages/cli/dist).Path
$env:OPENCODE_CLI_TARGET = "x86_64-pc-windows-msvc"

Push-Location packages/desktop

bun run build

.\resources\opencode-cli.exe --version

bun run package:win --x64 --publish never

Pop-Location
```

`bun run build` runs the Desktop prebuild automatically and copies the CLI built from the current checkout into the Desktop resources.

The packaged CLI should report the expected OpenCode version before creating the installer.

Installer artifacts are written to:

```text
packages/desktop/dist/
```

The Windows x64 installer is:

```text
packages/desktop/dist/opencode-desktop-win-x64.exe
```

Local builds do not use the CI Windows signing step.

### Reinstalling the same OpenCode version

Desktop stages the bundled CLI under:

```text
%APPDATA%\ai.opencode.desktop\cli\<version>\
```

If you rebuild and reinstall the same OpenCode version, Desktop may reuse the previously staged CLI instead of copying the newly bundled executable.

When testing a rebuilt installer with the same version number, stop OpenCode and remove or rename the cached version directory before launching the new installation:

```powershell
Get-Process OpenCode,opencode-cli -ErrorAction SilentlyContinue |
    Stop-Process -Force

$cache = Join-Path $env:APPDATA "ai.opencode.desktop\cli\2.0.11"

if (Test-Path -LiteralPath $cache) {
    Rename-Item `
        -LiteralPath $cache `
        -NewName "2.0.11-backup-$(Get-Date -Format yyyyMMdd-HHmmss)"
}
```

On the next launch, Desktop will stage the CLI bundled with the newly installed application.

### Verify background service registration

For a production `latest` CLI build, the background service should use the normal registration file:

```text
%USERPROFILE%\.local\state\opencode\service.json
```

It should not register as:

```text
service-prod.json
```

After starting OpenCode, verify the service:

```powershell
opencode service status
```

You can also inspect the registration:

```powershell
$state = Join-Path $env:USERPROFILE ".local\state\opencode"
Get-Content (Join-Path $state "service.json")
```

## Execution plugin

From the checkout root, build a new standalone package. The output directory must not already exist.

```powershell
Push-Location packages/superpowers-execution
bun run package:stage C:/opencode/superpowers-execution-next
Pop-Location
```

This includes the compiled plugin, skills, and runtime dependencies.

When ready to restart your server, replace the installed package while keeping a timestamped backup:

```powershell
opencode service stop

Rename-Item `
    C:/opencode/superpowers-execution `
    "superpowers-execution-backup-$(Get-Date -Format yyyyMMdd-HHmmss)"

Rename-Item `
    C:/opencode/superpowers-execution-next `
    superpowers-execution

opencode service start
opencode service status
```

Keep the existing plugin config entry:

```text
"C:/opencode/superpowers-execution"
```

No `AGENTS.md` entry is neede

# Windows desktop and execution plugin

Run in PowerShell with Bun 1.4.2 installed. From your checkout:

```powershell
git switch v2
git pull --ff-only origin v2
bun install
```

## Desktop (Windows x64, local production build)

Build the bundled CLI from this checkout, then the desktop and installer:

```powershell
$env:OPENCODE_CHANNEL = "prod"
$env:OPENCODE_VERSION = (Get-Content packages/cli/package.json | ConvertFrom-Json).version
bun run --cwd packages/cli script/build.ts --target=opencode-windows-x64-baseline
$env:OPENCODE_CLI_DIST = (Resolve-Path packages/cli/dist).Path
$env:OPENCODE_CLI_TARGET = "x86_64-pc-windows-msvc"
Push-Location packages/desktop
bun run build
bun run package:win --x64 --publish never
Pop-Location
```

`bun run build` runs the desktop prebuild automatically. Installer artifacts are in
`packages/desktop/dist/`. Local builds do not use the CI Windows signing step.

## Execution plugin

From the checkout root, build a new standalone package (the output must not exist):

```powershell
Push-Location packages/superpowers-execution
bun run package:stage C:/opencode/superpowers-execution-next
Pop-Location
```

This includes the compiled plugin, skills, and runtime dependencies. When ready to
restart your server, replace the installed package, keeping a timestamped backup:

```powershell
opencode service stop
Rename-Item C:/opencode/superpowers-execution "superpowers-execution-backup-$(Get-Date -Format yyyyMMdd-HHmmss)"
Rename-Item C:/opencode/superpowers-execution-next superpowers-execution
opencode service start
opencode service status
```

Keep the existing plugin config entry `"C:/opencode/superpowers-execution"`.
No `AGENTS.md` entry is needed.

# .NET / C# — Debug & Build Configuration

> **Runtime support:** .NET isolated worker Azure Functions (`FUNCTIONS_WORKER_RUNTIME=dotnet-isolated`).

## Prerequisites

| Tool | Detection Command | Required For | Install Link |
|------|-------------------|-------------|-------------|
| .NET SDK | `dotnet --version` | Build and run .NET projects | [dotnet.microsoft.com](https://dotnet.microsoft.com/download) |
| Azure Functions Core Tools | `func --version` | Run Functions host locally | [aka.ms/azure-functions-core-tools](https://aka.ms/azure-functions-core-tools) |

---

## Two Supported Debug Shapes

Functions Worker **2.x** (required for .NET 10 — `Microsoft.Azure.Functions.Worker >= 2.50.0`, `Microsoft.Azure.Functions.Worker.Sdk >= 2.0.5`) supports **two** debug shapes. Pick based on the project's trigger mix:

| Shape | When to use | preLaunchTask | `request` | How settings load |
|-------|-------------|---------------|-----------|-------------------|
| **A. `dotnet run` + `coreclr launch`** *(recommended for HTTP-only)* | Only HTTP triggers | `dotnet build` | `launch` | Worker.Sdk 2.0+ wires Functions Core Tools into `dotnet run`; `local.settings.json` loads automatically |
| **B. `func host start` + `coreclr attach`** *(required for non-HTTP triggers)* | Timer / Blob / Queue / Service Bus / Cosmos / Event Hub triggers | `func: host start` | `attach` | `func host start` loads settings as before |

> **Why two shapes?** `dotnet run` is simpler, faster, and breakpoints land on first F5 — but only the Functions host can emulate non-HTTP trigger sources (queues, timers, blob events, service-bus messages). For any mixed-trigger project, use Shape B.
>
> **Historical note:** In Worker **1.x**, Shape A was unsupported — `local.settings.json` was only loaded by the host, so launching the DLL directly caused missing-config errors. Worker.Sdk 2.0+ fixes this by integrating Core Tools into the `dotnet run` pipeline. Legacy docs that say "never use `coreclr launch`" refer to the 1.x limitation and no longer apply.

---

## ⛔ REQUIRED: Generate `.vscode/extensions.json`

The **attach** path (Shape B) uses the `"type": "func"` task type contributed by the **Azure Functions VS Code extension**. If the user doesn't have it installed, F5 fails with:

```
Could not find the task 'func: host start'.
```

Shape A (`dotnet run`) does **not** require the Functions extension for F5 — but we still recommend it for the CLI (`Create Function`, `Deploy to Azure`) features.

```json
{
  "recommendations": [
    "ms-azuretools.vscode-azurefunctions",
    "ms-dotnettools.csharp"
  ]
}
```

| Extension ID | Why Required |
|--------------|-------------|
| `ms-azuretools.vscode-azurefunctions` | Contributes the `"type": "func"` task type (Shape B only); also needed for deploy / create-function UX |
| `ms-dotnettools.csharp` | Contributes the `"type": "coreclr"` debugger used by both shapes |

> If `.vscode/extensions.json` already exists, **merge** the recommendations — do not overwrite.

---

## Debugger Fragments

### Shape A — `dotnet run` + `coreclr launch` (HTTP-only projects)

```jsonc
{
  "name": "Functions (dotnet run)",
  "type": "coreclr",
  "request": "launch",
  "program": "${workspaceFolder}/{functionsProjectDir}/bin/Debug/net10.0/{AssemblyName}.dll",
  "cwd": "${workspaceFolder}/{functionsProjectDir}",
  "preLaunchTask": "dotnet build",
  "stopAtEntry": false,
  "console": "integratedTerminal"
}
```

| Field | Value | Why |
|-------|-------|-----|
| `type` | `"coreclr"` | .NET debugger (requires C# extension) |
| `request` | `"launch"` | VS Code starts the worker directly; Worker.Sdk 2.0+ invokes Core Tools to load settings |
| `program` | path to built `.dll` | **Not** `.exe` — we're using `dotnet <dll>` mechanics, which work cross-platform |
| `cwd` | project directory | Required so `local.settings.json` and `host.json` are discovered |
| `console` | `"integratedTerminal"` | Surfaces the Functions banner + invocation logs in the VS Code terminal |

> The `.dll` + `cwd` combo is what lets Worker.Sdk 2.x find `local.settings.json`. If breakpoints don't bind on startup, verify `cwd` points at the folder containing `host.json`.

### Shape B — `func host start` + `coreclr attach` (non-HTTP triggers)

```jsonc
{
  "name": "Functions (attach to func host)",
  "type": "coreclr",
  "request": "attach",
  "processName": "{AssemblyName}.exe",
  "preLaunchTask": "func: host start"
}
```

| Field | Value | Why |
|-------|-------|-----|
| `type` | `"coreclr"` | .NET debugger |
| `request` | `"attach"` | Host spawns the worker; we attach to it |
| `processName` | `"{AssemblyName}.exe"` | Auto-attach by OS process name — avoids the picker dialog |
| `preLaunchTask` | `"func: host start"` | Starts the Functions host, which spawns the worker |

#### Determining `processName` (Shape B only)

> ⛔ **The `.exe` suffix is REQUIRED on Windows.** VS Code's coreclr `processName` is matched literally against the OS process name including extension. Writing `"Scrapbook.Api"` instead of `"Scrapbook.Api.exe"` produces:
>
> ```
> No process with the specified name is currently running.
> ```

The worker process name is **`{AssemblyName}.exe`**, derived from the project's `.csproj`:

1. If `<AssemblyName>` is defined, use that + `.exe`
2. Otherwise the `.csproj` filename (e.g., `Functions.csproj` → `Functions.exe`)
3. The `.csproj` **MUST** have `<OutputType>Exe</OutputType>` — otherwise no apphost `.exe` is produced and there's no attach target.

**Verify before writing `launch.json`:** after `dotnet build`, confirm `{functionsProjectDir}/bin/Debug/{tfm}/{AssemblyName}.exe` exists.

> ❌ **Do NOT use `${command:pickProcess}` or `${command:azureFunctions.pickProcess}`** — both pop a blocking dialog every F5. Use literal `processName`.

> ❌ **Do NOT drop the `.exe`** — the attach will silently fail at F5 time.

---

## Build Chain Tasks

Tasks.json must include entries for **both** debug shapes so either can be selected:

```
Shape A:  "dotnet build"
              └── dependsOn: "Start Emulators"

Shape B:  "func: host start"
              ├── dependsOn: "dotnet build"
              └── dependsOn: "Start Emulators"
```

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "shell",
      "label": "Start Emulators",
      "command": "docker compose up -d",
      "options": { "cwd": "${workspaceFolder}" },
      "problemMatcher": []
    },
    {
      "label": "dotnet build",
      "type": "process",
      "command": "dotnet",
      "args": [
        "build",
        "${workspaceFolder}/{path-to-csproj}",
        "--configuration",
        "Debug"
      ],
      "dependsOn": ["Start Emulators"],
      "problemMatcher": "$msCompile",
      "group": "build"
    },
    {
      "type": "func",
      "label": "func: host start",
      "command": "host start",
      "options": { "cwd": "${workspaceFolder}/{path-to-functions-project}" },
      "problemMatcher": "$func-dotnet-watch",
      "isBackground": true,
      "dependsOn": ["dotnet build"]
    }
  ]
}
```

| Task | Type | Used by | Purpose | Background? |
|------|------|---------|---------|------------|
| `Start Emulators` | `shell` | Both shapes | `docker compose up -d` — see [orchestrators/docker-compose.md](../orchestrators/docker-compose.md) for compose-file invariants | No |
| `dotnet build` | `process` | Both shapes | Restore + compile | No — `$msCompile` |
| `func: host start` | `func` | Shape B only | Launches the Functions host; worker attach target | ✅ Yes — `$func-dotnet-watch` |

> **Shape A** skips the `func: host start` task entirely — `dotnet run` is invoked implicitly by `coreclr launch` through Worker.Sdk 2.x.
>
> **Shape B** keeps the historical attach flow and requires the `ms-azuretools.vscode-azurefunctions` extension for the `"type": "func"` task.

---

## Convenience Scripts

Because .NET projects do not have a built-in script runner equivalent to `npm run`, convenience scripts live as shell scripts under `scripts/`:

```
scripts/
  emulators-start.sh    # docker compose up -d
  emulators-stop.sh     # docker compose down
  emulators-clean.sh    # docker compose down && rm -rf {data-dirs}
  db-migrate.sh         # (only if migrations detected)
```

Each script is a plain `#!/bin/bash` wrapper around the docker/psql/dotnet commands. See [project-types/functions.md](../project-types/functions.md) for the full docker-compose integration.

---

## Checklist — What to Verify Before Marking Launch Config `Implemented`

After generating `launch.json`, `tasks.json`, and `extensions.json`, Phase 3 validation MUST confirm (apply the row for whichever shape was emitted):

**Both shapes:**
1. ✅ `.vscode/extensions.json` includes `ms-dotnettools.csharp` (and `ms-azuretools.vscode-azurefunctions` if Shape B)
2. ✅ `dotnet build` succeeds (0 errors)
3. ✅ `curl http://localhost:7071/api/health` → `200`
4. ✅ No missing-config errors (`ConnectionStrings:AppDb is required`, etc.) — confirms `local.settings.json` was loaded by whichever shape is in use

**Shape A additionally:**
5. ✅ The `program` path in `launch.json` resolves to the built `.dll` on disk (not `.exe`)
6. ✅ `cwd` in `launch.json` points at the folder containing `host.json`

**Shape B additionally:**
5. ✅ `func host start` runs and **all functions register** in the terminal output
6. ✅ `processName` in `launch.json` matches the actual built `.exe` on disk, **including the `.exe` suffix** (`ls bin/Debug/{tfm}/*.exe` → file must exist, basename + `.exe` must equal `processName` character-for-character)

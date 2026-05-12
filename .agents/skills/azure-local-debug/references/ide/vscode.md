# VS Code — IDE Configuration

## Overview

VS Code uses two JSON files under `.vscode/` to configure debugging and build tasks.

## Configuration File Paths

| Purpose | File Path |
|---------|-----------|
| Debug configuration | `.vscode/launch.json` |
| Build configuration | `.vscode/tasks.json` |

---

## Debug Configuration — `launch.json`

The skill assembles `launch.json` entries by combining **debugger properties** from `runtimes/{rt}.md` (port) and **request mode** from `project-types/{type}.md` with the VS Code-specific wrapper below.

### Template

```json
{
  "name": "{service-id} (debug)",
  "type": "<vs-code-debugger-type>",
  "request": "<attach-or-launch>",
  "port": "<base-debug-port>",
  "restart": true,
  "preLaunchTask": "<startup-task-label>"
}
```

### Field Reference

| Field | Source | Notes |
|-------|--------|-------|
| `name` | Service ID from [multi-service.md](../multi-service.md) or project name | Always suffixed with `(debug)` |
| `type` | **Server-side:** Debug protocol from `runtimes/{rt}.md` → mapped to VS Code adapter in runtime table below. **Browser-based:** Project type from `project-types/{type}.md` → mapped in project-type table below. | The source depends on whether the debugger attaches to a runtime process or launches a browser. |
| `request` | `project-types/{type}.md` → Runtime Wiring → `Request Mode` | `"attach"` when the host command spawns the runtime; `"launch"` when the IDE starts it directly |
| `port` | `runtimes/{rt}.md` → `Base debug port` | Incremented per service in monorepos (see [multi-service.md § Port Assignment](../multi-service.md)). Not used for browser-based configs (use `url` instead). |
| `restart` | Always `true` | Re-attach after the host restarts on file changes |
| `preLaunchTask` | `project-types/{type}.md` → startup task label | e.g. `"func: host start"` — rendered via the per-project-type subsection in Build Configuration below |

### Runtime Debug Protocol → VS Code Adapter Mapping

> For server-side project types (Functions, App Service, etc.) where the debugger attaches to a runtime process.
>
> **To add a new runtime:** add a row here, then create or update the corresponding `runtimes/{rt}.md` with the debugger properties.

| Runtime | Debug Protocol → VS Code Debugger Type | Extra Fields | Status |
|---------|----------------------|--------------|--------|
| node | Node Inspector → `node` | — | ✅ Implemented |
| dotnet | CoreCLR DAP → `coreclr` | Shape A (`launch`): `program`, `cwd`, `console`. Shape B (`attach`): literal `processName` **with `.exe` suffix on Windows**. ⛔ Never use `${command:pickProcess}`. See [runtimes/dotnet.md](../runtimes/dotnet.md). | ✅ Implemented |
| python | debugpy DAP → `debugpy` | `"connect": { "host": "localhost", "port": 5678 }` | ⛔ Not yet implemented |
| java | JDWP → `java` | `"hostName": "localhost"` | ⛔ Not yet implemented |
| go | Delve DAP → `go` | `"mode": "remote"` | ⛔ Not yet implemented |

### Project-Type → VS Code Adapter Mapping

> For browser-based project types where the IDE launches a browser instead of attaching to a runtime process. These adapters are determined by the project type, not the runtime.

| Project Type | VS Code Debugger Type | Extra Fields | Reference |
|-------------|----------------------|--------------|-----------|
| Frontend SPA | `chrome` | `"url"`, `"webRoot"` | [project-types/frontend-spa.md](../project-types/frontend-spa.md) |

### Example: Frontend SPA

> See [project-types/frontend-spa.md](../project-types/frontend-spa.md) for detection signals and framework table.

The debug adapter is `chrome` (built-in JS Debugger). The request mode is `launch` (VS Code opens the browser).

```json
{
  "name": "{id} (debug)",
  "type": "chrome",
  "request": "launch",
  "url": "http://localhost:{dev-server-port}",
  "webRoot": "${workspaceFolder}/{service-root}",
  "preLaunchTask": "{id} dev"
}
```

---

## Build Configuration — `tasks.json`

VS Code tasks are defined in `.vscode/tasks.json`. The skill builds a dependency chain with two kinds of tasks:

1. **Runtime build tasks** — install, clean, watch, build (commands from `runtimes/{rt}.md`, problem matchers from this file)
2. **Project type top-level task** — the task that starts the application (e.g., `func host start`). This is the task that `preLaunchTask` in the launch configuration points to. It sits at the top of the chain and depends on the runtime build tasks.

### Chain Shape

```
"<top-level-task>"                ← project-type-specific (see per-project-type subsections below)
       ├── dependsOn: "<watch-task>"       ← from Runtime Task Reference
       │                └── dependsOn: "<clean-task>"
       │                               └── dependsOn: "<install-task>"
       └── dependsOn: "Start Emulators"    ← only when emulators are required
```

### Emulator Task

This task is present only when emulators are required (i.e., when a `docker-compose.yml` is generated during Phase 1):

```json
{
  "type": "shell",
  "label": "Start Emulators",
  "command": "docker compose down && docker compose up -d",
  "problemMatcher": []
}
```

### Runtime Task Reference

> **To add a new runtime:** add a row here with VS Code-specific problem matchers, then create or update the corresponding `runtimes/{rt}.md` with the build commands and chain shape.
>
> Build commands (install, clean, watch, build) are **not** listed here — they are runtime properties sourced from each `runtimes/{rt}.md` Build Chain section.

| Runtime | Watch Problem Matcher | Build Problem Matcher | Status |
|---------|----------------------|----------------------|--------|
| node-ts | `$tsc-watch` | `$tsc` | ✅ Implemented |
| node-js | — | — | ✅ Implemented |
| dotnet | `$msCompile` | `$msCompile` | ✅ Implemented |
| python | — | — | ⛔ Not yet implemented |
| java | — | — | ⛔ Not yet implemented |
| go | — | — | ⛔ Not yet implemented |

> **Monorepo / alternative package managers:** Adjust task labels and commands as needed (e.g., `yarn`, `pnpm`, `gradle`). The key invariant is the chain shape: **install → clean → build/watch → top-level task**. Some runtimes skip clean or watch steps — use only what applies.

### Working Directory (`cwd`) Rules

> ⚠️ **CRITICAL for multi-service repos.** Without correct `cwd`, commands like `npm install` or `func host start` will run from the workspace root and fail.

| Task Scope | `cwd` Setting | Example |
|------------|--------------|---------|
| **Per-service tasks** (install, clean, watch, build, top-level) | `"options": { "cwd": "${workspaceFolder}/{service-root}" }` | `"cwd": "${workspaceFolder}/api"` |
| **Shared tasks** (Start Emulators) | Workspace root (omit `cwd` — it defaults to workspace root) | — |
| **Single-service repos** | Omit `cwd` — workspace root is the service root | — |

### Per Project Type: Azure Functions

> See [project-types/functions.md](../project-types/functions.md) for startup command, request mode, and per-runtime notes. This subsection covers VS Code-specific rendering only. Standard `"type": "shell"` project types do not need a subsection — the generic chain shape above already covers them.

The top-level task uses the VS Code `func` task type provided by the Azure Functions extension. The launch configuration's `preLaunchTask` points to this task.

| Runtime | Task Type | Problem Matcher | Status |
|---------|-----------|----------------|--------|
| node-ts | `func` | `$func-node-watch` | ✅ Implemented |
| node-js | `func` | `$func-node-watch` | ✅ Implemented |
| dotnet  | `func` | `$func-dotnet-watch` | ✅ Implemented — see [runtimes/dotnet.md](../runtimes/dotnet.md) for Shape A vs Shape B |
| python  | `func` | `$func-python-watch` | ⛔ Not yet implemented |
| java    | `func` | `$func-java-watch` | ⛔ Not yet implemented |

**node-ts** (has watch task):

```json
{
  "type": "func",
  "label": "func: host start",
  "command": "host start",
  "problemMatcher": "$func-node-watch",
  "isBackground": true,
  "dependsOn": ["npm watch", "Start Emulators"]
}
```

**node-js** (no compile/watch step):

```json
{
  "type": "func",
  "label": "func: host start",
  "command": "host start",
  "problemMatcher": "$func-node-watch",
  "isBackground": true,
  "dependsOn": ["npm install", "Start Emulators"]
}
```

> `dependsOn`: first entry is the runtime-specific prerequisite — watch task for TypeScript, install task for JavaScript. Include `"Start Emulators"` only when emulators are required.

### Example: frontend SPA tasks

> See [project-types/frontend-spa.md](../project-types/frontend-spa.md) for detection signals, framework table, and startup task label derivation.

The top-level task is the framework's dev server (e.g., `npm run dev` for Vite). No runtime build chain — the dev server handles everything. The task label follows the pattern `"{id} dev"` (see [project-types/frontend-spa.md](../project-types/frontend-spa.md) Runtime Wiring).

```json
{
  "type": "shell",
  "label": "{id} dev",
  "command": "npm run dev",
  "options": { "cwd": "${workspaceFolder}/{service-root}" },
  "isBackground": true,
  "problemMatcher": {
    "owner": "vite",
    "pattern": { "regexp": "^$" },
    "background": {
      "activeOnStart": true,
      "beginsPattern": "VITE",
      "endsPattern": "ready in \\d+"
    }
  }
}
```

> ⚠️ **IMPORTANT: Background tasks MUST have a real `problemMatcher`.**
> Avoid `"problemMatcher": []` on a task with `"isBackground": true`.
> An empty matcher causes VS Code to display a blocking dialog:
> *"The task has not exited and doesn't have a 'problemMatcher' defined."*
> Always use a framework-specific background matcher from [project-types/frontend-spa.md](../project-types/frontend-spa.md).

---

## Multi-Service / Compound Configuration

> ⛔ **MANDATORY:** When 2+ service roots are detected (including Frontend SPA projects), a compound launch configuration **must** be generated.

```json
{
  "name": "Start All",
  "configurations": ["{id} (debug)", "..."],
  "preLaunchTask": "Start Emulators",
  "stopAll": true
}
```

One entry per service using its assigned ID from [multi-service.md](../multi-service.md).

> ⚠️ `preLaunchTask` is **conditional**: include `"preLaunchTask": "Start Emulators"` only when emulators are required (i.e., the `"Start Emulators"` task exists in `tasks.json`). Omit the field entirely when no emulators are needed.

---

## Debug Configuration Checklist Validation

> ⛔ **MANDATORY.** You MUST execute every step below for each launch configuration. Do NOT skip, assume, or approximate results. Do NOT proceed to the closing message until every checklist entry has a real ✅ or ❌ result.

For each **non-compound** launch configuration in `.vscode/launch.json`:

1. Read the config's `preLaunchTask` value
2. Trace the full `dependsOn` chain in `.vscode/tasks.json` to resolve the dependency order
3. Run prerequisite tasks first (install, clean, emulators), then start the `preLaunchTask` itself as a background process
4. If a `docker-compose.yml` was generated, verify all services started correctly after `docker compose up -d`. Long-running services (e.g., database emulators, Azurite) should be running and healthy; one-shot services (e.g., `db-migrate`) should have exited with code 0. Use `docker compose ps` and `docker compose logs <service>` to check. If any service failed, diagnose the issue, fix the configuration, and re-run until all services are healthy or exited cleanly. Only mark the config ❌ after exhausting reasonable fix attempts.
5. Confirm the ready signal in stdout from the **top-level task**, for example:
   - Azure Functions host → `"Host lock lease acquired"` or `"Functions host started"`
   - Vite / webpack → `"ready in"` or `"Local:"`
   - Node HTTP server → `"listening on"` or `"Server running"`
6. After the ready signal, confirm with `curl` using the **application HTTP port** (not the debug port):
   - For `node`-type configs (Functions): `curl -s -o /dev/null -w "%{http_code}" http://localhost:7071/api/health` → expect `200` (port `7071` is the Functions host HTTP port; debug port `9229` is for the debugger only)
   - For `chrome`-type configs (browser dev servers): `curl -s -o /dev/null -w "%{http_code}" http://localhost:<url port from launch config>` → expect `200` or `301`
   - **Note:** For `chrome`-type configs you are validating that the dev server started and is reachable — you do NOT need to launch a browser. The `preLaunchTask` is a shell task (`npm run dev` / Vite) that runs in the terminal like any other.
7. **For `coreclr` attach configs (`"request": "attach"` + `processName`):** after the ready signal is observed, list running OS processes and confirm a process with the exact name in `processName` exists. If it doesn't, the F5 attach WILL fail with `"No process with the specified name is currently running"`.
   - Windows PowerShell: `Get-Process -Name "<processName without .exe>" -ErrorAction SilentlyContinue` → must return a process
   - macOS/Linux: `pgrep -x "<processName>"` → must return a PID
   - The `processName` field MUST include the `.exe` suffix on Windows (e.g., `"Scrapbook.Api.exe"`, NOT `"Scrapbook.Api"`). If this check fails, fix `launch.json` before marking the config ✅.
8. Kill background processes, then move to the next config
9. For compound configs: skip running them; mark ✅ if all named member configs passed, ❌ if any failed

**You MUST then edit the `## Debug Configuration Checklist` section in `.azure/local-development-plan.md`:**

```
Debug Configuration Checklist:
✅ <config-name> — <ready signal + curl result>
✅ <config-name> — <ready signal + curl result>
```

One line per config (non-compound and compound). ✅ requires the ready signal observed AND curl confirmed.

> ⛔ Do NOT set status to `Implemented` until every stub in the Debug Configuration Checklist has been replaced with a real ✅ or ❌ result. A checklist with any remaining stubs is incomplete — go back and validate.

---

## Quick Start

After Phase 3 validation, include this in the closing message:

> Press **F5** in VS Code and select **Start All** to launch the full application with debugging.

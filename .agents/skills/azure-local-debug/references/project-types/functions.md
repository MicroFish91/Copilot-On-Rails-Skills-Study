# Project Type: Azure Functions

Reference guide for local development setup of Azure Functions projects.

---

## Detection Signals

| Signal | Notes |
|--------|-------|
| `host.json` present | Primary signal — required |
| Azure Functions SDK in dependencies | Confirms it's a Functions project — see [classify.md § Functions SDK Detection](../classify.md) |

---

## Runtime Support Matrix

| Runtime | Status | Reference |
|---------|--------|-----------|
| node-ts | ✅ Implemented | [runtimes/node.md](../runtimes/node.md) |
| node-js | ✅ Implemented | [runtimes/node.md](../runtimes/node.md) |
| dotnet (Functions isolated) | ✅ Implemented | [runtimes/dotnet.md](../runtimes/dotnet.md) |
| dotnet (Aspire AppHost) | ✅ Implemented — handled by [orchestrators/aspire.md](../orchestrators/aspire.md), **not** this project type | See orchestrator doc |
| python  | 🔲 Planned | [limited-support.md](../limited-support.md) |
| java    | 🔲 Planned | [limited-support.md](../limited-support.md) |

> **⚠️ Emulators only:** When a runtime with limited support is detected, proceed with emulator setup (docker-compose) — this is language-agnostic. Skip IDE debug/launch configuration generation and inform the user to configure those manually or notify and provide best effort attempt.
>
> **If the workspace contains an Aspire AppHost**, this project-type doc does NOT apply even if `host.json` is also present — the classifier matches the Aspire row first. Aspire supersedes the Functions-isolated flow for that workspace.

---

## Dependency Discovery

Scan every `function.json` for its `"type"` binding field, **or** scan Python/Java source files for trigger decorator/attribute names. Each binding maps to an emulator.

### Binding → Emulator Mapping

| Binding Type(s) | Azure Service | Default Ports | Connection String | Status | Reference |
|----------------|---------------|---------------|-------------------|--------|-----------|
| `blobTrigger`, `blob` | Blob Storage | 10000 | `UseDevelopmentStorage=true` | ✅ Implemented | [emulators/azurite.md](../emulators/azurite.md) |
| `queueTrigger`, `queue` | Queue Storage | 10001 | `UseDevelopmentStorage=true` | ✅ Implemented | [emulators/azurite.md](../emulators/azurite.md) |
| `table` | Table Storage | 10002 | `UseDevelopmentStorage=true` | ✅ Implemented | [emulators/azurite.md](../emulators/azurite.md) |
| `cosmosDBTrigger`, `cosmosDB` | Cosmos DB | 8081, 10250–10254 | — | 🔲 Planned | [limited-support.md](../limited-support.md) |
| `serviceBusTrigger`, `serviceBus` | Service Bus | 5672 | — | 🔲 Planned | [limited-support.md](../limited-support.md) |
| `eventHubTrigger`, `eventHub` | Event Hubs | 9093 | — | 🔲 Planned | [limited-support.md](../limited-support.md) |
| `sql`, `sqlTrigger` | Azure SQL | 1433 | — | 🔲 Planned | [limited-support.md](../limited-support.md) |
| `httpTrigger` | (built-in) | — | — | ✅ Implemented | — |
| `timerTrigger` | (built-in) | — | — | ✅ Implemented | — |

> **Azurite consolidation:** If multiple storage bindings (blob + queue + table) are detected, create a **single** Azurite service — not one per binding type.

### Services Without Azure Emulators

| Binding Type | Azure Service | Recommendation |
|-------------|---------------|----------------|
| `signalR` | Azure SignalR | Use a dev-tier Azure SignalR instance |
| PostgreSQL (SDK, not a binding) | Azure Database for PostgreSQL | [emulators/postgres.md](../emulators/postgres.md) |

---

## Startup Command

```
func host start
```

> The Azure Functions Core Tools handle debug flag injection for the appropriate runtime automatically (e.g., `--inspect` for Node.js).

> **Two debug shapes for .NET.** Functions Worker 2.x supports both `dotnet run` + `coreclr launch` (HTTP-only projects) **and** `func host start` + `coreclr attach` (any trigger mix). See [runtimes/dotnet.md § Two Supported Debug Shapes](../runtimes/dotnet.md). For non-.NET runtimes the attach-by-`func host start` shape is the only path.

---

## Runtime Wiring

<!-- Combines with runtimes/{rt}.md (protocol, port) and ide/{ide}.md to produce IDE debug config.
     Debug port values come from each runtimes/{rt}.md Debugger Properties table. -->

| Runtime | Startup task label | Task type | Problem matcher | Request mode | Status | Reference |
|---------|--------------------|-----------|-----------------|--------------|--------|-----------|
| node-ts | `func: host start` | `func` | `$func-node-watch` | `attach` | ✅ Implemented | [runtimes/node.md](../runtimes/node.md) |
| node-js | `func: host start` | `func` | `$func-node-watch` | `attach` | ✅ Implemented | [runtimes/node.md](../runtimes/node.md) |
| dotnet  | `func: host start` (Shape B) **or** `dotnet build` (Shape A) | `func` / `process` | `$func-dotnet-watch` / `$msCompile` | `attach` (B) / `launch` (A) | ✅ Implemented | [runtimes/dotnet.md](../runtimes/dotnet.md) |
| python  | `func: host start` | `func` | `$func-python-watch` | `attach` | 🔲 Planned | [limited-support.md](../limited-support.md) |
| java    | `func: host start` | `func` | `$func-java-watch` | `attach` | 🔲 Planned | [limited-support.md](../limited-support.md) |

> **dotnet `processName` warning.** Shape B (`coreclr attach`) requires the literal `processName` in `launch.json` to include the `.exe` suffix on Windows (e.g., `"Scrapbook.Api.exe"`, NOT `"Scrapbook.Api"`). Without it, F5 fails with `"No process with the specified name is currently running"`. Do NOT use `${command:pickProcess}`. See [runtimes/dotnet.md § Determining processName](../runtimes/dotnet.md).

The startup step depends on:
1. The runtime-specific build/watch step from `runtimes/{rt}.md` (e.g., `npm watch` for node-ts, `dotnet build` for dotnet)
2. The `Start Emulators` step (only when emulators are required)

`dependsOn` list: first entry is the runtime-specific build/watch task label from `runtimes/{rt}.md`; second is `"Start Emulators"` when emulators are required.

Place emulator connection strings in `local.settings.json` under `"Values"`:

| Emulator | Key | Value | Status | Reference |
|----------|-----|-------|--------|-----------|
| Azurite (storage) | `AzureWebJobsStorage` | `UseDevelopmentStorage=true` | ✅ Implemented | [emulators/azurite.md](../emulators/azurite.md) |
| Cosmos DB | {detected from bindings} | — | 🔲 Planned | [limited-support.md](../limited-support.md) |
| Service Bus | {detected from bindings} | — | 🔲 Planned | [limited-support.md](../limited-support.md) |
| Event Hubs | {detected from bindings} | — | 🔲 Planned | [limited-support.md](../limited-support.md) |
| SQL Edge | {detected from bindings} | — | 🔲 Planned | [limited-support.md](../limited-support.md) |
| PostgreSQL | {detected from code} | See [emulators/postgres.md](../emulators/postgres.md) | ✅ Implemented | [emulators/postgres.md](../emulators/postgres.md) |

> **Key discovery:** Except for `AzureWebJobsStorage` (a well-known Azure Functions convention), connection string key names are not fixed. The inventory phase ([inventory.md](../inventory.md)) detects the actual key names from bindings, SDK usage, and existing configuration files. Use the detected names — do not invent defaults.

> **Never overwrite** existing values in `local.settings.json` — only add missing keys.

---

## API Test Collections

See [api-test-collections.md](../api-test-collections.md) for all test script patterns. For this project type, generate tests for:

- HTTP triggers → HTTP patterns with `baseUrl: http://localhost:7071/api`
- Blob triggers → Storage § Blob trigger pattern
- Queue triggers → Storage § Queue trigger pattern
- Timer triggers → Timer § admin API pattern (only if explicitly requested)
- Cosmos DB triggers → Cosmos DB pattern
- Service Bus triggers → Service Bus pattern
- Event Hub triggers → Event Hubs pattern

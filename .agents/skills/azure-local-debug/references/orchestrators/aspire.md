# Orchestrator: .NET Aspire

> Used instead of [docker-compose](docker-compose.md) when a .NET project selects `Orchestration: Aspire` (or when `*.AppHost.csproj` is detected in the workspace).

## Detection Signals

| Signal | Notes |
|--------|-------|
| `*.AppHost.csproj` exists | Primary signal |
| `<Sdk Name="Aspire.AppHost.Sdk"` in csproj | Confirms Aspire AppHost SDK |
| `<IsAspireHost>true</IsAspireHost>` | Alt signal |
| `Aspire.Hosting.AppHost` PackageReference | Fallback |

## Phase 2 Behavior — SHORT-CIRCUIT

When Aspire is the orchestrator, **do NOT generate**:

- `docker-compose.yml` — AppHost owns container lifecycle
- `scripts/emulators-*.sh` / `.ps1` — replaced by F5 on AppHost
- `.vscode/tasks.json` emulator / build / `func host start` chain
- Custom attach-by-processName `launch.json` configs

**DO generate** a minimal `.vscode/launch.json`:

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Run AppHost",
      "type": "dotnet",
      "request": "launch",
      "projectPath": "${workspaceFolder}/src/AppHost/AppHost.csproj"
    }
  ]
}
```

And `.vscode/extensions.json`:

```json
{
  "recommendations": [
    "ms-dotnettools.csdevkit",
    "ms-dotnettools.csharp"
  ]
}
```

## Prerequisites Check (surface to user)

| Tool | Command | Required |
|------|---------|----------|
| .NET SDK 10+ | `dotnet --version` | Yes — .NET 10 is mandatory; downgrade only on explicit user request (see [shared-references/runtimes/dotnet-aspire.md](../../../shared-references/runtimes/dotnet-aspire.md)) |
| Aspire project templates | `dotnet new list aspire` | Yes — install once with `dotnet new install Aspire.ProjectTemplates`. Aspire 9+ ships as a NuGet template/package, **not** a workload — do NOT run `dotnet workload install aspire` |
| Docker (or Podman) | `docker info` | Yes — for container resources |
| C# Dev Kit (VS Code) | recommended in `extensions.json` | Yes — for auto-attach debugging |

## API Test Collection

Still emit `api-test-collections/` as normal. The base URL is whatever the Aspire dashboard shows for the API resource (dynamic local port). Document that users should check the dashboard's Endpoints panel for the actual port.

## Reference

See [`../../../shared-references/runtimes/dotnet-aspire.md`](../../../shared-references/runtimes/dotnet-aspire.md) for the full scaffolded structure, AppHost wiring, ServiceDefaults, and testing patterns.

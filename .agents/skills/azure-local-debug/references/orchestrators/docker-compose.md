# Orchestrator: docker-compose

> **Default orchestrator.** Selected when no Aspire AppHost is detected in the workspace and the project plan does not specify `Orchestration: Aspire AppHost`. The symmetric pair of [aspire.md](aspire.md).

---

## When This Orchestrator Is Active

| Condition | Result |
|-----------|--------|
| `*.AppHost.csproj` / `<Sdk Name="Aspire.AppHost.Sdk">` / `Aspire.Hosting.AppHost` PackageReference present | ❌ Use [aspire.md](aspire.md) instead |
| Project plan Section 2 specifies `Orchestration: Aspire AppHost` | ❌ Use [aspire.md](aspire.md) instead |
| Anything else (including plain Azure Functions, Container Apps, App Service, Node, Python, Java, Go workspaces) | ✅ This orchestrator |

> Resolution is binary: classification picks **exactly one** orchestrator per workspace. See [classify.md](../classify.md) Step 1.

---

## Prerequisites

| Tool | Required | Notes |
|------|----------|-------|
| Compose-spec runtime | ✅ | Docker Engine, Docker Desktop, **or** Podman with `podman compose` (drop-in compatible) |
| `docker compose` v2 plugin | ✅ | `docker compose version` MUST report `v2.x` or higher. Never use the legacy hyphenated `docker-compose` binary in scripts or tasks. |
| WSL2 (Windows hosts) | ✅ on Windows | Docker Desktop requires WSL2 backend; Linux-container mode only |

---

## Generated Artifact Contract

When this orchestrator is active, Phase 2 **MUST** emit:

| Artifact | Path | Owner |
|----------|------|-------|
| Compose file | `docker-compose.yml` (workspace root) | This file (invariants) + [emulators/*.md](../emulators/) (per-service blocks) |
| Emulator scripts | `scripts/emulators-{start,stop,clean}.sh` (+ `.ps1` on Windows-friendly projects) | [generate.md](../generate.md) |
| `Start Emulators` VS Code task | `.vscode/tasks.json` | [runtimes/{rt}.md](../runtimes/) — task **label** is canonical; runtimes hard-depend on this name |
| Connection strings | `local.settings.json` / `.env` / `appsettings.Development.json` pointing at emulator ports | [runtimes/{rt}.md](../runtimes/) |
| Migrations service | `db-migrate` one-shot service in compose (only when migrations detected) | [migrations.md](../migrations.md) |

---

## Compose-File Invariants

These rules apply to **every** `docker-compose.yml` produced by this skill. They are collected here so individual emulator/runtime files can stay focused on their service blocks.

1. **No top-level `version:` key.** Deprecated since the Compose Spec 2023 rev. Emit only `services:`, `volumes:`, `networks:` at the top level.
2. **`docker compose` (v2 plugin) only.** Scripts, tasks, and documentation MUST invoke `docker compose ...` (space). Never the legacy `docker-compose ...` (hyphen) binary.
3. **Azurite MUST include `--skipApiVersionCheck`.** Required to avoid `x-ms-version` 400 errors when the Azure Storage SDK ships ahead of the Azurite image. See [emulators/azurite.md](../emulators/azurite.md).
4. **Bind-mount data dirs use `./.{service-name}` convention** (e.g., `./.azurite`, `./.postgres`, `./.cosmos`). The `emulators-clean` script derives wipe paths from `volumes:` — non-conforming mounts will not be cleaned.
5. **`container_name` — prefer to omit.** Default Compose names (`{project}-{service}-1`) are unique per workspace and avoid clashes when multiple workspaces run side-by-side. If a fixed name is required, use `{service}-emulator`. Never use bare service names.
6. **Declare a `healthcheck` on any emulator with dependents.** Migrations and dependent services rely on `depends_on: condition: service_healthy`; without a healthcheck the dependent races startup. Postgres is the canonical example (see [emulators/postgres.md](../emulators/postgres.md)). Healthchecks are optional for emulators with no dependents (e.g., Azurite).
7. **`restart` policy:**
   - `unless-stopped` — long-lived emulators (Azurite, Postgres, Cosmos, Service Bus, etc.)
   - `"no"` — one-shot services (`db-migrate`, schema bootstrap)
8. **Merge — never overwrite.** When a `docker-compose.yml` already exists, merge new services into it. Never silently replace. See [SKILL.md § Global Rules](../../SKILL.md#global-rules-no-exceptions).

---

## Symmetry vs Aspire

| Aspire feature | docker-compose equivalent |
|---|---|
| Aspire dashboard / OTel viewer | None — wire to App Insights or run external Jaeger/Prometheus if needed |
| Service discovery (`WithReference`) | Hard-coded `localhost:{port}` in connection strings; explicit env var wiring |
| `RunAsEmulator(...)` | A YAML service block from [emulators/*.md](../emulators/) |
| `WaitFor(...)` | `depends_on: condition: service_healthy` (requires the dependency to declare a healthcheck — see invariant 6) |
| `azd up` | `docker compose up -d` (local only); cloud deployment via separate IaC (Bicep / azd) |
| AppHost owns lifecycle | `Start Emulators` task + emulator scripts own lifecycle |
| F5 launches AppHost | F5 launches the service; `Start Emulators` runs as `preLaunchTask`/`dependsOn` |

---

## Cross-References

This file is an **index + invariants doc only**. Detailed content lives elsewhere — do not duplicate.

| Topic | Reference |
|-------|-----------|
| Per-emulator YAML blocks | [emulators/{name}.md](../emulators/) |
| Migration one-shot service | [migrations.md](../migrations.md) |
| Port conflict resolution | [generate.md](../generate.md) § Port Scan |
| Volume wipe protocol | [generate.md](../generate.md) § Wiping |
| Multi-service emulator dedup | [multi-service.md](../multi-service.md) |
| Runtime task wiring (`Start Emulators` consumer) | [runtimes/dotnet.md](../runtimes/dotnet.md), [runtimes/node.md](../runtimes/node.md) |
| Aspire alternative | [aspire.md](aspire.md) |

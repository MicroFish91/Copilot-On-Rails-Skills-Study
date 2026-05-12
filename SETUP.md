# 🚂 Copilot on Rails

**Copilot on Rails** is delivered through the [Azure Resource Groups extension](https://github.com/microsoft/vscode-azureresourcegroups). It walks you through scaffolding a new Azure project and setting up local debugging using two skills: **azure-project-scaffold** and **azure-local-debug**.

## Prerequisites

| Tool | Required for |
|------|-------------|
| [VS Code](https://code.visualstudio.com/) | IDE |
| [GitHub Copilot](https://github.com/features/copilot) | Agent mode |
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | Local emulators |
| [Node.js / npm](https://nodejs.org/) | Runtime |

---

## Skills

Both skills are triggered naturally through the Copilot on Rails guided agent flow.

| Skill | What it does |
|-------|-------------|
| `azure-project-scaffold` | Plans and scaffolds a new Azure-centric project end-to-end |
| `azure-local-debug` | Sets up local dev environment — emulators, IDE configs, and API test collections |

---

## Test Paths

There are two ways to test the experience.

### 🟢 Test A — Full Journey (Greenfield)

| | |
|---|---|
| **Repo** | [Copilot-On-Rails-Skills-Study](https://github.com/MicroFish91/Copilot-On-Rails-Skills-Study) |
| **Skills** | `azure-project-scaffold` → `azure-local-debug` |
| **Time** | Longer — end-to-end |

Open this repo in VS Code and run the extension.

### 🟡 Test B — Local Debug Only (Brownfield)

| | |
|---|---|
| **Repo** | _TBD_ |
| **Skills** | `azure-local-debug` only |
| **Time** | Shorter — focused on local dev setup |

Open the brownfield project repo in VS Code and run the extension. Useful for saving time or focusing on the debugging experience.
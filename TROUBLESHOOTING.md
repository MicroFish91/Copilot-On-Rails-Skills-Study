# Troubleshooting

Common pitfalls that may surface while running this project.

## 1. Cleaning Up Generated/Untracked Files

You'll often need to remove all untracked and modified files to get back to a clean state. The quickest way is:

```bash
git checkout . && git clean -fd
```

- `git checkout .` — reverts all tracked file modifications
- `git clean -fd` — removes all untracked files (`-f`) and directories (`-d`)

## 2. Persistent File Generation (Azurite, Postgres, etc.)

Sometimes files keep getting generated even after you think things are stopped — for example, Azurite storage files or Postgres data files.

**To resolve:**

1. **Stop/Remove all Docker containers** via Docker Desktop.
2. **Delete the generated files manually** once the containers are stopped.
3. **If Azurite files are still being generated**, the VS Code Azurite extension may be the cause. Open the **Command Palette** (F1) and run **"Azurite: Stop"** to shut down the extension's Azurite instance.

## 3. Ensure You're Logged In to CLI Tools

Many operations require active authentication. Make sure you're logged in to all relevant tools before running:

- **Azure CLI:**
  ```bash
  az login
  ```
- **Azure Developer CLI:**
  ```bash
  azd auth login
  ```
- **Azure Resource Groups Extension**

Verify your login status at any time:

```bash
az account show
azd auth login --check-status
gh auth status
```

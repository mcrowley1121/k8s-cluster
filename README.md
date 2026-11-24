# k8s-cluster

GitOps repository for managing Kubernetes cluster configuration with Flux.

## Development Environment

This repo uses a [Dev Container](https://containers.dev/) to provide a reproducible development environment with all required tools pre-installed.

### Prerequisites

One of:
- [VS Code](https://code.visualstudio.com/) with the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
- [devcontainer CLI](https://github.com/devcontainers/cli) (`npm install -g @devcontainers/cli`)

### Starting the container

**VS Code:** Open the repo folder and click **Reopen in Container** when prompted (or run `Dev Containers: Reopen in Container` from the command palette).

**CLI:**
```bash
devcontainer up --workspace-folder .
```

### Exec into the container

**VS Code:** The editor reconnects automatically inside the container.

**CLI:**
```bash
devcontainer exec --workspace-folder . bash
```

## Tool Installation

Tool installation happens in two stages.

### Stage 1 — Image build

The `Dockerfile` starts from `mcr.microsoft.com/devcontainers/base:debian`, copies the `mise` binary, and bakes all tools into the image at build time. This means container starts are fast — tools are already present and don't need to be downloaded each time. The layer only rebuilds when `mise.toml` changes.

Tools installed via `mise.toml`:

| Tool | Purpose |
|------|---------|
| `age` | Encryption for SOPS secrets |
| `direnv` | Per-directory environment variables |
| `flux2` | Flux GitOps CLI |
| `kubectl` | Kubernetes CLI |
| `kubens` | Fast namespace switching |
| `k9s` | Terminal Kubernetes UI |
| `sops` | Secret encryption/decryption |
| `pre-commit` | Git hook management |
| `uv` | Fast Python package manager (required by the pipx backend) |
| `commitizen` | Conventional commit tooling (installed via `pipx:commitizen`) |

Mise shims are on `PATH` for every process in the container — including git hooks and VS Code extensions — not just interactive shells. Direnv is wired into `/etc/bash.bashrc` so `.envrc` loads automatically in every terminal.

### Stage 2 — Container creation (`scripts/setup`)

After the container is created, `scripts/setup` runs once via `postCreateCommand`. It:

1. Creates a writable `~/.gitconfig` that includes the host's config (for git identity — see [Git Identity](#git-identity))
2. Runs `mise install` — a fast no-op if the image is current, catches drift if `mise.toml` changed since the last build
3. Installs pre-commit hooks:

```bash
pre-commit install
pre-commit install --hook-type commit-msg
```

4. Runs `gh auth setup-git` to wire `gh` as the HTTPS credential helper for GitHub

`safe.directory` and `push.autoSetupRemote` are set at the system level in the image so they apply regardless of what's in `~/.gitconfig`.

The commit-msg hook enforces [Conventional Commits](https://www.conventionalcommits.org/) via commitizen.

## GitHub Authentication (commit & push)

Browsers don't work inside containers, so authenticate on the **host first** — the container mounts `~/.config/gh` from the host, so credentials are present from the moment the container starts.

**One-time setup (host):**

Install `gh` if not already present:

| OS | Command |
|----|---------|
| macOS | `brew install gh` |
| Windows | `winget install GitHub.cli` |
| Debian/Ubuntu | `sudo apt install gh` |
| Fedora/RHEL | `sudo dnf install gh` |

Other platforms: [cli.github.com](https://cli.github.com)

Then authenticate — a browser window will open on the host. Use `--insecure-storage` so the token is written to `~/.config/gh` rather than the OS keychain — containers are Linux and cannot reach the macOS Keychain:

```bash
gh auth login --insecure-storage
```

Then open (or reopen) the container. `scripts/setup` calls `gh auth setup-git` automatically on each container start, which configures git to use `gh` as the HTTPS credential helper. No further action needed to commit and push.

> **Note:** If your `.envrc` sets a `GITHUB_TOKEN` environment variable, `gh` will prefer it over the stored credentials. An invalid or revoked token will silently break pushes. Remove it from `.envrc` if it is no longer needed.

## Git Identity

`user.email` and `user.name` are carried in automatically from the host's `~/.gitconfig`. The file is bind-mounted read-only and included by the container's git config, so whatever is set on the host is visible inside.

If you haven't configured them on the host yet, do so there first:

```bash
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

Then rebuild the container so the mount picks up the new values.

> **Windows (native, no WSL2):** `gh` stores credentials at `%APPDATA%\GitHub CLI` rather than `~/.config/gh`. The mount in `devcontainer.json` uses `${localEnv:HOME}/.config/gh` which works on macOS, Linux, and Windows with WSL2. On native Windows you will need to update the `source` path in the mount to point to `${localEnv:APPDATA}/GitHub CLI`.

## Personal Dotfiles (chezmoi)

Personal dotfiles managed by chezmoi are configured at the client level, not in this repo's `devcontainer.json`. This keeps the shared baseline clean and lets each contributor use their own shell config.

**VS Code:** Add to your user `settings.json`:
```json
"dotfiles.repository": "https://github.com/<you>/dotfiles",
"dotfiles.installCommand": "chezmoi apply"
```

**CLI:**
```bash
devcontainer up --workspace-folder . \
  --dotfiles-repository https://github.com/<you>/dotfiles \
  --dotfiles-install-command "chezmoi apply"
```

Dotfiles are applied before `postCreateCommand` runs, so your shell config is in place by the time the workspace is ready.

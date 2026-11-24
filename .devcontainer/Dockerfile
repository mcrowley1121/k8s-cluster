FROM mcr.microsoft.com/devcontainers/base:debian

# mise binary from the official image. Pin a tag (jdxcode/mise:vYYYY.M.P)
# so a rebuild next month doesn't silently change mise itself.
COPY --from=jdxcode/mise /usr/local/bin/mise /usr/local/bin/

# Activate direnv for interactive shells so the repo's .envrc is loaded
# automatically. direnv is a mise-managed tool; this installs the per-shell
# hook that actually evaluates .envrc. /etc/bash.bashrc is sourced by
# interactive non-login shells (what `devcontainer exec bash` starts) and
# survives personal dotfiles.
RUN echo 'eval "$(direnv hook bash)"' >> /etc/bash.bashrc

# Trust all directories (safe for a container the user controls) and set
# push.autoSetupRemote at the system level so they work regardless of what's
# in the user's bind-mounted ~/.gitconfig — and without writing to that file.
RUN git config --system --add safe.directory '*' \
    && git config --system push.autoSetupRemote true

USER vscode

# Shims on PATH make mise-managed tools visible to every process —
# git hooks, VS Code extensions — not just interactive shells.
# Trusting /workspaces lets mise read the repo's mise.toml without a prompt.
ENV PATH="/home/vscode/.local/share/mise/shims:${PATH}" \
    MISE_TRUSTED_CONFIG_PATHS="/workspaces"

# Install tools at image build time so they live in a cached layer.
# This layer only rebuilds when mise.toml changes; container recreates stay fast.
# uv must be installed before pipx: tools because the pipx backend uses uvx.
WORKDIR /tmp/mise-bootstrap
COPY --chown=vscode:vscode mise.toml .
RUN mise trust mise.toml \
    && mise install uv \
    && mise install
WORKDIR /

# Chezmoi Migration Design

**Date:** 2026-03-14
**Status:** Approved

## Overview

Migrate from a bare git repo dotfiles approach to chezmoi as the single dotfiles manager across multiple machines. Start fresh — the existing repo contents (`.zshrc`, `.cheat/`) are abandoned.

## Goals

- Manage all configuration from a single chezmoi source repo
- Support two Macs (work + personal) with tag-based conditional templating
- Use 1Password for secrets management (SSH keys, API tokens)
- Automate Homebrew package installation via chezmoi scripts
- Enable one-command bootstrap on a fresh machine
- Follow chezmoi best practices throughout

## Machines

| Machine | Tag | Notes |
|---------|-----|-------|
| Work Mac | `work` | Has company-internal tools, work email, proxy settings |
| Personal Mac | `personal` | Standard personal setup |
| Windows | — | Not used for development; low priority, deferred |

## Repository Structure

```
~/.local/share/chezmoi/                              # chezmoi source directory
├── .chezmoi.toml.tmpl                               # config template (prompts for machineType)
├── .chezmoiignore                                   # OS/machine-conditional ignores
├── dot_zshrc.tmpl                                   # shell config (built fresh)
├── dot_zprofile.tmpl                                # login shell env (Homebrew PATH, etc.)
├── dot_gitconfig.tmpl                               # git config (name/email per machine)
├── private_dot_ssh/
│   └── config.tmpl                                  # SSH config with 1Password agent
├── dot_config/
│   ├── starship.toml                                # prompt theme
│   └── ...                                          # other app configs
├── dot_Brewfile.tmpl                                # shared + conditional packages → ~/.Brewfile
├── run_onchange_after_install-packages.sh.tmpl      # Homebrew bundle (after, so ~/.Brewfile exists)
├── run_onchange_after_configure-macos.sh.tmpl       # macOS defaults
└── README.md                                        # setup instructions
```

Note: All source files live directly in the chezmoi source root — no `home/` subdirectory. Chezmoi maps source paths directly to the target (`$HOME`) by stripping prefixes like `dot_` and `private_`.

### Naming Conventions

- `private_` prefix — chezmoi sets restrictive file permissions (0600/0700)
- `dot_` prefix — maps to `.` in the target path
- `.tmpl` suffix — processed as a Go template
- `run_onchange_` prefix — script re-runs only when its content changes
- `before_` / `after_` — controls execution order relative to dotfile application

### Shell Startup File Split

- `dot_zprofile.tmpl` → `~/.zprofile` — runs once per login shell. Contains environment setup: Homebrew PATH (`eval "$(/opt/homebrew/bin/brew shellenv)"`), and other PATH modifications.
- `dot_zshrc.tmpl` → `~/.zshrc` — runs per interactive shell. Contains aliases, plugins, prompt config, and runtime secrets via `op read`.

## Configuration & Templating

### `.chezmoi.toml.tmpl`

Prompts once during `chezmoi init`, persists to `~/.config/chezmoi/chezmoi.toml`:

```toml
{{- $machineType := promptStringOnce . "machineType" "Machine type (personal/work)" -}}

[data]
    machineType = {{ $machineType | quote }}
    email = {{ if eq $machineType "work" }}"your.name@company.com"{{ else }}"your.name@personal.com"{{ end }}

[onepassword]
    command = "op"
```

### Template Usage

All machine-specific logic flows from the single `machineType` variable. Example in `dot_gitconfig.tmpl`:

```toml
[user]
    name = "Your Name"
    email = {{ .email | quote }}
{{ if eq .machineType "work" }}
[http]
    proxy = http://internal-proxy:8080
{{ end }}
```

### `.chezmoiignore`

Prevents irrelevant files from being applied:

```
{{ if ne .chezmoi.os "darwin" }}
.Brewfile
run_onchange_after_install-packages.sh.tmpl
run_onchange_after_configure-macos.sh.tmpl
{{ end }}

{{ if eq .machineType "personal" }}
.config/internal-tool/
{{ end }}
```

## Secrets Management — 1Password

Secrets are never stored in the repo. They are resolved at `chezmoi apply` time via the 1Password CLI.

### SSH Config (`private_dot_ssh/config.tmpl`)

```
{{ if eq .machineType "work" }}
Host github.work.com
    IdentityFile ~/.ssh/id_work
    IdentityAgent "~/Library/Group Containers/2BUA8C4S2C.com.1password/t/agent.sock"
{{ end }}

Host github.com
    IdentityFile ~/.ssh/id_personal
    IdentityAgent "~/Library/Group Containers/2BUA8C4S2C.com.1password/t/agent.sock"
```

### API Tokens in Shell Config (`dot_zshrc.tmpl`)

Secrets are resolved on-demand at shell startup via `op read`, rather than baked into the file at `chezmoi apply` time. This avoids writing plaintext secrets to disk:

```zsh
{{ if eq .machineType "work" }}
if command -v op &>/dev/null; then
    export INTERNAL_API_TOKEN=$(op read "op://Work/internal-api/token" 2>/dev/null)
fi
{{ end }}
```

Note: The `op` availability guard prevents shell startup errors when 1Password CLI is not installed or the user is not signed in.

### Git Commit Signing (in `dot_gitconfig.tmpl`)

```toml
{{ if eq .chezmoi.os "darwin" }}
[gpg]
    format = ssh

[gpg "ssh"]
    program = "/Applications/1Password.app/Contents/MacOS/op-ssh-sign"

[commit]
    gpgsign = true
{{ end }}
```

Note: The `op-ssh-sign` path is macOS-specific. Guarded by an OS conditional for future Windows support.

## Homebrew & System Setup

### `dot_Brewfile.tmpl`

Managed as a chezmoi target file at `~/.Brewfile`. Shared base with conditional sections:

```ruby
# Core tools
brew "chezmoi"
brew "git"
brew "fzf"
brew "ripgrep"
brew "jq"
brew "gh"
brew "starship"

# 1Password
cask "1password"
cask "1password-cli"

{{ if eq .machineType "work" }}
# Work-specific
brew "internal-tool"
cask "slack"
cask "zoom"
{{ end }}

{{ if eq .machineType "personal" }}
# Personal
cask "spotify"
cask "discord"
{{ end }}
```

### `run_onchange_after_install-packages.sh.tmpl`

Runs after dotfiles are applied (so `~/.Brewfile` already exists). Installs Homebrew if missing, then runs `brew bundle`. Re-runs only when Brewfile content or machineType changes:

```bash
#!/bin/bash
# Brewfile hash: {{ include "dot_Brewfile.tmpl" | sha256sum }}
# machineType: {{ .machineType }}

if ! command -v brew &> /dev/null; then
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
fi

brew bundle --global --no-lock
```

Note: `brew bundle --global` reads from `~/.Brewfile`, which chezmoi renders from `dot_Brewfile.tmpl`. The hash + machineType comments trigger `run_onchange_` re-execution when either the template content or the machine tag changes.

### `run_onchange_after_configure-macos.sh.tmpl`

Idempotent macOS defaults:

```bash
#!/bin/bash

defaults write NSGlobalDomain KeyRepeat -int 2
defaults write NSGlobalDomain InitialKeyRepeat -int 15
defaults write com.apple.finder AppleShowAllFiles -bool true

{{ if eq .machineType "work" }}
# Work-specific macOS settings
{{ end }}

# Restart Finder to apply changes
killall Finder

# Note: Keyboard repeat settings require logout/restart to take effect
```

## Bootstrap & Workflow

### Fresh Machine Bootstrap

Single command:

```bash
sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply your-github-username
```

Execution order:
1. Installs chezmoi
2. Clones dotfiles repo from GitHub
3. Prompts for `machineType` (personal/work)
4. Applies all templated dotfiles with correct permissions (including `~/.Brewfile`)
5. `run_onchange_after_install-packages` — installs Homebrew + packages (including 1Password CLI)
6. `run_onchange_after_configure-macos` — sets macOS defaults

Post-bootstrap manual step: run `chezmoi doctor` to verify all dependencies are present.

**Bootstrap prerequisite:** The `op read` calls in `.zshrc` run at shell startup, not at `chezmoi apply` time — so `chezmoi apply` itself does not require 1Password authentication. However, the first new shell session after apply will invoke `op read`, which requires 1Password to be signed in. On a fresh machine, consider a two-step bootstrap:

```bash
# Step 1: Clone dotfiles repo (no apply yet)
sh -c "$(curl -fsLS get.chezmoi.io)" -- init your-github-username
# Step 2: Apply dotfiles and install packages
chezmoi apply
# Step 3: Sign into 1Password (needed before opening a new shell)
eval $(op signin)
# Step 4: Verify
chezmoi doctor
```

### Day-to-Day

```bash
chezmoi edit ~/.zshrc     # edit a managed file
chezmoi diff              # preview changes
chezmoi apply             # apply changes
chezmoi add ~/.config/X   # add a new file to management
chezmoi cd && git add -A && git commit -m "update" && git push
```

### Sync Between Machines

```bash
chezmoi update            # git pull + chezmoi apply
```

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Config tool | chezmoi | Templating, multi-machine, 1Password integration, active community |
| Machine differentiation | Tag-based (`machineType`) | Portable, doesn't depend on hostname |
| Secrets backend | 1Password | Already in use, native chezmoi support, works on both Macs |
| Homebrew management | Shared Brewfile + conditional extras | DRY, single source of truth |
| Shell config | Start fresh | Old `.zshrc` had hardcoded paths, outdated setup |
| Windows | Deferred | Not used for development |

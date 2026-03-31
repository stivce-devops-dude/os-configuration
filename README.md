# os-configuration

Ansible project for configuring macOS and Arch Linux machines from a single codebase. macOS runs locally (`connection: local`), Arch Linux runs over SSH.

## Prerequisites

### Control node (macOS)
- Ansible installed (`brew install ansible`)
- Galaxy collections: `ansible-galaxy collection install -r requirements.yml`

### Arch target
- Fresh Arch install with `base`, `linux`, `openssh`, `python`, `sudo`, `networkmanager`
- SSH key deployed for the target user
- User in `wheel` group with passwordless sudo
- NetworkManager and sshd enabled

> Use the companion [arch.install](https://github.com/stivce-devops-dude/arch.install) script to bootstrap a fresh Arch install that meets these requirements.

## Quick start

```bash
# Install dependencies
ansible-galaxy collection install -r requirements.yml

# Create vault password file
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass

# Run against Arch (skip gaming packages)
./runit-minimal

# Run against Arch (full desktop + gaming)
./runit-full

# Run against macOS
ansible-playbook playbooks/macos.yml --diff
```

## Inventory

| Host | Group | IP | Connection |
|------|-------|----|------------|
| macbook | macos | localhost | local |
| archbox | archlinux, gaming | 10.0.0.133 | SSH (id_ed25519) |

Edit `inventory/hosts.yml` and `inventory/host_vars/` to match your setup.

### Group vars

| File | Purpose |
|------|---------|
| `group_vars/all/vars.yml` | Shared: user, timezone, home directories, shared packages |
| `group_vars/all/vault.yml` | Encrypted secrets (become password) |
| `group_vars/macos.yml` | Homebrew casks/formulas, `connection: local` |
| `group_vars/archlinux.yml` | Arch base packages, hostname, GPU flags |
| `group_vars/gaming.yml` | Enables NVIDIA by default for gaming hosts |

## Roles

### Shared
| Role | Description |
|------|-------------|
| `common` | Creates standard `$HOME` directories (Projects, Documents, etc.) |
| `packages-base` | Installs shared CLI tools (neovim, tmux, fzf, ripgrep, etc.) via Homebrew or pacman with cross-platform name mapping |

### macOS only
| Role | Description |
|------|-------------|
| `macos-update` | Runs `softwareupdate -ia` and installs Xcode CLI tools |
| `homebrew` | Installs Homebrew if not present |
| `macos-defaults` | Configures system preferences (Dock, Finder, keyboard, trackpad, screenshots, dark mode) |

### Arch only
| Role | Description |
|------|-------------|
| `pacman-optimization` | Enables multilib, reflector mirrors, parallel downloads, color, paccache timer |
| `makepkg-optimization` | Optimizes AUR build times: `MAKEFLAGS=-j$(nproc)`, tmpfs builddir, ccache, `-march=native`, multi-threaded zstd |
| `system-configuration` | Hostname, timezone, default editor (neovim), default shell (zsh), SSH hardening, sudo, GRUB config |
| `bootstrap-deps` | Installs yay (AUR helper) from source |
| `boot-optimization` | Masks `systemd-resolved`, disables `NetworkManager-wait-online` |
| `gpu` | NVIDIA and/or AMD driver stack with DKMS support, GRUB kernel params, initramfs modules |
| `packages-hyprland` | Full Wayland/Hyprland desktop: compositor, waybar, PipeWire audio, fonts, themes, Qt support, ly display manager |
| `packages-gaming` | Wine, Steam, gamemode, mangohud, gamescope (tagged `gaming`, opt-in) |

## Execution order

### macOS
1. System update + Xcode CLI tools
2. Homebrew installation
3. macOS defaults
4. Home directories
5. Packages (formulas + casks)

### Arch Linux
1. Pacman optimization (mirrors, multilib, parallel downloads)
2. Makepkg optimization (build flags, ccache)
3. System configuration (hostname, timezone, editor, shell, SSH, sudo, GRUB)
4. Bootstrap yay
5. Boot optimization
6. Home directories
7. Base packages
8. GPU drivers (conditional: `enable_nvidia` or `enable_amd`)
9. Hyprland desktop stack
10. Gaming packages (conditional: host in `gaming` group)

## Tags

Run specific parts of the playbook:

```bash
ansible-playbook playbooks/archlinux.yml --tags pacman      # mirrors + pacman.conf only
ansible-playbook playbooks/archlinux.yml --tags gpu          # GPU drivers only
ansible-playbook playbooks/archlinux.yml --tags gaming       # gaming packages only
ansible-playbook playbooks/archlinux.yml --tags hyprland     # desktop stack only
ansible-playbook playbooks/archlinux.yml --skip-tags gaming  # everything except gaming
```

## Vault

Sensitive values go in `inventory/group_vars/all/vault.yml`:

```bash
# Encrypt
ansible-vault encrypt inventory/group_vars/all/vault.yml

# Edit
ansible-vault edit inventory/group_vars/all/vault.yml
```

The vault password file path is configured in `ansible.cfg` as `.vault_pass`.

## Customization

- **Packages**: Edit `group_vars/all/vars.yml` (shared), `group_vars/macos.yml` (casks/formulas), `group_vars/archlinux.yml` (arch base)
- **GPU**: Set `enable_nvidia: true` or `enable_amd: true` in host/group vars. DKMS is default, override with `use_nvidia_dkms: false`
- **Timezone**: Change `timezone` in `group_vars/all/vars.yml`
- **Home directories**: Edit `home_directories` list in `group_vars/all/vars.yml`
- **macOS defaults**: Edit `roles/macos-defaults/defaults/main.yml` to add/remove preferences
- **AUR packages**: Listed in `roles/packages-hyprland/defaults/main.yml` and `roles/packages-gaming/defaults/main.yml`, prefer `-bin` variants

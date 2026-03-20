# How the Installation Scripts Work

This document explains how the installation and setup scripts in this repository work, covering control flow, key functions, required dependencies, environment variables, side effects, and safety notes.

---

## Table of Contents

1. [Overview](#overview)
2. [Main Entry Scripts](#main-entry-scripts)
   - [Distro-Hyprland.sh (Bootstrap)](#distro-hyprlandsh-bootstrap)
   - [copy.sh (Dotfiles Installer)](#copysh-dotfiles-installer)
3. [Helper Scripts](#helper-scripts)
4. [Required Dependencies](#required-dependencies)
5. [Optional / Auto-Detected Dependencies](#optional--auto-detected-dependencies)
6. [Environment Variables](#environment-variables)
7. [Files and Directories Modified or Created](#files-and-directories-modified-or-created)
8. [Startup Applications Enabled](#startup-applications-enabled)
9. [Backup Strategy](#backup-strategy)
10. [Safety Notes](#safety-notes)

---

## Overview

The repository contains two distinct stages of installation:

| Stage | Script | Purpose |
|-------|--------|---------|
| 1 | `Distro-Hyprland.sh` | Bootstrap: detects your distro, clones the matching package-install repo, and runs its `install.sh` to install Hyprland and all required packages |
| 2 | `copy.sh` | Dotfiles: copies the pre-configured files in this repository into `~/.config`, handling backups, user preferences, and hardware detection |

> **Important:** This repository contains *only dotfiles*. Package installation happens in separate, distro-specific repositories. `Distro-Hyprland.sh` clones those for you automatically.

---

## Main Entry Scripts

### `Distro-Hyprland.sh` (Bootstrap)

**Purpose:** Detects the running Linux distribution, installs `git` if missing, clones the matching distro-specific Hyprland install repository, and starts its `install.sh`.

**How to run:**

```bash
sh <(curl -L https://raw.githubusercontent.com/LinuxBeginnings/Hyprland-Dots/main/Distro-Hyprland.sh)
```

Or, after cloning this repo:

```bash
bash Distro-Hyprland.sh
```

**Control flow, step by step:**

```
1. Reads /etc/os-release to identify distro name and version.
   Exits with an error if /etc/os-release is not found.

2. Sets distro-specific variables based on detected OS:
   - Debian GNU/Linux  → PACKAGE_MANAGER=apt,    Distro=Debian-Hyprland
   - Ubuntu 24.04      → PACKAGE_MANAGER=apt,    Distro=Ubuntu-Hyprland, branch=24.04
   - Ubuntu 24.10      → PACKAGE_MANAGER=apt,    Distro=Ubuntu-Hyprland, branch=24.10
   - Ubuntu 25.04      → PACKAGE_MANAGER=apt,    Distro=Ubuntu-Hyprland, branch=25.04
   - Ubuntu 25.10      → PACKAGE_MANAGER=apt,    Distro=Ubuntu-Hyprland, branch=25.10
   - Ubuntu 26.04-dev  → PACKAGE_MANAGER=apt,    Distro=Ubuntu-Hyprland, branch=26.04-development
   - Arch-based        → PACKAGE_MANAGER=pacman, Distro=Arch-Hyprland
   - Fedora            → PACKAGE_MANAGER=dnf,    Distro=Fedora-Hyprland
   - OpenSUSE          → PACKAGE_MANAGER=zypper, Distro=OpenSUSE-Hyprland
   - NixOS             → PACKAGE_MANAGER=nix,    Distro=NixOS-Hyprland
   - Anything else     → prints error and exits

3. Checks whether git is installed.
   If not, installs it using the distro's package manager.

4. Determines the clone destination:
   - $HOME/<Distro-name>             (most distros)
   - $HOME/<Distro-name>-<version>   (Ubuntu, to allow multiple versions)

5. If the destination directory already exists:
   - cd into it
   - git stash (saves any local changes)
   - git pull (fetches the latest version)
   - Runs install.sh

6. If the destination directory does not exist:
   - git clone --depth=1 <repo-url> [with -b <branch> for Ubuntu]
   - cd into it
   - chmod +x install.sh
   - Runs install.sh
```

**Key variables:**

| Variable | Description |
|----------|-------------|
| `distro_name` | Full distro name from `/etc/os-release` |
| `distro_version` | Version ID from `/etc/os-release` |
| `PACKAGE_MANAGER` | Package manager command for this distro |
| `GIT_INSTALL_CMD` | Command to install git (e.g. `sudo apt install -y git`) |
| `Distro` | Short name of the distro install repo (e.g. `Arch-Hyprland`) |
| `Github_URL` | Full URL to the distro-specific GitHub repo |
| `Github_URL_branch` | Branch name (Ubuntu only) |
| `Distro_DIR` | Local path where the repo is cloned |

**Side effects:**
- May install `git` using the system package manager (requires `sudo`).
- Clones an external GitHub repository to `$HOME/<Distro-name>[-version]/`.
- Runs that repository's `install.sh`, which will install Hyprland packages (see the individual distro repos for details).

---

### `copy.sh` (Dotfiles Installer)

**Purpose:** Copies the dotfiles from this repository into `~/.config`. Handles fresh installs, upgrades, hardware detection, user preference prompts, backups, and restores.

**How to run:**

```bash
# Interactive menu (recommended):
./copy.sh

# Non-interactive modes:
./copy.sh --upgrade          # Upgrade an existing install
./copy.sh --express-upgrade  # Quick upgrade (requires v2.3.18+ already installed)
./copy.sh --tty              # Force plain text prompts instead of graphical menu
./copy.sh --help             # Show usage
```

**Run modes:**

| Mode | Flag | Behavior |
|------|------|----------|
| Install | _(menu or default)_ | Full fresh install with all prompts |
| Upgrade | `--upgrade` | Upgrades configs; preserves user customizations; prompts before overwriting |
| Express | `--express-upgrade` | Like upgrade but skips restore prompts and auto-cleans old backups (requires existing v2.3.18+) |
| Update | _(menu option)_ | Runs `git pull` to update the dotfiles repository itself, then returns to the menu |

**Detailed control flow:**

```
── INITIALIZATION ──────────────────────────────────────────────────────────────
1. Sources 6 helper scripts (exits if any required helper is missing):
     scripts/copy_menu.sh    – interactive menu UI
     scripts/lib_backup.sh   – backup and cleanup functions
     scripts/lib_detect.sh   – hardware/distro detection
     scripts/lib_prompts.sh  – user-facing prompts
     scripts/lib_apps.sh     – application enablement and editor selection
     scripts/lib_copy.sh     – copy phases and restore logic
     scripts/lib_update.sh   – git pull / repo update helper

2. Parses CLI arguments (--upgrade, --express-upgrade, --tty, --help).

3. Checks whether express mode is supported by reading the version file
   at ~/.config/hypr/vX.Y.Z. Express requires v2.3.18 or newer.

4. If no mode is given via CLI, shows the interactive copy_menu.
   Menu choices: Install / Upgrade / Express / Update / Quit.

── SAFETY CHECKS ───────────────────────────────────────────────────────────────
5. Exits immediately if run as root (EUID = 0).
6. Exits if rsync is not installed.

── DISTRO WARNING ──────────────────────────────────────────────────────────────
7. On Debian or Ubuntu: shows a warning that Hyprland v0.50+ is required.
   Asks the user to confirm before proceeding.

── HARDWARE DETECTION & AUTO-CONFIGURATION ─────────────────────────────────────
8. detect_nvidia_adjust()
     Uses lspci to detect an NVIDIA GPU.
     If found: uncomments NVIDIA-specific environment variables in
     ~/.config/hypr/configs/ENVariables.conf
     (LIBVA_DRIVER_NAME, __GLX_VENDOR_LIBRARY_NAME, NVD_BACKEND, GSK_RENDERER).

9. detect_vm_adjust()
     Uses hostnamectl to detect a virtual machine.
     If found: enables WLR_RENDERER_ALLOW_SOFTWARE=1 and adds a virtual monitor.

10. detect_nixos_adjust()
     If NixOS is detected: enables the NixOS-specific Polkit script.

── APP DETECTION & AUTO-ENABLEMENT ─────────────────────────────────────────────
11. enable_asusctl()    – if rog-control-center is installed → adds it to startup
12. enable_blueman()    – if blueman-applet is installed → adds it to startup
13. enable_ags()        – if ags is installed → enables it in startup + refresh scripts
14. enable_quickshell() – if qs is installed → enables it in startup
15. ensure_keybinds_init() – adds KeybindsLayoutInit.sh to Startup_Apps.conf
16. install_waybar_weather() – installs pre-built binary to /usr/local/bin
                               (or builds from Go source on NixOS)

── USER PROMPTS ────────────────────────────────────────────────────────────────
17. Keyboard layout:
     Auto-detects via localectl or setxkbmap.
     Prompts user to confirm or override.
     Writes result to ~/.config/hypr/configs/SystemSettings.conf.

18. Default editor:
     Checks for nvim, then vim.
     Prompts user to confirm choice.
     Writes EDITOR= to ~/.config/hypr/UserConfigs/01-UserDefaults.conf.

19. Monitor resolution:
     Asks whether the primary monitor is < 1440p or ≥ 1440p.
     Adjusts: kitty font size (14 vs 16), rofi font size, hyprlock config.

20. Clock format:
     Default is 24-hour.
     Optionally switches to 12-hour (AM/PM).
     Modifies: waybar modules JSON, hyprlock config, SDDM theme.conf (via sudo -n).

21. Express upgrade offer (upgrade mode only):
     If installed version is ≥ 2.3.18 and user hasn't used --express-upgrade,
     asks whether to proceed in express mode.

── COPY PHASE 1 (with prompts) ─────────────────────────────────────────────────
22. For each of: fastfetch, kitty, rofi, swaync
     - If ~/.config/<app> already exists: asks user to replace it.
         Yes → backs up the old directory with a timestamp, copies the new one.
         No  → skips.
     - If it does not exist: copies directly.
     - Rofi: restores any custom themes from the backup.

── WAYBAR (special handling) ────────────────────────────────────────────────────
23. Backs up existing ~/.config/waybar (copy, not move, so originals remain).
    Copies new waybar configuration.
    Restores: custom configs, styles, and user-modified modules from backup.

── COPY PHASE 2 (automatic, no prompts) ────────────────────────────────────────
24. For each of: btop, cava, hypr, Kvantum, qt5ct, qt6ct, swappy, wallust, wlogout
     Moves existing directory to <name>-backup-MMDD_HHMM, then copies new config.

25. Terminal configs (if the terminal is installed):
     ghostty → copies config/ghostty/ to ~/.config/ghostty/
     wezterm → copies config/wezterm/ to ~/.config/wezterm/

── WAYBAR-WEATHER CONFIG ────────────────────────────────────────────────────────
26. Install mode:  always copies fresh; prompts for Fahrenheit vs Celsius.
    Upgrade mode:  copies only if the config is missing.
    Express mode:  copies if missing; skips the F/C prompt.

── AGS CONFIG ───────────────────────────────────────────────────────────────────
27. If ags is installed:
     - Not present → copies directly.
     - Present     → asks user; Yes → backup and replace, No → skip.

── QUICKSHELL CONFIG ────────────────────────────────────────────────────────────
28. If qs is installed:
     - Removes default shell.qml if present (it blocks overview detection).
     - Not present → copies directly.
     - Present     → asks user; Yes → backup, copy, remove shell.qml, No → skip.
     - Ensures the overview/ subdirectory exists.
     - Migrates old "qs" startup commands to "qs -c overview".

── RESTORE HYPR ASSETS ──────────────────────────────────────────────────────────
29. restore_hypr_assets():
     Restores from backup: Monitor_Profiles/, animations/.
     Restores: monitors.conf, workspaces.conf.
     Does NOT restore wallpaper_effects/ on a fresh install
     (allows the default wallpaper to apply on first run).
     Skips restore prompts in express mode.

── RESTORE USER CONFIGS ─────────────────────────────────────────────────────────
30. restore_user_configs():
     Extracts non-duplicate exec-once lines from old Startup_Apps.conf.
     Extracts custom windowrule/layerrule lines.
     Writes extracted customizations back to UserConfigs/.

31. cleanup_duplicate_userconfigs():
     If installed version is ≤ 2.3.19: uses AWK to strip duplicate entries
     between base and user config files for Startup_Apps, WindowRules,
     and UserKeybinds.

32. restore_user_scripts():
     Copies custom scripts from the hypr backup back to UserScripts/.

── ROFI THEMES SYMLINKS ─────────────────────────────────────────────────────────
33. Ensures ~/.local/share/rofi/themes/ exists.
    Creates symlinks from ~/.config/rofi/themes/ into that directory.

── WALLPAPERS ────────────────────────────────────────────────────────────────────
34. Copies default wallpapers (5 files) to ~/Pictures/wallpapers/.
    Optionally offers to download ~1 GB of additional wallpapers from
    https://github.com/LinuxBeginnings/Wallpaper-Bank.git
    (clones the repo, copies contents, then deletes the clone).

── PERMISSIONS ──────────────────────────────────────────────────────────────────
35. chmod +x ~/.config/hypr/scripts/*
    chmod +x ~/.config/hypr/UserScripts/*
    chmod +x ~/.config/hypr/initial-boot.sh

── WAYBAR CONFIG SYMLINK ────────────────────────────────────────────────────────
36. Detects chassis type (desktop or laptop) via hostnamectl.
    Desktop → symlinks ~/.config/waybar/config → TOP-Default
    Laptop  → symlinks ~/.config/waybar/config → TOP-Default-Laptop
    Removes the variant not in use.

── SDDM WALLPAPER ────────────────────────────────────────────────────────────────
37. If /usr/share/sddm/themes/simple_sddm_2 exists:
     Copies the current wallpaper as the SDDM background (uses sudo -n,
     so it silently skips if sudo requires a password at that point).

── BACKUP CLEANUP ────────────────────────────────────────────────────────────────
38. Express mode: automatically removes all but the newest backup per app.
    Standard mode: prompts for each app that has more than one backup.

── WAYBAR STYLE SYMLINK ─────────────────────────────────────────────────────────
39. Creates/resets symlink:
     ~/.config/waybar/style.css → Extra-Prismatic-Glow.css

── FINAL INITIALIZATION ─────────────────────────────────────────────────────────
40. Runs: wallust run -s <wallpaper>
     Generates color schemes consumed by waybar, kitty, and rofi.

41. Checks that waybar-weather binary is available (prints a NixOS-specific
    warning if it is missing).

42. Prints success message and recommends logging out or rebooting.
```

---

## Helper Scripts

All helpers live in `scripts/` and are **sourced** (not executed) by `copy.sh`.

### `scripts/lib_backup.sh`

| Function | What it does |
|----------|-------------|
| `backup_dir <path>` | Moves a directory to `<path>-backup-MMDD_HHMM` |
| `get_backup_dirname` | Returns a timestamp string `back-up_MMDD_HHMM` |
| `cleanup_backups <prompt\|auto>` | In `auto` mode removes all but the newest backup per app; in `prompt` mode asks for each |

### `scripts/lib_detect.sh`

| Function | What it does |
|----------|-------------|
| `detect_nvidia_adjust` | Runs `lspci`; if NVIDIA GPU found, uncomments NVIDIA env vars in `ENVariables.conf` |
| `detect_vm_adjust` | Runs `hostnamectl`; if virtual machine found, enables software renderer |
| `detect_nixos_adjust` | If NixOS detected, enables the NixOS-specific Polkit startup script |
| `detect_waybar_config` | Returns `"desktop"` or `"laptop"` based on `hostnamectl` chassis type |

### `scripts/lib_prompts.sh`

| Function | What it does |
|----------|-------------|
| `prompt_detect_layout` | Auto-detects keyboard layout via `localectl` or `setxkbmap` |
| `prompt_keyboard_layout` | Prompts for confirmation/override; writes to `SystemSettings.conf` |
| `prompt_resolution_choice` | Prompts for < 1440p vs ≥ 1440p; adjusts font and hyprlock config |
| `prompt_clock_12h` | Toggles 24H → 12H; modifies waybar modules, hyprlock, and SDDM config |
| `apply_sddm_12h_format` | Edits `/usr/share/sddm/themes/simple_sddm_2/theme.conf` |
| `apply_sddm_12h_format_sequoia` | Edits the sequoia_2 SDDM variant |
| `prompt_express_upgrade` | Offers express mode if installed version ≥ 2.3.18 |

### `scripts/lib_apps.sh`

| Function | What it does |
|----------|-------------|
| `enable_asusctl` | Detects `asusctl`; appends `exec-once = rog-control-center` to startup |
| `enable_blueman` | Detects `blueman-applet`; appends it to startup |
| `enable_ags` | Detects `ags`; enables it in startup and waybar refresh scripts |
| `enable_quickshell` | Detects `qs`; enables it in startup |
| `ensure_keybinds_init` | Adds `KeybindsLayoutInit.sh` to `Startup_Apps.conf` |
| `choose_default_editor` | Prompts nvim/vim; writes `EDITOR=` to `01-UserDefaults.conf` |
| `install_terminal_configs` | Copies `ghostty.config` and `wezterm.lua` if those terminals are present |
| `install_waybar_weather_binary` | Installs prebuilt binary to `/usr/local/bin` (or uses AUR on Arch) |
| `install_waybar_weather_nixos` | Builds waybar-weather from Go source |
| `install_waybar_weather` | Wrapper that calls the binary or NixOS variant as appropriate |

### `scripts/lib_copy.sh`

| Function | What it does |
|----------|-------------|
| `copy_phase1` | Interactively replaces fastfetch, kitty, rofi, swaync |
| `copy_waybar` | Backs up and replaces waybar; restores user-modified configs/styles |
| `copy_phase2` | Automatically replaces btop, cava, hypr, Kvantum, qt5ct, qt6ct, swappy, wallust, wlogout |
| `restore_hypr_assets` | Restores animations/ and Monitor_Profiles/ from backup |
| `restore_user_configs` | Extracts custom startup/window-rule entries from backup and writes them to `UserConfigs/` |
| `cleanup_duplicate_userconfigs` | For installs ≤ v2.3.19: removes duplicate entries between base and user config files |
| `restore_user_scripts` | Copies custom scripts from backup back to `UserScripts/` |
| `restore_hypr_files` | Restores key Hyprland files (monitors.conf, workspaces.conf) |

### `scripts/lib_update.sh`

| Function | What it does |
|----------|-------------|
| `run_repo_update` | Verifies the working directory is the Hyprland-Dots root, stashes uncommitted changes, runs `git pull --ff-only`, logs results, and waits for a keypress |

### `scripts/copy_menu.sh`

Provides `show_copy_menu()`, which renders an interactive whiptail (or basic TTY) menu offering: Install / Upgrade / Express / Update / Quit.

---

## Required Dependencies

These must be present **before** running `copy.sh`:

| Dependency | Why it is needed |
|------------|-----------------|
| `rsync` | All config copying is done with rsync; `copy.sh` exits immediately if it is missing |
| `git` | Needed to clone/update the repository; `Distro-Hyprland.sh` installs it automatically |
| `bash` | The scripts use Bash-specific features (`[[ ]]`, `declare -f`, etc.) |
| `tput` | Used for colored terminal output |

---

## Optional / Auto-Detected Dependencies

These are detected at runtime. The scripts adjust behavior automatically:

| Dependency | Effect if present |
|------------|------------------|
| `lspci` | Enables NVIDIA GPU detection |
| `hostnamectl` | Enables VM detection and desktop/laptop waybar profile selection |
| `localectl` | Enables automatic keyboard layout detection |
| `setxkbmap` | Fallback keyboard layout detection |
| `wallust` | Required to generate color schemes at the end of `copy.sh`; `copy.sh` will attempt to run it |
| `ags` | AGS config is copied and AGS is added to startup |
| `qs` (Quickshell) | Quickshell config is copied and added to startup |
| `asusctl` | `rog-control-center` is added to startup |
| `blueman-applet` | Blueman is added to startup |
| `nvim` / `vim` | Offered as the default `$EDITOR` |
| `ghostty` | ghostty terminal config is installed |
| `wezterm` | WezTerm config is installed |
| `go` | Needed to build `waybar-weather` from source on NixOS |
| `yay` / `paru` | Used on Arch to install `waybar-weather` from the AUR |
| `sudo` | Used non-interactively (`sudo -n`) for SDDM config changes |

---

## Environment Variables

### Set by `copy.sh` at runtime (internal)

| Variable | Description |
|----------|-------------|
| `DOTFILES_DIR` | Absolute path to the cloned Hyprland-Dots repo (exported) |
| `UPGRADE_MODE` | `1` if upgrading an existing install |
| `EXPRESS_MODE` | `1` if running in express upgrade mode |
| `RUN_MODE` | `"install"`, `"upgrade"`, or `"express"` |
| `COPY_TUI_BACKEND` | Set to `"basic"` when `--tty` is passed, to force plain prompts |

### Written to Hyprland config files

These are written into `~/.config/hypr/` files and read by Hyprland on startup:

| Variable / Setting | File | Notes |
|--------------------|------|-------|
| `$kb_layout` | `configs/SystemSettings.conf` | Keyboard layout (e.g. `us`) |
| `$kb_variant` | `configs/SystemSettings.conf` | Keyboard variant |
| `EDITOR` | `UserConfigs/01-UserDefaults.conf` | Default text editor |
| `$term` | `UserConfigs/01-UserDefaults.conf` | Terminal (default: `kitty`) |
| `$files` | `UserConfigs/01-UserDefaults.conf` | File manager (default: `thunar`) |
| `$Search_Engine` | `UserConfigs/01-UserDefaults.conf` | Web search URL |
| `HYPRCURSOR_THEME` | `configs/ENVariables.conf` | Cursor theme |
| `HYPRCURSOR_SIZE` | `configs/ENVariables.conf` | Cursor size |
| `WLR_RENDERER_ALLOW_SOFTWARE` | `configs/ENVariables.conf` | Set to `1` for VMs |
| `LIBVA_DRIVER_NAME` | `configs/ENVariables.conf` | Uncommented automatically for NVIDIA |
| `__GLX_VENDOR_LIBRARY_NAME` | `configs/ENVariables.conf` | Uncommented automatically for NVIDIA |
| `NVD_BACKEND` | `configs/ENVariables.conf` | Uncommented automatically for NVIDIA |
| `GSK_RENDERER` | `configs/ENVariables.conf` | Uncommented automatically for NVIDIA |

---

## Files and Directories Modified or Created

### Directories written to `~/.config/`

| Path | Notes |
|------|-------|
| `~/.config/hypr/` | Main Hyprland config (copied in Phase 2) |
| `~/.config/waybar/` | Status bar config (smart merge with backups) |
| `~/.config/rofi/` | App launcher themes (Phase 1, with prompt) |
| `~/.config/kitty/` | Terminal config (Phase 1, with prompt) |
| `~/.config/fastfetch/` | System info display (Phase 1, with prompt) |
| `~/.config/swaync/` | Notification center (Phase 1, with prompt) |
| `~/.config/btop/` | System monitor (Phase 2, automatic) |
| `~/.config/cava/` | Audio visualizer (Phase 2, automatic) |
| `~/.config/Kvantum/` | Qt theme engine (Phase 2, automatic) |
| `~/.config/qt5ct/` | Qt5 settings (Phase 2, automatic) |
| `~/.config/qt6ct/` | Qt6 settings (Phase 2, automatic) |
| `~/.config/swappy/` | Screenshot tool (Phase 2, automatic) |
| `~/.config/wallust/` | Dynamic color scheme generator (Phase 2, automatic) |
| `~/.config/wlogout/` | Logout menu (Phase 2, automatic) |
| `~/.config/ghostty/` | ghostty terminal (if installed) |
| `~/.config/wezterm/` | WezTerm terminal (if installed) |
| `~/.config/ags/` | AGS widget shell (if installed, with prompt) |
| `~/.config/quickshell/` | Quickshell widget shell (if installed, with prompt) |

### Other locations

| Path | Notes |
|------|-------|
| `~/.local/share/rofi/themes/` | Symlinks to `~/.config/rofi/themes/` |
| `~/.local/bin/waybar-weather` | Binary installed by `install_waybar_weather_binary` |
| `~/Pictures/wallpapers/` | Default wallpapers (and optional ~1 GB pack) |
| `~/Pictures/Screenshots/` | Created on first screenshot |
| `/usr/local/bin/waybar-weather` | Alternate binary install location |
| `/usr/share/sddm/themes/*/theme.conf` | Modified for 12-hour clock (uses `sudo -n`) |

### Backup directories

Old configs are moved to timestamped directories before being replaced:

```
~/.config/<app>-backup-MMDD_HHMM/
```

Examples: `~/.config/hypr-backup-0320_1430/`, `~/.config/waybar-backup-0320_1430/`

### Version and flag files

| File | Purpose |
|------|---------|
| `~/.config/hypr/v2.3.21` | Version marker; used to detect the installed dotfiles version |
| `~/.config/hypr/.initial_startup_done` | Prevents `initial-boot.sh` from running on subsequent boots |

### Installation log

Every run of `copy.sh` appends output to:

```
Copy-Logs/install-<timestamp>.log
```

(created in the Hyprland-Dots repository directory)

### User customization files (never overwritten during upgrades)

| Path | Purpose |
|------|---------|
| `~/.config/hypr/UserConfigs/*.conf` | User settings, keybinds, window rules, env vars |
| `~/.config/hypr/UserScripts/*.sh` | User-added scripts |

---

## Startup Applications Enabled

`copy.sh` does not enable systemd services. Instead it edits Hyprland's `exec-once` startup config. The following programs are started automatically when Hyprland launches:

| Program | Config file | Condition |
|---------|------------|-----------|
| `swww-daemon` | `configs/Startup_Apps.conf` | Always |
| `waybar` | `configs/Startup_Apps.conf` | Always |
| `swaync` | `configs/Startup_Apps.conf` | Always |
| `hypridle` | `configs/Startup_Apps.conf` | Always |
| `xdg-desktop-portal-hyprland` | `configs/Startup_Apps.conf` | Always |
| `$scriptsDir/Polkit.sh` | `configs/Startup_Apps.conf` | Non-NixOS |
| `$scriptsDir/Polkit-NixOS.sh` | `configs/Startup_Apps.conf` | NixOS only |
| `$scriptsDir/KeybindsLayoutInit.sh` | `configs/Startup_Apps.conf` | Always (added by `ensure_keybinds_init`) |
| `ags` | `configs/Startup_Apps.conf` | If `ags` is installed |
| `qs -c overview` | `configs/Startup_Apps.conf` | If `qs` is installed |
| `blueman-applet` | `configs/Startup_Apps.conf` | If `blueman-applet` is installed |
| `rog-control-center` | `configs/Startup_Apps.conf` | If `asusctl` is installed |

Users can add their own startup entries in `~/.config/hypr/UserConfigs/Startup_Apps.conf`; these are preserved across upgrades.

---

## Backup Strategy

`copy.sh` is designed to be **non-destructive**:

- Before replacing any config directory, the existing one is **renamed** (not deleted) to `<name>-backup-MMDD_HHMM/`.
- The restore logic then re-applies your personal customizations (keybinds, window rules, startup entries, monitor profiles, etc.) on top of the freshly copied defaults.
- **Backups are not deleted automatically** in standard mode; the script will prompt you if multiple backups accumulate. In express mode, all but the most recent backup per app are removed automatically.
- Backup directories remain at `~/.config/` and must be deleted manually if you no longer need them.

---

## Safety Notes

- **Do not run `copy.sh` as root.** The script explicitly checks for `EUID = 0` and exits.
- **`rsync` is required.** The script exits with a clear error if it is missing.
- **`Distro-Hyprland.sh` uses `sudo`** to install `git` if missing and to run the distro-specific `install.sh`. Ensure you have sudo privileges.
- **SDDM modifications use `sudo -n`** (non-interactive). If sudo requires a password at that point, the SDDM clock format change is silently skipped—no harm done.
- **Ubuntu / Debian:** These dotfiles require Hyprland v0.50 or newer. `copy.sh` displays a warning and confirmation prompt before proceeding.
- **NVIDIA users:** After installation, check `~/.config/hypr/UserConfigs/ENVariables.conf`. The NVIDIA environment variables are uncommented automatically if an NVIDIA GPU is detected, but you may need to adjust them for your driver version.
- **The wallpaper bank download is optional** (~1 GB). You will be prompted before any download begins.
- **Logs** are written to `Copy-Logs/` inside the Hyprland-Dots repo directory, not to your home directory.

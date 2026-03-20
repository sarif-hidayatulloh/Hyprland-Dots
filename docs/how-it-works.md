# How the Scripts Work

> A complete reference for every installer, copy, and runtime script in **KooL's Hyprland-Dots**.

---

## Table of Contents

1. [Script Tree](#script-tree)
2. [Entry Points](#entry-points)
3. [End-to-End Installation Flow](#end-to-end-installation-flow)
   - [Stage 0 – Bootstrap (Distro-Hyprland.sh)](#stage-0--bootstrap-distro-hyprlandsh)
   - [Stage 1 – Menu Selection (copy.sh + copy_menu.sh)](#stage-1--menu-selection-copysh--copy_menush)
   - [Stage 2 – Safety Checks & Environment Setup](#stage-2--safety-checks--environment-setup)
   - [Stage 3 – Interactive Prompts](#stage-3--interactive-prompts)
   - [Stage 4 – Config Copy Phases](#stage-4--config-copy-phases)
   - [Stage 5 – App-Specific Extras](#stage-5--app-specific-extras)
   - [Stage 6 – Restore & Cleanup](#stage-6--restore--cleanup)
   - [Stage 7 – Final Symlinks & Wallust Init](#stage-7--final-symlinks--wallust-init)
4. [Modes of Operation](#modes-of-operation)
5. [Library Scripts Reference](#library-scripts-reference)
6. [Archive Scripts Reference](#archive-scripts-reference)
7. [Runtime / UserScripts Reference](#runtime--userscripts-reference)
8. [Prerequisites](#prerequisites)
9. [What Gets Changed on Your System](#what-gets-changed-on-your-system)
   - [Directories Created / Replaced in ~/.config](#directories-created--replaced-in-config)
   - [Backups Created](#backups-created)
   - [Symlinks Created](#symlinks-created)
   - [System Files Modified (sudo)](#system-files-modified-sudo)
   - [Executables Installed to /usr/bin](#executables-installed-to-usrbin)
10. [Environment Variables & Config Toggles](#environment-variables--config-toggles)
11. [Safe-Run Guidance](#safe-run-guidance)
12. [Reverting Changes](#reverting-changes)

---

## Script Tree

```
Hyprland-Dots/
│
├── Distro-Hyprland.sh          ← Bootstrap: detects distro, clones matching
│                                  installer repo, runs its install.sh
│
├── copy.sh                     ← Main dotfiles installer/upgrader
│   ├── sources: scripts/copy_menu.sh
│   ├── sources: scripts/lib_backup.sh
│   ├── sources: scripts/lib_detect.sh
│   ├── sources: scripts/lib_prompts.sh
│   ├── sources: scripts/lib_apps.sh
│   ├── sources: scripts/lib_copy.sh
│   └── sources: scripts/lib_update.sh
│
├── scripts/
│   ├── copy_menu.sh            ← Interactive menu (whiptail or plain text)
│   ├── lib_backup.sh           ← Backup / cleanup helpers
│   ├── lib_detect.sh           ← Nvidia / VM / NixOS / waybar chassis detection
│   ├── lib_prompts.sh          ← Keyboard, resolution, clock, express prompts
│   ├── lib_apps.sh             ← App enablement (ags, quickshell, asusctl…)
│                                  + waybar-weather installer
│   ├── lib_copy.sh             ← Copy phases 1 & 2; waybar; restore helpers
│   └── lib_update.sh           ← git stash + pull (repo update helper)
│
├── archive/
│   ├── release.sh              ← Download & install from GitHub Releases tarball
│   ├── upgrade.sh              ← Semi-manual per-directory rsync upgrade
│   └── update-dots.sh          ← Lightweight git fetch/pull updater
│
└── config/hypr/
    ├── initial-boot.sh         ← Runs once on first Hyprland login (wallpaper,
    │                              GTK theme, Kvantum, cursor)
    ├── scripts/                ← Runtime helper scripts (keybinds, audio,
    │                              brightness, screenshots, wallpaper…)
    └── UserScripts/            ← User-facing scripts (rofi beats, wallpaper
                                   selection, weather, ZSH theme…)
```

---

## Entry Points

| Script | When to use |
|--------|------------|
| `Distro-Hyprland.sh` | **Fully automated first-time setup** – run via `curl` on a supported distro. Clones the matching distro installer and launches it. |
| `copy.sh` | **Recommended** install/upgrade of dotfiles only (no package installation). Run after cloning this repo. |
| `archive/release.sh` | Install from the latest GitHub *Release* tarball (one version behind `main`). |
| `archive/upgrade.sh` | Semi-manual per-directory `rsync` upgrade with interactive diff review. |
| `archive/update-dots.sh` | Lightweight `git fetch + pull` to refresh the local Hyprland-Dots clone. |

---

## End-to-End Installation Flow

### Stage 0 – Bootstrap (`Distro-Hyprland.sh`)

```bash
sh <(curl -L https://raw.githubusercontent.com/LinuxBeginnings/Hyprland-Dots/main/Distro-Hyprland.sh)
```

1. Reads `/etc/os-release` to identify the distribution.
2. Selects the matching install repo URL and local directory name:

   | Distro | Repo cloned |
   |--------|------------|
   | Arch / Manjaro (pacman) | `LinuxBeginnings/Arch-Hyprland` |
   | Fedora (dnf) | `LinuxBeginnings/Fedora-Hyprland` |
   | OpenSUSE Tumbleweed (zypper) | `LinuxBeginnings/OpenSUSE-Hyprland` |
   | Debian GNU/Linux | `LinuxBeginnings/Debian-Hyprland` |
   | Ubuntu 24.04 / 25.10 / 26.04-dev | `LinuxBeginnings/Ubuntu-Hyprland` (branch-specific) |
   | NixOS | `LinuxBeginnings/NixOS-Hyprland` |

3. Installs `git` if missing (using the distro's package manager).
4. Clones the installer repo (shallow clone, `--depth=1`) or `git pull`-updates it if already present.
5. Runs `install.sh` inside the cloned installer directory — *that* script installs Hyprland packages and optionally pulls this dotfiles repo.

> **Note:** `Distro-Hyprland.sh` does **not** install packages itself; it delegates to the distro-specific installer.

---

### Stage 1 – Menu Selection (`copy.sh` + `copy_menu.sh`)

Running `./copy.sh` without arguments presents an interactive menu (using **whiptail** if available, otherwise plain text):

```
      KooL's Hyprland Dotfiles
  Select what you would like to do:
  1) Install  – Fresh copy
  2) Upgrade  – Backups + prompts
  3) Express  – Skips restores & wallpapers (requires v2.3.18+)
  4) Update   – Stash + git pull
  5) Quit
```

CLI flags bypass the menu:

```bash
./copy.sh --upgrade           # Go straight to upgrade mode
./copy.sh --express-upgrade   # Go straight to express upgrade
./copy.sh --tty               # Force plain-text prompts (no whiptail)
./copy.sh --help              # Print usage
```

---

### Stage 2 – Safety Checks & Environment Setup

After mode selection, `copy.sh` performs the following before touching any files:

1. **Root guard** – exits immediately if `EUID == 0`.
2. **`rsync` check** – exits if `rsync` is not installed.
3. **Ubuntu/Debian version warning** – warns about minimum Hyprland v0.50 requirement and asks the user to confirm.
4. **Log file creation** – `Copy-Logs/install-<timestamp>_dotfiles.log` is created in the repo directory.
5. **`xdg-user-dirs-update`** – refreshes XDG home directories if the command is available.
6. **GPU / VM / NixOS detection** (via `lib_detect.sh`):
   - Nvidia GPU → uncomments Nvidia env vars in `config/hypr/configs/ENVariables.conf`, sets `no_hardware_cursors = 1`.
   - Virtual machine → enables software renderer env var, uncomments a virtual monitor config.
   - NixOS → adds `Polkit-NixOS.sh` to the startup overlay and disables the default `Polkit.sh`.
7. **Bibata cursor** – if `~/.icons/Bibata-Modern-Ice/hyprcursors` exists, activates `HYPRCURSOR_THEME` in `ENVariables.conf`.

---

### Stage 3 – Interactive Prompts

These prompts gather user preferences *before* any files are copied:

| Prompt | Effect |
|--------|--------|
| **Keyboard layout** | Auto-detected via `localectl`/`setxkbmap`; user confirms or overrides. Written to `config/hypr/configs/SystemSettings.conf`. |
| **Optional app enablement** | Detects `asusctl`, `blueman-applet`, `ags`, `quickshell` and adds `exec-once` entries to the startup overlay. |
| **KeybindsLayoutInit** | Always added to startup overlay. |
| **waybar-weather** | Installed to `/usr/bin/waybar-weather` (prebuilt binary or AUR on Arch; built from source on NixOS). |
| **Default editor** | Prompts to set `nvim` or `vim` as the `EDITOR` env in `UserDefaults.conf`. |
| **Monitor resolution** | `< 1440p` reduces Kitty font size, swaps hyprlock config, and reduces Rofi font sizes. `≥ 1440p` (default) leaves defaults. |
| **12-hour clock** | Edits `config/waybar/Modules`, `hyprlock.conf`, and SDDM `theme.conf` (if installed) to use 12H/AM-PM format. |
| **Express upgrade** | In upgrade mode, asks whether to skip restore prompts and trim old backups. |

---

### Stage 4 – Config Copy Phases

#### Phase 1 (`copy_phase1`)

For each of `fastfetch`, `kitty`, `rofi`, `swaync`:
- If `~/.config/<app>` already exists → prompt user; on **Yes**, move existing to `~/.config/<app>-backup-<timestamp>` then copy fresh.
- On **No** → skip (existing config kept).
- If no existing config → copy directly from repo.
- Special case for `rofi`: custom themes from the backup's `themes/` and `0-shared-fonts.rasi` are re-merged into the new config.

#### Waybar (`copy_waybar`)

- Same backup-or-skip prompt as Phase 1.
- After replacing, restores any extra configs/styles found in the backup that are not in the new copy (non-destructive merge).
- Re-applies `UserModules` from backup if it exists.

#### Phase 2 (`copy_phase2`)

For each of `btop`, `cava`, `hypr`, `Kvantum`, `qt5ct`, `qt6ct`, `swappy`, `wallust`, `wlogout`:
- Existing directories are backed up **automatically** (no prompt).
- Fresh copy is made from repo.
- Calls `install_terminal_configs` to install `ghostty` and `wezterm` configs to `~/.config/ghostty/` and `~/.config/wezterm/`.

---

### Stage 5 – App-Specific Extras

After the core copy phases, `copy.sh` handles optional extras in order:

1. **waybar-weather config** (`config/waybar-weather/`) – copied to `~/.config/waybar-weather/`; prompts for Fahrenheit/Celsius on first install.
2. **ags** – if `ags` binary is present, copies `config/ags/` to `~/.config/ags/` (with backup prompt).
3. **quickshell** – if `qs` binary is present, copies `config/quickshell/` to `~/.config/quickshell/`, ensures `overview` sub-config exists, and updates any old `exec-once = qs` startup commands to `exec-once = qs -c overview`.

---

### Stage 6 – Restore & Cleanup

After copies are complete:

1. **Restore hypr assets** (`restore_hypr_assets`) – copies `Monitor_Profiles/`, `animations/`, and (in upgrade mode) `wallpaper_effects/` from the hypr backup back to `~/.config/hypr/`. Also restores `monitors.conf` and `workspaces.conf`.

2. **Restore UserConfigs** (`restore_user_configs`) – restores your personal settings from backup:
   - For installs from **v2.3.19+**: prompts once then uses `rsync` to restore the entire `UserConfigs/` directory.
   - For older installs: prompts per file for `01-UserDefaults.conf`, `ENVariables.conf`, `LaptopDisplay.conf`, `Laptops.conf`, `UserDecorations.conf`, `UserAnimations.conf`, `UserKeybinds.conf`, `UserSettings.conf`. For `Startup_Apps.conf` and `WindowRules.conf`, uses `compose_overlay_from_backup` to extract only user-added entries.

3. **Restore UserScripts** (`restore_user_scripts`) – prompts to restore these specific scripts if found in backup: `RofiBeats.sh`, `Weather.py`, `Weather.sh`. (Skipped in express mode.)

4. **Restore hypr files** (`restore_hypr_files`) – prompts to restore `hyprlock.conf` and `hypridle.conf` from the backup if they exist. (Skipped in express mode.)

5. **Duplicate UserConfigs cleanup** – for upgrades from versions ≤ 2.3.19, removes `exec-once` / `windowrule` lines from UserConfigs that are exact duplicates of the base configs.

6. **Rofi themes symlinks** – creates `~/.local/share/rofi/themes/` and symlinks all `~/.config/rofi/themes/*` into it.

7. **Wallpapers** – copies `wallpapers/` from the repo to `~/Pictures/wallpapers/`. Optionally offers to download an additional ~1 GB wallpaper pack from `LinuxBeginnings/Wallpaper-Bank`.

8. **Script permissions** – makes `~/.config/hypr/scripts/*`, `~/.config/hypr/UserScripts/*`, and `~/.config/hypr/initial-boot.sh` executable.

9. **SDDM wallpaper** – if `/usr/share/sddm/themes/simple_sddm_2` exists, applies the current wallpaper as the SDDM background using `sudo -n` (non-interactively; skipped if sudo is not cached).

10. **Backup cleanup** – in Express mode: auto-deletes all but the newest backup for each app. In standard mode: prompts per app.

---

### Stage 7 – Final Symlinks & Wallust Init

1. **Waybar config symlink** – creates `~/.config/waybar/config` → `TOP-Default` (desktop) or `TOP-Default-Laptop` (other chassis).
2. **Waybar style symlink** – creates `~/.config/waybar/style.css` → `style/Extra-Prismatic-Glow.css`.
3. **Removes legacy Waybar configs** – deletes old version-tagged configs (`[TOP] Default (old v1–v4)`, etc.).
4. **`wallust run`** – initialises wallust colour theme from the current wallpaper (`~/.config/hypr/wallpaper_effects/.wallpaper_current`), which propagates colours to Waybar, Kitty, Rofi, etc.
5. Prints final success message and recommends a logout/reboot.

---

## Modes of Operation

| Mode | What it does |
|------|-------------|
| **install** | Full fresh copy. Phase 1 apps prompt before replacing. Phase 2 apps auto-backup and replace. |
| **upgrade** | Same as install, but prompts for express mode; restores user configs/scripts from backup. |
| **express** | Skips restore prompts, SDDM questions, and wallpaper download; auto-trims old backups. Requires currently installed version ≥ 2.3.18. |
| **update** (menu option 4) | Runs `run_repo_update`: stashes local changes, `git pull`, shows a summary. Does **not** copy any configs. |

---

## Library Scripts Reference

### `scripts/copy_menu.sh`

Provides `show_copy_menu(express_supported)`. Displays the main menu using **whiptail** (TUI dialog) if present, otherwise falls back to a plain numbered list. Sets the global `COPY_MENU_CHOICE`.

---

### `scripts/lib_backup.sh`

| Function | What it does |
|----------|-------------|
| `get_backup_dirname` | Returns a timestamp string: `back-up_MMDD_HHMM` |
| `backup_dir(dir, [log])` | Moves `dir` to `dir-backup-<timestamp>` |
| `cleanup_backups(mode, [log])` | Finds multiple backups for the same base dir under `~/.config`; in `auto` mode deletes all but the newest; in `prompt` mode asks the user |

---

### `scripts/lib_detect.sh`

| Function | Trigger condition | Effect |
|----------|------------------|--------|
| `detect_nvidia_adjust` | `lspci` shows Nvidia GPU | Uncomments `LIBVA_DRIVER_NAME`, `__GLX_VENDOR_LIBRARY_NAME`, `NVD_BACKEND`, `GSK_RENDERER` env vars; sets `no_hardware_cursors = 1` |
| `detect_vm_adjust` | `hostnamectl` shows `Chassis: vm` | Enables `WLR_RENDERER_ALLOW_SOFTWARE`; uncomments Virtual-1 monitor entry; sets `no_hardware_cursors = 1` |
| `detect_nixos_adjust` | `hostnamectl` shows `NixOS` | Adds `Polkit-NixOS.sh` to startup overlay; adds `Polkit.sh` to disable list |
| `detect_waybar_config` | `hostnamectl Chassis: desktop` | Returns `"desktop"` (TOP-Default) vs `"laptop"` (TOP-Default-Laptop) |

---

### `scripts/lib_prompts.sh`

| Function | What it prompts / does |
|----------|----------------------|
| `prompt_detect_layout` | Auto-detects keyboard layout via `localectl` or `setxkbmap` |
| `prompt_keyboard_layout(layout, log)` | Confirms/overrides layout; writes `kb_layout` to `SystemSettings.conf` |
| `prompt_resolution_choice` | Asks `< 1440p` or `≥ 1440p`; returns choice string |
| `prompt_clock_12h(log)` | Edits `waybar/Modules`, `hyprlock.conf`, SDDM `theme.conf` for 12H format |
| `apply_sddm_12h_format(dir, log)` | Edits `simple-sddm` / `simple_sddm_2` theme.conf using `sudo -n` |
| `apply_sddm_12h_format_sequoia(dir, log)` | Edits `sequoia_2` SDDM theme.conf |
| `prompt_express_upgrade(supported, log)` | Offers express mode in upgrade flow |

---

### `scripts/lib_apps.sh`

| Function | What it does |
|----------|-------------|
| `enable_asusctl(log)` | Adds `exec-once = rog-control-center` to startup overlay if `asusctl` is installed |
| `enable_blueman(log)` | Adds `exec-once = blueman-applet` to startup overlay if `blueman-applet` is installed |
| `enable_ags(log)` | Adds `exec-once = ags`; uncomments `ags -q && ags &` in `Refresh.sh` / `RefreshNoWaybar.sh` |
| `enable_quickshell(log)` | Adds `exec-once = qs`; uncomments `pkill qs && qs &` in refresh scripts |
| `ensure_keybinds_init(log)` | Ensures `KeybindsLayoutInit.sh` is in the startup overlay |
| `install_terminal_configs(log)` | Copies `ghostty.config` → `~/.config/ghostty/config`; copies `wezterm.lua` → `~/.config/wezterm/wezterm.lua` |
| `choose_default_editor(log)` | Prompts to set `nvim` or `vim` as `EDITOR` in `UserDefaults.conf` |
| `install_waybar_weather_binary(log)` | On Arch: tries AUR via `yay`; falls back to installing bundled compressed binary from `assets/waybar-weather.gz` to `/usr/bin/waybar-weather` |
| `install_waybar_weather_nixos(log)` | Builds `waybar-weather` from source using `go build`; installs to `~/.local/bin/waybar-weather` |
| `install_waybar_weather(log)` | Dispatcher: calls NixOS or non-NixOS variant automatically |

---

### `scripts/lib_copy.sh`

| Function | What it does |
|----------|-------------|
| `copy_phase1(log, mode)` | Prompts-backed copy of `fastfetch`, `kitty`, `rofi`, `swaync` |
| `copy_waybar(log)` | Prompts-backed copy of `waybar` with backup-merge logic |
| `copy_phase2(log)` | Auto-backed copy of `btop`, `cava`, `hypr`, `Kvantum`, `qt5ct`, `qt6ct`, `swappy`, `wallust`, `wlogout`; then calls `install_terminal_configs` |
| `restore_hypr_assets(log, express)` | Restores `Monitor_Profiles/`, `animations/`, `wallpaper_effects/`, `monitors.conf`, `workspaces.conf` from the hypr backup |
| `compose_overlay_from_backup(type, base, old, new, disable)` | Extracts user-only additions from old UserConfigs vs base (for Startup_Apps and WindowRules) |
| `cleanup_duplicate_userconfigs(version, log)` | Removes duplicate `exec-once` / `windowrule` lines from UserConfigs (only for versions ≤ 2.3.19) |

---

### `scripts/lib_update.sh`

Provides `run_repo_update(repo_dir)`:

1. Verifies directory is the `Hyprland-Dots` root.
2. Stashes any local changes (`git stash -u`).
3. Runs `git pull --ff-only`.
4. Prints a summary (commit before/after, stash status, pull status).
5. Optionally runs `cleanup_duplicate_userconfigs` for existing installs.
6. Waits for a keypress before returning to the menu.

---

## Archive Scripts Reference

> These scripts are in `archive/` and are considered legacy alternatives to `copy.sh`.

### `archive/release.sh`

Downloads from the latest GitHub **Release** (one version behind `main`):

1. Checks if `Hyprland-Dots.tar.gz` already exists; if so, compares its version to the latest API tag and optionally re-downloads.
2. Fetches `tarball_url` from the GitHub API, downloads and extracts it.
3. Renames the extracted directory to `LinuxBeginnings-Hyprland-Dots`, `cd`s into it, and runs `./copy.sh`.

### `archive/upgrade.sh`

Semi-manual per-directory upgrade using `rsync`:

1. Checks for a version marker file in `~/.config/hypr/` to confirm dots are installed.
2. Compares source (`config/`) to target (`~/.config/`) for each tracked directory using `rsync --dry-run`.
3. For each directory with differences, shows the diff and prompts the user.
4. On confirmation: creates an `*-b4-upgrade` rsync backup, then syncs with exclusions (e.g., `UserConfigs/`, `UserScripts/`).

### `archive/update-dots.sh`

Lightweight `git` updater:

1. Verifies it is inside a git repo.
2. Fetches tags and checks upstream.
3. Reports `behind` / `ahead` counts.
4. If behind: stashes local changes and `git pull`.
5. Reminds user to run `./copy.sh` after pulling.

---

## Runtime / UserScripts Reference

These scripts live under `~/.config/hypr/` after installation and are called by Hyprland keybinds or startup entries.

### `config/hypr/initial-boot.sh`

Runs **once** on the first Hyprland session after installation (guarded by `~/.config/hypr/.initial_startup_done`):

- Runs `wallust` and `swww` to apply the default wallpaper.
- Sets GTK dark mode, GTK theme, icon theme, and cursor theme via `gsettings` (and `dconf` on NixOS).
- Sets the Kvantum theme via `kvantummanager`.
- Creates the marker file so it never runs again.

### Notable `scripts/` runtime scripts

| Script | What it does |
|--------|-------------|
| `WallustSwww.sh` | Applies wallust colours + swww wallpaper transition |
| `ScreenShot.sh` | Screenshot via `hyprshot`/`grimblast` with region/window/monitor modes |
| `Volume.sh` | Audio volume control via `wpctl` (PipeWire) with OSD notifications |
| `Brightness.sh` | Screen brightness via `brightnessctl` with OSD |
| `BrightnessKbd.sh` | Keyboard backlight brightness |
| `LockScreen.sh` | Locks screen via `hyprlock` |
| `Wlogout.sh` | Launches `wlogout` power menu |
| `Refresh.sh` | Kills and restarts Waybar, `swaync`, AGS/Quickshell |
| `RefreshNoWaybar.sh` | Same as above, but without restarting Waybar |
| `ThemeChanger.sh` | Rofi-based theme selector (wallust colour schemes) |
| `WaybarLayout.sh` | Rofi menu to switch Waybar layout configs |
| `WaybarStyles.sh` | Rofi menu to switch Waybar CSS styles |
| `DarkLight.sh` | Toggles GTK dark/light mode |
| `GameMode.sh` | Toggles Hyprland animations/blur off for gaming |
| `ChangeBlur.sh` | Rofi menu to cycle Hyprland blur settings |
| `Animations.sh` | Rofi menu to switch animation presets |
| `KeyBinds.sh` | Shows keybind cheat-sheet via `rofi` |
| `ClipManager.sh` | Clipboard manager via `cliphist` + `rofi` |
| `MediaCtrl.sh` | Media player controls (play/pause/next/prev) via `playerctl` |
| `MonitorProfiles.sh` | Loads/saves multi-monitor hyprland profiles |
| `Distro_update.sh` | Launches a terminal running the distro package manager update |
| `KooLsDotsUpdate.sh` | Pulls latest Hyprland-Dots and runs `copy.sh --express-upgrade` |
| `sddm_wallpaper.sh` | Copies current wallpaper to SDDM theme background (requires sudo) |

### Notable `UserScripts/`

| Script | What it does |
|--------|-------------|
| `WallpaperSelect.sh` | Rofi file picker for wallpapers; runs wallust + swww |
| `WallpaperRandom.sh` | Picks a random wallpaper from `~/Pictures/wallpapers/` |
| `WallpaperAutoChange.sh` | Loops random wallpaper changes at an interval |
| `WallpaperEffects.sh` | Applies image effects (blur, pixelate, etc.) to wallpaper |
| `RofiBeats.sh` | Rofi-based internet radio player via `mpv` |
| `RofiCalc.sh` | Rofi calculator |
| `Weather.sh` / `WeatherWrap.sh` | Fetches weather data for Waybar module |
| `ZshChangeTheme.sh` | Switches Zsh prompt theme |
| `RainbowBorders.sh` | Animates rainbow Hyprland border colours |

---

## Prerequisites

### Supported Distributions

| Distribution | Status |
|-------------|--------|
| Arch Linux / Manjaro (pacman) | ✅ Fully supported |
| Fedora (dnf) | ✅ Fully supported |
| OpenSUSE Tumbleweed (zypper) | ✅ Fully supported |
| Debian GNU/Linux (Trixie/SID) | ✅ Supported (Hyprland built from source) |
| Ubuntu 24.04 LTS | ✅ Supported (via PPA, Hyprland v0.50+) |
| Ubuntu 25.10 / 26.04-dev | ✅ Supported |
| NixOS (25.05+) | ✅ Supported |

### Required System Packages (before running `copy.sh`)

`copy.sh` **does not install packages**. These must already be present:

- `hyprland` v0.50+ (v0.51.1+ for Debian/Ubuntu)
- `rsync` (required; copy.sh exits if missing)
- `git` (required to clone this repo)
- `wallust` (required for final colour init)
- `swww` (wallpaper daemon)
- `waybar`, `rofi`, `kitty`, `swaync` (core UI components)
- `hyprlock`, `wlogout`, `hypridle` (lock/logout/idle)
- `gzip` (required if installing `waybar-weather` from bundled asset)
- `whiptail` (optional; provides TUI menu)
- `xdg-user-dirs` (optional; updates home directories)

The per-distro installer repos (`Arch-Hyprland`, `Fedora-Hyprland`, etc.) install all required packages automatically.

### Permissions

- `copy.sh` **must not** be run as root (enforced by the script).
- `sudo` is used only for specific non-interactive operations (`sudo -n`):
  - Copying wallpaper to SDDM theme directory.
  - Editing SDDM `theme.conf` for 12H clock format.
  - Installing `waybar-weather` binary to `/usr/bin/`.
- If `sudo` is not cached (passwordless sudo not available), these steps are silently skipped with a warning.

---

## What Gets Changed on Your System

### Directories Created / Replaced in `~/.config`

| Directory | Phase | Behaviour |
|-----------|-------|-----------|
| `~/.config/fastfetch` | Phase 1 | Prompted replace or skip |
| `~/.config/kitty` | Phase 1 | Prompted replace or skip |
| `~/.config/rofi` | Phase 1 | Prompted replace or skip; custom themes re-merged |
| `~/.config/swaync` | Phase 1 | Prompted replace or skip |
| `~/.config/waybar` | Waybar | Prompted replace or skip; extra configs/styles re-merged |
| `~/.config/btop` | Phase 2 | Auto-backup + replace |
| `~/.config/cava` | Phase 2 | Auto-backup + replace |
| `~/.config/hypr` | Phase 2 | Auto-backup + replace; user configs/scripts restored |
| `~/.config/Kvantum` | Phase 2 | Auto-backup + replace |
| `~/.config/qt5ct` | Phase 2 | Auto-backup + replace |
| `~/.config/qt6ct` | Phase 2 | Auto-backup + replace |
| `~/.config/swappy` | Phase 2 | Auto-backup + replace |
| `~/.config/wallust` | Phase 2 | Auto-backup + replace |
| `~/.config/wlogout` | Phase 2 | Auto-backup + replace |
| `~/.config/ghostty` | Phase 2 | Created or overwritten (no backup) |
| `~/.config/wezterm` | Phase 2 | Created or overwritten (no backup) |
| `~/.config/waybar-weather` | Post-phase | Created on install; skipped on upgrade if exists |
| `~/.config/ags` | Optional | Prompted replace or skip (only if `ags` installed) |
| `~/.config/quickshell` | Optional | Prompted replace or skip (only if `qs` installed) |

### Backups Created

Backups are timestamped and placed adjacent to the original:

```
~/.config/hypr-backup-MMDD_HHMM/
~/.config/waybar-backup-MMDD_HHMM/
~/.config/kitty-backup-MMDD_HHMM/
...
```

> Backups are **not** automatically deleted (except in express mode or when the user chooses to during cleanup).

### Symlinks Created

| Symlink | Target |
|---------|--------|
| `~/.config/waybar/config` | `~/.config/waybar/configs/TOP-Default` (desktop) or `TOP-Default-Laptop` (laptop) |
| `~/.config/waybar/style.css` | `~/.config/waybar/style/Extra-Prismatic-Glow.css` |
| `~/.local/share/rofi/themes/*` | All files in `~/.config/rofi/themes/` |

### System Files Modified (sudo)

| File | Condition | Change |
|------|-----------|--------|
| `/usr/share/sddm/themes/simple_sddm_2/Backgrounds/default` | `simple_sddm_2` theme installed | Replaced with current wallpaper |
| `/usr/share/sddm/themes/simple-sddm/theme.conf` | Theme installed + 12H chosen | `HourFormat` switched to 12H |
| `/usr/share/sddm/themes/simple_sddm_2/theme.conf` | Theme installed + 12H chosen | `HourFormat` switched to 12H |
| `/usr/share/sddm/themes/sequoia_2/theme.conf` | Theme installed + 12H chosen | `clockFormat` switched to 12H |
| `/usr/bin/waybar-weather` | Non-NixOS, `waybar-weather` not present | Installed from bundled asset or AUR |

### Executables Installed to `/usr/bin`

- `waybar-weather` (non-NixOS only; from bundled `assets/waybar-weather.gz` or AUR)

---

## Environment Variables & Config Toggles

These are set/modified in `config/hypr/configs/ENVariables.conf` and `config/hypr/UserConfigs/01-UserDefaults.conf` during installation:

| Variable | Default state | Activated when |
|----------|--------------|----------------|
| `LIBVA_DRIVER_NAME=nvidia` | Commented out | Nvidia GPU detected |
| `__GLX_VENDOR_LIBRARY_NAME=nvidia` | Commented out | Nvidia GPU detected |
| `NVD_BACKEND=direct` | Commented out | Nvidia GPU detected |
| `GSK_RENDERER=ngl` | Commented out | Nvidia GPU detected |
| `WLR_RENDERER_ALLOW_SOFTWARE=1` | Commented out | Running in VM |
| `HYPRCURSOR_THEME=Bibata-Modern-Ice` | Commented out | `~/.icons/Bibata-Modern-Ice/hyprcursors` exists |
| `HYPRCURSOR_SIZE=24` | Commented out | Same as above |
| `EDITOR=<editor>` | Commented out | User chooses nvim/vim during install |
| `no_hardware_cursors = 1` | 2 (auto) | Nvidia GPU or VM detected |

---

## Safe-Run Guidance

### Running in a Virtual Machine

The scripts are safe to run inside a VM — `detect_vm_adjust` even auto-configures the required software renderer. Use a VM snapshot to test without risk:

1. Create a VM running a supported distro with Hyprland installed.
2. Clone this repo: `git clone --depth=1 https://github.com/LinuxBeginnings/Hyprland-Dots.git`
3. Run `./copy.sh` and choose **Install** mode.
4. Review the results, then snapshot or discard the VM.

### Dry-Run / Safe Preview

There is no built-in dry-run flag in `copy.sh`. To preview what would change:

```bash
# Compare your current config vs the repo
diff -rq --exclude='*.log' ~/.config/hypr config/hypr
diff -rq --exclude='*.log' ~/.config/waybar config/waybar
# ...etc
```

Or use the `upgrade.sh` archive script which shows `rsync --dry-run` diffs interactively before applying anything.

### Partial / Selective Update

If you only want to update one application's config:

```bash
# Example: update only Waybar
cp -r ~/.config/waybar ~/.config/waybar-manual-backup
cp -r config/waybar ~/.config/waybar
```

Or use `archive/upgrade.sh` which offers per-directory confirmation.

---

## Reverting Changes

All Phase 1 and Phase 2 configs are backed up before replacement. To revert:

1. Find the backup: `ls ~/.config/ | grep backup`
2. Remove the new config and restore the backup:
   ```bash
   rm -rf ~/.config/hypr
   mv ~/.config/hypr-backup-MMDD_HHMM ~/.config/hypr
   ```
3. Repeat for any other directory you want to revert (`waybar`, `kitty`, etc.).

To revert the `initial-boot.sh` one-time setup, delete the marker file (this re-enables it on next login):

```bash
rm ~/.config/hypr/.initial_startup_done
```

To revert system-level changes (SDDM theme, `/usr/bin/waybar-weather`):

```bash
# Restore SDDM theme.conf from your distro's package
sudo pacman -S sddm   # or appropriate package manager

# Remove waybar-weather
sudo rm /usr/bin/waybar-weather
```

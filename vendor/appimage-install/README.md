# User-Space AppImage Installer for Qubes OS (`appimage-install`)

## Goal
Install, integrate, and update arbitrary AppImages entirely within user space (`$HOME`) in a Qubes AppVM, with zero modifications to the underlying TemplateVM, full persistence across VM reboots, and automatic desktop and icon integration for dom0 application menus.

## Context

In Qubes OS:
1. **Template immutability:** Applications installed inside an AppVM should ideally avoid modifying the root filesystem or TemplateVM so that templates remain clean and generic.
2. **FUSE availability:** Debian 13 (Trixie) does not ship `libfuse2` by default (only `libfuse3`). Standard Type 2 AppImages fail to launch directly unless extracted or supplied with `libfuse.so.2`. Extracting once to a persistent folder avoids FUSE issues and startup extraction churn.
3. **AppDir specification:** Upstream AppImages already bundle an official `.desktop` entry, application icons (`usr/share/icons/hicolor/...` or root `.DirIcon`), and window manager classes (`StartupWMClass`). Hand-rolling `.desktop` files frequently results in missing icons, broken window grouping, or wrong MIME associations.

`appimage-install` automates this entire extraction and desktop integration workflow.

## Usage

```bash
appimage-install <path-to-AppImage> [app-name]
```

### Examples

Install OpenCode Desktop:
```bash
./appimage-install ~/Downloads/OpenCode-2.1.3.AppImage opencode-desktop
```

Install Obsidian:
```bash
./appimage-install ~/Downloads/Obsidian-1.5.8.AppImage obsidian
```

### Updating an Application

To update an existing app, simply run `appimage-install` pointing to the new AppImage:
```bash
./appimage-install ~/Downloads/Obsidian-1.6.0.AppImage obsidian
```
The script replaces the extracted payload, refreshes desktop files and icons, and keeps the wrapper in `~/.local/bin/obsidian` intact.

## What It Does

1. **Stages the binary:** Saves the AppImage to `~/Applications/<app-name>.AppImage` as an executable payload.
2. **Extracts payload:** Extracts the SquashFS root to `~/Applications/<app-name>-extracted/` so it runs immediately without FUSE.
3. **Creates a CLI wrapper:** Drops an executable launcher script in `~/.local/bin/<app-name>` pointing to the extracted `AppRun`.
4. **Installs icons:** Copies upstream `usr/share/icons/hicolor/` to `~/.local/share/icons/hicolor/` (or falls back to extracting the root `.DirIcon`/PNG), ensuring dom0 icon synchronizers find the icon at all resolutions.
5. **Installs desktop entry:** Copies the bundled upstream `*.desktop` entry to `~/.local/share/applications/` and points `Exec=` to the `~/.local/bin/<app-name>` wrapper while preserving any flags (e.g. `--no-sandbox %U`).
6. **Refreshes caches:** Updates `update-desktop-database` and `gtk-update-icon-cache`.

## dom0 Integration

After running `appimage-install` in the AppVM:
1. Open **Qube Manager** in dom0.
2. Select the AppVM and open **Qube Settings** → **Applications** tab.
3. Click **Refresh Applications**.
4. Select the newly installed application and move it to the right pane to pin it to your desktop menu.

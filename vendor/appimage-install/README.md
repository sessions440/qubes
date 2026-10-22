# Qubes OS AppImage Installer (`appimage-install`)

## Goal
Install, integrate, and update arbitrary AppImages either in an individual AppVM (user space, under `$HOME`) or system-wide across all derived AppVMs via a TemplateVM (under `/opt` and `/usr`). Reuses bundled upstream `.desktop` entries, hicolor icons, and `StartupWMClass` metadata for seamless dom0 application menu and window manager integration.

## Context

In Qubes OS, AppImages can be installed in two distinct scopes:

1. **AppVM (User space):**
   - Installed entirely under `$HOME` (`~/Applications/`, `~/.local/bin/`, `~/.local/share/`).
   - Requires no root privilege and leaves the TemplateVM completely untouched.
   - Ideal for single-qube applications.

2. **TemplateVM (System-wide):**
   - Software must live in the root filesystem (`/opt/`, `/usr/local/bin/`, `/usr/share/`).
   - **Why `/home` does not work in a TemplateVM:** In Qubes OS, `/home/user/` in a TemplateVM is *never* inherited by AppVMs. AppVMs have private `/rw/home/user/` filesystems. Only the TemplateVM's root filesystem (`/`) is mounted read-only into derived AppVMs.
   - **Permissions:** Root extraction of AppImages preserves permissions that may prevent non-root `user` in derived AppVMs from reading or executing files. The script automatically sets `chmod -R a+rX`.
   - **Network isolation:** TemplateVMs do not have a NetVM by default. The cleanest workflow is to download the AppImage in an AppVM, then transfer it into the TemplateVM via `qvm-copy`.

## Usage

```bash
# AppVM (User Space):
appimage-install <path-to-AppImage> [app-name]

# TemplateVM (System-wide):
sudo appimage-install --system <path-to-AppImage> [app-name]
```

### Examples

**Install in an AppVM (User space):**
```bash
appimage-install ~/Downloads/OpenCode-2.1.3.AppImage opencode-desktop
```

**Install in a TemplateVM (System-wide):**
```bash
sudo appimage-install --system ~/QubesIncoming/work/Obsidian-1.5.8.AppImage obsidian
```

### Updating an Application

To update an existing app, re-run `appimage-install` pointing to the new release file. The payload is re-extracted, desktop files and icons are refreshed, and existing command wrappers and menu shortcuts remain valid.

## What It Does

| Step | User Mode (AppVM) | System Mode (`--system`, TemplateVM) |
|---|---|---|
| **Payload** | `~/Applications/<app>-extracted/` | `/opt/<app>/` |
| **Wrapper** | `~/.local/bin/<app>` | `/usr/local/bin/<app>` |
| **Desktop Entry** | `~/.local/share/applications/` | `/usr/share/applications/` |
| **Icons** | `~/.local/share/icons/hicolor/` | `/usr/share/icons/hicolor/` |
| **Permissions** | User permissions | `chmod -R a+rX` for global read/execute |
| **Root required?** | No | Yes (`sudo`) |

## dom0 Integration

### If installed in an AppVM:
1. Open **Qube Manager** in dom0.
2. Select the AppVM → **Qube Settings** → **Applications** tab.
3. Click **Refresh Applications**, select the app, and move it to the right pane.

### If installed in a TemplateVM:
1. Shut down the TemplateVM: `qvm-shutdown <template>`
2. Sync template appmenus in dom0: `qvm-sync-appmenus <template>`
3. In any derived AppVM: Open **Qube Settings** → **Applications** tab → **Refresh Applications**, and add the app to the menu.

## Status

- **AppVM (user space):** Confirmed working on `debian-13-xfce` (`id0-agents`).
- **TemplateVM (`--system` mode):** Design proposed, not yet field-tested in a live TemplateVM.

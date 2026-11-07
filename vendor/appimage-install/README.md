# Qubes OS AppImage Installer (`appimage-install`)

## Goal
Install, integrate, update, and remove arbitrary AppImages either in an individual AppVM (user space, under `$HOME`) or system-wide across all derived AppVMs via a TemplateVM (under `/opt` and `/usr`). Reuses bundled upstream `.desktop` entries, hicolor icons, and `StartupWMClass` metadata for seamless dom0 application menu and window manager integration.

## Installing the Script

### In an AppVM (User Space)
Symlink or copy the script to a directory on your `$PATH` (e.g. `~/.local/bin`):

```bash
mkdir -p ~/.local/bin
ln -sf ~/qubes/vendor/appimage-install/appimage-install ~/.local/bin/appimage-install
```

### In a TemplateVM (System-Wide)
Copy the script from an AppVM into the TemplateVM:

1. In the source AppVM:
   ```bash
   qvm-copy ~/qubes/vendor/appimage-install/appimage-install
   ```
2. In the TemplateVM (as root):
   ```bash
   sudo cp ~/QubesIncoming/<source-vm>/appimage-install /usr/bin/
   sudo chmod +x /usr/bin/appimage-install
   rm -rf ~/QubesIncoming/<source-vm>/
   ```

Now `appimage-install` is available in `/usr/bin/` across the template and all derived AppVMs.

---

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

---

## Filesystem Layout & What It Does

| Component | User Mode (AppVM) | System Mode (`--system`, TemplateVM) |
|---|---|---|
| **Payload** | `~/Applications/<app>-extracted/` | `/opt/<app>/` |
| **Wrapper** | `~/.local/bin/<app>` | `/usr/bin/<app>` |
| **Desktop Entry** | `~/.local/share/applications/<id>.desktop` | `/usr/share/applications/<id>.desktop` |
| **Icons** | `~/.local/share/icons/hicolor/` | `/usr/share/icons/hicolor/` |
| **Permissions** | User ownership | `chmod -R a+rX` for global read/execute |
| **Privilege** | Unprivileged (`user`) | Root (`sudo`) |

> **Why `/usr/bin/` instead of `/usr/local/bin/` for TemplateVMs:**
> In Qubes OS, every AppVM bind-mounts its private volume `/rw/usrlocal` over `/usr/local` at boot (`usr-local.mount`). Any binary placed in `/usr/local/bin/` in a TemplateVM is shadowed and completely invisible inside existing AppVMs. System binaries inherited from the template must live in `/usr/bin/`.

---

## Updating an Application

To update an existing app, re-run `appimage-install` pointing to the new release file. The payload is re-extracted, desktop files and icons are refreshed, and existing command wrappers and menu shortcuts remain valid.

- **In an AppVM:**
  ```bash
  appimage-install ~/Downloads/MyApp-1.3.0.AppImage myapp
  ```
- **In a TemplateVM:**
  ```bash
  sudo appimage-install --system ~/QubesIncoming/<source-vm>/MyApp-1.3.0.AppImage myapp
  ```
  Then in dom0, sync appmenus while the template is running, shut down the template, and restart derived AppVMs:
  ```bash
  qvm-sync-appmenus <template>
  qvm-shutdown <template>
  ```

---

## Removing / Deleting an Application

Removal is a straightforward filesystem deletion of the files created during installation.

### From an AppVM (User Space)

1. Remove the application files:
   ```bash
   # Remove payload, staged AppImage, wrapper, and desktop entry:
   rm -rf ~/Applications/<app>.AppImage ~/Applications/<app>-extracted
   rm -f ~/.local/bin/<app>
   rm -f ~/.local/share/applications/<app>*.desktop

   # Refresh the desktop database:
   update-desktop-database ~/.local/share/applications/
   ```
2. Refresh dom0 menus:
   In dom0, open **Qube Manager** → select the AppVM → **Applications** tab → **Refresh Applications** (or run `qvm-sync-appmenus <appvm>` in dom0).

### From a TemplateVM (System-Wide)

1. In the TemplateVM (as root):
   ```bash
   sudo rm -rf /opt/<app>
   sudo rm -f /usr/bin/<app>
   sudo rm -f /usr/share/applications/<app>*.desktop
   sudo update-desktop-database /usr/share/applications/
   ```
2. In dom0 (while the template is still running):
   ```bash
   qvm-sync-appmenus <template>
   qvm-shutdown <template>
   ```
3. In derived AppVMs: Open **Qube Manager** → select the AppVM → **Applications** tab → **Refresh Applications** to remove the app from the menu, and restart the AppVMs.

---

## dom0 Integration

### If installed in an AppVM:
1. Open **Qube Manager** in dom0.
2. Select the AppVM → **Qube Settings** → **Applications** tab.
3. Click **Refresh Applications**, select the app, and move it to the right pane.

### If installed in a TemplateVM:
1. Sync template appmenus in dom0 while the template is still running:
   ```bash
   qvm-sync-appmenus <template>
   ```
   *(Note: `qvm-sync-appmenus` communicates via qrexec and requires the qube to be running).*
2. Shut down the TemplateVM:
   ```bash
   qvm-shutdown <template>
   ```
3. Restart derived AppVMs so they mount the new template root volume snapshot.
4. In any derived AppVM: Open **Qube Settings** → **Applications** tab → **Refresh Applications**, and add the app to the menu.

---

## Status

- **AppVM (user space):** Confirmed working on `debian-13-xfce` (`id0-agents`).
- **TemplateVM (`--system` mode):** Confirmed working on `debian-13-xfce` (verified via Obsidian AppImage install and menu launch across multiple AppVMs).

# AppImage Installation in Qubes OS (AppVM vs. TemplateVM)

## Context

Goal: install and integrate an arbitrary AppImage in Qubes OS cleanly, reliably, and with
proper dom0 application menu and window manager integration.

Two distinct deployment models exist in Qubes OS:

1. **AppVM (User space):** Installed entirely under `$HOME`, zero changes to the underlying
   TemplateVM, no `apt` installs, no root privileges required. Best for single-qube applications.
2. **TemplateVM (System-wide):** Installed in the TemplateVM's root filesystem (`/opt/` and
   `/usr/`) so that the application is inherited across all derived AppVMs.

Every standard AppImage is an AppDir filesystem containing an upstream `.desktop` entry,
multi-resolution hicolor icons, and window manager classes (`StartupWMClass`). Reusing these
bundled assets ensures correct dom0 icon resolution and window grouping without hand-crafting
desktop files.

For full script options, filesystem layout, and update/deletion details, see
`vendor/appimage-install/README.md`.

---

## Part 1 — AppVM Installation (User Space)

Installs entirely under `/home/user`, persisting across AppVM reboots without touching the TemplateVM.

### Method A — Automated (`appimage-install` helper)

The helper script `vendor/appimage-install/appimage-install` automates staging, extraction,
icon promotion, wrapper creation, and desktop integration. See `vendor/appimage-install/README.md`
for details.

`[Agent/qrexec]` or `[Human]` inside the AppVM:

```bash
# Ensure script is available on PATH:
mkdir -p ~/.local/bin
ln -sf ~/qubes/vendor/appimage-install/appimage-install ~/.local/bin/appimage-install

# Install or update:
appimage-install ~/Downloads/MyApp-1.2.3.AppImage myapp
```

### Method B — Manual Streamlined Steps

`[Agent/qrexec]` or `[Human]` inside the AppVM:

1. **Stage and extract payload:**
   ```bash
   mkdir -p ~/Applications ~/.local/bin ~/.local/share/applications ~/.local/share/icons
   mv ~/Downloads/MyApp-1.2.3.AppImage ~/Applications/myapp.AppImage
   chmod +x ~/Applications/myapp.AppImage

   # Extract once to avoid FUSE dependencies and runtime extraction lag:
   rm -rf ~/Applications/myapp-extracted
   cd ~/Applications
   ./myapp.AppImage --appimage-extract
   mv squashfs-root myapp-extracted
   ```

2. **Create launcher wrapper:**
   ```bash
   cat > ~/.local/bin/myapp << 'EOF'
   #!/bin/bash
   exec "$HOME/Applications/myapp-extracted/AppRun" "$@"
   EOF
   chmod +x ~/.local/bin/myapp
   ```

3. **Install bundled icons:**
   ```bash
   if [ -d ~/Applications/myapp-extracted/usr/share/icons/hicolor ]; then
       cp -r ~/Applications/myapp-extracted/usr/share/icons/hicolor ~/.local/share/icons/
   elif [ -f ~/Applications/myapp-extracted/.DirIcon ]; then
       mkdir -p ~/.local/share/icons/hicolor/256x256/apps
       cp ~/Applications/myapp-extracted/*.png ~/.local/share/icons/hicolor/256x256/apps/ 2>/dev/null || true
   fi
   ```

4. **Install and patch bundled desktop entry:**
   ```bash
   DESKTOP_SRC="$(find ~/Applications/myapp-extracted -maxdepth 1 -type f -name "*.desktop" -exec grep -l '^\[Desktop Entry\]' {} + | head -n 1)"
   cp "$DESKTOP_SRC" ~/.local/share/applications/

   DESKTOP_DEST=~/.local/share/applications/"$(basename "$DESKTOP_SRC")"
   sed -i -E 's|^Exec=[^ ]+|Exec=/home/user/.local/bin/myapp|' "$DESKTOP_DEST"

   update-desktop-database ~/.local/share/applications/
   gtk-update-icon-cache -f ~/.local/share/icons/hicolor/ 2>/dev/null || true
   ```

5. **Sync with dom0 application menu:**
   `[Human/dom0]`:
   In **Qube Manager** → select the AppVM → **Applications** tab → **Refresh Applications** → add app to menu. (Or run `qvm-sync-appmenus <qube-name>` in dom0).

---

## Part 2 — TemplateVM Installation (System-Wide)

Use when an application should be available across all AppVMs derived from a template.

### Key TemplateVM Constraints

- **`/home/user/` is NOT inherited:** AppVMs get private `/rw/home/user/` filesystems. Software
  intended for AppVMs must live in `/opt/<app>`, `/usr/bin/<app>`, and `/usr/share/`.
- **`/usr/bin/` vs. `/usr/local/bin/`:** AppVMs bind-mount `/rw/usrlocal` over `/usr/local`. Any
  binary in `/usr/local/bin` in a TemplateVM is shadowed and invisible in AppVMs. System wrappers
  must go in `/usr/bin/`.
- **Network isolation:** TemplateVMs lack network access by default. Download the AppImage
  in an AppVM and transfer it with `qvm-copy`, avoiding attaching network interfaces to the template.
- **Permissions:** Root extraction of AppImages creates files readable only by root. Permissions
  must be normalized with `chmod -R a+rX /opt/<app>` so unprivileged AppVM users can execute it.

### Method A — Automated (`appimage-install --system`)

1. **Transfer AppImage to TemplateVM:**
   `[Agent/qrexec]` or `[Human]` in source AppVM:
   ```bash
   qvm-copy ~/Downloads/MyApp-1.2.3.AppImage
   ```

2. **Run helper in TemplateVM:**
   `[Human/TemplateVM]` (run as root / with `sudo`):
   ```bash
   sudo appimage-install --system ~/QubesIncoming/<source-vm>/MyApp-1.2.3.AppImage myapp
   ```
   *(To install `appimage-install` in the template, see `vendor/appimage-install/README.md#installing-the-script`).*

3. **Sync in dom0 while template is running, then shut down:**
   `[Human/dom0]`:
   ```bash
   qvm-sync-appmenus <template-name>
   qvm-shutdown <template-name>
   ```
   *(Note: `qvm-sync-appmenus` communicates via qrexec and requires the qube to be running).*

4. **Restart AppVMs:**
   `[Human/dom0]`:
   Restart derived AppVMs so they pick up the updated root volume snapshot. In **Qube Manager**,
   open **Applications** tab → **Refresh Applications** to pin the new shortcut.

### Method B — Manual Streamlined Steps

`[Human/TemplateVM]` (run as root in the TemplateVM):

1. **Extract payload to `/opt` and fix permissions:**
   ```bash
   chmod +x ~/QubesIncoming/<source-vm>/MyApp-1.2.3.AppImage
   cd /opt
   ~/QubesIncoming/<source-vm>/MyApp-1.2.3.AppImage --appimage-extract
   rm -rf /opt/myapp
   mv squashfs-root /opt/myapp
   chmod -R a+rX /opt/myapp
   ```

2. **Create launcher wrapper:**
   ```bash
   cat > /usr/bin/myapp << 'EOF'
   #!/bin/bash
   exec /opt/myapp/AppRun "$@"
   EOF
   chmod a+rx /usr/bin/myapp
   ```

3. **Install system icons and desktop entry:**
   ```bash
   # Icons:
   if [ -d /opt/myapp/usr/share/icons/hicolor ]; then
       cp -r /opt/myapp/usr/share/icons/hicolor /usr/share/icons/
       chmod -R a+rX /usr/share/icons/hicolor
   fi

   # Desktop file:
   DESKTOP_SRC="$(find /opt/myapp -maxdepth 1 -type f -name "*.desktop" -exec grep -l '^\[Desktop Entry\]' {} + | head -n 1)"
   cp "$DESKTOP_SRC" /usr/share/applications/
   DESKTOP_DEST=/usr/share/applications/"$(basename "$DESKTOP_SRC")"
   sed -i -E 's|^Exec=[^ ]+|Exec=/usr/bin/myapp|' "$DESKTOP_DEST"
   chmod 644 "$DESKTOP_DEST"

   update-desktop-database /usr/share/applications/
   gtk-update-icon-cache -f /usr/share/icons/hicolor/ 2>/dev/null || true
   rm -rf ~/QubesIncoming/<source-vm>/
   ```

4. **Sync in dom0 while template is running, shut down, and restart AppVMs:**
   `[Human/dom0]`:
   ```bash
   qvm-sync-appmenus <template-name>
   qvm-shutdown <template-name>
   ```
   Then restart derived AppVMs and refresh applications in Qube Manager.

---

## Verification

`[Agent/qrexec]` or `[Human]` inside the AppVM (or TemplateVM during install):

1. **Verify desktop syntax:**
   ```bash
   desktop-file-validate ~/.local/share/applications/*.desktop
   # Or for TemplateVM:
   desktop-file-validate /usr/share/applications/<app>.desktop
   ```
2. **Verify dom0 icon resolution:**
   ```bash
   /usr/lib/qubes/xdg-icon <icon-name> 128
   ```
   *(Check `Icon=` in the `.desktop` file for `<icon-name>`, e.g. `ai.opencode.desktop` or `obsidian`)*.
3. **Verify wrapper execution:**
   ```bash
   test -x ~/.local/bin/myapp && echo "User wrapper is executable"
   # Or for TemplateVM:
   test -x /usr/bin/myapp && echo "System wrapper is executable"
   ```
   *(Note: Avoid running graphical apps directly in non-interactive/agent shell sessions; test interactive launch as a human).*

---

## Updates and Removal

- **Updating:** Re-running `appimage-install` (or re-extracting) against a newer release updates the
  underlying payload while preserving existing wrapper scripts and dom0 menu shortcuts.
- **Removing:** Deleting an installed application is a simple removal of its payload, wrapper, and
  desktop entry, followed by a cache and dom0 menu refresh.

See `vendor/appimage-install/README.md#updating-an-application` and
`vendor/appimage-install/README.md#removing--deleting-an-application` for exact commands.

---

## Rationale & Gotchas

### Why FUSE matters (`debian-13` vs `whonix-workstation-18`)

AppImages mount via FUSE at runtime. `whonix-workstation-18` includes `libfuse2`, but
Debian 13 (Trixie) ships only `libfuse3` by default. Running traditional Type 2 AppImages
directly on Debian 13 fails with `dlopen(): error loading libfuse.so.2`. Extracting once
completely avoids FUSE requirements and eliminates startup lag.

### Why template binaries must live in `/usr/bin`, not `/usr/local/bin`

In Qubes OS, every AppVM bind-mounts its private volume `/rw/usrlocal` over `/usr/local`
at boot (`usr-local.mount`). Any binary installed to `/usr/local/bin/` in a TemplateVM is
shadowed and invisible inside existing AppVMs. System binaries inherited from the template
must be installed in `/usr/bin/`.

### Reusing upstream desktop entries vs. hand-crafting

Hand-crafted desktop files invariably miss key metadata:
1. **Icon name mismatches:** Packagers use reverse-DNS names (e.g. `ai.opencode.desktop.png`).
2. **`StartupWMClass`:** Essential for window managers to associate running windows with panel launchers.
3. **MIME & URL schemes:** Preserves deep-linking handlers (e.g. `x-scheme-handler/opencode;`).

---

## Status

- **AppVM installation (user space):** Confirmed working on `debian-13-xfce` (`id0-agents`).
- **TemplateVM installation (`--system` mode & manual):** Confirmed working on `debian-13-xfce`
  (verified via Obsidian AppImage install and menu launch across multiple AppVMs).

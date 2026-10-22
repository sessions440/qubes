# AppImage Installation in Qubes OS (AppVM vs. TemplateVM)

## Context

Goal: install and integrate an arbitrary AppImage in Qubes OS cleanly, reliably, and with
proper dom0 application menu and window manager integration.

Two distinct deployment models exist in Qubes OS:

1. **AppVM (User space):** Installed entirely under `$HOME`, zero changes to the underlying
   TemplateVM, no `apt` installs, no root privileges required. Best for single-qube applications.
2. **TemplateVM (System-wide):** Installed in the TemplateVM's root filesystem (`/opt/` and
   `/usr/`) so that the application is inherited across all derived AppVMs.

Every standard AppImage is an AppDir filesystem containing:
- An upstream-tested `.desktop` file at the root.
- Multi-resolution application icons in `usr/share/icons/hicolor/` (or root `.DirIcon` / `*.png`).
- Window manager mapping (`StartupWMClass`), file associations, and MIME types.

Reusing these bundled assets instead of hand-crafting `.desktop` files ensures
accurate icon display in dom0, correct window grouping on taskbars, and zero
procedural friction.

---

## Part 1 — AppVM Installation (User Space)

Ideal for single-qube applications. Everything lives under `/home/user`, persisting
across AppVM reboots without touching the TemplateVM.

### Method A — Automated (`appimage-install` helper)

The convenience script `vendor/appimage-install/appimage-install` handles staging,
extraction, icon promotion, wrapper creation, and desktop integration in a single command.

`[Agent/qrexec]` or `[Human]` inside the AppVM:

```bash
# If not already on PATH:
mkdir -p ~/.local/bin
ln -sf ~/qubes/vendor/appimage-install/appimage-install ~/.local/bin/appimage-install

# Install or update an AppImage in user space:
appimage-install ~/Downloads/MyApp-1.2.3.AppImage myapp
```

Arguments:
- `$1`: Path to the downloaded `.AppImage` file.
- `$2` *(optional)*: Command slug used for the binary wrapper (`~/.local/bin/myapp`),
  payload directory (`~/Applications/myapp-extracted`), and AppImage staging.

### Method B — Manual Streamlined Steps

If installing manually without the helper script, follow these steps to extract and
wire up the bundled assets.

#### Step 1 — Stage and extract the AppImage

`[Agent/qrexec]` or `[Human]`:

```bash
mkdir -p ~/Applications ~/.local/bin ~/.local/share/applications ~/.local/share/icons
mv ~/Downloads/MyApp-1.2.3.AppImage ~/Applications/myapp.AppImage
chmod +x ~/Applications/myapp.AppImage

# Extract payload once to avoid FUSE dependencies and startup extraction churn:
rm -rf ~/Applications/myapp-extracted
cd ~/Applications
./myapp.AppImage --appimage-extract
mv squashfs-root myapp-extracted
```

#### Step 2 — Create the launcher wrapper

`[Agent/qrexec]` or `[Human]`:

```bash
cat > ~/.local/bin/myapp << 'EOF'
#!/bin/bash
exec "$HOME/Applications/myapp-extracted/AppRun" "$@"
EOF
chmod +x ~/.local/bin/myapp
```

#### Step 3 — Install icons

`[Agent/qrexec]` or `[Human]`:

If the AppImage bundles standard hicolor icons (most Electron and Qt apps):

```bash
if [ -d ~/Applications/myapp-extracted/usr/share/icons/hicolor ]; then
    cp -r ~/Applications/myapp-extracted/usr/share/icons/hicolor ~/.local/share/icons/
fi
```

If the AppImage only provides a root `.DirIcon` or `myapp.png`:

```bash
mkdir -p ~/.local/share/icons/hicolor/256x256/apps
cp ~/Applications/myapp-extracted/*.png ~/.local/share/icons/hicolor/256x256/apps/ 2>/dev/null || true
```

#### Step 4 — Install and patch the bundled `.desktop` file

`[Agent/qrexec]` or `[Human]`:

Copy the upstream `.desktop` file and point its `Exec=` command to the wrapper,
preserving all arguments and flags (e.g. `--no-sandbox %U`):

```bash
# Locate upstream desktop entry:
DESKTOP_SRC="$(find ~/Applications/myapp-extracted -maxdepth 1 -type f -name "*.desktop" -exec grep -l '^\[Desktop Entry\]' {} + | head -n 1)"
cp "$DESKTOP_SRC" ~/.local/share/applications/

# Patch Exec line to use ~/.local/bin/myapp:
DESKTOP_DEST=~/.local/share/applications/"$(basename "$DESKTOP_SRC")"
sed -i -E 's|^Exec=[^ ]+|Exec=/home/user/.local/bin/myapp|' "$DESKTOP_DEST"

# Refresh desktop & icon caches:
update-desktop-database ~/.local/share/applications/
gtk-update-icon-cache -f ~/.local/share/icons/hicolor/ 2>/dev/null || true
```

#### Step 5 — Add to the Qubes app menu in dom0

`[Human/dom0]`:

1. Open **Qube Manager** → select the AppVM → **Qube Settings** → **Applications** tab.
2. Click **Refresh Applications**.
3. Select your application from the left column and move it to the right column to pin it.

Alternatively, refresh via dom0 terminal:

```bash
qvm-sync-appmenus <qube-name>
```

---

## Part 2 — TemplateVM Installation (System-Wide)

Use when an application should be available to multiple AppVMs based on the same template.

### Key TemplateVM Constraints

1. **`/home/user/` is NOT inherited:** In Qubes OS, a TemplateVM's `/home` is private to
   the template and is discarded for derived AppVMs. AppVMs receive their own private
   `/rw/home/user/`. Software intended for AppVMs must live in the template's root filesystem:
   - Payload: `/opt/<app>/`
   - Wrapper binary: `/usr/local/bin/<app>`
   - Desktop entry: `/usr/share/applications/<id>.desktop`
   - Icons: `/usr/share/icons/hicolor/`
2. **Network isolation:** TemplateVMs do not have a NetVM attached by default. Rather than
   weakening security by temporarily attaching `sys-firewall`, download the AppImage in an
   AppVM and transfer it into the TemplateVM via `qvm-copy`.
3. **Permissions:** Root extraction of an AppImage preserves squashfs permissions that may
   prevent non-root `user` in derived AppVMs from reading or executing files. Always run
   `chmod -R a+rX /opt/<app>`.

### Method A — Automated (`appimage-install --system`)

*(Design proposed, not yet field-tested in a live TemplateVM)*

1. **Transfer the AppImage into the TemplateVM:**
   `[Agent/qrexec]` or `[Human]` in the source AppVM:
   ```bash
   qvm-copy ~/Downloads/MyApp-1.2.3.AppImage
   ```
   Select your TemplateVM (e.g. `debian-13-xfce`) in the dom0 prompt. It lands in
   `~/QubesIncoming/<source-vm>/` inside the TemplateVM.

2. **Run the helper in system mode inside the TemplateVM:**
   `[Human/TemplateVM]` (in a TemplateVM terminal):
   ```bash
   # If you copied or shared appimage-install, run with sudo --system:
   sudo ./appimage-install --system ~/QubesIncoming/<source-vm>/MyApp-1.2.3.AppImage myapp
   ```
   This extracts the payload to `/opt/myapp`, creates `/usr/local/bin/myapp`, installs
   system icons to `/usr/share/icons/hicolor/`, installs `/usr/share/applications/<id>.desktop`,
   and normalizes permissions (`chmod -R a+rX`).

3. **Shut down template and sync menus:**
   `[Human/dom0]`:
   ```bash
   qvm-shutdown <template-name>
   qvm-sync-appmenus <template-name>
   ```

4. **Add to menu in derived AppVMs:**
   `[Human/dom0]`:
   In **Qube Manager**, open any derived AppVM → **Qube Settings** → **Applications** tab →
   **Refresh Applications** → select the app to pin it to that qube's menu.

### Method B — Manual Streamlined Steps

*(Design proposed, not yet field-tested in a live TemplateVM)*

`[Human/TemplateVM]` (run as root / with `sudo` in the TemplateVM):

#### Step 1 — Extract payload to `/opt/<app>`

```bash
chmod +x ~/QubesIncoming/<source-vm>/MyApp-1.2.3.AppImage
cd /opt
~/QubesIncoming/<source-vm>/MyApp-1.2.3.AppImage --appimage-extract
rm -rf /opt/myapp
mv squashfs-root /opt/myapp

# Ensure all users can read and execute:
chmod -R a+rX /opt/myapp
```

#### Step 2 — Create the system launcher wrapper

```bash
cat > /usr/local/bin/myapp << 'EOF'
#!/bin/bash
exec /opt/myapp/AppRun "$@"
EOF
chmod a+rx /usr/local/bin/myapp
```

#### Step 3 — Install system icons

```bash
if [ -d /opt/myapp/usr/share/icons/hicolor ]; then
    cp -r /opt/myapp/usr/share/icons/hicolor /usr/share/icons/
    chmod -R a+rX /usr/share/icons/hicolor
elif [ -f /opt/myapp/.DirIcon ]; then
    mkdir -p /usr/share/icons/hicolor/256x256/apps
    cp -L /opt/myapp/.DirIcon /usr/share/icons/hicolor/256x256/apps/myapp.png
    chmod 644 /usr/share/icons/hicolor/256x256/apps/myapp.png
fi
```

#### Step 4 — Install and patch the bundled `.desktop` file

```bash
DESKTOP_SRC="$(find /opt/myapp -maxdepth 1 -type f -name "*.desktop" -exec grep -l '^\[Desktop Entry\]' {} + | head -n 1)"
cp "$DESKTOP_SRC" /usr/share/applications/

DESKTOP_DEST=/usr/share/applications/"$(basename "$DESKTOP_SRC")"
sed -i -E 's|^Exec=[^ ]+|Exec=/usr/local/bin/myapp|' "$DESKTOP_DEST"
chmod 644 "$DESKTOP_DEST"

update-desktop-database /usr/share/applications/
gtk-update-icon-cache -f /usr/share/icons/hicolor/ 2>/dev/null || true

# Clean up transferred file:
rm -rf ~/QubesIncoming/<source-vm>/
```

#### Step 5 — dom0 Sync & Shutdown

`[Human/dom0]`:

```bash
qvm-shutdown <template-name>
qvm-sync-appmenus <template-name>
```

Then refresh applications in the desired derived AppVMs.

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
   Test that Qubes' internal icon lookup resolves the icon:
   ```bash
   /usr/lib/qubes/xdg-icon <icon-name> 128
   ```
   *(Replace `<icon-name>` with the value of `Icon=` from the `.desktop` file, e.g. `ai.opencode.desktop` or `obsidian`)*.
3. **Verify wrapper execution:**
   ```bash
   test -x ~/.local/bin/myapp && echo "Wrapper is executable"
   # Or for TemplateVM:
   test -x /usr/local/bin/myapp && echo "System wrapper is executable"
   ```
   *(Note: For graphical Electron/desktop apps, running `myapp` directly in a shell opens the full GUI window. Test interactive launch from the desktop launcher or terminal as a human; avoid invoking GUI binaries in non-interactive/automated agent sessions as they will block until the window is closed).*

---

## Updates

- **AppVM update:**
  Re-run `appimage-install ~/Downloads/MyApp-new.AppImage myapp` in the AppVM. The wrapper
  in `~/.local/bin/myapp` and dom0 menu setup remain unchanged.
- **TemplateVM update:**
  `qvm-copy` the new release into the TemplateVM and re-run:
  ```bash
  sudo appimage-install --system ~/QubesIncoming/<source-vm>/MyApp-new.AppImage myapp
  ```
  Then shut down the template and run `qvm-sync-appmenus <template-name>` in dom0. All
  derived AppVMs immediately get the updated version without individual qube modifications.

---

## Rationale & Gotchas

### Why FUSE matters (`debian-13` vs `whonix-workstation-18`)

AppImages mount via FUSE at runtime. `whonix-workstation-18` includes `libfuse2`, but
Debian 13 (Trixie) ships only `libfuse3` by default. Running traditional Type 2 AppImages
directly on Debian 13 fails with:
```
dlopen(): error loading libfuse.so.2
```
Do **not** install `libfuse2` into the template or via root hooks. Extracting once (to
`~/Applications/myapp-extracted/` in an AppVM or `/opt/myapp` in a TemplateVM) completely
avoids FUSE requirements, eliminates startup extraction lag, and keeps the TemplateVM clean.

### Why you should never write `.desktop` files by hand

Hand-crafted desktop files invariably miss key metadata:
1. **Icon name mismatches:** Packagers name icons using reverse-DNS or AppStream IDs
   (e.g. `ai.opencode.desktop.png`), not simple names like `myapp.png`. Hand-written
   names cause missing icons in dom0 and application menus.
2. **`StartupWMClass`:** Crucial for XFCE, GNOME, and KDE. Without this, window managers
   cannot match running windows to the desktop launcher, breaking panel icons and window grouping.
3. **MIME & URL schemes:** Misses registered URI handlers (e.g. `x-scheme-handler/opencode;`
   or `x-scheme-handler/obsidian;`) needed for browser integration and deep linking.

### TemplateVM persistence and permission gotchas

- **`/home` is ephemeral across template-to-AppVM inheritance:** Putting files in
  `/home/user/` in a TemplateVM does not propagate to AppVMs. Files must go to root
  mounts (`/opt/`, `/usr/local/bin/`, `/usr/share/`).
- **Permissions:** Squashfs extraction run under `sudo` can create files with restricted
  permissions (e.g. `drwx------`). `chmod -R a+rX` ensures the unprivileged `user`
  account inside derived AppVMs has read and traversal access.
- **Network isolation:** Keeping TemplateVMs disconnected from NetVMs is a core Qubes
  security principle. Transferring the AppImage payload via `qvm-copy` avoids temporarily
  attaching a NetVM to the template.

---

## Status

- **AppVM installation (user space):** Confirmed working on `debian-13-xfce` (`id0-agents`)
  using OpenCode Desktop AppImage, including icon resolution via `/usr/lib/qubes/xdg-icon`
  and dom0 application menu sync.
- **TemplateVM installation (`--system` mode & manual):** Design proposed, not yet
  field-tested in a live TemplateVM. The existing Obsidian installation in the template
  demonstrates the `/opt` pattern, but the streamlined asset-reuse flow and `--system`
  script option have not yet been verified live in a TemplateVM session.

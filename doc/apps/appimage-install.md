# Generic AppImage Installation (Qubes AppVM)

## Context

Goal: install an arbitrary AppImage in an AppVM (Whonix-based or `debian-13`-based)
entirely under `$HOME`, with no changes to the underlying TemplateVM and no `apt`
install of anything. Consistent with the minimal-footprint discipline applied
elsewhere in this environment (see `pass-install.md`, `opencode-install.md`).

Every standard AppImage is an AppDir filesystem containing:
- An upstream-tested `.desktop` file at the root.
- Multi-resolution application icons in `usr/share/icons/hicolor/` (or root `.DirIcon` / `*.png`).
- Window manager mapping (`StartupWMClass`), file associations, and MIME types.

Reusing these bundled assets instead of hand-crafting `.desktop` files ensures
accurate icon display in dom0, correct window grouping on taskbars, and zero
procedural friction.

---

## Method A — Automated (`appimage-install` helper)

The convenience script `vendor/appimage-install/appimage-install` handles staging,
extraction, icon promotion, wrapper creation, and desktop integration in a single command.

`[Agent/qrexec]` or `[Human]` inside the AppVM:

```bash
# If not already on PATH:
mkdir -p ~/.local/bin
ln -sf ~/qubes/vendor/appimage-install/appimage-install ~/.local/bin/appimage-install

# Install or update an AppImage:
appimage-install ~/Downloads/MyApp-1.2.3.AppImage myapp
```

Arguments:
- `$1`: Path to the downloaded `.AppImage` file.
- `$2` *(optional)*: Command slug used for the binary wrapper (`~/.local/bin/myapp`),
  payload directory (`~/Applications/myapp-extracted`), and AppImage staging.

---

## Method B — Manual Streamlined Steps

If installing manually without the helper script, follow these steps to extract and
wire up the bundled assets.

### Step 1 — Stage and extract the AppImage

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

### Step 2 — Create the launcher wrapper

`[Agent/qrexec]` or `[Human]`:

```bash
cat > ~/.local/bin/myapp << 'EOF'
#!/bin/bash
exec "$HOME/Applications/myapp-extracted/AppRun" "$@"
EOF
chmod +x ~/.local/bin/myapp
```

### Step 3 — Install icons

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

### Step 4 — Install and patch the bundled `.desktop` file

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

---

## Step 5 — Add to the Qubes app menu in dom0

`[Human/dom0]`:

1. Open **Qube Manager** → select the AppVM → **Qube Settings** → **Applications** tab.
2. Click **Refresh Applications**.
3. Select your application from the left column and move it to the right column to pin it.

Alternatively, refresh via dom0 terminal:

```bash
qvm-sync-appmenus <qube-name>
```

---

## Verification

`[Agent/qrexec]` or `[Human]` inside the AppVM:

1. **Verify desktop syntax:**
   ```bash
   desktop-file-validate ~/.local/share/applications/*.desktop
   ```
2. **Verify dom0 icon resolution:**
   Test that Qubes' internal icon lookup resolves the icon:
   ```bash
   /usr/lib/qubes/xdg-icon <icon-name> 128
   ```
   *(Replace `<icon-name>` with the value of `Icon=` from the `.desktop` file, e.g. `ai.opencode.desktop`)*.
3. **Verify wrapper execution:**
   ```bash
   test -x ~/.local/bin/myapp && echo "Wrapper is executable"
   ```
   *(Note: For graphical Electron/desktop apps, running `myapp` directly in a shell opens the full GUI window. Test interactive launch from the desktop launcher or terminal as a human; avoid invoking GUI binaries in non-interactive/automated agent sessions as they will block until the window is closed).*

---

## Updates

To update an application:

- **With helper script:**
  ```bash
  appimage-install ~/Downloads/MyApp-1.3.0.AppImage myapp
  ```
- **Manually:**
  Replace `~/Applications/myapp.AppImage`, re-extract to `~/Applications/myapp-extracted`,
  and refresh icons/desktop files. The wrapper `~/.local/bin/myapp` and dom0 menu
  configuration remain unchanged.

---

## Rationale & Gotchas

### Why FUSE matters (`debian-13` vs `whonix-workstation-18`)

AppImages mount via FUSE at runtime. `whonix-workstation-18` includes `libfuse2`, but
Debian 13 (Trixie) ships only `libfuse3` by default. Running traditional Type 2 AppImages
directly on Debian 13 fails with:
```
dlopen(): error loading libfuse.so.2
```
Do **not** install `libfuse2` into the template or via root hooks. Extracting once to
`~/Applications/myapp-extracted/` completely avoids FUSE requirements, eliminates
startup extraction lag, and keeps the TemplateVM pristine.

### Why you should never write `.desktop` files by hand

Hand-crafted desktop files invariably miss key metadata:
1. **Icon name mismatches:** Packagers name icons using reverse-DNS or AppStream IDs
   (e.g. `ai.opencode.desktop.png`), not simple names like `myapp.png`. Hand-written
   names cause missing icons in dom0 and application menus.
2. **`StartupWMClass`:** Crucial for XFCE, GNOME, and KDE. Without this, window managers
   cannot match running windows to the desktop launcher, breaking panel icons and window grouping.
3. **MIME & URL schemes:** Misses registered URI handlers (e.g. `x-scheme-handler/opencode;`)
   needed for web browser integrations or OAuth callbacks.

---

## Status

Confirmed working on `debian-13-xfce` (`id0-agents`) using OpenCode Desktop AppImage,
including icon resolution via `/usr/lib/qubes/xdg-icon` and dom0 application menu sync.

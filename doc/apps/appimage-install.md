# Generic AppImage Installation (Qubes AppVM)

## Context

Goal: install an arbitrary AppImage in an AppVM (Whonix-based or `debian-13`-based)
entirely under `$HOME`, with no changes to the underlying TemplateVM and no `apt`
install of anything. Consistent with the minimal-footprint discipline applied
elsewhere in this environment (see `pass-install.md`, `opencode-install.md`).

This is the generic pattern for any single-file AppImage app (Obsidian, Jan, etc.).
App-specific quirks get their own doc; this one covers the common path.

## Step 1 — Place the file

```bash
mkdir -p ~/.local/bin ~/Applications
mv ~/Downloads/MyApp-1.2.3.AppImage ~/Applications/myapp.AppImage
chmod +x ~/Applications/myapp.AppImage
```

Two conventions worth keeping:

- Store the AppImage itself in `~/Applications/`, not `~/.local/bin/` — it's a
  payload, not a script you'd expect on `$PATH` directly.
- Rename to a stable, version-free filename (`myapp.AppImage`). Upstream release
  filenames carry version strings; a stable name means the `.desktop` file and any
  wrapper script never need to change across updates.

Everything here lives under `$HOME`, so it persists across AppVM reboots without
touching the TemplateVM — same pattern as the `pass` and OpenCode installs.

## Step 2 — FUSE: know before you run

AppImages normally mount themselves via FUSE. **`debian-13` templates do not ship
FUSE by default; `whonix-workstation-18` does.** This is the main fork in the road:

| Template | FUSE present | Action needed |
|---|---|---|
| `whonix-workstation-18` | yes | run directly, no extra step |
| `debian-13` (and derivatives) | no | extract first, or run with `--appimage-extract-and-run` |

Newer AppImages bundle FUSE 3 statically and may run on `debian-13` with no extra
step regardless — try running it directly first (Step 3) and only fall back to
extraction if you see `dlopen(): error loading libfuse.so.2` or similar.

Do **not** install `libfuse2`/`fuse` into the template or via a per-boot `apt`
hook to work around this — that's the wrong tradeoff here.
The extraction path below is the supported, no-template-touch answer.

## Step 3 — Run it

**If FUSE is present (Whonix), or the AppImage doesn't need it:**

```bash
~/Applications/myapp.AppImage
```

Nothing further needed — the file runs and self-mounts as designed.

**If FUSE is absent (`debian-13`) and the app requires it, extract once:**

```bash
cd ~/Applications
./myapp.AppImage --appimage-extract
mv squashfs-root myapp-extracted
```

Then launch via the extracted `AppRun`:

```bash
~/Applications/myapp-extracted/AppRun
```

This produces a small wrapper script so the launch command stays stable regardless
of which path you're on:

```bash
cat > ~/.local/bin/myapp << 'EOF'
#!/bin/bash
exec ~/Applications/myapp-extracted/AppRun "$@"
EOF
chmod +x ~/.local/bin/myapp
```

(If instead you're on the direct-run path, point this wrapper at
`~/Applications/myapp.AppImage` instead of an `AppRun` path.)

### Why extract-and-keep, not `--appimage-extract-and-run` every time

`--appimage-extract-and-run` re-extracts to a fresh temp directory on *every*
launch — correct for a one-off/build-script context, wasteful for an app you'll
launch repeatedly. Extracting once into a persistent `~/Applications/myapp-extracted/`
and launching `AppRun` directly gets the same FUSE-free behavior with instant
startup thereafter. Reserve `--appimage-extract-and-run` for apps you run rarely
enough that the disk churn doesn't matter.

## Step 4 — Desktop integration

Icon first — extract it so the `.desktop` file has something to point at:

- If you extracted in Step 3, the icon is already sitting in
  `~/Applications/myapp-extracted/` (usually `.DirIcon` or a `myapp.png` at the
  top level).
- If you're running the AppImage directly (no extraction), pull the icon out
  without a full extract:
  ```bash
  cd /tmp
  ~/Applications/myapp.AppImage --appimage-extract '*.png' '*.svg' >/dev/null
  mkdir -p ~/.local/share/icons/hicolor/256x256/apps
  cp squashfs-root/myapp.png ~/.local/share/icons/hicolor/256x256/apps/myapp.png
  rm -rf squashfs-root
  ```

Then the `.desktop` file:

```ini
[Desktop Entry]
Name=MyApp
Exec=/home/user/.local/bin/myapp %U
Icon=myapp
Type=Application
Categories=Utility;
```

Notes:

- `Exec` should point at the wrapper script from Step 3, not the raw
  `.AppImage`/`AppRun` path — that's the whole reason the wrapper exists: one
  stable target regardless of which FUSE path you took.
- `Icon=myapp` refers to the icon by name (matches the filename dropped into
  `~/.local/share/icons/...`), not a full path — lets the icon theme resolve it
  at whatever resolution is needed.
- `%U` passes through file/URL args from the file manager or a URL handler (e.g.
  double-clicking a `.md` file associated with Obsidian launches
  `myapp %U /path/to/file.md`). Omit it for apps that never take a file argument.

Save as `~/.local/share/applications/myapp.desktop`, then refresh:

```bash
update-desktop-database ~/.local/share/applications/
```

## Step 5 — Add to the Qubes app menu

In dom0: **Qube Manager → (select the AppVM) → Applications tab → Refresh
Applications**, then select `MyApp` from the list to pin it into the qube's menu
entry in dom0's app launcher.

If it doesn't show up after refresh, confirm the `.desktop` file has `Type=Application`
and no syntax errors (`desktop-file-validate ~/.local/share/applications/myapp.desktop`
inside the AppVM), then refresh again.

## Updates

There is no update mechanism here — deliberately. This whole approach trades
auto-updates for zero template footprint. To update:

```bash
# download the new release, then:
mv ~/Downloads/MyApp-1.3.0.AppImage ~/Applications/myapp.AppImage
chmod +x ~/Applications/myapp.AppImage

# if you're on the extraction path, re-extract:
rm -rf ~/Applications/myapp-extracted
./myapp.AppImage --appimage-extract
mv squashfs-root myapp-extracted
```

Because the wrapper script, `.desktop` file, and icon all reference the stable
`myapp`/`myapp.AppImage` names rather than versioned filenames, none of them need
to change on update — only the binary/extracted tree underneath gets replaced.

## Status

Confirmed working for both the direct-run (Whonix) and extract (`debian-13`)
paths, including desktop menu integration in dom0.

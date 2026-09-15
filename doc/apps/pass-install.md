# Installing `pass` in an AppVM without modifying the template

## Context

Goal: run [`pass`](https://www.passwordstore.org/) in a `debian-13`-based AppVM without installing anything into the shared
`debian-13` template (used by other AppVMs) and without relying on the AppVM's root filesystem.
**Everything must live under `$HOME` (or another already-persistent
path) to survive a reboot, and nothing may be installed via `apt` in this AppVM.**

## Dependency audit

`pass`'s real hard dependencies, confirmed from `src/password-store.sh` upstream:

| Dependency | Status in this AppVM | Notes |
|---|---|---|
| `bash` | present | default shell |
| GNU `getopt` (util-linux) | present | essential package |
| `git` | present | already in this `debian-13` template |
| `gnupg2` | present | already in this `debian-13` template |
| `tree` (≥1.7.0) | **missing** — installed locally, see below | hard dependency, no fallback: used unconditionally by `cmd_show()`/`cmd_find()` (bare `pass`, `pass find`) |
| `xclip`/`wl-clipboard` | `xclip` present | only required for `pass -c` |
| `qrencode` | not installed, not needed | only required for QR-code display |

## Step 1 — `tree` via user-space `dpkg -x` extraction

`tree` is a small, self-contained binary with a shallow dependency footprint (glibc only),
so it's a good candidate for extraction rather than a full `apt install`:

```bash
cd /tmp
apt-get download tree
dpkg -x tree_*.deb ~/.local/tree-pkg
ln -s ~/.local/tree-pkg/usr/bin/tree ~/.local/bin/tree
```

This unpacks the package contents without registering it in `dpkg`'s database and without
touching anything outside `$HOME`. Verified working: `tree --version`.

**Caveat:** this approach sacrifices automatic updates for `tree` -- updates are manual-only. That's fine for simple utilities like `tree` with no network exposure, no real attack surface of any kind, and whose dependencies are already satisfied by the base image. It's not a general substitute for `apt install` —
packages with agent/daemon/socket integration (e.g. `gnupg2` itself) are much more fragile
extracted this way, which is part of why we're glad `git`/`gnupg2` didn't need this treatment.

## Step 2 — `pass` via `make install PREFIX=~/.local`

### Why not just copy `password-store.sh` into `~/.local/bin`?

The upstream script does this near the top:

```
source "$(dirname "$0")/platform/$(uname | cut -d _ -f 1 | tr '[:upper:]' '[:lower:]').sh" 2>/dev/null
```

It expects a `platform/linux.sh` file *relative to the script's own location*. A bare copy
or symlink breaks this `source` — **silently**, since stderr is discarded — leaving
platform functions (clipboard copy/clear, used by `pass -c`) undefined. The upstream
`Makefile` avoids this by `sed`-patching the script at install time to embed absolute paths.
So: use `make install`, not a raw copy.

### Install

```bash
git clone https://git.zx2c4.com/password-store ~/src/password-store
cd ~/src/password-store
make install PREFIX="$HOME/.local" WITH_ALLCOMP=yes
```

`WITH_ALLCOMP=yes` is required, not optional, on a fresh prefix: the Makefile only
auto-enables completion installation if the target completion directory already exists
(`wildcard $(BASHCOMPDIR)`), which it won't on a first install into `~/.local`.

This installs, entirely under `$HOME`:

- `~/.local/bin/pass` — patched, location-independent script
- `~/.local/share/man/man1/pass.1`
- `~/.local/lib/password-store/platform.sh` + `extensions/` — platform functions and
  system-extension dir, wired via absolute paths baked in at install time
- `~/.local/share/bash-completion/completions/pass` (+ zsh/fish equivalents)

No `sudo`, no `apt`, nothing outside `$HOME` touched.

### Verification

```bash
which pass      # ~/.local/bin — Debian's default ~/.profile already adds ~/.local/bin to PATH
pass --version
man pass        # man-db auto-maps ~/.local/bin -> ~/.local/share/man; no config needed
```

Bash completion loads automatically if the `bash-completion` package is present (scans
`~/.local/share/bash-completion/completions` by default).

## Result

`pass` + all hard dependencies are installed and persistent, entirely within the AppVM's
private volume. The shared `debian-13` template is untouched. Nothing here requires `apt`,
root, or a template rebuild to survive a reboot.

## Forward reference: extensions

`pass` extensions (e.g. `pass-export`, used for a planned Vaultwarden migration) are
enabled via `PASSWORD_STORE_ENABLE_EXTENSIONS=true` and read from the **user** extensions
directory, which defaults to `~/.password-store/.extensions/<name>.bash` — distinct from
the `LIBDIR/password-store/extensions` system dir this install created. Drop extensions
there, not under `~/.local/lib`.

## If long-term persistence beyond this AppVM's lifetime is needed

This install is scoped to a single AppVM's private volume. If `pass` needs to outlive the
AppVM (e.g. rebuilt from a fresh template clone) or be reproducible elsewhere, this
document is the reproduction recipe — re-run steps 1–2 verbatim in any `debian-13`-based
AppVM. No template-level or Salt-level changes are implied or required.

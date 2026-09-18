# QR Scanning in Qubes (`zbar-tools`)

## Context

Goal: scan QR codes from inside Qubes AppVMs, from two sources:

- **Images** — screenshots or image files (typically PNG).
- **The laptop camera** — attached manually from `sys-usb` to the qube doing
  the scanning.

Scanning is occasional, not routine. That shapes the design: proportionate
hardening, no custom plumbing.

Chosen design:

1. Install `zbar-tools` **with `--no-install-recommends`** in the
   `debian-13-xfce-brave` template.
2. Create a dedicated **no-network AppVM, `qr-scan`**, on that template. Use it
   for camera scanning and for images from untrusted sources.

Other qubes on `debian-13-xfce-brave` also get the tool (see Gotchas), so a
quick decode of a trusted screenshot can happen in place.

`debian-13-xfce-brave` is a clone of `debian-13-xfce` that carries Brave and a
small handful of other apps. This is a deliberate exception to the
community-templates-only policy. The doc describing that exception is
`<not yet written>`.

## Dependency audit

Package facts below were checked against packages.debian.org (trixie) on
2026-09-18. Re-verify with `apt-get install -s` (Step 1) before trusting them.

| Item | Detail |
|---|---|
| Package name | `zbar-tools` — **not** `zbarimg`. `zbarimg` and `zbarcam` are binaries inside it. |
| `zbar-tools` depends on | `libzbar0t64`, and ImageMagick 6's wand library (`libmagickwand-6.q16-7t64`). `zbarimg` uses ImageMagick to load images. |
| `libzbar0t64` depends on | `libjpeg62-turbo`, `libv4l-0t64`, `libx11-6`, `libdbus-1-3` |
| `libzbar0t64` **recommends** | `libmagickcore-6.q16-7-extra` — adds SVG, WMF, OpenEXR, DjVu and Graphviz support to ImageMagick |
| `zbar-tools` suggests | `zbarcam-gtk`, `zbarcam-qt` — not needed |

Which code path touches what:

- **`zbarimg` (images):** goes through ImageMagick's image loading. This is the
  large attack surface of the whole setup.
- **`zbarcam` (camera):** as far as I know, reads the camera through libzbar's
  video path (`libv4l`, plus `libjpeg` for MJPEG frames) and does not use
  ImageMagick coders. **Unverified** — the `ldd` check in Verification
  confirms it.

## Design rationale

- **Template install, not a user-space venv.** A QR decoder parses untrusted
  images, so it should receive Debian security updates through the template.
  `pyzbar` is only a ctypes wrapper and still needs the system `libzbar0`.
  The `opencv-python` alternative is a large bundled binary wheel with no
  automatic updates. This is the opposite of the `pass`/`tree` case in
  `pass-install.md`, where skipping updates was acceptable for a tool with no
  attack surface.
- **`--no-install-recommends`.** A full `debian-13-xfce` clone installs
  Recommends by default. Without the flag, the extra-codecs package (SVG, WMF,
  OpenEXR, DjVu, Graphviz) would come along. PNG and JPEG, the formats QR
  images almost always use, do not depend on it. Excluding it removes the
  exotic-format decoders that ImageMagick has historically had the most
  trouble with.
- **Not in `debian-13-minimal-net`.** That template backs `lan-proxy` and
  `sys-vpn-id0-proton`, the most sensitive qubes. See `lan-restricted-proxyvm.md`
  and `vpn-proxyvm.md`. ImageMagick libraries do not belong there.
- **`qr-scan` has no network.** A hostile image or QR payload can compromise
  the parser, but a qube with no network has nowhere to send anything. Isolation
  here is of *execution*, not *installation*: the package still sits on disk in
  the shared template.
- **Camera goes to `qr-scan`, not a Brave qube.** Attaching the camera passes
  the whole USB device to that qube, which runs the kernel's USB video driver
  and can use the camera for as long as it is attached. A network-connected
  browser qube should not hold that. Attach only for the duration of a scan,
  then detach.

## Step 1 — Install in the template `[Human/dom0]`

**This step modifies a template shared by other qubes.** Every qube based on
`debian-13-xfce-brave` gets `zbar-tools` and ImageMagick core libraries.
Whonix-based qubes are unaffected.

```bash
# [Human/dom0]
qvm-run -u root debian-13-xfce-brave xterm
```

In that root shell:

```bash
# [Human/dom0] — typed inside the template's root shell
apt-get update
apt-get install -s --no-install-recommends zbar-tools   # simulate; review the package list
apt-get install --no-install-recommends zbar-tools
```

The simulated run should not list `libmagickcore-6.q16-7-extra`. If it does,
stop and investigate before installing.

```bash
# [Human/dom0]
qvm-shutdown debian-13-xfce-brave
```

Restart any qubes based on the template that were running.

## Step 2 — Create `qr-scan` `[Human/dom0]`

```bash
# [Human/dom0]
qvm-create --class AppVM --template debian-13-xfce-brave --label <label> qr-scan
qvm-prefs qr-scan netvm ""
qvm-prefs qr-scan netvm      # confirm: no NetVM set
```

`<label>` is the color you choose; not yet decided.

## Step 3 — Scan an image or screenshot `[Human]`

Get the image into `qr-scan`:

```bash
# [Human] — run in the source qube; dom0 shows its standard copy prompt
qvm-copy ~/path/to/screenshot.png      # choose qr-scan as the target
```

Take the screenshot **inside the qube that is displaying the QR code**. An
AppVM's screenshot tool only sees that qube's own windows, so a QR shown in
another qube can't be captured from here.

In `qr-scan`, the file lands in `~/QubesIncoming/<source-qube>/`:

```bash
# [Human] — inside qr-scan
zbarimg --quiet --raw ~/QubesIncoming/<source-qube>/screenshot.png
```

Copy the decoded text out with the Qubes clipboard: select it, `Ctrl+Shift+C`,
then `Ctrl+Shift+V` in the destination qube. Treat the result as untrusted
data; do not paste it into a shell without reading it.

## Step 4 — Scan with the camera `[Human]`

Attach the camera from `sys-usb` to `qr-scan`, using the Qubes Devices widget
or from dom0:

```bash
# [Human/dom0]
qvm-usb                                              # find the camera's device ID
qvm-usb attach qr-scan sys-usb:<camera-device-id>
```

In `qr-scan`:

```bash
# [Human] — inside qr-scan
ls /dev/video*
zbarcam --raw /dev/video0
```

Laptop cameras often expose more than one `/dev/video*` node. If `zbarcam`
fails on the first, try the next. Then detach immediately:

```bash
# [Human/dom0]
qvm-usb detach qr-scan sys-usb:<camera-device-id>
```

## Verification

`qr-scan` has no network, so there is no SSH path into it for an agent. All
checks below are `[Human]` unless tagged otherwise.

1. **No network** (`[Human/dom0]`): `qvm-prefs qr-scan netvm` shows no NetVM.
   Inside `qr-scan`, `ping -c1 1.1.1.1` must fail.
2. **Install as intended:** in `qr-scan`, `zbarimg --version` works and
   `dpkg -l libmagickcore-6.q16-7-extra` reports it is **not** installed.
3. **Image decode:** decode a PNG screenshot of any known QR code with
   `zbarimg --quiet --raw`.
4. **Camera path avoids ImageMagick coders** (confirms an unverified claim
   above): `ldd "$(command -v zbarcam)" | grep -i magick` should print nothing.
   If it prints ImageMagick libraries, revisit the Design rationale.
5. **Camera decode:** attach, scan a QR code with `zbarcam`, detach.
6. **After every template update:** repeat the `dpkg -l
   libmagickcore-6.q16-7-extra` check to confirm the extra-codecs package did
   not arrive as a new Recommends of a later package.

## Gotchas

- **Missing decoders fail closed.** With `--no-install-recommends`, SVG, WMF,
  OpenEXR, DjVu and Graphviz inputs are refused. The error should name a
  missing decode delegate (unverified wording). Workaround: re-capture the QR
  as PNG. Adding `libmagickcore-6.q16-7-extra` to the template is possible but
  widens the parser surface for every qube on it. That is a human decision,
  not a quiet fix.
- **Screenshots are per-qube.** See Step 3.
- **The camera is a whole-device attachment.** Attach only for the scan and
  detach after.
- **Template scope.** The tool is present in every qube on
  `debian-13-xfce-brave`, including Brave qubes. Only `qr-scan` combines it
  with no network.
- **Version drift.** The dependency data above is a 2026-09-18 snapshot; sid had
  already moved to ImageMagick 7. Re-check with the simulated install.

## Considered and not adopted

- **Python venv (`pyzbar` / `opencv-python`).** Needs the system `libzbar0`
  anyway, or a large bundled wheel with no automatic updates, plus PyPI supply
  chain exposure.
- **`dpkg -x` user-space extraction** (as in `pass-install.md`). The ImageMagick
  dependency chain is too deep for this to be practical.
- **Install in `debian-13-minimal-net`.** Rejected; see Design rationale.
- **`qubes-video-companion` for the camera.** Streams the camera to a qube as
  one-way raw video, so the qube never handles the USB device. It needs a dom0
  package plus a package in the qube, and I have not confirmed how well it
  works on 4.3. The current manual `sys-usb` attach is simpler for occasional
  use. Revisit if camera scanning becomes routine.
- **qrexec disposable decode service.** A custom service in a `template_for_dispvms`
  qube: an image goes in on stdin, decoded text comes out, and a fresh no-network
  disposable is created per call. Not adopted, for several reasons: every call
  boots a disposable; it cannot serve a live camera stream; it needs a custom
  service script and a dom0 policy line; and its text output returns into the
  calling qube, so it would need control-character stripping and a length cap.
  The package would still have to live in some template. Illustrative shape only,
  to verify against 4.3 docs before use:

  ```bash
  # in the template behind the disposable: /etc/qubes-rpc/my.QRDecode
  #!/bin/sh
  t=$(mktemp) && cat > "$t" && zbarimg --quiet --raw "$t" | head -c 4096 | tr -cd '[:print:]\n'

  # dom0: /etc/qubes/policy.d/30-user.policy
  my.QRDecode  *  <client-qube>  @dispvm:<qr-dvm>  allow

  # caller
  qrexec-client-vm @dispvm:<qr-dvm> my.QRDecode < shot.png
  ```

  Qubes 4.3 preloaded disposables (the `preload-dispvm-max` feature on the
  disposable template) reduce the startup delay, at the cost of RAM per
  preloaded qube.

## Forward references

- `pass-install.md` — the user-space-install tradeoffs and the update caveat
  that does not carry over to this tool.
- `lan-restricted-proxyvm.md`, `vpn-proxyvm.md` — the `debian-13-minimal-net`
  template this design deliberately avoids.
- `<debian-13-xfce-brave template doc>` — not yet written.

## Status

Design proposed, not yet implemented.

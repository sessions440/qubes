# USB Printer Setup (Template-wide & AppVM)

## Status

`Confirmed working` — Live verified on Qubes OS 4.3 using Debian 13 (`debian-13-xfce-brave`) and a Brother HL-L2320D laser printer.

## Context

When attaching a USB printer to a Qubes OS AppVM via `sys-usb`, printing fails out of the box for two reasons:
1. **CUPS is gated by Qubes services**: `/usr/lib/systemd/system/cups.service.d/30_qubes.conf` and `cups.socket.d/30_qubes.conf` enforce `ConditionPathExists=/var/run/qubes-service/cups`. By default, CUPS will not start in an AppVM unless dom0 explicitly enables the `cups` service for that specific VM.
2. **Missing printer driver and queue**: Many laser printers (such as host-based Brother GDI printers) require specialized drivers (e.g., `printer-driver-brlaser`) and a pre-configured CUPS queue to translate print jobs.

This document describes the **core setup**: installing the driver, queue, and systemd socket-activation override inside the **TemplateVM**. This makes printing completely zero-touch for any child AppVM: simply attach the USB printer to any running AppVM and print immediately.

Alternative setups (persistent USB auto-attachment to a preferred qube, and isolating the printer to a single AppVM without template modifications) are documented at the end.

---

## Prerequisites & Dependency Audit

- **TemplateVM**: Debian 13 (e.g. `debian-13-xfce-brave` or standard Debian template).
- **Driver package**: `printer-driver-brlaser` (for Brother laser printers, including HL-L2320D). Other printer families use `printer-driver-gutenprint`, `hplip`, etc.
- **Hardware**: Brother USB printer attached to machine.

---

## Core Setup: Template-Wide "Plug-and-Print"

All steps are performed once in the TemplateVM.

### 1. Clear the Qubes service condition `[Human/dom0]`

Inside a root shell in the TemplateVM (via `qvm-run -u root <template> bash` from dom0):

Add systemd overrides to clear `ConditionPathExists` on both the CUPS socket and service:

```bash
mkdir -p /etc/systemd/system/cups.socket.d /etc/systemd/system/cups.service.d
printf "[Unit]\nConditionPathExists=\n" > /etc/systemd/system/cups.socket.d/qubes-override.conf
printf "[Unit]\nConditionPathExists=\n" > /etc/systemd/system/cups.service.d/qubes-override.conf
systemctl daemon-reload
```

*Note: Clearing this condition allows systemd's built-in socket activation (`cups.socket`) to handle CUPS on-demand across all child AppVMs without modifying dom0 services.*

### 2. Install driver and configure the printer queue `[Human/dom0]`

In the same root shell:

```bash
# Install driver
apt update && apt install -y printer-driver-brlaser

# Start CUPS temporarily to register the printer
systemctl start cups cups.socket

# Add the printer queue with default US Letter paper size
lpadmin -p Brother_HL-L2320D -E \
  -v "usb://Brother/HL-L2320D%20series?serial=U63877C9N736113" \
  -m drv:///brlaser.drv/brl2320d.ppd \
  -o PageSize=Letter

# Set as default printer destination
lpadmin -d Brother_HL-L2320D

# Stop CUPS before shutting down the template
systemctl stop cups
```

*(Note: To default to another paper size such as US Legal or A4, replace `-o PageSize=Letter` with `-o PageSize=Legal` or `-o PageSize=A4`.)*

### 3. Shut down the template `[Human/dom0]`

In **dom0**:

```bash
qvm-shutdown <template>
```

---

## Verification

1. Start any AppVM based on the updated template.
2. `[Human]` Attach the USB printer to that AppVM using the Qubes Devices tray widget (`sys-usb:X-X - Brother ...`).
3. Inside the AppVM, verify the printer queue status:
   ```bash
   lpstat -p -d
   ```
   Output should show:
   ```text
   printer Brother_HL-L2320D is idle. enabled since ...
   system default destination: Brother_HL-L2320D
   ```
4. Verify default paper size:
   ```bash
   lpoptions -p Brother_HL-L2320D -l | grep PageSize
   ```
   Output should highlight `*Letter` (or your chosen size):
   ```text
   PageSize/Media Size: A4 A5 A6 B5 B6 EnvC5 EnvMonarch EnvDL Executive Legal *Letter
   ```
5. `[Human]` Open Brave (or any application) in the AppVM, press `Ctrl + P`, verify `Brother_HL-L2320D` appears in the Destination dropdown, and print a test page.

---

## Rationale & Gotchas

### Socket activation efficiency
Enabling CUPS via `cups.socket` means the background daemon (`cupsd`) does not sit active eating memory or CPU when idle. Systemd holds the Unix socket (`/run/cups/cups.sock`) and automatically launches `cupsd` only when an application opens a print dialog or submits a print job.

### CUPS PPD deprecation warning
When running `lpadmin`, CUPS may output:
```text
lpadmin: Printer drivers are deprecated and will stop working in a future version of CUPS.
```
This is a standard upstream notice from OpenPrinting indicating that future major releases (CUPS 3.x) will transition from PPD files to IPP "Printer Applications". On Debian 13 (Trixie) and CUPS 2.4.x, PPD drivers are fully supported and stable; this warning can be safely ignored.

### USB Device URI
The URI `usb://Brother/HL-L2320D%20series?serial=...` includes the hardware serial number. If you replace the physical printer, query the new URI in an AppVM with `sudo lpinfo -v` and update the queue.

---

## Alternative Setups

### Alternative 1: Persistent USB auto-attachment to a preferred qube `[Human/dom0]`

If you primarily print from a single qube (e.g. `id0-agents` or `work`) and do not want to manually attach the USB printer each time:

1. Identify the printer's device ID in dom0:
   ```bash
   qvm-usb list
   ```
   Look for the entry corresponding to your Brother printer (e.g., `sys-usb:1-2`).
2. Attach it persistently to the preferred qube:
   ```bash
   qvm-usb attach --persistent <target-appvm> sys-usb:<device-id>
   ```
Whenever the qube boots and the printer is plugged in, Qubes will automatically attach it to that qube.

### Alternative 2: Single AppVM only (unmodified template)

If you follow strict minimal-footprint discipline and prefer **not** to touch the shared template:

1. **Enable CUPS in dom0 for that AppVM only** `[Human/dom0]`:
   ```bash
   qvm-service <target-appvm> cups on
   ```
2. **Persist `/etc/cups` across reboots via `bind-dirs`** `[Human]` or `[Agent/qrexec]`:
   Inside `<target-appvm>`:
   ```bash
   sudo mkdir -p /rw/config/qubes-bind-dirs.d
   echo "binds+=( '/etc/cups' )" | sudo tee -a /rw/config/qubes-bind-dirs.d/50_user.conf
   ```
3. **Install driver and configure queue in the AppVM**:
   *Note: Because packages in `/usr` do not persist in AppVMs across reboots, `printer-driver-brlaser` must either be reinstalled on boot via `/rw/config/rc.local`, or placed in a dedicated standalone clone template.*

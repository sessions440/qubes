# Troubleshooting

Dated entries, newest first. Each entry has **Symptom**, **Cause** and **Fix**.
Commands are role-tagged as in `AGENTS.md`. Add an entry only after the fix has
been verified on the real system.

## 2026-11-09: sys-net won't start: "Unable to reset PCI device"

**Symptom**

`qvm-start sys-net` fails in dom0:

```
Start failed: internal error: Unable to reset PCI device 0000:00:1f.6: no FLR, PM reset or bus reset available, see /var/log/libvirt/libxl/libxl-driver.log for details
```

`00:1f.6` is the Intel Ethernet controller (`8086:550a`) assigned to sys-net.
Noticed after a suspend/resume cycle; suspend was not shown to be the trigger.

Clues that mislead:

- `/var/log/libvirt/libxl/libxl-driver.log` and `sys-net.log` were not updated
  (last write days earlier) and `journalctl -b -u libvirtd` was empty. The
  failure is raised before libxl logs anything, so these logs show nothing.
- `qvm-pci ls sys-net` printed nothing for the halted qube.
  `qvm-pci ls` (no argument) showed the device as not used by any qube.
- Old "kernel doesn't support reset from sysfs" lines in the log named
  `0000:00:14.0` (USB controller). Unrelated.
- A device that is healthy looks like this, and this one was healthy:
  bound to `pciback`, power state D0, `power/control` = `on`, listed in
  `xl pci-assignable-list`. A reboot would not have changed any of it.

**Cause**

sys-net's persistent assignment of `00_1f.6` in `/var/lib/qubes/qubes.xml` had
no options. Without `no-strict-reset`, libvirt insists on a reset method it
recognizes (FLR, PM reset or bus reset). The kernel did offer resets for this
device (`reset_method` = `device_specific af_flr`), but libvirt's check
evidently does not accept them, so it refused.

Not known: why sys-net started fine before Nov 6. The option may never have
been set while the check passed, or something changed since. Suspend/resume was
not shown to be the cause.

**Fix**

1. `[Human/dom0]` Check the assignment (the `<device>` line has no `<option>`
   child when the option is missing):
   ```bash
   grep -n -A3 'id="00_1f.6"' /var/lib/qubes/qubes.xml
   ```

2. `[Human/dom0]` Replace the assignment with one that has the option:
   ```bash
   qvm-pci unassign sys-net dom0:00_1f.6
   qvm-pci assign --required --option no-strict-reset=True sys-net dom0:00_1f.6
   qvm-start sys-net
   ```
   If the `assign` fails, rerun it before anything else: sys-net has no
   Ethernet until it succeeds.

3. `[Human/dom0]` Confirm the option stuck: repeat step 1; it should now show
   `no-strict-reset` set to `True`.

Notes:

- **Qubes 4.3 syntax.** Persistent assignment is `qvm-pci assign` / `unassign`.
  `attach` / `detach` only affect a running qube, so
  `attach --persistent` and `detach` both failed with misleading errors
  ("already assigned", "not attached").
- **Security trade-off.** `no-strict-reset` skips the reset check, so the
  device may not be reset between uses. As I understand the Qubes docs, the
  risk is state carrying over between qubes that share the device. This
  device is only ever assigned to sys-net, so the exposure is small. Don't
  copy the option onto devices that move between qubes without thinking it
  through. (Not verified against the current docs.)
- **Wi-Fi (`00_14.3`)** is assigned without the option and starts fine. Leave
  it alone unless it produces the same error.
- **Unrelated.** `xl dmesg` showed CPU thermal-throttling messages during the
  session.

Verified on the real system: fresh `qvm-start sys-net` succeeded with the
option; after a suspend/resume cycle Wi-Fi recovered, and
`qvm-shutdown --wait --force sys-net` followed by `qvm-start sys-net` was
clean. `--force` stopped only sys-net, not the qubes using it.

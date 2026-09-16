# Fix: disable NetworkManager autoconnect on `sys-vpn`

## Context

`nmcli connection import type wireguard` sets `connection.autoconnect=yes` by
default on imported profiles. Autoconnect *is* honored for native WireGuard
device connections (unlike stacked VPN-type connections), so NetworkManager
will silently re-activate the default connection whenever it sees the device
idle and the profile eligible — including right after a multi-connection
conflict forces things down. This causes unexpected reconnections outside of
the explicit boot-time `nmcli connection up` in `/rw/config/rc.local`.

Target qube: `sys-vpn` (AppVM, `debian-13-minimal-net`-derived template).
Real connection names on this host: `qubes-CA-1040`, `qubes-CA-1048`,
`qubes-US-MI-19`. Confirm these still match before proceeding — do not
assume placeholder names like `proton-ca` from older design docs.

## Steps

1. **Inventory current state.** Run and record the output:
   ```bash
   nmcli -f NAME,TYPE,DEVICE,AUTOCONNECT,AUTOCONNECT-PRIORITY connection show
   ```
   Confirm all three connections above are present and note which (if any)
   currently show `AUTOCONNECT: yes`.

2. **Disable autoconnect on all three** — the default connection included.
   Going forward, connection state changes should only happen explicitly
   (via `nmcli`, a wrapper script, or the boot-time `rc.local` line), never
   via NM's own autoconnect logic:
   ```bash
   sudo nmcli connection modify qubes-CA-1040 connection.autoconnect no
   sudo nmcli connection modify qubes-CA-1048 connection.autoconnect no
   sudo nmcli connection modify qubes-US-MI-19 connection.autoconnect no
   ```

3. **Inspect `/rw/config/rc.local`** (not `$HOME` — that's the wrong
   location and was a source of confusion earlier):
   ```bash
   cat /rw/config/rc.local
   ```
   Confirm the explicit `nmcli connection up <name>` line references one of
   the three real connection names above. If it references a stale
   placeholder name, fix it to the intended real default connection.

4. **Also verify the kill switch wildcard in the same file matches reality.**
   The nftables rule should use `oifname` matching the actual interface
   names. Check what NM actually names the interfaces:
   ```bash
   nmcli -f NAME,DEVICE connection show
   ```
   If the existing rule in `rc.local` uses a pattern like `"proton*"`, it
   will **not** match interfaces named `qubes-CA-1040` etc., meaning the
   kill switch silently fails to enforce fail-closed behavior for these
   connections. Update the pattern (e.g. to `"qubes-*"`, or list all three
   names explicitly) so it actually matches.

5. **Apply and reboot** `sys-vpn` (or cycle the connection manually with
   `nmcli connection down`/`up`) to test in a clean state.

6. **Verify all of the following:**
   - Exactly one WireGuard connection is active after boot:
     ```bash
     nmcli connection show --active
     ```
   - From a downstream AppVM: `curl https://ifconfig.me` shows the VPN IP.
   - Bringing the active tunnel down from `sys-vpn`
     (`sudo nmcli connection down <name>`) causes downstream connectivity to
     fail outright — not fall back to clearnet. This confirms the kill
     switch fires correctly with the corrected interface-name pattern.

7. **Report back:**
   - Final `AUTOCONNECT` state of all three connections.
   - Final content of `/rw/config/rc.local`.
   - Confirmation that the three checks in step 6 passed.

## Do not

- Re-enable `autoconnect` on any of the three connections.
- Rely on NM's own autoconnect logic to bring up the default connection —
  the explicit line in `rc.local` is the only intended activation path at
  boot.

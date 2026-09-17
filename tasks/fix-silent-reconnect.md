# Fix: NetworkManager multi-connection conflicts on `sys-vpn-id0-proton`

## Context

Two related NetworkManager (NM) issues on `sys-vpn-id0-proton`:

1. **Silent default reconnection.** `nmcli connection import type wireguard`
   sets `connection.autoconnect=yes` by default on imported profiles.
   Autoconnect *is* honored for native WireGuard device connections
   (unlike stacked VPN-type connections), so NM will silently re-activate
   the default connection whenever it sees the device idle and the
   profile eligible — including right after a multi-connection conflict
   (below) forces things down.

2. **Multiple simultaneously-active connections.** Each imported WireGuard
   profile creates its own independent virtual device with
   `AllowedIPs = 0.0.0.0/0`, so NM has no built-in reason to prevent two
   from being active at once (they're different devices, not competing
   for one slot). Two simultaneously-active default-route tunnels
   produces total connectivity loss. This can happen via the Qubes
   network widget, `nmcli`, or any other NM frontend — the fix must live
   at the NM layer, not depend on remembering to use one specific tool.

**Verify real names before proceeding.** Docs elsewhere in this repo have
been manually updated to real deployed names, but some may have been
missed — do not assume any name below (or any interface/connection name
referenced in other repo docs) is still a placeholder from an earlier
draft unless you've checked it against the live system with the commands
in Step 1.

Target qube: `sys-vpn-id0-proton`, template `debian-13-minimal-net`
(`debian-13-minimal`-derived). Real connection names confirmed on this
host: `qubes-CA-1040`, `qubes-CA-1048`, `qubes-US-MI-19`.

## Role tags

Every command below is tagged:
- **[Human/dom0]** — requires dom0 privilege (`qvm-*` commands, or opening
  a shell inside a TemplateVM). An agent operating over SSH inside a qube
  has no path to run these — they need a human at a dom0 terminal.
- **[Agent/SSH]** — runs inside an already-reachable, already-running qube
  over SSH. Safe for an agent to execute once network + SSH access to
  that qube is established.

## Part A — Disable autoconnect

1. **[Agent/SSH]** Inventory current state. Run and record the output:
   ```bash
   nmcli -f NAME,TYPE,DEVICE,AUTOCONNECT,AUTOCONNECT-PRIORITY connection show
   ```
   Confirm all three connections above are present; note which (if any)
   currently show `AUTOCONNECT: yes`.

2. **[Agent/SSH]** Disable autoconnect on all three — the default
   connection included. Going forward, connection state changes should
   only happen explicitly (via `nmcli`, the dispatcher script in Part B,
   or the boot-time `rc.local` line), never via NM's own autoconnect
   logic:
   ```bash
   sudo nmcli connection modify qubes-CA-1040 connection.autoconnect no
   sudo nmcli connection modify qubes-CA-1048 connection.autoconnect no
   sudo nmcli connection modify qubes-US-MI-19 connection.autoconnect no
   ```

3. **[Agent/SSH]** Inspect `/rw/config/rc.local` (not `$HOME` — that
   location has no relevance to boot-time bring-up; this path is
   per-AppVM persistent storage, editable directly over SSH, no dom0
   needed):
   ```bash
   cat /rw/config/rc.local
   ```
   Confirm the explicit `nmcli connection up <name>` line references one
   of the three real connection names above. Fix it if it references a
   stale placeholder.

4. **[Agent/SSH]** Verify the kill switch wildcard in the same file
   matches reality. Check what NM actually names the interfaces:
   ```bash
   nmcli -f NAME,DEVICE connection show
   ```
   If the existing nftables rule in `rc.local` uses a pattern like
   `"proton*"`, it will **not** match interfaces named `qubes-CA-1040`
   etc. — meaning the kill switch silently fails to enforce fail-closed
   behavior. Update the pattern (e.g. to `"qubes-*"`, or list all three
   names explicitly) so it actually matches. This edit is also a direct
   file edit under `/rw/config/rc.local` — no dom0 needed.

## Part B — Exclusive-activation dispatcher script

This is the proper fix for multi-connection conflicts: a NetworkManager
dispatcher script that fires on the generic `up` event for any interface
(native WireGuard connections don't emit `vpn-up`/`vpn-down`, so the
generic per-interface events are the correct hook) and brings down any
other `qubes-*` connection. This makes the *naive* action — picking a
different connection via the Qubes network widget, `nmcli`, or any other
frontend — correct by construction, rather than requiring a separate
CLI habit to remember.

**This must be added to the template (`debian-13-minimal-net`), not the
running `sys-vpn-id0-proton` AppVM** — `/etc` is part of the ephemeral
per-boot overlay in an AppVM and reverts to whatever the template
provides; changes need to be baked into the template image to persist.

Editing a template requires a dom0 session — there is no SSH-reachable
path into the template for this step. A human runs this part.

1. **[Human/dom0]** Shut down `sys-vpn-id0-proton`, then open a root
   shell in the template:
   ```bash
   qvm-shutdown sys-vpn-id0-proton
   qvm-run -u root debian-13-minimal-net xterm
   ```

2. **[Human/dom0]**, typed inside that root shell — create the
   dispatcher script:
   ```bash
   cat > /etc/NetworkManager/dispatcher.d/90-exclusive-vpn.sh << 'EOF'
   #!/bin/bash
   # Enforce exclusive WireGuard connection: bringing one up takes down the rest.
   IFACE="$1"
   ACTION="$2"
   VPN_PREFIX="qubes-"

   case "$ACTION" in
     up)
       [[ "$IFACE" == ${VPN_PREFIX}* ]] || exit 0
       for name in $(nmcli -t -f NAME connection show --active); do
         [[ "$name" == ${VPN_PREFIX}* ]] || continue
         dev=$(nmcli -t -f GENERAL.DEVICES connection show "$name" 2>/dev/null | cut -d: -f2)
         [[ "$dev" == "$IFACE" ]] && continue
         nmcli connection down "$name"
       done
       ;;
   esac
   EOF
   ```

3. **[Human/dom0]**, same shell — permissions matter, NM silently
   refuses to run scripts that don't meet these:
   ```bash
   chown root:root /etc/NetworkManager/dispatcher.d/90-exclusive-vpn.sh
   chmod 750 /etc/NetworkManager/dispatcher.d/90-exclusive-vpn.sh
   ```

4. **[Human/dom0]** Shut down the template and restart
   `sys-vpn-id0-proton` to inherit the change:
   ```bash
   qvm-shutdown debian-13-minimal-net
   qvm-start sys-vpn-id0-proton
   ```

## Verification

Once `sys-vpn-id0-proton` is back up and SSH-reachable, these are all
**[Agent/SSH]** except where noted:

1. **Autoconnect check:** confirm exactly one WireGuard connection is
   active and it's the intended default:
   ```bash
   nmcli connection show --active
   ```

2. **Exclusivity check:** manually bring up a second `qubes-*` connection
   while the first is active (`nmcli connection up qubes-CA-1048` while
   `qubes-CA-1040` is up) and confirm the dispatcher script brings the
   first one down automatically — check `nmcli connection show --active`
   immediately after, and check dispatcher logs if it doesn't:
   ```bash
   sudo journalctl -u NetworkManager | grep -i dispatcher
   ```

3. **Downstream connectivity** — requires access to a downstream AppVM,
   which the agent may or may not have; do via **[Human]** if not:
   `curl https://ifconfig.me` should show the VPN IP.

4. **Kill switch check** (**[Agent/SSH]** on `sys-vpn-id0-proton`, plus
   the same downstream check as #3): bring the active tunnel down
   (`sudo nmcli connection down <name>`) and confirm downstream
   connectivity fails outright — not falling back to clearnet.

5. **No silent reactivation:** leave the qube idle for several minutes
   with no connection active and confirm nothing comes back up on its
   own.

## Report back

- Final `AUTOCONNECT` state of all three connections.
- Final content of `/rw/config/rc.local`, including the corrected kill
  switch pattern.
- Confirmation the dispatcher script is in place with correct ownership
  and permissions.
- Results of all five verification checks above.

## Do not

- Re-enable `connection.autoconnect` on any of the three connections.
- Rely on NM's own autoconnect logic to bring up the default connection —
  the explicit line in `rc.local` is the only intended activation path at
  boot.
- Add the dispatcher script directly to the running `sys-vpn-id0-proton`
  AppVM's `/etc` — it will be silently lost on the next reboot. It
  belongs in the template.
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
   produce total connectivity loss. This can happen via the Qubes
   network widget, `nmcli`, or any other NM frontend — the fix must live
   at the NM layer, not depend on remembering to use one specific tool.

**Verify real names before proceeding.** Docs elsewhere in this repo have
been manually updated to real deployed names, but some may have been
missed — do not assume any name below (or any interface/connection name
referenced in other repo docs) is still a placeholder from an earlier
draft unless you've checked it against the live system with the commands
in Part A, step 1.

Target qube: `sys-vpn-id0-proton`, template `debian-13-minimal-net` (`debian-13-minimal`-derived).
Real connection names confirmed on this host: `qubes-CA-1040`,
`qubes-CA-1048`, `qubes-US-MI-19`.

## Prerequisites and privileges

- Shell access from the agent qube to `sys-vpn-id0-proton` over qrexec
  (`qubes.VMShell`), set up per `agent-qube-access.md`. Send a command
  with
  `printf '%s\n' '<command>' | qrexec-client-vm sys-vpn-id0-proton qubes.VMShell`.
- **The agent has `user` only, no root.** `debian-13-minimal-net` does not
  ship passwordless `sudo` (by design), so `sudo ...` fails in this qube.
  Everything that needs root is run by a human from dom0 with
  `qvm-run -u root sys-vpn-id0-proton '<command>'`.
- Files executed as root at boot (`/rw/config/rc.local`, dispatcher
  scripts) are root-equivalent. The agent **drafts** them; a human reviews
  and installs them. The agent never writes them in place.

## Role tags

- **[Human/dom0]** — requires dom0 privilege, root inside the qube via
  `qvm-run -u root`, or a shell inside the TemplateVM.
- **[Agent/qrexec]** — runs as `user` from the agent qube through
  `qrexec-client-vm`. Read-only checks and drafting only.
- **[Human]** — interactive step or review that should not be delegated.

## Part A — Disable autoconnect

1. **[Agent/qrexec]** Inventory current state (read-only). Record the output:
   ```bash
   nmcli -f NAME,TYPE,DEVICE,AUTOCONNECT,AUTOCONNECT-PRIORITY connection show
   ```
   Confirm all three connections above are present; note which currently
   show `AUTOCONNECT: yes`.

2. **[Agent/qrexec]** Read the current boot script and the interface names
   (read-only):
   ```bash
   cat /rw/config/rc.local
   nmcli -f NAME,DEVICE connection show
   ```
   Check that the `nmcli connection up <name>` line names one of the three
   real connections, and that the kill-switch `oifname` pattern matches
   the real interface names. A pattern like `"proton*"` does **not** match
   `qubes-CA-1040` etc., which would mean the kill switch silently fails
   open. Expected fix: `"qubes*"` (or list all three names).

3. **[Agent/qrexec]** Draft the corrected file as
   `~/rc.local.proposed` (same content as the current file with only the
   fixes from step 2). Do not touch `/rw/config/rc.local`. Write the draft
   with a `cat > ~/rc.local.proposed` command, sending the file text on
   stdin (untested; if it misbehaves, draft in the agent qube and
   `qvm-copy` it, then adjust the path in step 5). Produce a unified diff
   against it:
   ```bash
   diff -u /rw/config/rc.local ~/rc.local.proposed
   ```

4. **[Human]** Review the diff. It will run as root at every boot.

5. **[Human/dom0]** Back up, then install the reviewed file:
   ```bash
   qvm-run -u root sys-vpn-id0-proton 'cp -a /rw/config/rc.local /rw/config/rc.local.bak'
   qvm-run -u root sys-vpn-id0-proton 'install -m 0755 -o root -g root /home/user/rc.local.proposed /rw/config/rc.local'
   ```

6. **[Human/dom0]** Disable autoconnect on all three — the default
   included. From here on, connection state changes happen only
   explicitly (`nmcli`, the dispatcher script in Part B, or the boot-time
   line in `rc.local`), never via NM's own autoconnect logic:
   ```bash
   qvm-run -u root sys-vpn-id0-proton 'nmcli connection modify qubes-CA-1040 connection.autoconnect no'
   qvm-run -u root sys-vpn-id0-proton 'nmcli connection modify qubes-CA-1048 connection.autoconnect no'
   qvm-run -u root sys-vpn-id0-proton 'nmcli connection modify qubes-US-MI-19 connection.autoconnect no'
   ```

### Open risk: boot ordering

`rc.local` can run **before NetworkManager is ready** (a known gotcha on
minimal Debian templates), in which case `nmcli connection up ...` in
`rc.local` silently fails. If so, the VPN has so far been coming up at
boot because of **autoconnect**, not because of `rc.local` — and step 6
would remove that. The kill switch keeps this fail-closed (no leak), but
downstream qubes would have no connectivity after boot.

**Test before calling Part A done:** restart `sys-vpn-id0-proton` and
check `nmcli connection show --active`. If no WireGuard connection is
active, `rc.local` is losing the race. Candidate fixes (human decision,
not yet tried): wait for NM in a backgrounded line, e.g.
`( nm-online -q -t 60 && nmcli connection up qubes-CA-1040 ) &`, or bring
the default connection up from a dispatcher script on the uplink's `up`
event. Do **not** re-enable autoconnect as the workaround.

## Part B — Exclusive-activation dispatcher script

This is the proper fix for multi-connection conflicts: a NetworkManager
dispatcher script that fires on the generic `up` event for any interface
(native WireGuard connections don't emit `vpn-up`/`vpn-down`, so the
generic per-interface events are the correct hook) and brings down any
other `qubes-*` connection. This makes the *naive* action — picking a
different connection via the Qubes network widget, `nmcli`, or any other
frontend — correct by construction.

**This must be added to the template (`debian-13-minimal-net`), not the
running `sys-vpn-id0-proton` AppVM** — `/etc` is part of the ephemeral
per-boot overlay in an AppVM and reverts to whatever the template
provides. Editing a template requires dom0; a human runs this part.
Note that `debian-13-minimal-net` also backs `lan-proxy`; the script only
acts on interfaces named `qubes-*`, so it is inert there.

1. **[Human/dom0]** Shut down `sys-vpn-id0-proton`, then open a root
   shell in the template:
   ```bash
   qvm-shutdown sys-vpn-id0-proton
   qvm-run -u root debian-13-minimal-net xterm
   ```

2. **[Human/dom0]**, typed in that root shell — create the dispatcher
   script:
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

3. **[Human/dom0]**, same shell — NM silently refuses to run scripts that
   don't meet these:
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
   The agent's qrexec calls fail while the qube is down; retry after the
   restart.

## Verification

Once `sys-vpn-id0-proton` is back up and reachable over qrexec:

1. **[Agent/qrexec]** Autoconnect and boot state: exactly one WireGuard
   connection is active and it is the intended default:
   ```bash
   nmcli -f NAME,AUTOCONNECT connection show
   nmcli connection show --active
   ```

2. **[Human/dom0]** Exclusivity: with the default active, bring up a
   second one and confirm the first goes down automatically:
   ```bash
   qvm-run -u root sys-vpn-id0-proton 'nmcli connection up qubes-CA-1048'
   ```
   then **[Agent/qrexec]** `nmcli connection show --active` should list only
   `qubes-CA-1048`. If not, **[Human/dom0]**
   `qvm-run -u root sys-vpn-id0-proton 'journalctl -u NetworkManager | grep -i dispatcher'`.
   Also try switching through the Qubes network widget, since that is the
   path the fix exists for.

3. **[Human]** Downstream connectivity: from a downstream AppVM,
   `curl https://ifconfig.me` shows the VPN IP.

4. **[Human/dom0] + [Human]** Kill switch: bring the active tunnel down
   (`qvm-run -u root sys-vpn-id0-proton 'nmcli connection down <name>'`)
   and confirm downstream connectivity fails outright rather than falling
   back to clearnet.

5. **[Agent/qrexec]** No silent reactivation: with no connection active,
   leave the qube idle for several minutes and confirm nothing comes back
   up (`nmcli connection show --active`).

## Report back

- Final `AUTOCONNECT` state of all three connections.
- The diff applied to `/rw/config/rc.local`, and the final kill-switch
  pattern.
- Whether a WireGuard connection is active after a fresh boot (the
  boot-ordering test).
- Confirmation the dispatcher script is in place with `root:root` and
  mode `750`.
- Results of all five verification checks.

## Do not

- Re-enable `connection.autoconnect` on any of the three connections.
- Rely on NM's own autoconnect logic to bring up the default connection.
- Write `/rw/config/rc.local` (or any file run as root at boot) directly
  as the agent.
- Weaken the kill switch (for example `policy drop` → `policy accept`) to
  get past an error; flag it for a human decision.
- Add the dispatcher script to the running AppVM's `/etc`; it is lost on
  reboot. It belongs in the template.

## Status

Design proposed, not yet implemented. The dispatcher approach rests on
NM's generic dispatcher events; it has not yet been run on this system.

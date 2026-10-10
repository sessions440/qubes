# GUI ProxyVM for ProtonVPN (`sys-vpn-id0-proton-gui`) — Design & Setup

## Context

Purpose: recover ProtonVPN's load-aware server selection and one-click
switching UX for **temporary, one-off country routing** from a given
AppVM — something the static-`.conf` approach in `sys-vpn-id0-proton`
can't offer.

Kept **completely separate** from `sys-vpn-id0-proton` rather than merged
into it. Two independent reasons:

- Proton's official Linux app has an open, unresolved bug (v4.14.0, as of
  Feb 2026) where it sometimes fails to recognize its own connection
  state correctly. Running it in the same qube as the manually-imported
  `qubes-*` configs + exclusive-activation dispatcher script would mean
  reasoning about interaction between two different connection-management
  layers. Full separation means two independent NM instances in two
  independent qubes — nothing to reconcile.
- Keeping Proton's account credentials in a qube that does nothing else
  (no browser, no editor, no untrusted content) preserves the isolation
  property that makes a ProxyVM-based kill switch meaningful in the first
  place: a compromised downstream qube has no code-execution path into
  this one. See the discussion in this repo on running ProtonVPN directly
  inside a general-purpose AppVM for the fuller reasoning — the risk
  identified there (credential colocation with untrusted content,
  self-enforced kill switch defeatable by whatever compromised the same
  qube) is exactly what a dedicated, nothing-else-runs-here qube avoids.

Normally kept **shut down**. Started on demand; a target AppVM's `netvm`
is pointed at it temporarily, then reverted.

Chain: `AppVM → sys-vpn-id0-proton-gui → sys-firewall → sys-net`

## Role tags

Same convention as `fix-silent-reconnect.md`:
- **[Human/dom0]** — requires dom0 privilege, root inside a qube via
  `qvm-run -u root`, or a shell opened inside a TemplateVM. No agent path to dom0 or a TemplateVM.
- **[Agent/qrexec]** — runs as `user` from the agent qube through
  `qrexec-client-vm`, in an already-running qube that a dom0 policy line
  allows.
- **[Human]** — requires interactive input (credentials, GUI clicks) that
  should not be delegated to an agent regardless of access level.
- **[Human or Agent/qrexec]** — a step done inside this qube that either can
  run. Root inside this qube is `sudo` (full template, passwordless), shown
  as *(root)*; no dom0 involved.

## Why a dedicated template, not the existing browsing/coding template

**Do not install this into the existing customized `debian-13-xfce`
template used for Brave/Codium AppVMs.** That would recreate exactly the
credential-colocation problem described above. Clone a fresh template
instead.

## Dependency notes

- Full desktop templates already ship `qubes-core-agent-network-manager`
  and start a real desktop/D-Bus session at qube boot — this is why
  `nm-applet` and notifications work normally in `sys-net`/`sys-usb` even
  though you never see a full desktop shell for those qubes. This means
  the "headless" failure mode that blocks Proton's official CLI (no
  D-Bus session bus, no unlocked keyring) does **not** apply here.
- Install Proton's official Linux app per their current published
  instructions at setup time — packaging has changed before and may
  again; verify the package signature/checksum against Proton's
  published values rather than trusting a raw download.
- `wireguard-tools` should be installed explicitly regardless of what
  Proton's installer pulls in. `openssh-server` is needed only for the SSH
  backup path in `agent-qube-access.md`; it is not installed by default.

## Setup

1. **[Human/dom0]** Clone a fresh template — not the loaded
   browsing/coding one:
   ```bash
   qvm-clone debian-13-xfce debian-13-xfce-net
   ```

2. **[Human/dom0]** Open a root shell in the new template and install
   base packages:
   ```bash
   qvm-run -u root debian-13-xfce-net xfce4-terminal
   ```
   Inside that shell:
   ```bash
   apt update
   apt install wireguard-tools
   ```

3. **[Human/dom0]**, same shell — install the ProtonVPN Linux app per
   Proton's current official instructions. Verify the package signature
   before installing. Skip instructions to enable split tunneling.

4. **[Human/dom0]** Shut down the template:
   ```bash
   qvm-shutdown debian-13-xfce-net
   ```

5. **[Human/dom0]** Create the ProxyVM:
   ```bash
   qvm-create sys-vpn-id0-proton-gui \
     --class AppVM \
     --template debian-13-xfce-net \
     --label orange \
     --prop provides_network=True \
     --prop netvm=sys-firewall

   qvm-service sys-vpn-id0-proton-gui network-manager on
   ```
   If instead you use the dom0 "Create New Qube" GUI, you may add Proton VPN to the applications list now, so it's available in the Qubes menu.

6. **[Human]** Start the qube and log into the Proton
   app interactively — this is credential entry (possibly with 2FA) and
   should not be scripted or delegated to an agent.

   If prompted to choose a password for a new keyring, use an empty password and select "Continue". If you encounter trouble (Proton VPN GUI says "unexpected error occurred") then clear Proton cache and config, and reset the local keyrings directory:
   ```bash
   rm -rf ~/.cache/Proton ~/.config/Proton ~/.local/share/keyrings/*
   ```
   Then reboot the qube and try again.
   
   This session lives in
   `sys-vpn-id0-proton-gui`'s own persistent `$HOME`/keyring, not the
   template — logging in during template customization would bake the
   session into the shared image instead, which is the wrong place for
   it (every AppVM cloned from that template would inherit the same
   session).

7. **[Human/dom0, then Agent/qrexec]** Decide whether the agent gets any
   shell in this qube at all. It is on a full template
   (`debian-13-xfce-net`), which has passwordless `sudo`, so a
   `qubes.VMShell` shell as `user` is root-equivalent here, and this qube
   holds Proton's account credentials. If yes, add a `qubes.VMShell` allow
   line for it per `agent-qube-access.md`; root-off is moot on this
   template, so keep the `qubes.VMRootShell` line commented out and treat
   the VMShell line as the real decision. Administering this qube by hand
   is a reasonable alternative.

8. **[Human]** Enable Proton's own kill switch in the app, in **Advanced**
   mode: Menu > Settings > Features, turn on the Kill switch toggle, then
   click **Advanced** (Standard is selected by default).
   - **Standard** only engages when an established connection drops by
     accident. It does nothing while the VPN is deliberately disconnected
     or before it has connected.
   - **Advanced** blocks all traffic outside the VPN interface at all
     times, including after a manual disconnect, and stays active across a
     restart of the qube.
   - Advanced fits this qube: it exists only to carry downstream traffic
     through the tunnel, so the cost (no internet in the qube itself unless
     connected) doesn't matter. If the qube ever needs the network while
     disconnected (e.g. a re-login), switching the kill switch off for that
     is a deliberate human decision; switch it back on afterward.
   - Not compatible with split tunneling, which this setup skips anyway.
   - This is the app-level kill switch, enforced by the app inside this
     qube. Step 9 adds an independent one that does not depend on the app.
   - Verified on this system: the app's kill switch also blocks traffic
     forwarded from downstream qubes, not only this qube's own traffic.
   - Sources: Proton's "Kill switch" and "Advanced kill switch" support
     pages (protonvpn.com/support/what-is-kill-switch,
     protonvpn.com/support/advanced-kill-switch).

   **NetShield (optional; recommend leaving it off).** NetShield is
   Proton's DNS-level ad/tracker/malware blocking, available on paid
   plans. Not needed here: this qube is for occasional one-off routing, DNS
   filtering can break sites and make a routing problem harder to
   diagnose, and downstream qubes can run their own blockers. Turn it on
   only if you want it for a specific session.

9. **Backstop kill switch (nftables, via `/rw/config/rc.local`).** Keep
   this even with Advanced kill switch on, as an independent layer. The
   app's kill switch is enforced by the app inside this qube (which has a
   known state-tracking bug, see Known risks); this is a `forward` chain
   with `policy drop`, loaded at boot before the app starts, that lets
   forwarded traffic leave only through the tunnel. Same fail-closed
   pattern as `sys-vpn-id0-proton`. Scope: forwarded traffic (downstream
   qubes), not this qube's own traffic.

   **Where it runs.** Every command below runs inside
   `sys-vpn-id0-proton-gui`, so either a human (terminal in the qube) or an
   agent (`qubes.VMShell`, if step 7 granted one) can do it. dom0 is needed
   only for the restart in step h. Commands marked *(root)* are shown
   without `sudo`: in a terminal run `sudo -i` first; an agent prefixes each
   with `sudo`. This full template has passwordless `sudo`, so `user` is
   root-equivalent here. `/rw/config/rc.local` runs as root at every boot,
   so the file is drafted without privileges, a human reads it in full, and
   it is installed only after that approval (see `AGENTS.md`).

   a. **[Human]** Connect to any server once through the Proton app.

   b. **[Human or Agent/qrexec]** Find the tunnel interface name (no root
      needed):
      ```bash
      nmcli -f NAME,TYPE,DEVICE connection show --active
      ip -br link
      ```
      The tunnel is the active connection of type `wireguard` (or
      `tun`/`vpn` if you use an OpenVPN-based protocol); its `DEVICE` is
      the name you want. Ignore `eth0`, `lo`, `vif*`, and the
      `pvpn-killswitch` / `pvpn-ipv6leak-protection` entries, which are
      Proton's kill-switch dummies, not the tunnel. Connect to a second
      server (and each protocol you plan to use) and re-run the commands:
      if the name stays the same, use it exactly; if it changes, use the
      narrowest wildcard that matches every tunnel name and not the
      dummies. Don't guess a name: earlier notes mention
      `pvpnrouteintrf0`, which is unconfirmed.

   c. **[Human or Agent/qrexec]** *(root)* Look at the current file and back
      it up:
      ```bash
      cat /rw/config/rc.local
      cp -a /rw/config/rc.local /rw/config/rc.local.bak
      ```
      On a fresh qube it should hold only the stock comments. If it has
      anything else, merge the block below into it instead of replacing it.

   d. **[Human or Agent/qrexec]** Draft the new file in the home directory
      (no root). Replace `<tunnel-iface>` with the name from step b first
      (if left unreplaced, the rule matches nothing and all forwarding is
      dropped: fail-closed, but useless):
      ```bash
      cat > ~/rc.local.proposed <<'RC_FILE'
      #!/bin/bash
      # Backstop kill switch, independent of the Proton app.
      # Forwarded traffic may leave only through the VPN tunnel.
      TUNNEL_IFACE="<tunnel-iface>"

      nft -f - <<NFT
      table inet qubes-vpn-gui
      delete table inet qubes-vpn-gui
      table inet qubes-vpn-gui {
        chain forward {
          type filter hook forward priority 0; policy drop;
          oifname "${TUNNEL_IFACE}" accept
          ct state established,related oifname "vif*" accept
        }
      }
      NFT
      RC_FILE

      diff -u /rw/config/rc.local ~/rc.local.proposed
      ```
      The `table` / `delete table` / `table { ... }` sequence makes the
      file safe to re-run. The second rule accepts only reply traffic
      heading back to downstream qubes (`vif*`). Don't widen it to a bare
      `ct state established,related accept`: that would also pass packets
      of an already-established flow out of the uplink if the tunnel
      dropped. This `vif*` form is a tightening over the rule in
      `vpn-proxyvm.md`; step g confirms the interface naming it assumes.

   e. **[Human]** Read the diff in full and approve it. An agent does not
      continue until the human has approved.

   f. **[Human or Agent/qrexec]** *(root)* Install and load it:
      ```bash
      install -m 0755 -o root -g root ~/rc.local.proposed /rw/config/rc.local
      /rw/config/rc.local
      nft list table inet qubes-vpn-gui
      ```
      The listing should show `policy drop` and your interface name.

   g. **[Human or Agent/qrexec]** With a downstream qube running (its
      `vif` exists only while it runs), confirm the downstream interface
      naming that the `vif*` pattern assumes:
      ```bash
      ip -br link | grep vif
      ```

   h. **[Human/dom0]** Restart the qube to prove the file survives a
      reboot (an agent's calls fail while it is down):
      ```bash
      qvm-shutdown --wait sys-vpn-id0-proton-gui
      qvm-start sys-vpn-id0-proton-gui
      ```
      Then **[Human or Agent/qrexec]** *(root)*:
      ```bash
      nft list table inet qubes-vpn-gui
      ```
      The table must be present before you connect in the app. Then do the
      Verification checks below.

10. **[Human/dom0]** Return the qube to its normal at-rest state:
    ```bash
    qvm-shutdown sys-vpn-id0-proton-gui
    ```

## Usage (on demand)

1. **[Human/dom0]** `qvm-start sys-vpn-id0-proton-gui`
2. **[Human]** Open its window and pick a country/server via the Proton
   app's own UI.
3. **[Human/dom0]** Point the target AppVM at it:
   ```bash
   qvm-prefs <target-appvm> netvm sys-vpn-id0-proton-gui
   ```
4. Use it.
5. **[Human/dom0]** Revert the target AppVM's `netvm` to its normal
   value, then shut this qube back down:
   ```bash
   qvm-prefs <target-appvm> netvm <original-netvm>
   qvm-shutdown sys-vpn-id0-proton-gui
   ```

Changing `netvm` on a running qube is supported but has occasional
reports of getting stuck — test on a low-stakes AppVM first; a qube
restart is the fallback if it wedges.

## Verification

- From the target AppVM: `curl https://ifconfig.me` shows the IP of the
  country selected in the Proton app.
- Kill switch check: disconnect from within the Proton app (or bring the
  interface down via `nmcli`) and confirm the target AppVM's connectivity
  fails outright rather than falling back to clearnet. Repeat after
  rebooting the qube and before connecting, to cover the boot window.
- Backstop check: `nft list table inet qubes-vpn-gui` (root) in this qube shows
  `policy drop`, and downstream traffic works while connected (so the
  `oifname` rule matches the real tunnel name). Do this for every server
  and protocol you use.
- Optional, human decision: to show the backstop works without the app's
  kill switch, switch the app's kill switch off with the tunnel
  disconnected, test from the downstream AppVM that connectivity still
  fails, then switch it back on. This briefly weakens a control, so only
  do it deliberately.
- Confirm this qube's own entry in the Qubes network widget shows a
  single managed connection, not a flood of pre-imported server entries —
  this is expected behavior for the modern official app (it reuses one
  connection object per selection) rather than the older, unofficial
  community import scripts that pre-populate hundreds of static profiles.

## Known risks / open questions

- Proton's Linux GUI app has an open bug (as of v4.14.0) where it
  sometimes fails to recognize its own successful connection and spins
  indefinitely; a restart of the app may be needed to resync. Not fatal,
  but expect occasional flakiness distinct from anything in
  `sys-vpn-id0-proton`.
- The tunnel interface name in step 9 must be taken from this system,
  not from notes. Until it is confirmed, the backstop either blocks
  everything (fail-closed) or matches too broadly.
- Proton's kill switch was verified to block forwarded downstream traffic
  (see step 8). The step 9 backstop is kept anyway as an independent
  layer that doesn't depend on the app's behavior or its known bugs.
- Not verified: the `vif*` reply rule in step 9.

## Status

Design proposed, not yet implemented. Update this doc once fielded.

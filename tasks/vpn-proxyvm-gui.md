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
- **[Human or Agent/qrexec]** — a step done inside this qube that either
  can run. `sudo` is passwordless here (full template), so `user` is
  root-equivalent; no dom0 involved.

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

8. **[Human]** Enable Proton's kill switch in **Advanced** mode: Menu >
   Settings > Features > Kill switch on, then click **Advanced**. Leave
   NetShield off. (Why: [Kill switch notes](#kill-switch-notes).)

9. Add a backstop kill switch: forwarded traffic may leave only through
   the VPN tunnel, enforced by nftables from boot, independent of the
   app. (Why, and how it differs from `vpn-proxyvm.md`:
   [Kill switch notes](#kill-switch-notes).)

   a. **[Human]** Connect to any server in the Proton app.

   b. **[Human or Agent/qrexec]** Find the tunnel interface name: the
      `DEVICE` of the `wireguard` connection.
      ```bash
      nmcli -f NAME,TYPE,DEVICE connection show --active
      ```

   c. **[Human or Agent/qrexec]** Append the rules to `rc.local`, with the
      name from step b in place of `<tunnel-iface>`. An agent shows the
      human the full command and gets approval first (see `AGENTS.md`).
      ```bash
      sudo tee -a /rw/config/rc.local <<'EOF'

      # Backstop kill switch: forward only via the VPN tunnel
      nft add table inet qubes-vpn-gui
      nft add chain inet qubes-vpn-gui forward { type filter hook forward priority 0 \; policy drop \; }
      nft add rule inet qubes-vpn-gui forward oifname "<tunnel-iface>" accept
      nft add rule inet qubes-vpn-gui forward ct state established,related oifname "vif*" accept
      EOF
      sudo chmod +x /rw/config/rc.local
      ```

   d. **[Human/dom0]** Restart the qube so the rules load from boot:
      ```bash
      qvm-shutdown --wait sys-vpn-id0-proton-gui
      qvm-start sys-vpn-id0-proton-gui
      ```

   e. **[Human or Agent/qrexec]** Confirm the table is loaded, with
      `policy drop` and your interface name:
      ```bash
      sudo nft list table inet qubes-vpn-gui
      ```
      Then run the [Verification](#verification) checks.

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
- Kill switch: disconnect in the Proton app and confirm the target AppVM
  loses connectivity rather than falling back to clearnet. Repeat after
  rebooting the qube, before connecting.
- Backstop: step 9e shows `policy drop`, and downstream traffic works while
  connected, on every server and protocol you use. To test it alone, turn
  the app's kill switch off with the tunnel disconnected, confirm the
  downstream AppVM still has no connectivity, then turn it back on (this
  briefly weakens a control: a human decision).
- Confirm this qube's own entry in the Qubes network widget shows a
  single managed connection, not a flood of pre-imported server entries —
  this is expected behavior for the modern official app (it reuses one
  connection object per selection) rather than the older, unofficial
  community import scripts that pre-populate hundreds of static profiles.

## Kill switch notes

**Standard vs Advanced (step 8).** Standard engages only when an
established connection drops by accident; it does nothing during a
deliberate disconnect or before the first connect. Advanced blocks all
traffic outside the VPN interface at all times and persists across
restarts. This qube exists only to carry downstream traffic through the
tunnel, so Advanced costs nothing. If the qube itself needs the network
while disconnected (e.g. a re-login), turn the kill switch off
deliberately, then back on. Advanced is incompatible with split tunneling,
which this setup skips. Verified on this system: the app's kill switch also
blocks forwarded downstream traffic. Sources:
protonvpn.com/support/what-is-kill-switch,
protonvpn.com/support/advanced-kill-switch.

**NetShield (step 8; leave off).** DNS-level ad/tracker/malware blocking,
paid plans only. Not needed for occasional one-off routing: DNS filtering
can break sites and obscure routing problems, and downstream qubes can run
their own blockers. Enable it only for a specific session.

**Why a backstop (step 9).** The app's kill switch is enforced by the app
inside this qube, which has a known state-tracking bug (see Known risks).
The nftables rules load at boot before the app starts and don't depend on
it. Same fail-closed pattern as step 6 of `vpn-proxyvm.md`, with three
differences:
- The app, not you, creates and names the tunnel interface, so the name is
  read off the system (step 9b) instead of following a `qubes*` naming
  convention. If it changes between servers or protocols, use the
  narrowest wildcard that matches every tunnel name. Proton's kill-switch
  `pvpn-*` connections are not the tunnel, and interface names reported
  elsewhere (e.g. `pvpnrouteintrf0`) are unconfirmed.
- There is no `nmcli connection up` line: the app manages the connection.
- The reply rule accepts established traffic only toward downstream qubes
  (`vif*`). A bare `ct state established,related accept`, as in
  `vpn-proxyvm.md`, could pass an already-established flow out the uplink
  if the tunnel dropped.

If `<tunnel-iface>` is left unreplaced, nothing matches and all forwarding
is dropped: fail-closed, but useless.

## Known risks / open questions

- Proton's Linux GUI app has an open bug (as of v4.14.0) where it
  sometimes fails to recognize its own successful connection and spins
  indefinitely; a restart of the app may be needed to resync. Not fatal,
  but expect occasional flakiness distinct from anything in
  `sys-vpn-id0-proton`.
- The tunnel interface name in step 9 must come from this system, not
  from notes.
- Not verified: the `vif*` reply rule in step 9. If the name is wrong,
  connected downstream qubes lose connectivity (fail-closed), which the
  Verification checks will show.

## Status

Design proposed, not yet implemented. Update this doc once fielded.

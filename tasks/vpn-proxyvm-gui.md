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
- **[Human/dom0]** — requires dom0 privilege, or a shell opened inside a
  TemplateVM. No SSH-based path for an agent to run these.
- **[Agent/SSH]** — runs inside an already-running, SSH-reachable qube.
- **[Human]** — requires interactive input (credentials, GUI clicks) that
  should not be delegated to an agent regardless of access level.

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
- `wireguard-tools` and `openssh-server` (for agent access) should be
  installed explicitly regardless of what Proton's installer pulls in.

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
   apt install openssh-server wireguard-tools
   systemctl enable ssh
   ```

3. **[Human/dom0]**, same shell — install the ProtonVPN Linux app per
   Proton's current official instructions. Verify the package signature
   before installing.

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

6. **[Human]** Start the qube and open its window. Log into the Proton
   app interactively — this is credential entry (possibly with 2FA) and
   should not be scripted or delegated to an agent. This session lives in
   `sys-vpn-id0-proton-gui`'s own persistent `$HOME`/keyring, not the
   template — logging in during template customization would bake the
   session into the shared image instead, which is the wrong place for
   it (every AppVM cloned from that template would inherit the same
   session).

7. **[Human, then Agent/SSH once reachable]** Add your agent's public key
   to this qube's own `~/.ssh/authorized_keys` (parallels the
   `sys-vpn-id0-proton` setup) for future agent-driven administration.

8. **[Human]** Enable Proton's own kill switch in its app settings, if
   available in the installed version.

9. **[Agent/SSH]** Add a qube-level nftables backstop kill switch via
   `/rw/config/rc.local`, following the same fail-closed pattern as
   `sys-vpn-id0-proton`. **The interface name needs empirical
   confirmation on this system** — check with `ip link` after connecting
   through the app once; Proton's managed interface is reported
   elsewhere as something like `pvpnrouteintrf0`, but confirm the real
   name before finalizing the `oifname` pattern (a `"pvpn*"` wildcard is
   a reasonable starting point, verified against the actual name).

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
  fails outright rather than falling back to clearnet.
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
- The kill switch interface-name pattern in step 9 is provisional until
  confirmed against the real interface name on this system.

## Status

Design proposed, not yet implemented. Update this doc once fielded.

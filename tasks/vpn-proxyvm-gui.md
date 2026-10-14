# GUI ProxyVM for ProtonVPN (`sys-vpn-id0-proton-gui`) — Always-On Default Design & Setup

## Context

Purpose: make `sys-vpn-id0-proton-gui`, a dedicated qube running Proton's
official Linux GUI app, the **default, always-on VPN NetVM** for the AppVMs
that currently use `sys-vpn-id0-proton`. This recovers ProtonVPN's load-aware
server selection and interactive switching, which the static-`.conf`
approach in `sys-vpn-id0-proton` cannot offer, and avoids the maintenance
burden of that approach (see `fix-silent-reconnect.md`, now shelved).

History: this doc originally described an **on-demand** qube, normally shut
down and pointed at temporarily. Decision of 2026-10-13: it becomes the
default instead, started automatically by downstream qubes. `sys-vpn-id0-proton`
is demoted to a cold fallback (see `vpn-proxyvm.md`).

Chain: `AppVM → sys-vpn-id0-proton-gui → sys-firewall → sys-net`

Kept **completely separate** from `sys-vpn-id0-proton` rather than merged
into it. Two independent reasons:

- Proton's official Linux app has an open, unresolved bug (v4.14.0, as of
  Feb 2026) where it sometimes fails to recognize its own connection
  state correctly. Running it in the same qube as the manually-imported
  `qubes-*` configs would mean reasoning about two connection-management
  layers. Two qubes means two independent NM instances; nothing to reconcile.
- Keeping Proton's session in a qube that does nothing else (no browser,
  no editor, no untrusted content) preserves the isolation property that
  makes a ProxyVM-based kill switch meaningful: a compromised downstream
  qube has no code-execution path into this one.

## Role tags

Same convention as `AGENTS.md`:
- **[Human/dom0]** — requires dom0 privilege, root inside a qube via
  `qvm-run -u root`, or a shell opened inside a TemplateVM. No agent path.
- **[Agent/qrexec]** — runs as `user` from the agent qube through
  `qrexec-client-vm`, in an already-running qube that a dom0 policy line
  allows. **No such policy line exists for this qube by design** (step 8).
- **[Human]** — interactive input (credentials, GUI clicks) that should
  not be delegated regardless of access level.

## Why the GUI app (decision record, 2026-10-13)

- **Versus static WireGuard configs** (`vpn-proxyvm.md`): the original
  rejection of Proton's tooling rested on the CLI needing a D-Bus session
  and keyring, which a minimal loginless qube lacks. That argument does not
  apply to a full desktop template, which starts a desktop session at boot.
  The claim that the Qubes community converges on static configs is
  overstated: the Qubes forum has a long-running guide for running the Proton
  app in a dedicated qube, and a separate WireGuard-without-the-app guide.
  Static configs also go stale: Proton says a retired server means new
  config files must be downloaded.
- **Versus the Proton CLI:** finely balanced, either is defensible. The
  GUI wins on the official auto-connect setting and the server list with
  load. The CLI wins on scriptable switching. See "CLI qube sketch" below.
- **Provider independence** is the real advantage of static configs, and is
  retained by keeping `sys-vpn-id0-proton` as a cold fallback.

## Architecture options

Stated during the design discussion; **option 1 is the plan**. Options 2–4
are alternatives for the Proton app itself, and option 5 is orthogonal and
combines with any of them.

1. **GUI only, one qube.** The plan in this doc. Simplest.
2. **CLI only, one qube.** Scriptable, but loses the official auto-connect
   and the load-aware list; the autostart launcher has to be self-built.
3. **Two qubes on the same template.** `sys-vpn-id0-proton-gui` stays the
   interactive default and a second qube is used for CLI experiments. The
   GUI/CLI exclusivity applies per qube, so there is no conflict. The cost
   is two qubes holding Proton sessions. Both qubes share one template, so
   there is no template drift. This is the planned route for experimenting
   with the CLI; sketch below.
4. **One CLI qube per exit** (`...-ca`, `...-us`, and so on), each pinned with
   `protonvpn connect --country XX`. Switching exits means changing an
   AppVM's `netvm`, so other downstream qubes are not dropped. Costs: RAM
   per qube, and the plan's simultaneous-connection limit (not checked).
5. **Keep `sys-vpn-id0-proton` as a cold fallback**, with a single config
   (`vpn-proxyvm.md`).

## Why a dedicated template, not the existing browsing/coding template

**Do not install this into the existing customized `debian-13-xfce`
template used for Brave/Codium AppVMs.** That would put Proton's session
next to untrusted content. Clone a fresh template instead.

## Dependency notes

- Full desktop templates already ship `qubes-core-agent-network-manager`
  and start a real desktop/D-Bus session at qube boot. This is why
  `nm-applet` works in `sys-net`/`sys-usb`. The "headless" failure mode
  that blocks Proton's CLI on minimal templates does **not** apply here.
- Proton's Debian package `proton-vpn-gnome-desktop` installs **both** the
  GUI app and the CLI. They cannot run at the same time in one qube.
- Proton officially supports the latest stable Debian and Ubuntu running
  GNOME. Other desktops (this template is XFCE) are not officially
  supported; community reports say the app works there.
- Both the GUI and the CLI require NetworkManager and a keyring
  (gnome-keyring); the CLI is documented as not working headless.
- Install per Proton's current official instructions at setup time
  (as of 2026-10-13: add Proton's repo via their release package, verify its
  checksum against the value on their page, then
  `apt install proton-vpn-gnome-desktop`). Do not trust a raw download.
- `wireguard-tools` should be installed explicitly regardless of what
  Proton's installer pulls in. `openssh-server` is needed only for the SSH
  backup path in `agent-qube-access.md`; it is not installed by default.

## Setup

If the qube already exists, check Part A against the live system and
continue at Part B.

### Part A — Create the qube

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
   Proton's current official instructions (see Dependency notes). Verify
   the checksum before installing. Skip instructions to enable split
   tunneling.

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
   Community guides also set initial memory to about 400 MB, uncheck
   "Include in memory balancing", and add `qubes-firewall` under Services.
   Verify the services against the live qube. If you use the "Create New
   Qube" GUI, you can add Proton VPN to the applications list now.

6. **[Human]** Start the qube and sign in to the Proton app interactively.
   This is credential entry (possibly with 2FA) and must not be delegated
   to an agent. The session lives in this qube's own `$HOME`/keyring, not
   the template: signing in during template customization would bake the
   session into every qube cloned from that template.

   If prompted for a keyring password, the older guidance was an empty
   password and "Continue". **Caution:** one GitHub issue (age unknown,
   possibly fixed) reports that with an empty-password keyring the app
   deleted its entry at launch and recreated the keyring every reboot. This
   matters for unattended boot (Part B), so test it (Part B, step 3).
   If the app says "unexpected error occurred", clear its state and retry:
   ```bash
   rm -rf ~/.cache/Proton ~/.config/Proton ~/.local/share/keyrings/*
   ```
   Then reboot the qube.

7. **[Human/dom0, then Agent/qrexec]** Decide whether the agent gets any
   shell in this qube. **Decision: no.** It is on a full template with
   passwordless `sudo`, so a `qubes.VMShell` shell as `user` is
   root-equivalent here, and this qube holds a Proton session (see
   "Session credentials"). Do not add a `qubes.VMShell` line for it and
   keep the `qubes.VMRootShell` line commented out. Administer it by hand.

### Part B — Make it the always-on default

1. **[Human]** Autostart the app at qube boot. Autostart is plain XDG
   autostart, not a Proton feature; community Qubes guides symlink the
   app's `.desktop` file. The filename has changed before
   (`protonvpn-app.desktop` → `proton.vpn.app.gtk.desktop`, reported
   Sept 2025), so check the real name first. In the qube:
   ```bash
   ls /usr/share/applications | grep -i proton
   mkdir -p ~/.config/autostart
   ln -s /usr/share/applications/<real-name>.desktop ~/.config/autostart/
   ```
   This lives in the AppVM's private volume, so it needs no template change.

2. **[Human]** In the app: Settings → General → **Auto connect**, and
   optionally **Start app minimized**. Auto-connect is an official feature;
   it is documented as connecting when the app starts, not on every
   re-link after a drop.

3. **[Human]** **Cold-boot test (main risk).** Shut the qube down, start it,
   and confirm the tunnel comes up with no clicks. Repeat a few times. A
   forum report says the app opens but never connects; the cause was not
   found and the keyring is only a suspect. If it fails, check the keyring
   (Part A step 6 caution) before anything else.

4. **[Agent/qrexec drafts, Human/dom0 installs]** Kill-switch backstop.
   Reports say the app's own kill switch has not worked reliably, and one
   guide warns it is not guaranteed if the app crashes. Add a qube-level
   rule that never forwards downstream traffic out the uplink, so only the
   tunnel can carry it. The community pattern (verify the exact syntax on
   this system) in `/rw/config/qubes-firewall-user-script`:
   ```bash
   #!/bin/bash
   # Fail-closed: never forward downstream traffic out the uplink.
   nft add rule qubes custom-forward oifname eth0 counter drop
   nft add rule ip6 qubes custom-forward oifname eth0 counter drop
   ```
   This avoids depending on the tunnel interface's name. Confirm the uplink
   is `eth0` with `ip -br link`, and make the file executable. The file
   runs as root at boot, so a human reviews and installs it (`AGENTS.md`).
   Open issue: `agent-qube-access.md` notes reports of this hook being
   ignored, so verify it applies after a reboot
   (`nft list chain ip qubes custom-forward`). The alternative is an
   `rc.local` table as in `vpn-proxyvm.md`; pick one mechanism, not both.
   Also keep Proton's own kill switch enabled if the app offers it, but
   test with and without it for interference with forwarded traffic
   (untested).

5. **[Human]** DNS check after connect. From a downstream AppVM, run a DNS
   leak test and confirm resolution goes through the tunnel.

6. **[Human/dom0]** Cutover, one AppVM at a time, lowest-stakes first.
   Shut the AppVM down first, since changing `netvm` on a running qube
   occasionally wedges:
   ```bash
   qvm-ls -O NAME,NETVM | awk '$2=="sys-vpn-id0-proton"{print $1}'
   qvm-shutdown <appvm>
   qvm-prefs <appvm> netvm sys-vpn-id0-proton-gui
   ```
   Rollback is the same command pointing back at `sys-vpn-id0-proton`. Do
   not change `qubes-prefs default_netvm` unless that is how these qubes
   are configured.

7. **[Human/dom0]** Stop treating the qube as normally shut down. Starting a
   downstream qube starts it automatically. Downstream qubes have **no
   connectivity until the tunnel is up**; avoid network-dependent
   autostarts in them.

### Switching, reconnect and the tray

- **Server switching:** open the qube's window and pick a server. Downstream
  qubes lose connectivity briefly. This is expected behavior.
- **Reconnect (decision: no watchdog).** With the backstop verified, a lost
  tunnel fails closed and downstream qubes see no connectivity. The cost is
  a manual reconnect. The likely triggers on a laptop are suspend/resume and
  Wi-Fi changes; whether the app reconnects by itself then is not known.
  Test suspend/resume before treating this as rare.
- **Window and tray.** The app has "Start app minimized" and an optional
  tray icon (show/hide, connect/disconnect, quit). On GNOME the tray icon
  needs an AppIndicator extension; whether it appears in Qubes is untested,
  and a StatusNotifier-to-XEmbed bridge such as `snixembed` may be needed.
  Hiding the window does not free memory; the process keeps running.

## Session credentials

- The username and password are needed for sign-in. Proton's Python client
  library authenticates with them and then works with an access token and a
  refresh token, and a stored session can be reloaded and refreshed without
  the password. The Linux app stores its credentials through a system-keyring
  library. **What the app writes to disk was not confirmed.**
- To check, a **[Human]** can search the keyring entries and
  `~/.config/Proton` for the password after signing in, without pasting any
  secret into an agent's context.
- Even if only tokens are stored, they are bearer credentials, the keyring
  is unlocked inside the running session, and `user` is root-equivalent on
  this template. Token-only storage limits damage (the Proton password,
  which also covers other Proton services, is not on disk, and the session
  should be revocable from the Proton account; verify the revoke option in
  the UI) but does not make agent shell access safe. Step 8 stands.

## CLI qube sketch (option 3; for later experiments)

Not a complete procedure. Facts as of 2026-10-13:

- Install: `proton-vpn-gnome-desktop` already includes the CLI.
- Commands: `protonvpn signin [username]`, `protonvpn connect` (fastest;
  or `--country`, `--city`, a server name), `protonvpn disconnect`,
  `protonvpn signout`, `protonvpn config set <option> <value>`. Settings
  reset to defaults on sign-out.
- Features per the README: kill switch, NetShield, port forwarding, custom
  DNS, IPv6, VPN Accelerator, NAT type; WireGuard only; no split tunneling.
- No documented auto-connect or start-at-boot setting. Proton's roadmap says
  auto-launch is coming for the regular Linux app. A startup script is a
  well-worn path, and here it belongs in desktop-session autostart so it has
  D-Bus and the keyring. A root systemd unit probably would not see the
  user's keyring.

Sketch:

1. **[Human/dom0]** Create a **second qube** on the same template
   (`debian-13-xfce-net`) with the same properties as Part A step 5. Prefer
   a fresh qube and a separate sign-in. Cloning `sys-vpn-id0-proton-gui`
   (a known community pattern) also copies its session; if you clone, remove
   the GUI's autostart symlink and its Auto connect setting, since only one
   of GUI/CLI can run per qube.
2. **[Human]** `protonvpn signin <username>`, then `protonvpn connect --country XX`.
3. **[Human]** Autostart (untested): an entry in `~/.config/autostart` whose
   `Exec` is `protonvpn connect --country XX`.
4. **[Agent/qrexec drafts, Human/dom0 installs]** The same kill-switch
   backstop as Part B step 4.
5. **[Human/dom0]** Point a test AppVM's `netvm` at it.

Open questions to settle first: does `protonvpn` work when launched through
`qvm-run` (does it get the session's D-Bus environment)? Does a clone share
the GUI's login session? If both work, a dom0 one-liner such as
`qvm-run <qube> 'protonvpn connect --country DE'` switches exits without
opening a window.

## Considered and not adopted

- **DispVM for the GUI qube.** A disposable is stateless and loses changes
  on shutdown; persistent changes belong in the disposable template. For
  this app the login session is the state, so each disposable would need a
  copy of the session baked into the template. Disposables that refresh
  tokens would leave the template's copy stale (not tested whether Proton
  rotates refresh tokens). Clones of an AppVM already share one template, so
  a DispVM would not reduce template drift.
- **One DispVM per country with a static WireGuard config each.** Stateless
  suits this, and one connection per qube avoids the problems in
  `fix-silent-reconnect.md`. Costs: stale configs, key in the template. Not
  pursued for now. The one-ProxyVM-per-exit idea is provider-agnostic;
  Mullvad's CLI logs in with an account number and has auto-connect and a
  lockdown mode, but its account number is a long-lived secret on disk and
  its Qubes specifics were not checked.
- **Proton CLI as the default qube.** See option 2 above.
- **A status watchdog.** Declined: fail-closed plus manual reconnect.

## Verification

- **[Human]** Cold boot: the tunnel comes up with no clicks, repeated.
- From a target AppVM: `curl https://ifconfig.me` shows the IP of the
  server the app connected to.
- **[Human]** Kill switch: kill the app process (not just disconnect) and
  confirm the target AppVM's connectivity fails outright rather than
  falling back to clearnet. Repeat after a reboot to confirm the backstop
  is applied.
- **[Human]** DNS leak test from the target AppVM.
- **[Human]** Suspend/resume and a Wi-Fi change: record what happens.
- **[Human]** The qube's own entry in the Qubes network widget shows a single
  managed connection, not hundreds of pre-imported profiles (expected for
  the official app).
- **[Human]** `qvm-ls -O NAME,NETVM` shows the intended AppVMs on this qube.

## Known risks / open questions

- The app's connection-state bug (v4.14.0, Feb 2026); check the installed
  version's current behavior.
- Keyring behavior with an empty password, and cold-boot connect reliability.
- Whether the tray icon works in Qubes.
- Whether the app reconnects after suspend/resume or a network change.
- The backstop syntax and hook behavior on this system, and any interaction
  with Proton's own kill switch.
- Recovery if Proton's kill switch is left behind (for example after
  uninstalling the app): delete the leftover NetworkManager connections
  whose names start with `pvpn-` (`pvpn-killswitch`,
  `pvpn-ipv6leak-protection`, possibly `pvpn-routed-killswitch`) using
  `nmcli connection show --active` and `nmcli connection delete <name>`.

## Forward references

- `vpn-proxyvm.md` — the cold-fallback ProxyVM with static WireGuard.
- `fix-silent-reconnect.md` — shelved.
- `agent-qube-access.md` — why this qube has no VMShell line.
- `AGENTS.md` — dom0 boundary, no secrets, don't weaken security controls.
- Proton docs consulted 2026-10-13: Linux GUI usage, Linux CLI usage,
  Debian/Ubuntu install; Qubes forum guides for the Proton app; Qubes docs
  "How to use disposables".

## Status

Design proposed, not yet implemented. Nothing here has been verified on the
real system as the default NetVM. Unverified: all of Part B, the tray in
Qubes, the keyring behavior, what the app stores on disk, the backstop
syntax, and every item under "CLI qube sketch".

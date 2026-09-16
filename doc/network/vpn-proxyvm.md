# VPN ProxyVM (`sys-vpn-id0-proton`) — Design & Configuration

## Context

Goal: a dedicated ProxyVM that routes downstream AppVMs through a ProtonVPN
WireGuard tunnel, built on a `debian-13-minimal` clone rather than a full
`debian-13` template — consistent with the minimal-footprint discipline
applied elsewhere in this environment (see `pass-install.md`,
`lan-restricted-proxyvm.md`).

Chain: `AppVM → sys-vpn-id0-proton → sys-firewall → sys-net`

Rejected alternative: Proton's official Linux CLI. It depends on
`gnome-keyring` / a D-Bus session bus, which never exists in an
auto-started, loginless ProxyVM — it's effectively a headless environment
from the CLI's point of view. It's also the wrong tool anyway: it's built
for interactive server-switching on a desktop, and a ProxyVM is meant to
pin to one (or a small rotating set of) static configs. Static WireGuard
`.conf` files via NetworkManager avoid the dependency entirely and are the
approach the Qubes community converges on for this.

## Dependency audit — why `debian-13-minimal` needs three packages, not just NetworkManager

A minimal template lacks the Qubes network-qube integration layer by
default. Installing `network-manager` alone is not sufficient; the
Qubes-specific agent packages are what wire NM into the qube's role as a
`provides_network=True` VM.

| Package | Why needed |
|---|---|
| `qubes-core-agent-networking` | Qubes networking integration layer for a minimal template; without it the qube isn't a proper Qubes network node |
| `qubes-core-agent-network-manager` | Makes NetworkManager actually function correctly under Qubes (pulls in `network-manager` as a dependency) |
| `wireguard-tools` | Required for NM to import/handle WireGuard `.conf` files (`nmcli connection import type wireguard`) |

**Known gotcha:** `qubes-core-agent-network-manager` has been reported
unavailable ("package not found") on some fully-updated
`debian-13-minimal` clones. If hit, verify
`/etc/apt/sources.list.d/qubes-r4.list` is present and enabled before
assuming the package doesn't exist.

## Step 1 — Install the base template (if not already present)

```bash
sudo qubes-dom0-update qubes-template-debian-13-minimal
```

## Step 2 — Clone and provision the template

Clone rather than modify the shared minimal template directly:

```bash
qvm-clone debian-13-minimal debian-13-minimal-net
qvm-run -u root debian-13-minimal-net xterm
```

Inside the template:

```bash
apt update
apt install qubes-core-agent-networking qubes-core-agent-network-manager wireguard-tools
```

Shut down before proceeding:

```bash
qvm-shutdown debian-13-minimal-net
```

## Step 3 — Create the ProxyVM

```bash
qvm-create sys-vpn-id0-proton \
  --class AppVM \
  --template debian-13-minimal-net \
  --label orange \
  --prop provides_network=True \
  --prop netvm=sys-firewall

qvm-service sys-vpn-id0-proton network-manager on
```

`netvm=sys-firewall` is required — `provides_network=True` only makes
`sys-vpn-id0-proton` an upstream for *other* qubes; it still needs its own upstream to
reach the internet. Omitting this leaves `sys-vpn-id0-proton` with no network at all.
Standard topology keeps `sys-firewall` between `sys-vpn-id0-proton` and `sys-net`
rather than pointing directly at `sys-net`.

## Step 4 — Generate WireGuard configs (ProtonVPN)

Via account.proton.me → VPN → Downloads → WireGuard configuration →
platform Linux. Generate one `.conf` per server/region desired for
rudimentary server switching (a ProxyVM otherwise pins to whatever server
was baked into the config at generation time — there is no live
server-switching without multiple configs).

**Naming constraint:** keep filenames ≤15 characters, no spaces
(`qubes-CA-1040.conf`, `qubes-CA-1048.conf`, `qubes-US-MI-19.conf`). The filename becomes the WireGuard
interface name; longer names are silently truncated by the kernel.

## Step 5 — Import configs into `sys-vpn-id0-proton`

From the qube holding the downloaded configs:

```bash
qvm-copy-to-vm sys-vpn-id0-proton qubes-CA-1040.conf qubes-CA-1048.conf qubes-US-MI-19.conf
```

In `sys-vpn-id0-proton`:

```bash
mv ~/QubesIncoming/<source-qube>/qubes-*.conf ~/
sudo nmcli connection import type wireguard file ~/qubes-CA-1040.conf
sudo nmcli connection import type wireguard file ~/qubes-CA-1048.conf
sudo nmcli connection import type wireguard file ~/qubes-US-MI-19.conf

sudo nmcli connection modify qubes-CA-1040 connection.autoconnect yes
sudo nmcli connection modify qubes-CA-1048 connection.autoconnect no
sudo nmcli connection modify qubes-US-MI-19 connection.autoconnect no
```

## Step 6 — Kill switch (nftables, via `rc.local`)

`/etc` is ephemeral in AppVMs, so persistence goes through
`/rw/config/rc.local` — same pattern used for the LAN-restricted ProxyVM
and NM dispatcher scripts elsewhere in this environment.

```bash
sudo nano /rw/config/rc.local
```

```bash
#!/bin/bash

# Kill switch: drop all forwarded traffic not going through a VPN tunnel
nft add table inet qubes-vpn
nft add chain inet qubes-vpn forward { type filter hook forward priority 0 \; policy drop \; }
nft add rule inet qubes-vpn forward oifname "qubes*" accept
nft add rule inet qubes-vpn forward ct state established,related accept

# Bring up the default VPN connection
nmcli connection up qubes-CA-1040
```

```bash
sudo chmod +x /rw/config/rc.local
```

The `oifname "qubes*"` wildcard is deliberate — it must match *every*
imported connection's interface name, not just the default, or switching
to a non-default server trips the kill switch. This mirrors the
`oifname "qubes*"` wildcard already used in the WireGuard ProxyVM's
nftables kill switch elsewhere in this setup.

**Fail-closed by design:** if the tunnel drops or fails to come up, the
`policy drop` default means downstream AppVMs lose connectivity entirely
rather than falling back to clearnet.

## Step 7 — Attach downstream AppVMs

```bash
qvm-prefs <appvm-name> netvm sys-vpn-id0-proton
```

## Step 8 — Verification

```bash
# From a downstream AppVM — should show a ProtonVPN IP
curl https://ifconfig.me

# Confirm kill switch — bring the tunnel down in sys-vpn-id0-proton, then retry curl
# from the downstream AppVM. Expect a timeout, not a fallback to the real IP.
sudo nmcli connection down qubes-CA-1040
```

Restore:

```bash
sudo nmcli connection up qubes-CA-1040
```

## Server switching

```bash
sudo nmcli connection down qubes-CA-1040
sudo nmcli connection up qubes-US-MI-19
```

Downstream AppVMs briefly lose connectivity during switchover — expected
fail-closed behavior, not a bug.

## Rejected/considered alternatives

- **Proton official CLI** — depends on `gnome-keyring`/D-Bus session,
  unavailable in a headless, auto-started ProxyVM. Also solves a problem
  (interactive load-based server switching) that a static-config ProxyVM
  doesn't need solved the same way.
- **OpenVPN via `.ovpn` + daemon in `rc.local`** — works, used in some
  community guides, but WireGuard is simpler and better-integrated with NM
  in current templates; no reason to prefer it here.
- **Full `debian-13` template instead of minimal clone** — simpler (NM
  present out of the box) but larger attack surface for a qube that
  handles all downstream network traffic; three extra packages on a
  minimal clone is a small cost for a meaningfully smaller image.
- **Community-maintained "sys-vpn-id0-proton template"** — does not exist as a
  distributable artifact; Qubes' local-build trust model discourages
  pre-built template distribution, and the official minimal-template docs
  already enumerate the needed packages, so the gap is small enough that
  no one has wrapped it (Qusal covers `sys-tailscale`/`sys-mirage-firewall`
  but not a generic WireGuard VPN formula as of this writing).

## Known gotchas summary

- `qubes-core-agent-network-manager` may report as unavailable on a fully
  updated minimal clone if Qubes repos aren't enabled — check
  `/etc/apt/sources.list.d/qubes-r4.list` first.
- WireGuard config filenames >15 chars get truncated as interface names —
  silently, no error.
- The nftables kill switch rule must use a wildcard (`qubes*`) covering
  *all* imported connection names, not just the default one, or switching
  servers breaks connectivity even with a healthy tunnel.
- `provides_network=True` does not give `sys-vpn-id0-proton` its own uplink — `netvm`
  must be set explicitly (`sys-firewall`), or the qube has no network.

## Status

Implemented and confirmed working, including kill-switch fail-closed
behavior and multi-config server switching.

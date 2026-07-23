# LAN-Restricted ProxyVM Setup

A ProxyVM (`lan-proxy`) that only permits outbound traffic to the home LAN
(`192.168.2.0/24`), with all other traffic dropped. Used to isolate qubes
that should only ever talk to home services (e.g. `gitserver.home.arpa`),
with no path to the WAN.

Built on the existing `debian-13-minimal-net` template.

## 1. Create the ProxyVM

```
qvm-create --class AppVM --template debian-13-minimal-net --label green --prop provides_network=True lan-proxy
```

Set its NetVM to your normal uplink (same NetVM your other AppVMs use to
reach the LAN):

```
qvm-prefs lan-proxy netvm sys-firewall
```

## 2. Kill the default route, keep only LAN

Instead of a VPN tunnel, this qube just blocks any route to WAN via
nftables. Add to `/rc.local` inside `lan-proxy` (persists via qubes
bind-dirs, same pattern as the WireGuard ProxyVM):

```bash
#!/bin/bash
# Allow LAN, drop everything else outbound
nft add table inet filter
nft add chain inet filter output '{ type filter hook output priority 0; policy drop; }'
nft add rule inet filter output ip daddr 192.168.2.0/24 accept
nft add rule inet filter output oif lo accept
nft add rule inet filter output ct state established,related accept
```

Make it executable:

```
chmod +x /rc.local
```

This acts as a real kill switch — if the LAN rule fails to apply for any
reason, the qube has no network at all rather than silently leaking to WAN.

## 3. Qube-to-qube LAN routing

`lan-proxy` needs a route to 192.168.2.x through its NetVM. If
`sys-net`/`sys-firewall` already bridges to the physical LAN (true if
other AppVMs already reach `.home.arpa` services), no extra NAT config is
needed. Verify with:

```
ping 192.168.2.1
```

## 4. Point client AppVMs at it

```
qvm-prefs <your-git-appvm> netvm lan-proxy
```

## 5. DNS

DNS must resolve via the LAN resolver (AdGuard Home / dnsmasq), not a WAN
DNS IP — forwarding to a WAN DNS server's IP will be dropped by the
firewall rules above (as intended). Point `lan-proxy`'s resolv.conf /
qubes DNS service at the LAN resolver so `.home.arpa` names resolve
correctly.

## 6. Test

```
qvm-start lan-proxy
qvm-run -p lan-proxy 'ping -c2 192.168.2.1'      # should work
qvm-run -p lan-proxy 'ping -c2 1.1.1.1'          # should fail/timeout
```

From the client qube, confirm the target service (e.g. `git clone` against
`gitserver.home.arpa`) works and confirm no WAN reachability.

## Status

Confirmed working, including `.home.arpa` DNS resolution.

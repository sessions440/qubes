# Coding-agent SSH access to a qube — generic setup

## Context

Goal: let a coding agent (Claude Code, OpenCode, etc.) running in a
dedicated **agent qube** administer another qube over SSH, without giving
the agent dom0 access and (by default) without root in the target.

Worked example in this repo: agent qube `id0-agents` administering
`sys-vpn-id0-proton` (template `debian-13-minimal-net`). Real names are
used where they are known; anything in `<angle brackets>` is a
placeholder to fill in.

## Role tags

- **[Human/dom0]** — requires dom0 privilege, a shell inside a TemplateVM,
  or `qvm-run -u root <qube> ...`. An agent has no path to these.
- **[Human]** — interactive input or a judgment call that should not be
  delegated (key handling, reviewing root-executed files).
- **[Agent/SSH]** — runs as `user` inside an already SSH-reachable qube.

## Design decisions

- **dom0 is off the table.** dom0 has no network by design and controls
  every qube, disk, and USB device. An agent with `qvm-*` access has
  de facto control of the whole machine; one prompt-injected instruction
  has a system-wide blast radius. Root inside a *single* qube is a much
  smaller prize.
- **SSH from a separate agent qube is the right call.** It keeps the
  agent and its credentials out of the qube being administered, and out
  of dom0.
- **The agent gets `user`, not root** (see "Root access" below).
- **An IP-based SSH path only exists to a qube that is upstream of the
  agent qube** — i.e. the agent qube's `netvm`. Sibling qubes cannot
  reach each other (the ProxyVM's `forward` chain drops traffic to
  downstream interfaces). Targets that are not on the agent qube's
  netvm path need a different mechanism (see "Considered and not
  adopted").
- **Side effect:** pointing the agent qube's `netvm` at the target routes
  *all* of the agent qube's traffic through it. For `sys-vpn-id0-proton`
  that means the agent's traffic goes through the VPN.

## Dependency audit

| Package | Where | Why |
|---|---|---|
| `openssh-server` | target's **template** | the SSH daemon; must be in the template because the AppVM root filesystem is inherited read-only |
| `openssh-client` | agent qube | client side (present on full templates; check on minimal) |
| `qubes-core-agent-networking` | target's template, if minimal | Qubes network integration and the `qubes-firewall` service; already required by any networked minimal qube |

## Setup

1. **[Human/dom0]** Install the server in the target's template:
   ```bash
   qvm-run -u root <target-template> xterm
   ```
   In that root shell:
   ```bash
   apt update
   apt install openssh-server
   systemctl is-enabled ssh     # Debian enables it on install
   ```
   Optional hardening (recommended; **not yet verified** in this repo) —
   disable password login so only keys work:
   ```bash
   echo 'PasswordAuthentication no' > /etc/ssh/sshd_config.d/50-no-password.conf
   ```
   Shut the template down, then restart the target qube. Edits made
   directly in the running AppVM's `/etc` are lost on reboot.

   Host keys generated at install time live in the template image, so
   they are stable across reboots of the target — and, as far as I
   can tell, shared by every qube on that template.

2. **[Human]** In the agent qube, generate a dedicated key pair (this
   deployment uses `~/.ssh/ai_homelab`):
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/<agent-key> -C '<agent-qube> agent key'
   ```
   The private key stays in the agent qube. It never enters this repo.

3. **[Human/dom0 prompt, then Human]** Copy the **public** key to the
   target with Qubes' own mechanism (no dom0 file handling):
   ```bash
   # in the agent qube
   qvm-copy ~/.ssh/<agent-key>.pub      # choose <target> in the dom0 prompt
   ```
   In the target (as `user`):
   ```bash
   mkdir -p ~/.ssh && chmod 700 ~/.ssh
   cat ~/QubesIncoming/<agent-qube>/<agent-key>.pub >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```
   `~/.ssh` lives in the target's persistent private volume, so this is
   per-instance and survives reboots. (Not for disposables.) sshd ignores
   `authorized_keys` if permissions are too loose.

4. **[Human/dom0]** Make the target the agent qube's netvm and get the
   target's address:
   ```bash
   qvm-prefs <agent-qube> netvm <target>
   qvm-prefs <target> ip                # e.g. 10.137.0.29 for sys-vpn-id0-proton
   ```
   At this point `ping <target-ip>` works but `ssh` fails with "No route
   to host". That is expected; do the firewall step in the last section.

5. **[Human]** Client config in the agent qube. Add to
   `~/.ssh/config`:
   ```
   Host <alias>
       HostName <target-ip>
       User user
       IdentityFile ~/.ssh/<agent-key>
       IdentitiesOnly yes
   ```
   
6. **[Human]** First connection: compare the host-key fingerprint with
   the server's (`ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` in the
   target) before accepting it.

## Verification

1. **[Agent/SSH]** `ssh <alias> true` exits 0 with no password prompt.
2. **[Human]** With password auth disabled, confirm a login attempt
   without the key is refused.
3. **[Human/dom0]** Reboot the target and repeat 1. (Verifies that both
   the daemon and the firewall rule persist.)

## Templates: what differs

| | `debian-13-minimal` (and clones such as `debian-13-minimal-net`) | `debian-13-xfce` |
|---|---|---|
| `sudo` for `user` inside the qube | **Not available** — `qubes-core-agent-passwordless-root` is not installed by design | **Passwordless by default** |
| SSH server | install `openssh-server` | install `openssh-server` (same steps) |
| Qubes networking agent | install `qubes-core-agent-networking` | included |
| What an SSH login gives the agent | unprivileged `user` | `user` with passwordless `sudo`, i.e. **root-equivalent inside that qube** |

The last row matters: on a full template an "unprivileged" agent is not
unprivileged. Treat an agent SSH login to an xfce-based qube as root in
that qube, and don't use such a qube as a target for anything holding
credentials you wouldn't hand to the agent.

Footprint note: installing `openssh-server` in a *shared* template runs
sshd in every qube on it. Inbound connections are blocked by Qubes'
default-deny firewall until a rule is added to a specific qube (see the
last section), but it is still an extra daemon. Prefer a dedicated or
minimal template for qubes the agent administers. Gating the unit on a
per-qube Qubes service flag is a possible refinement; not tried here.

## Root access

- **Default (used here): no root for the agent.** Root steps are run by a
  human from dom0: `qvm-run -u root <target> '<command>'`. The agent
  drafts the exact command or file; the human reviews and runs it.
- **Files executed as root at boot are root-equivalent.** Anything the
  agent can write that runs at boot — `/rw/config/rc.local`,
  `/rw/config/qubes-firewall-user-script`, NetworkManager dispatcher
  scripts — is a privilege-escalation path even without `sudo`. Check who
  can write them (`ls -ld /rw/config /rw/config/rc.local`; ownership not
  yet verified here). Agent proposes, human installs.
- **Future options, if the agent ever needs real root:**
  - Install `qubes-core-agent-passwordless-root` in the template. Scope is
    every qube on that template (for `debian-13-minimal-net` that includes
    `lan-proxy`).
  - Replace passwordless root with a dom0 yes/no prompt (Qubes community
    guide; no guarantee of safety). Keeps a human in the loop per action.
  Not adopted for ProxyVMs, which hold VPN keys and the kill switch.

## Gotcha: Qubes' default-deny inbound firewall

**Symptom:** `ping <target-ip>` works, `ssh` fails with "No route to
host" (not a timeout), and `systemctl status ssh` in the target is
healthy.

**Cause:** Qubes' `qubes` nftables table has an `input` chain with
`policy drop`. Its first rule is `jump custom-input` (an empty,
user-editable chain), followed by rules that accept established traffic
and ICMP and then `reject with icmp host-prohibited` for new connections
from downstream interfaces. The reject is what the client reports as "No
route to host"; the ICMP accept is why ping works. You can see the reject
counter rise on each failed attempt:
```bash
nft list table qubes        # [Human/dom0] run via: qvm-run -u root <target> '...'
```

**Fix:** add an accept rule to `custom-input`, scoped to the agent qube's
address. Plain `add` is enough; the chain is evaluated before the default
reject, so `insert ... index 0` is unnecessary:
```bash
nft add rule qubes custom-input ip saddr <agent-qube-ip> tcp dport 22 accept comment '"agent SSH from <agent-qube> (see doc/agents/agent-ssh-access.md)"'
```
The single-quoted double-quoted comment is required for nft to parse it;
comments are limited to 128 bytes. Get the agent qube's address with
`qvm-prefs <agent-qube> ip`. This takes effect immediately (verified live
for `id0-agents` → `sys-vpn-id0-proton`).

**Persistence (not yet verified):** the rule is lost on reboot. Because
the target is necessarily a network-providing qube (see Design
decisions), the documented home is `/rw/config/qubes-firewall-user-script`
(make it executable), not `rc.local`. There are long-standing reports of
that hook being ignored in some versions, so also test after both a full
reboot and `systemctl restart qubes-firewall`:
```bash
nft list chain qubes custom-input
```
If the rule appears twice (both hooks fired), keep only the one that
works. Consider an idempotent form so repeated runs don't stack rules:
```bash
#!/bin/sh
# Allow SSH from <agent-qube> so a coding agent can administer this qube.
# Source IP derives from <agent-qube>'s QID; re-derive if it is recreated.
nft list chain qubes custom-input | grep -q 'saddr <agent-qube-ip> tcp dport 22' || \
  nft add rule qubes custom-input ip saddr <agent-qube-ip> tcp dport 22 accept comment '"agent SSH from <agent-qube>"'
```
Writing this file is a **[Human/dom0]** step (`qvm-run -u root <target>
xterm`); it runs as root at boot.

**Ongoing maintenance:**
- Regular qubes get `10.137.0.<qid>`; the address depends on the qube's
  own QID, **not** on its netvm (Qube Manager shows `10.137.0.x` across
  qubes with different netvms). So one rule source address works for every
  target the agent qube is attached to, and switching its netvm does not
  change it. Each target still needs its own copy of the rule.
- The address changes only if the agent qube is **deleted and recreated**
  (new QID) — and possibly if restored from a backup (old report, not
  verified on 4.3). Re-derive it and update the rule then.
- **Disposables** get `10.138.x.y`, which differs per instance, so an
  exact-address rule cannot be used for a disposable agent qube. Use a
  regular AppVM for the agent qube.

## Considered and not adopted

- **Agent in dom0 / dom0 `qvm-run` driven by an agent.** See Design
  decisions.
- **Passwordless root in shared templates.** See Root access.
- **Subnet-wide accept rule** (survives recreating the agent qube without
  maintenance, but opens port 22 to any qube attached to the target).
- **`qvm-connect-tcp` for targets not on the agent qube's netvm path**
  (for example an ordinary AppVM). It tunnels over qrexec and needs a
  dom0 policy entry. Not tried for SSH here.

## Status

Partially confirmed. Verified live: SSH key login from `id0-agents` to
`sys-vpn-id0-proton` (`debian-13-minimal-net`) after a manually added
`custom-input` rule. **Not yet verified:** rule persistence across reboot
and `qubes-firewall` reload; password-auth-off drop-in; the
`debian-13-xfce` variant; ownership of `/rw/config`.

## Forward references

- `fix-silent-reconnect.md` — first task planned for an agent over this
  path.
- `vpn-proxyvm.md`, `vpn-proxyvm-gui.md` — the ProxyVMs this access
  targets.
- `AGENTS.md` — agent security rules (dom0 boundary, least privilege).

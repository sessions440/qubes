# Coding-agent access to a qube — qrexec shell (preferred), SSH over qrexec (backup)

Suggested repo path: `doc/agents/agent-qube-access.md` (replaces `agent-ssh-access.md`; see
"Doc housekeeping").

## Context

Goal: let a coding agent (Claude Code, OpenCode, etc.) running in a dedicated **agent qube**
administer another qube (the **target**) through a shell-like interface, without giving the
agent dom0 access. A human can use the same path.

Worked example in this repo: agent qube `id0-agents` administering `sys-vpn-id0-proton`
(template `debian-13-minimal-net`). Real names are used where known; anything in
`<angle brackets>` is a placeholder.

The transport is **qrexec**, Qubes' inter-qube RPC, not IP. Two services are documented:

- **`qubes.VMShell`** (preferred): a shell as `user` in the target, reached with
  `qrexec-client-vm`. One dom0 policy line per target; nothing to install in the target.
- **`qubes.ConnectTCP` + SSH** (backup): a TCP pipe to the target's sshd, for when real SSH
  features are needed (`scp`, `sftp`, TTY, SSH-based tooling).

This replaces the earlier IP-based design (SSH to the agent qube's upstream netvm plus a
`custom-input` firewall rule). See "Considered and not adopted".

## Role tags

- **[Human/dom0]**: requires dom0 privilege, root inside a qube via `qvm-run -u root`, or a
  shell inside a TemplateVM. An agent has no path to these.
- **[Human]**: interactive input or a judgment call that should not be delegated.
- **[Agent/qrexec]**: runs as `user` from the agent qube through `qrexec-client-vm`. Works only
  if a dom0 policy line already allows it; the agent cannot create one. *(New tag: the VMShell
  counterpart of `[Agent/SSH]`; add it to `AGENTS.md`.)*
- **[Agent/SSH]**: runs as `user` over SSH. Backup path only.

## Design decisions

- **dom0 stays off the table.** Same rationale as `AGENTS.md`: an agent with `qvm-*` access
  controls the whole machine.
- **qrexec instead of IP.** dom0 policy is the one place that says which qube may call which
  service on which target. No per-qube firewall rules, no netvm swap, and it works for any
  target regardless of network topology (sibling, upstream, no netvm). The target sees the
  caller's qube name as supplied by dom0, which the caller cannot influence.
- **Policy is the sole gate for VMShell.** Use exact source and target names; never `@anyvm`
  or wildcards. Rules are matched first-to-last across all policy files (lexical order); the
  first match wins and no match means deny. A `*`-argument line shadows specific-argument
  lines below it. Keep our rules in a file that sorts before `90-default.policy`.
- **Explicit targets, not `@default ... target=`.** With a redirect, the action is decided
  before redirection, so rules about the redirected target are never consulted. Explicit
  names are easier to audit.
- **VMShell preferred, SSH as backup.** VMShell needs no template change, no sshd, no keys and
  no host-key trust, and works for any qube including disposables. Its costs are no TTY, no
  `scp`/`rsync`, and a stdin quirk (see Gotchas). Qubes' docs classify VMShell as giving full
  control over the target; that is the intent here, so scope the policy line accordingly.
- **The agent gets `user`.** Root is a per-target opt-in (see "Root access").

| | `qubes.VMShell` | `qubes.ConnectTCP` + SSH |
|---|---|---|
| Needed in target | nothing (service ships with the core agent) | `openssh-server` in the target's template, `authorized_keys` |
| Keys / host-key trust | none | agent key, per-target `authorized_keys`, first-use fingerprint check |
| Setup per target | 1 policy line | policy line, key copy, ssh config entry |
| Disposable targets | work | per-instance keys do not persist |
| stdout, stderr, exit code | yes (verified) | yes (verified) |
| TTY | none | possible with `ssh -t` (not tested) |
| `scp` / `sftp` | no | yes (`scp` verified) |
| `rsync`, port forwarding, IDE remote | no | possible (not tested) |
| Root path | `qubes.VMRootShell` policy line | `qubes.VMRootShell` alongside, or sudo (template-wide package) |

## Dependency audit

| Item | Where | Needed for |
|---|---|---|
| `qubes.VMShell`, `qubes.VMRootShell` services | target (ship with the Qubes core agent) | preferred path; present in a `debian-13-minimal-net`-based qube (verified) |
| `qrexec-client-vm` | agent qube | both paths (verified) |
| `openssh-server` | target's template | **backup path only** |
| `openssh-client` | agent qube | backup path only |

Consequence: if the backup path is not used, `openssh-server` can be purged from templates
where it was installed only for the earlier design (see "Cleanup").

## Setup: preferred path (VMShell)

1. **[Human/dom0]** Create `/etc/qubes/policy.d/30-user-agents.policy`:
   ```
   # Agent qube -> target: unprivileged shell. One line per target, exact names, no wildcards.
   qubes.VMShell  *  id0-agents  sys-vpn-id0-proton  allow

   # Root shell opt-in: leave commented out. Uncomment for a work session, re-comment after.
   #qubes.VMRootShell  *  id0-agents  sys-vpn-id0-proton  allow
   ```
   The target is started automatically on `allow` if it is halted. Nothing needs persisting
   in the target: the policy lives in dom0.

2. **[Agent/qrexec]** Run one command per invocation:
   ```bash
   printf '%s\n' '<command>' | qrexec-client-vm sys-vpn-id0-proton qubes.VMShell
   ```
   Verified behavior: stdout and stderr text both reach the caller, the exit status
   propagates, the shell runs as `user`, and there is no TTY. (Whether stderr arrives on a
   separate stream was not checked.)

3. **Files.** `scp` and `rsync` do not work over this path.
   - Small files: send them on stdin after the command line. **Untested:**
     ```bash
     { echo 'cat > /tmp/f'; cat local-file; } | qrexec-client-vm <target> qubes.VMShell
     ```
   - Human-approved transfers: `qvm-copy <file>` from the agent qube; dom0 prompts, and the
     file lands in `~/QubesIncoming/id0-agents/` in the target.

## Root access

- **Default: root off, as a speed bump, not a security boundary.** It guards against agent
  mistakes (a stray `rm`, an unintended `apt`), not against an attacker. Qubes' own docs argue
  that separating root from user inside a VM gives no meaningful protection. On full templates
  (`debian-13-xfce` family) `user` has passwordless sudo, so root-off is moot there. On minimal
  clones it is a real but modest hurdle: an attacker needs a local exploit or a root-run path
  that `user` can write.
- **The real boundary is which targets have a policy line at all**, and what lives in them.
  Weigh "is a compromised agent holding `user` here acceptable?" before adding a VMShell line,
  not only before enabling root.
- **Mechanism.** Every target gets a commented-out `qubes.VMRootShell` allow line (step 1
  above). Verified on a throwaway: with the line allowed, `VMRootShell` returns `uid=0` and
  `VMShell` still returns `user`; with the line commented out, `VMRootShell` is refused
  immediately and no restart is needed.
  - **[Human/dom0]** Switch on: uncomment the line. Switch off: comment it out.
  - Works alongside either transport.
- **No eligibility list.** The dom0 policy file is the record of which targets are root-on.
  Decide case by case, weighing: what secrets live on the target; the template type (on an
  xfce-based target root-off is moot, so the question is whether the agent gets any shell at
  all); and whether the tedium of root-off outweighs the benefit.
- **Agent check.** **[Agent/qrexec]** `echo id | qrexec-client-vm <target> qubes.VMRootShell`.
  `uid=0` means root is on; `Request refused` means off. A refusal is final: ask the human.
  Do not look for a workaround.
- **Approval practice when root is on.** Anything that persists as root-run at boot or alters
  a security control (`/rw/config/rc.local`, `/rw/config/qubes-firewall-user-script`,
  NetworkManager dispatcher scripts, sudoers, the kill switch) is shown to the human in full
  and approved before it is written. "Run script.sh" without showing the content does not
  count. With root off, the agent cannot write these files anyway: it drafts, and a human
  installs with `qvm-run -u root`. Note the limit: with root on, this rule is enforced by the
  agent harness and instructions inside the agent qube, not by dom0.
- **Harness permissions (unverified idea).** Auto-allow user-level `VMShell` calls but always
  prompt for `VMRootShell` calls. How well this works depends on how the harness matches piped
  commands. If matching is awkward, a tiny wrapper with separate user and root names would be
  the one place a wrapper earns its place.
- **`ask` caveat.** As far as I know, a dom0 `ask` prompt shows the service, source and target,
  not the command, so it is consent to a shell, not review of what runs. Per-command prompts
  are impractical for an agent. (Not verified.)
- **Not adopted for root:** the `qubes-core-agent-passwordless-root` package (template-wide
  scope), and the community dom0-prompt guide for sudo (the Qubes docs say not to rely on it
  for extra security).

## Setup: backup path (ConnectTCP + SSH)

Use when a task needs `scp`/`sftp`, a TTY, port forwarding or SSH-based tooling.

1. **[Human/dom0]** Install the server in the target's template. This modifies a shared
   template; prefer a dedicated or minimal one.
   ```bash
   qvm-run -u root <target-template> xterm
   # in that root shell:
   apt update && apt install openssh-server
   ```
   Shut the template down, then restart the target. Host keys come from the template, so they
   are identical across qubes on it; they cannot tell you which qube you reached. Routing is
   decided by dom0 policy and the target named in the `ProxyCommand`.

2. **[Human]** In the agent qube, create a dedicated key (this deployment uses
   `~/.ssh/ai_homelab`):
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/ai_homelab -C 'id0-agents agent key'
   qvm-copy ~/.ssh/ai_homelab.pub        # choose the target in the dom0 prompt
   ```
   In the target as `user`:
   ```bash
   mkdir -p ~/.ssh && chmod 700 ~/.ssh
   cat ~/QubesIncoming/id0-agents/ai_homelab.pub >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```
   The private key stays in the agent qube and never enters this repo.

3. **[Human/dom0]** Add a port-specific line to the same policy file (never `*`):
   ```
   qubes.ConnectTCP  +22  id0-agents  sys-vpn-id0-proton  allow
   ```

4. **[Human]** In `~/.ssh/config` in the agent qube (not `/etc/ssh/ssh_config`; a forum report
   says a 4.3 upgrade overwrote an `/etc` edit):
   ```
   Host sys-vpn-id0-proton
       User user
       IdentityFile ~/.ssh/ai_homelab
       IdentitiesOnly yes
       ProxyCommand qrexec-client-vm sys-vpn-id0-proton qubes.ConnectTCP+22
   ```
   The port attaches to the service name with `+`, with no space. The first connection shows
   `<no hostip for proxy command>` in the host-key prompt; that is expected. Compare the
   fingerprint with `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` in the target.

5. **[Human/dom0], optional hardening.** Bind sshd to loopback so it is only reachable through
   qrexec:
   ```bash
   qvm-run -u root <target-template> xterm
   # in that root shell:
   printf 'ListenAddress 127.0.0.1\n' > /etc/ssh/sshd_config.d/60-loopback.conf
   ```
   Verified on a throwaway (applied in the running qube): `ss -ltn` showed port 22 only on
   `127.0.0.1`, and SSH through `ConnectTCP` still worked.

6. **[Agent/SSH]** `ssh sys-vpn-id0-proton '<command>'`, `scp`, `sftp`.

## Cleanup of the earlier IP-based design

Do this after the preferred path is verified on the real target.

1. **[Human/dom0]** Restore the agent qube's netvm:
   ```bash
   qvm-prefs id0-agents netvm <normal-netvm>
   ```
2. **[Human/dom0]** Remove the old inbound rule. The live rule is gone after a reboot; also
   check for a persisted copy:
   ```bash
   qvm-run -p -u root sys-vpn-id0-proton 'nft list chain qubes custom-input'
   qvm-run -p -u root sys-vpn-id0-proton 'cat /rw/config/qubes-firewall-user-script'
   ```
   Delete any agent-SSH accept rule found.
3. **[Human/dom0]** Remove the agent's key from the target if the backup path is not kept:
   ```bash
   qvm-run -p sys-vpn-id0-proton "grep -n 'id0-agents agent key' ~/.ssh/authorized_keys"
   ```
   Edit out the matching line. In the agent qube, delete the `Host` block and
   `~/.ssh/ai_homelab*` if SSH is not kept.
4. **[Human/dom0]** Purge sshd from templates where it was installed only for this
   (`debian-13-minimal-net`, and `debian-13-xfce-net` if it was added there):
   ```bash
   qvm-run -u root debian-13-minimal-net xterm
   # in that root shell:
   apt purge openssh-server
   ls /etc/ssh/ssh_host_*      # confirm host keys are gone
   ```
   Shut the template down and restart dependent qubes. Skip this step if the backup path stays.

## Verification

Per target, after adding its policy line:

1. **[Human/dom0]** `cat /etc/qubes/policy.d/30-user-agents.policy` shows the intended lines.
2. **[Agent/qrexec]**
   ```bash
   printf 'echo out; echo err >&2; id -un; tty; exit 3\n' \
     | qrexec-client-vm <target> qubes.VMShell; echo "exit=$?"
   ```
   Expect `out`, `err`, `user`, `not a tty`, `exit=3`.
3. **[Agent/qrexec]** `echo id | qrexec-client-vm <target> qubes.VMRootShell` is refused by
   default.
4. **[Agent/qrexec]** A qube with no policy line is refused:
   `echo id | qrexec-client-vm <untargeted-qube> qubes.VMShell`.
5. **[Human/dom0] + [Agent/qrexec]** Root toggle: uncomment the `VMRootShell` line, expect
   `uid=0`; comment it out, expect `Request refused`.
6. Backup path only: **[Agent/SSH]** run `ssh <alias> 'echo out; echo err >&2; exit 3'`, then an
   `scp` round trip. **[Agent/qrexec]** `echo | qrexec-client-vm <target> qubes.ConnectTCP+80`
   must be refused.

## Gotchas

- **No TTY.** Interactive programs, pagers and `sudo` password prompts do not work over
  VMShell. For a human, open a terminal in the target or use `qvm-run` from dom0.
- **Stdin shares the stream with the command.** The target shell reads its script from the
  same pipe, so a command that reads stdin swallows the following lines. Verified:
  `printf 'cat\necho after\n' | ... qubes.VMShell` printed `echo after` instead of running it.
  Send one command per invocation, and treat the rest of the stream as that command's stdin.
- **Policy syntax.** The service name is case-sensitive; the argument is `+22` (no space) in
  `qrexec-client-vm`; a lowercase `qubes.connectTCP` policy line was reported ignored.
  Denials appear in dom0's journal (reported).
- **Policy order.** First match wins, and a `*` argument line shadows specific lines below it.
- **`allow` starts a halted target.**
- **Full templates.** On `debian-13-xfce`-based targets (for example `sys-vpn-id0-proton-gui`,
  which holds Proton account credentials) a `user` shell is root-equivalent through
  passwordless sudo, whichever transport is used. Decide whether the agent gets any shell
  there at all; see `vpn-proxyvm-gui.md`.
- **No per-command audit trail** is configured for either path. A custom logging service in
  `/usr/local/etc/qubes-rpc` of one qube could add it; not adopted.

## Considered and not adopted

- **IP-based SSH to the agent qube's upstream netvm.** Works only for that one target,
  requires swapping the agent qube's netvm (routing all its traffic through the target), and
  needs a `custom-input` nftables rule in every target plus a persistence hook
  (`/rw/config/qubes-firewall-user-script`), with open reports of that hook being ignored. The
  Qubes docs present this as the special-situations route.
- **Agent in dom0 / dom0 `qvm-run` driven by an agent.** See Design decisions.
- **Passwordless root in shared templates.** Template-wide scope.
- **Subnet-wide firewall accept.** Opens port 22 to any qube attached to the target.
- **`qvm-connect-tcp` local listener in the agent qube.** Any process there can use the bound
  port. The Qubes-documented permanent form is a systemd socket unit bound to `127.0.0.1`.
  `ProxyCommand` avoids the listener.
- **A wrapper script around `qrexec-client-vm`.** The raw one-liner works and avoids another
  maintenance entry. Revisit only if quoting mistakes recur or harness permission matching
  needs distinct command names.
- **`qubes.VMRootShell` as the default.** See "Root access".

## Doc housekeeping

- **`AGENTS.md`:** add the `[Agent/qrexec]` role tag. Rewrite "No root for agents by default":
  root off by default as a speed bump, opt-in via a commented-out `VMRootShell` policy line,
  the approval practice above, and "a refusal is final". Update the link that points at
  `agent-ssh-access.md`.
- **`fix-silent-reconnect.md`:** prerequisites and `[Agent/SSH]` tags become `[Agent/qrexec]`
  once the preferred path is verified on the real target.
- **`vpn-proxyvm-gui.md`:** step 2 no longer needs `openssh-server`; step 7 becomes a policy
  line plus the full-template caveat above.
- **`vpn-proxyvm.md`:** update the pointer to this doc.

## Forward references

- `vpn-proxyvm.md`, `vpn-proxyvm-gui.md`: the ProxyVMs this access targets.
- `fix-silent-reconnect.md`: first task planned for an agent over this path.
- `AGENTS.md`: agent security rules (dom0 boundary, least privilege).
- Qubes docs (4.3): firewall page (`ConnectTCP` section), qrexec page (policy syntax), and
  "Passwordless root access in qubes" (`doc.qubes-os.org/en/r4.3/user/security-in-qubes/vm-sudo.html`).

## Status

Design proposed. Behavior was verified on throwaway clones of `sys-vpn-id0-proton` (netvm
unset), not on the real target:

- **Verified:** `VMShell` stdout, stderr, exit code, `user`, no TTY, and the stdin quirk;
  `VMRootShell` refused by default, `uid=0` when allowed, refused again when commented out;
  `ConnectTCP` + SSH with stdout, stderr, exit code and `scp`; `ConnectTCP+80` refused;
  sshd bound to `127.0.0.1` still reachable through qrexec.
- **Not verified:** any of the above on the real `sys-vpn-id0-proton`; `rsync`; disposable
  or xfce-based targets; stdin-based file upload; harness permission matching; `ask` prompt
  contents; sshd purge side effects.

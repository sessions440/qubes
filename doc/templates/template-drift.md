# Cloned Template Drift — Keeping a Customized Template Reproducible

## Context

An AppVM boots from a read-only copy of its template's root volume. Only the
AppVM's private volume persists (`/home`, `/rw`, plus a few bind-mounted
paths). Anything that must exist in every qube sharing a base therefore must
be installed in the **template**.

This repo's policy is to rely on community-maintained templates and not
modify them in place, because they are shared and updated upstream. The
standard answer is to **clone the template and customize the clone**
(see "Minimal-footprint discipline" in `AGENTS.md`).

That answer has a cost, and this doc is about it: **a clone is a snapshot,
and nothing records what you did to it.** Without a declarative system (Nix
or similar), the only way to rebuild the clone faithfully, for example when
upstream ships a new OS release, is to have kept very detailed notes.

This doc is generic: it applies to any community template (Debian, Fedora,
Whonix, and so on) that you clone and customize. Template-specific upgrade
procedures are out of scope; follow the template maintainers' docs for those.

## The problem, stated precisely

"Drift" is two different problems:

1. **Delta loss.** You made changes to the clone (packages, files in `/etc`,
   unpackaged software) and cannot reliably list them. When the clone must be
   rebuilt, you are reverse-engineering your own work. This is the problem
   that hurts most, and it is entirely under your control.
2. **Upstream divergence.** The maintainers change things in the upstream
   template that your clone never receives.

What actually reaches a clone (my understanding; see Status for what is
unverified):

| Kind of change | Reaches the clone? | How |
|---|---|---|
| Package updates from repos the clone also uses | Yes | the clone's own `apt upgrade` / `dnf upgrade` |
| Changes baked into the template image at build time: default config not owned by a package, repo/sources lists, which packages are preinstalled, services enabled by default | **No** | only a new template release (a new image) carries these |
| Packaged config files you edited, when the package ships a new default | **Conflict** | the package manager prompts or keeps one side (see Gotchas) |
| Your own changes | n/a | exist only in your clone |

So "clone + `apt upgrade`" tracks upstream for *packages* and for nothing
else. How much the image-level changes matter depends on how much the
maintainers do beyond packaging. For a near-stock Debian or Fedora template
it is small. For a template whose maintainers ship a lot of custom
configuration it can be large. Check the template's release notes rather than
assuming.

### When it bites: the release cycle

Within one OS release, updates arrive through the package manager and drift is
mild. The painful event is a **new OS release** (Debian N to N+1, a new
Whonix release, and so on), which arrives as a *new template*. At that point
you either:

| | Rebuild | In-place upgrade of the clone |
|---|---|---|
| What you do | New upstream template, clone it, replay your delta | Upgrade the clone's OS release where the template docs describe it |
| Your delta survives by | being replayed | simply being there |
| Image-level upstream changes | picked up | **missed** (same as any drift) |
| Reproducible afterward | yes, if the delta is executable | no; accumulated hand-state carries forward |
| Effort | needs a recorded delta | low once, compounding debt |

Recommendation: **default to rebuild, and make the rebuild cheap.** Use
in-place upgrade only as a stopgap.

## What is state-of-the-art

There is no first-party "declare your template" equivalent of Nix. The
practical toolbox, from lightest to heaviest:

### 1. Shrink the delta (do this first)

Every change you keep out of the template is a change you never have to
replay. In order of preference:

1. **Don't put it in a template at all.** Software that can live entirely in
   an AppVM's `/home` (see `appimage-install.md`, `pass-install.md`) and
   per-AppVM config (`/rw/config/rc.local`, or bind-dirs for chosen `/etc`
   paths) never enter the delta.
2. **Split by purpose.** Several small clones, each with a delta of a few
   lines, are easier to reproduce than one large clone. The cost is more
   templates to update; the benefit is a smaller blast radius per template.
3. Only then, put it in the template.

### 2. Make the remaining delta executable: a build script

The lowest-tech option that actually works: a single, idempotent shell script
per clone, kept in this repo, that turns a fresh clone of the upstream
template into your customized template. The script **is** the notes: they
cannot go stale without the script failing.

Mapping your usual modification types to what the script records:

| Modification | Script records | Updates |
|---|---|---|
| `apt install` from the distro repo | the package name | with the template's normal updates |
| `apt install` from a third-party repo (e.g. a browser vendor) | repo URL, signing key (fingerprint verified against the vendor's published value), sources file, package name | with the template's normal updates, via the repo you added |
| Unpackaged software (e.g. an AppImage) | pinned version, download URL, sha256 | **none**; manual |
| Config tweak in `/etc` | the file, as a drop-in (see Gotchas) | n/a |

Skeleton (illustrative, adapt per clone; angle-bracket values are
placeholders, not real names):

```bash
#!/bin/bash
# <clone>.sh: idempotent delta from <upstream-template> to <clone>.
# Runs as root inside the template. A human reviews it before every run.
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

# --- 1. Packages from the distro's own repos ---
apt-get update
apt-get install -y <package-1> <package-2>

# --- 2. Third-party apt repo (key fingerprint checked by a human first) ---
install -m 0644 <key-file> /usr/share/keyrings/<vendor>.gpg
cat > /etc/apt/sources.list.d/<vendor>.sources <<'EOF'
Types: deb
URIs: <repo-url>
Suites: <suite>
Components: <component>
Signed-By: /usr/share/keyrings/<vendor>.gpg
EOF
apt-get update
apt-get install -y <vendor-package>

# --- 3. Config as drop-ins, not edits to packaged files ---
install -d -m 0755 /etc/<service>.d
cat > /etc/<service>.d/60-local.conf <<'EOF'
<directive>
EOF

# --- 4. Unpackaged software: pinned version + checksum ---
# download to /tmp, then: echo "<sha256>  <file>" | sha256sum -c -
# install under /opt/<app>  (not /usr/local; see Gotchas)
```

Running it. **Everything here is `[Human/dom0]`**: a template shell has no
agent path (see `AGENTS.md`). An agent may *draft* the script in this repo;
a human reviews and runs it. The script runs as root in the template, so it
is root-equivalent for every qube on that template.

```bash
# [Human/dom0] pull the reviewed script into dom0 (see vendor/qvm-pull)
qvm-run --pass-io <repo-appvm> 'cat <path>/<clone>.sh' > ~/<clone>.sh

# [Human/dom0] read it in full, then run it as root in the template
qvm-run --pass-io -u root <clone> 'bash -s' < ~/<clone>.sh
```

Keep **secrets out of the script** (`AGENTS.md`). Vendor signing keys are
public and fine; account tokens are not.

### 3. Salt, when the script outgrows itself

Qubes ships SaltStack integration (`qubesctl`, run from dom0, executed in
targets through a management disposable). States such as `pkg.installed`,
`file.managed` and `pkgrepo.managed` express the same delta declaratively, can
be re-applied, support a dry run (`test=True`), and Qubes' own `qvm.*` states
can also create the clone, so "clone and customize" becomes one command.
Qusal is a community library of Salt formulas for Qubes (browsing, coding,
network tunnels, and so on).

Use Salt over a script when you have many templates, want dom0-driven
end-to-end builds, or want dry-run drift reports. Costs:

- Steep learning curve and a long-standing reputation for it; several users
  report falling back to plain shell scripts.
- State files live in dom0 (`/srv/user_salt/`). A human installs them there;
  an agent never writes them (`AGENTS.md`, dom0 boundary). Pulling code into
  dom0 is a trust decision, so review it first.
- Qusal described itself as development-only, not production-ready, when last
  checked (mid-2026). Re-check before depending on it.

### 4. Audit an existing clone you never recorded

If you already have a customized clone with no script, reconstruct the delta
by comparing against a pristine clone of the same upstream template:

```bash
# [Human/dom0] one-off pristine reference
qvm-clone <upstream-template> <upstream-template>-pristine

# [Human/dom0] manually installed packages
qvm-run --pass-io <clone> 'apt-mark showmanual | sort' > clone.pkgs
qvm-run --pass-io <upstream-template>-pristine 'apt-mark showmanual | sort' > pristine.pkgs
diff pristine.pkgs clone.pkgs

# [Human/dom0] packaged files modified since install (conffiles flagged 'c')
qvm-run --pass-io <clone> 'dpkg -V'

# [Human/dom0] unpackaged files in the usual places
qvm-run --pass-io <clone> 'find /etc /opt /usr/local -xdev -type f | sort' > clone.files
qvm-run --pass-io <upstream-template>-pristine 'find /etc /opt /usr/local -xdev -type f | sort' > pristine.files
diff pristine.files clone.files
```

On Fedora-based templates the equivalents are `dnf repoquery --userinstalled`
and `rpm -Va`. Treat the output as a worklist to turn into the script; it is
not itself a complete record (a `dpkg -V` hit tells you a file changed, not
what the change was).

### 5. Prove it: rebuild from pristine and diff

Scripts and Salt states are **convergent, not hermetic**: they add what you
declared, but nothing removes a change you made by hand and forgot to
declare. Only a rebuild from pristine proves the record is complete.
Periodically (and before any new-release migration):

1. **[Human/dom0]** `qvm-clone <upstream-template> <clone>-test`
2. **[Human/dom0]** Run the script on `<clone>-test`.
3. **[Human/dom0]** Repeat the Section 4 comparisons between `<clone>` and
   `<clone>-test`. Any difference is state your script doesn't capture.
4. **[Human/dom0]** Delete `<clone>-test` once reviewed (confirm first;
   `AGENTS.md` asks for confirmation before destructive actions).

## Migration procedure (new upstream release)

1. **[Human/dom0]** Install the new upstream template per its maintainers'
   instructions.
2. **[Human/dom0]** Clone it to a new, release-named clone, e.g.
   `<clone>-<release>`.
3. **[Human/dom0]** Adapt the script if package names or repos changed (the
   script's failure messages are your changelog), then run it.
4. **[Human]** Test with a throwaway AppVM based on the new clone.
5. **[Human/dom0]** For each AppVM: shut it down, then
   `qvm-prefs <appvm> template <clone>-<release>`.
6. Keep the old clone until the new one has proven itself; delete it only
   after explicit human confirmation.

## Gotchas

- **Edited conffiles collide with upstream.** If you edit a packaged file in
  `/etc` and a later update ships a new default, the package manager must
  choose or prompt. Prefer **drop-in directories** (`*.d/`) so your changes
  live in files no package owns. How Qubes' update tooling handles a
  conffile prompt was not checked.
- **`/usr/local` is special in AppVMs.** As I understand it, an AppVM mounts
  its own persistent `/usr/local` over the template's, seeded from the
  template's contents on the AppVM's first start and not updated afterward.
  Unpackaged software installed in a template is safer under `/opt`.
  (Unverified; see Status.)
- **Unpackaged software never updates itself.** Anything in section 4 of the
  script (AppImages and similar) is frozen at its pinned version until you
  bump it. Put a version note in the script so a stale pin is visible.
- **Third-party repos move trust into your hands.** The key and sources file
  are part of your delta; verify fingerprints against the vendor's published
  value, and re-verify if the repo rotates its key.
- **Blast radius.** A template change affects every AppVM based on it. Test on
  a throwaway clone first; prefer more, smaller clones for risky changes.
- **A template can only be switched for a halted AppVM.**
- **Minimal templates may lack pieces.** A `*-minimal` clone may need extra
  Qubes agent packages to be a network qube or a Salt target; see
  `vpn-proxyvm.md` for the networking packages.

## Considered and not adopted

- **Relying on notes alone.** What the problem statement starts from. They
  drift from the template silently; a script fails loudly.
- **Backups or snapshots as the record.** `qvm-backup` and old templates are a
  good safety net, but they preserve state without explaining it.
- **In-place upgrades as policy.** Fine as a stopgap; they accumulate
  unrecorded hand-state and miss image-level changes.
- **Nix or Guix inside a template.** It could give hermetic user-level
  packages, but it does not manage the base OS or `/etc`, and I have not
  evaluated it on Qubes.
- **Building custom templates from source (qubes-builder).** Fully
  reproducible, but heavy, and at odds with relying on community templates.
- **NixOS-style whole-system templates.** I'm not aware of a mature,
  maintained option; not researched.

## Verification

For a given clone with a build script:

1. **[Human/dom0]** The script runs twice in a row on a clone without errors
   or changes the second time (idempotence).
2. **[Human/dom0]** Section 5's rebuild-and-diff shows no unexplained
   differences between `<clone>` and `<clone>-test`.
3. **[Human]** Each AppVM on the clone starts and its apps work.
4. **[Human/dom0]** After the next package update cycle, repeat step 2.

## Forward references

- `appimage-install.md`, `pass-install.md`: keeping software out of the
  template delta entirely.
- `vpn-proxyvm.md`, `lan-restricted-proxyvm.md`: minimal-template clones whose
  delta is a handful of packages plus `rc.local` and dispatcher files.
- `<debian-13-xfce-brave template doc>`: not yet written; a natural first
  candidate for a build script.
- `AGENTS.md`: dom0 and template boundary, no secrets, confirm-before-delete.
- Qubes docs: templates (updating, in-place upgrades) and Salt
  (`qubesctl`). Verify current URLs and wording.

## Status

Design proposed, not yet implemented. Nothing here has been run on the real
system.

Unverified, check before relying on it:

- The table of what reaches a clone, and in particular that image-level
  changes arrive only with a new template release.
- The `qvm-run --pass-io -u root <clone> 'bash -s' < file` delivery pattern.
- `/usr/local` seeding behavior in AppVMs, and conffile handling under the
  Qubes updater.
- Salt invocation details and `qvm.*` state names for creating clones.
- Which templates document a supported in-place upgrade path.
- Qusal's current status (last seen: development-only).

# OpenCode Installation (Qubes AppVM)

User-space install, no template modification required. Confirmed working in a
Whonix-based AppVM (debian-13 base should work identically).

## Install

```bash
curl -fsSL https://opencode.ai/install | bash
```

Installs to `~/.opencode/bin/opencode` (default fallback path — no
`$OPENCODE_INSTALL_DIR` or `$XDG_BIN_DIR` override needed). Everything lives
under `/home`, so it persists across AppVM reboots without touching the
TemplateVM.

Make sure `~/.opencode/bin` is on `$PATH` (add to `~/.bashrc` / `~/.profile`
if not already).

## Configure OpenRouter

1. Launch cold:
   ```bash
   opencode
   ```
2. Run `/connect`
3. Select **OpenRouter** from the provider list
4. Paste OpenRouter API key (from https://openrouter.ai/keys)
5. Run `/model` and select preferred model

That's it — no gotchas, no manual config file editing.

## What gets created

- `~/.local/share/opencode/auth.json` — stores the OpenRouter API key
  (created automatically by `/connect`, don't need to hand-write this)

## Notes

- No template changes, no root required — pure user-space, consistent with
  Claude Code (native binary) install pattern
- Same OpenRouter API key can be reused across Claude Code and OpenCode
  installs if desired

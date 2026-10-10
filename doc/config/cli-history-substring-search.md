# History Substring Search in Qubes OS Bash terminals

## Purpose

Bind the **Up** and **Down** arrow keys to search shell history by the
current line's prefix (substring match), rather than cycling through
entries linearly.

## Prerequisites

- Qubes OS dom0 shell: **Bash** (default XFCE Terminal)
- File: `/etc/inputrc` (system-wide) **or** `~/.inputrc` (per-user)

> `~/.inputrc` takes precedence over `/etc/inputrc` if both exist.

## Steps

Add the following to `~/.inputrc` (or `/etc/inputrc`):

```
# History substring search via arrow keys
"\e[A": history-search-backward
"\e[B": history-search-forward
```

## Usage

Type the beginning of a command, then press **↑** / **↓** to cycle
through history entries that start with (or contain, depending on
Readline version) that prefix.

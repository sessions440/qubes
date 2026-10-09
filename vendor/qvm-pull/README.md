# Safe Text & File Transfer Into dom0 (`qvm-pull`)

## Goal
The primary goal is to securely copy text, configuration snippets, logs, or file contents from an AppVM directly into `dom0` without exposing `dom0` to the risks of unrestricted cross-domain clipboards.

## The Idiomatic Solution: `qvm-run`
In Qubes OS, `dom0` is intentionally isolated from AppVM clipboards. The standard, secure way to bridge this gap without weakening isolation is an explicit, opt-in data pull initiated from `dom0`. 

The underlying tool for this is `qvm-run` with the `--pass-io` flag:
```bash
qvm-run --pass-io <source-appvm> 'cat /path/to/file' > /path/to/destination_in_dom0
```
This command runs a command inside the AppVM and streams its standard output straight back into `dom0`.

## The Convenience Wrapper: `qvm-pull`
While `qvm-run --pass-io` is powerful, typing out the full command repeatedly is tedious. The `qvm-pull` script acts as a lightweight wrapper to automate argument parsing, handle destination directory checks, and simplify daily usage.

### The Script
Save the following script as `qvm-pull`:

```bash
#!/bin/bash
set -euo pipefail

# Check for required arguments
if [ "$#" -lt 2 ]; then
    echo "Usage: $(basename "$0") <source-appvm> <source-path> [destination-path]"
    echo "Example: $(basename "$0") work-vm /home/user/snippets.txt ~/Desktop/"
    exit 1
fi

SOURCE_VM="$1"
SOURCE_PATH="$2"
DEST_PATH="${3:-.}"

# If the specified destination is a directory, append the base filename
if [ -d "$DEST_PATH" ]; then
    DEST_PATH="$DEST_PATH/$(basename "$SOURCE_PATH")"
fi

# Pull the file contents via pass-io from dom0
qvm-run --pass-io "$SOURCE_VM" "cat '$SOURCE_PATH'" > "$DEST_PATH"
echo "Successfully pulled '$SOURCE_PATH' from '$SOURCE_VM' to '$DEST_PATH'"
```

## Installation & Persistence in `dom0`
Unlike AppVMs, `dom0`'s file system changes are persistent across system reboots. To make the script globally executable from any terminal in `dom0`, place it in a directory included in your `PATH`, such as `/usr/local/bin`.

Run the following commands in a **`dom0` terminal**:

1. **Create and open the file:**
   ```bash
   sudo nano /usr/local/bin/qvm-pull
   ```
2. **Paste the script contents**, then save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).
3. **Make the script executable:**
   ```bash
   sudo chmod +x /usr/local/bin/qvm-pull
   ```

## Usage
Once installed, you can call `qvm-pull` from anywhere in `dom0`:

```bash
qvm-pull <source-appvm> <source-path> [destination-path]
```

* **Example 1 (Pulling to a specific path):**
  ```bash
  qvm-pull personal-vm /home/user/notes.txt ~/Desktop/notes.txt
  ```
* **Example 2 (Using destination directory shorthand—defaults to current directory):**
  ```bash
  qvm-pull work-vm /etc/hosts ~/work_hosts_backup
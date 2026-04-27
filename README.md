# Slackware multisbo

A bash wrapper for `sbopkg` that recursively resolves and installs SlackBuild packages and their dependencies from SlackBuilds.org.

## Overview

`multisbo` automates the installation of a target SlackBuild package and all its required dependencies as defined in the `.info` file’s `REQUIRES` field. It handles dependency trees, optional dependencies, and several edge cases that the standard `sbopkg` tool does not cover automatically.

## Features

- Recursive dependency resolution with cycle detection.
- Respects the `REQUIRES` field of each `.info` file.
- Optional dependencies can be included with `--with-optional`.
- Non‑interactive operation by default (using `sbopkg -B -k -e continue`); interactive mode available via `--interactive`.
- Caches all `.info` file locations after the first run to speed up subsequent invocations.
- Pre‑downloads all source tarballs (handles multi‑file `DOWNLOAD` entries).
- Falls back to a direct `SlackBuild` + `installpkg` when `sbopkg` fails (e.g., due to broken downloads or MD5 order issues).
- Logs all actions to `/tmp/multisbo.log`.
- Package detection: checks `/var/log/packages/` before attempting installation.

## Requirements

- Slackware Linux (tested on 15.0 and -current).
- `sbopkg` (installed and configured with a synced SBo repository).
- `sudo` privileges to run `sbopkg` and `installpkg`.
- Standard shell utilities: `grep`, `sed`, `tr`, `find`, `ls`, `wget`, `installpkg`, `bc`.

## Installation

1. Install `sbopkg`:
     ```bash
     wget https://github.com/sbopkg/sbopkg/releases/download/0.38.3/sbopkg-0.38.3-noarch-1_wsr.tgz
     sudo installpkg sbopkg-0.38.3-noarch-1_wsr.tgz
     ```
2. Download the script
   ```bash
   wget https://raw.githubusercontent.com/annemedia/mutlisbo/refs/heads/main/mutlisbo
   ```
3. Make it executable:
   ```bash
   chmod +x multisbo
   ```
4. Move it to bin (e.g., `/usr/sbin/`, `~/bin/` or `/usr/local/bin/`).
   ```bash
   sudo mv mutlisbo /usr/sbin/
   ``` 
5. Ensure `sbopkg` is synced:
   ```bash
   sudo sbopkg -r
   ```

## Usage

```
multisbo [OPTIONS] <package_name>
```

### Options

| Option | Description |
|--------|-------------|
| `--with-optional`   | Also install dependencies marked as optional in the `.info` file. |
| `--interactive`     | Run `sbopkg` in interactive mode (without `-B -k -e continue`). |
| `-h`, `--help`      | Display help and exit. |

### Examples

Install `letsencrypt` with all required (non‑optional) dependencies:
```bash
multisbo letsencrypt
```

Install `letsencrypt` including optional dependencies, interactively:
```bash
multisbo --with-optional --interactive letsencrypt
```

## Behaviour

1. Determines the Slackware version from `/etc/slackware-version` (fallback: 15.0).
2. Builds or loads a persistent cache of all `.info` file locations (stored in `~/.cache/multisbo/`).
3. Recursively examines the target package’s `REQUIRES` field.
4. For each package not already installed:
   - Downloads all source files defined in `DOWNLOAD` (and `DOWNLOAD_x86_64`, etc.) into `/var/cache/sbopkg/`.
   - Executes `sbopkg -B -k -e continue -i <package>`.
   - If `sbopkg` succeeds (exit 0) or fails with exit 1 but the package appears in `/var/log/packages/`, the package is considered installed.
   - If `sbopkg` fails and the package is not installed, the script falls back to a direct build: it runs the `SlackBuild` script and installs the resulting package with `installpkg`.
5. Records any failures and presents a summary at the end.

## Limitations

- `sbopkg` itself contains bugs with multi‑line `DOWNLOAD` and swapped MD5 checksums; `multisbo` works around these via the direct build fallback.
- The persistent cache is **not** automatically updated when the SBo repository changes (e.g., after `sbopkg -r`). If you sync the repository, delete the cache file (`~/.cache/multisbo/<version>.cache`) to force a rebuild.
- If a circular dependency is detected (e.g., package A requires B, and B requires A), the script aborts with an error message. It does not attempt to break the cycle automatically.

## Logging

All operations are logged to `/tmp/multisbo.log`. The log includes timestamps, executed commands, and all output from `sbopkg`, `wget`, and SlackBuild scripts.

## License

This script is released under the CC0 1.0 Universal. See the `LICENSE` file for details.

## Collab

`multisbo` was created to make SlackBuilds.org dependency handling more accessible to end users. It works alongside `sbopkg`, automating the recursive installation of required packages and providing a fallback build mechanism for edge cases where `sbopkg` alone would need manual intervention. Contributions and issue reports are welcome.

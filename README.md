# Slackware multisbo – recursive dependency resolver and source builder for SlackBuilds

`multisbo` is a Bash script that resolves and builds SlackBuild packages from **SlackBuilds.org** and **AlienBob** repositories, including recursive `REQUIRES` dependencies. It maintains its own local clones of the upstream repositories, caches `.info` files and source tarballs, and provides optional ELF‑level reverse dependency analysis for installed packages.

**Scope:** `multisbo` manages only third‑party packages from SBo and AlienBob. It does not interact with the Slackware base system (`a/`, `ap/`, `l/`, `n/`, etc.), which remains under Slackware’s native `pkgtools`. No part of the system is converted to source‑based management; `multisbo` coexists with binary package installation.

## Core capabilities

- **Recursive dependency resolution** – parses the `REQUIRES` field of `.info` files, builds a directed graph, and detects cycles.
- **Local repository management** – clones the SBo Git repository (shallow) and optionally syncs AlienBob build metadata via `rsync`.
- **Persistent cache** – stores parsed `.info` file paths and synthetic AlienBob info entries; rebuilds automatically when upstream files change.
- **Source handling** – downloads tarballs into a shared cache directory, verifies MD5 checksums against `.info` fields, supports multi‑file `DOWNLOAD` entries.
- **Direct build** – Executes the SlackBuild script and installs the resulting package with `installpkg` / `upgradepkg`.
- **Native optimisation injection** – adds `-march=native -mtune=native` to SlackBuild's `SLKCFLAGS` where possible (disabled with `-dn`).
- **Automated CMake policy version injection** – for outdated SlackBuilds that fail with modern CMake, injects `export CMAKE_POLICY_VERSION_MINIMUM=3.5` (override with option `-ecf[=VER]`).
- **ELF soname caching** – Scans installed packages (full scan once, then incremental updates) with readelf to record soname provides (`DT_SONAME`) and needs (`DT_NEEDED`).
- **Reverse dependency display** – `-d <pkg>` shows packages that depend on `<pkg>` either at build time (via `REQUIRES`) or at runtime (via ELF soname linkage).
- **Orphaned package detection** – `-r <pkg>` computes which runtime dependencies would become unused if `<pkg>` were removed, and optionally removes them.
- **Update checking** – `-c` compares installed package versions (from `/var/log/packages`) against the latest available versions in the local SBo/Alien metadata.
- **Upgrade mode** – `-u` reinstalls the target package and any outdated dependencies (or newly required ones) in correct build order.
- **Interactive build** – `-i` displays `README` or `slack-desc` before each build, allows editing the SlackBuild script (including setting `ALTERNATIVE_SOURCES`), and offers to skip or abort. When setting ALTERNATIVE_SOURCES you must manually update the VERSION variable in the SlackBuild to match the alternative tarball.

## Requirements

- Slackware Linux (15.0 or -current; older versions should work but are untested)
- `sudo` (for `installpkg`, `upgradepkg`, `removepkg`)
- `git` (to clone the SBo repository)
- `rsync` (optional; required only for `--sync-alien`)
- Standard POSIX tools: `bash`, `bc`, `grep`, `sed`, `tr`, `find`, `ls`, `wget`, `file`, `readelf`, `md5sum`, `installpkg`, `upgradepkg`, `removepkg`

## Installation

1. Download the script:
   ```bash
   wget https://github.com/annemedia/multisbo/releases/download/v2.0/multisbo
   ```
2. Mark executable and place in `PATH`:
   ```bash
   chmod +x multisbo
   sudo mv multisbo /usr/sbin/
   ```

The script stores its working directories under $HOME/.cache/multisbo/. Because multisbo must be run as root or with sudo to install packages, $HOME becomes /root; consequently the cache resides in /root/.cache/multisbo/.

## Command line reference

```
multisbo [OPTIONS] <package_name>
```

### Options

| Option | Description |
|--------|-------------|
| `-dn`, `--disable-native` | Do not add `-march=native -mtune=native` to `SLKCFLAGS`. |
| `-ecf[=VER]`, `--enable-cmake-fix[=VER]` | Inject `export CMAKE_POLICY_VERSION_MINIMUM=VER` (default 3.5) into SlackBuilds that invoke CMake. |
| `-c`, `--check-updates` | Check for newer versions of `<package_name>` and its dependencies; no installation. |
| `-u`, `--update` | Update `<package_name>` and all outdated dependencies to latest available versions. |
| `-i`, `--interactive` | Before building each package, show README/slack-desc, allow editing the SlackBuild, and prompt to continue/skip/abort. |
| `-a`, `--alien` | Prefer AlienBob’s SlackBuild over SBo when both provide the same package name. |
| `-fs`, `--force-single` | Ignore dependency resolution; build only the named package. |
| `-s [branch]`, `--sync-sbo[=branch]` | Clone or update the SBo Git repository (branch defaults to Slackware version, e.g., `15.0`). |
| `--sync-alien` | Sync AlienBob build metadata via `rsync`. |
| `-d <pkg>`, `--dependents <pkg>` | Show installed packages that depend on `<pkg>` (build‑time `REQUIRES` + runtime ELF linkage). |
| `-r <pkg>`, `--remove <pkg>` | Analyse runtime reverse dependencies of `<pkg>`, display breakage candidates and orphans, and optionally remove `<pkg>` together with orphaned packages. |
| `-h`, `--help` | Print usage summary. |

### Behaviour notes

- If the local SBo repository does not exist, `multisbo` clones it automatically (shallow clone, default branch = `$SLACKWARE_VERSION`).  
- The AlienBob mirror is not synced automatically. Use --sync-alien to sync explicitly. However, if a package is missing from SBo and the mirror is absent, the script will offer to sync it (requires confirmation).
- The ELF dependency cache is built incrementally. Full scan happens on first -d or -r request, an incremental scan occurs only when the set of installed packages changes or when timestamps indicate modifications.
- `-r` operates **only on runtime ELF dependencies**. Build‑time (`REQUIRES`) dependencies are not considered candidates for removal.
- `-c` and `-u` compare package versions by parsing the `VERSION` and `BUILD` fields from the `.info` file. The version comparison uses whatever SBo/Alien metadata is already present locally. For accurate results, run -s (and --sync-alien if needed) before -c or -u.
- When `-u` finds outdated packages, the script lists them and prompts with options to update all, update interactively (per‑package), or cancel. Interactive per‑package mode allows building, editing the SlackBuild, skipping, or quitting.

## Examples

**Interactive installation** (review README, edit SlackBuild per package):
```bash
multisbo -i <pkg>
```

**Non-interactive installation (wild run) - Install a package with its dependencies or fail** (auto‑clones SBo if needed, prompts for Alien if needed):
```bash
multisbo <pkg>
```

**Sync the SBo repository** (useful before running updates):
```bash
multisbo -s
```

**Sync AlienBob** (useful before running updates):
```bash
multisbo --sync-alien
```

**Install preferring AlienBob, with CMake fix, no native optimisations**:
```bash
multisbo -a -ecf=3.10 -dn <pkg>
```

**Check for updates and its dependencies**:
```bash
multisbo -c <pkg>
```

**Upgrade and all outdated dependencies** (interactive):
```bash
multisbo -u <pkg>
```

**Show reverse dependencies** (both build‑time and runtime):
```bash
multisbo -d <pkg>
```

**Analyse removal** (show breakage, then prompt):
```bash
multisbo -r <pkg>
```

**Force‑install a single package, ignoring dependencies**:
```bash
multisbo -fs <pkg>
```

## Directory structure

All data lives under `/root/.cache/multisbo/`:

| Path | Content |
|------|---------|
| `SBo/` | Shallow clone of `https://gitlab.com/SlackBuilds.org/slackbuilds.git` |
| `alien/` | Rsync mirror of `rsync://slackware.nl/mirrors/people/alien/slackbuilds/` (only build scripts, not full packages) |
| `sources/` | Source tarballs (shared across builds; symlinked into build directories) |
| `info_cache/` | Persistent cache of `.info` file paths and synthetic AlienBob `.info` files |
| `info_cache/elf_deps.txt` | Flat file mapping packages to soname provides/needs |
| `info_cache/elf_ts.cache` | Timestamps for ELF cache validation |
| `info_cache/elf_fingerprint` | MD5 fingerprint of `/var/log/packages` directory state |

## Limitations

- **No slotting** – multiple versions of the same library or application cannot coexist. Package upgrades replace the previous version directly.
- **No preserved library handling** – if a library updates its soname, dependents built against the old version will break. `multisbo` does not automatically rebuild dependents in this scenario.
- **ELF scanning** – only static `DT_NEEDED` entries from native ELF binaries/libraries are tracked. Does **not** detect:
  - `dlopen()`-ed modules (plugins)
  - Script interpreters (Python, Perl, Ruby, etc.)
  - Dynamic linker environment overrides (`LD_LIBRARY_PATH`)
- **No equivalent of Gentoo's `emerge --depclean`** – the `-r` option only offers to remove packages that are **exclusive** runtime orphans of the target. It does not scan the entire system for unused packages.
- **The ELF cache scan processes installed packages sequentially, not in parallel**. Within each package, file scanning uses only one thread by default (a hardcoded parallel_jobs=1). While increasing that value would parallelize intra‑package scans, the overall scan remains serial across packages and is slower than a fully parallelized implementation.
- **Version comparison for -c (check updates) and -u (update) is purely string‑based:** any difference between the installed version and the repo version triggers an "update available" message, even if the installed version is actually newer. The script does not perform semantic version comparison, epoch handling, or build number prioritization. Users must visually verify version numbers before proceeding.
- **ALTERNATIVE_SOURCES** - version detection from alternative source tarballs is heuristic. The script scans the downloaded filename for the first occurrence of X.Y or X.Y.Z. If no such pattern is found, it falls back to the VERSION from the original .info file. If the fallback does not match the actual version inside the tarball, the build will fail because the symlink name (and therefore the source directory) will be incorrect. Users setting ALTERNATIVE_SOURCES should verify that the downloaded tarball name contains a parsable version string, or manually adjust the symlink after download.

## Logging

All operations are logged to `/tmp/multisbo.log`. The log includes timestamps, command outputs, and error messages.

## License

`multisbo` is released under **CC0 1.0 Universal** (Public Domain). See the `LICENSE` file in the repository.

## Contributing

Issues, feature requests, and pull requests are welcome. When reporting problems, include:

- The exact command used and its output
- The relevant section of `/tmp/multisbo.log`
- The output of `cat /etc/slackware-version`

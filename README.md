# shell-discouraged-mispackager

## Overview

The **shell-discouraged-mispackager** installer is a sophisticated Bash-based package and dependency manager. It installs libraries and packages primarily from GitHub repositories (or direct ZIP archives) based on a declarative `packages.sources` file.

### Key Features

- **Declarative sources**: Uses `packages.sources` (and optional `packages.pinned.sources` for reproducibility) in a custom "sourcetree" format.
- **GitHub resolution**: Automatically resolves to the latest release tag when possible, or falls back to a specified/primary branch (default: `master`). Supports explicit tags, branches, commits, and direct ZIPs.
- **Nested dependencies**: Packages can declare their own `packages.sources`; these are processed recursively.
- **Pinning & reproducibility**: Generates and respects a pin file recording exact download URLs, commits, and versions.
- **Integrity**: Optional SHA-256 verification of downloaded archives.
- **Requirements**: Checks for required system programs or apt packages before proceeding.
- **Safe updates**: Backs up previous installations, supports forced reinstalls, and special handling for core/bootstrapped libraries (which may trigger a re-source "reboot").
- **Post-install handlers**: Sources and runs handler scripts from `packages.control/<project>` after installation.
- **Dynamic paths**: Produces a `package-paths.export` file exporting library path variables.
- **Conflict detection**: Detects version conflicts across the dependency tree.
- **Cleanup**: Archives downloaded ZIPs and previous installations; removes orphaned packages on explicit updates.

It depends on the external **ex10dead-common-library** (for logging, path helpers, confirmation prompts, etc.) and **sourcetree-datafile-format** (for parsing sources files). The `bootstrap_install` script handles the initial installation of these core components.

Typical usage is via an example wrapper (`examples/install`) that sets up the installation root, bootstraps the necessary components with pinned SHA-256s, and invokes the main installer.

## Usage

```bash
# Typical entry point (see examples/install)
./install [OPTIONS]

# Or, after bootstrapping, source the scripts and call:
install_package [OPTIONS]
```

The main entry point is the `install_package` function (defined in `bin/install-sourced-packages`). The example script passes through most options and specially handles core library updates (exit code 12).

## Options

The following options are recognized by `install_package`. They can be given in any order. Unknown options cause an error.

### General

| Option | Description |
|--------|-------------|
| `--debug` | Enable debug mode. Prints the parsed dependency tree and additional diagnostic information. |
| `--verbose` | Enable verbose logging. Shows detailed resolution and decision steps. |
| `--quiet` | Reduce logging output (suppresses info-level messages where supported by the common library). |
| `--print`, `--print-sources` | Parse the sources file(s) and print the dependency tree, then exit. Useful for inspecting what would be installed. |
| `--installation-root PATH` | Set the root directory for installations (default is usually configured in the calling script, e.g. `packages/`). Must contain a `FLAG_IS_MISPACKAGED` marker file. Can also be given as `--installation-root=PATH`. |
| `--` | End of options. Remaining arguments are ignored. |

### Installation & Update Behavior

| Option | Description |
|--------|-------------|
| `--force-install-all` | Force reinstallation of all packages even if they appear already installed and unchanged. Also treated as an update trigger by the example wrapper. |
| `--force-download-newer`, `--update` | Prefer the latest available release (or newer master commit for core libraries). For non-pinned packages this selects newer versions. Also enables checking for core-library updates. |
| `--update-pinned` | Like `--update`, but also updates packages that are explicitly pinned to a tag or branch when a newer release is available. |
| `--update-to-any-depth` | Apply update logic (`--force-download-newer`) not only to top-level packages but to nested dependencies at any depth. |

### Notes on Update Semantics

- Without an explicit update flag, the installer prefers the pin file (`packages.pinned.sources`) if it exists, ensuring reproducible installs.
- Core/bootstrapped libraries (installed via `bootstrap_install`) are updated specially. A change may cause the script to exit with code 12 (`EXIT_REBOOT`), signaling that the caller should re-source the updated libraries and re-invoke the installer.
- Explicit updates can remove packages that are no longer listed in any sources file (orphans are backed up first). Bootstrapped packages are never auto-removed.
- Version conflicts across the tree cause an immediate failure.

## Files and Directories

| Path (relative to installation root) | Purpose |
|--------------------------------------|---------|
| `packages.sources` | Declarative list of packages and requirements. |
| `packages.pinned.sources` | Generated pin file for reproducible installs. |
| `package-paths.export` | Generated file exporting `__<project>_library_path` variables. |
| `bootstrapped.dynamic` | Temporary file recording paths of bootstrapped libraries. |
| `FLAG_IS_MISPACKAGED` | Marker file required in the installation root. |
| `archive/downloads/` | Archived ZIP files of installed packages. |
| `archive/installations/` | Backups of previous installation directories. |
| `library/<project>/` | Default location for installed packages (configurable per entry). |
| `core-library/` | Location for bootstrapped core libraries. |
| `<package>/packages.control/<name>` | Optional post-install handler script. |
| `<package>/installation-settings.markdown` | Metadata written for each installed package. |

## Example packages.sources Snippet

```
# _type=package
# name=ex10dead_common_library
https://github.com/freywald/ex10dead-common-library

# _type=package
# name=desynchronizer
https://github.com/freywald/desynchronizer

# _type=requirement
# apt=curl

# _type=requirement
# program=sha256sum
# apt=coreutils
```

Metadata lines (`# key=value`) can specify `tag`, `branch`, `sha256`, `installation_path`, etc.

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success. |
| 1 | General failure (missing requirement, download error, conflict, handler failure, etc.). |
| 12 | Core libraries were updated; re-source and re-run the installer (handled automatically by the example wrapper). |
| 130 / 143 | Interrupted by SIGINT / SIGTERM (partial installs are cleaned up via traps). |

## Limitations and Notes

- Primarily targets GitHub. Direct ZIP URLs are supported but have reduced features.
- Assumes a Unix-like environment with `curl`, `unzip`, `git`, `sha256sum`, and `python3`.
- The primary branch is hardcoded as `master`.
- No built-in locking against concurrent runs.
- Heavy reliance on the external common library for many helper functions.

For a concrete working example, see the `examples/` directory in the repository, including `install` and `install.configuration`.

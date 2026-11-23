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
- **Dynamic paths**: Produces a `package-paths.export` file exporting library path variables (with commit/version comments for both normal and bootstrapped packages).
- **Conflict detection**: Detects version conflicts across the dependency tree.
- **Cleanup**: Archives downloaded ZIPs and previous installations; removes orphaned packages on explicit updates.
- **Incomplete bootstrap recovery**: Detects partial bootstrap directories (missing settings or only a lone script) and archives them before re-installing from the full archive.

It depends on the external **ex10dead-common-library** (for logging, path helpers, confirmation prompts, etc.) and **sourcetree-datafile-format** (for parsing sources files). The `bootstrap_inject_package` script handles the initial installation of these core components.

Typical usage is via an example wrapper (`examples/install`) that sets up the installation root, bootstraps the necessary components with pinned SHA-256s, and invokes the main installer.

## Usage

```bash
# Typical entry point (see examples/install)
./install [OPTIONS]

# Or, after bootstrapping, source the scripts and call:
install_package [OPTIONS]
```

The main entry point is the `install_package` function (defined in `bin/install-discouraged-packages`). The example script passes through most options and specially handles core library updates (exit code 12).

## Options

The following options are recognized by `install_package`. They can be given in any order. Unknown options cause an error.

## Usage

```bash
./install [OPTIONS]          # see examples/install
# or, after bootstrapping:
install_package [OPTIONS]
```

Main entry point: `install_package` in `bin/install-discouraged-packages`. Exit code 12 means core packages were updated; re-source and re-run.

## Options

| Option | Description |
|--------|-------------|
| `--debug` / `--verbose` / `--quiet` | Logging level. |
| `--print`, `--print-sources` | Parse and print the dependency tree, then exit. |
| `--installation-root PATH` | Installation root (must contain `FLAG_IS_MISPACKAGED`). |
| `--force-install-all` | Reinstall everything. |
| `--force-download-newer`, `--update` | Prefer newer releases / master commits. |
| `--update-pinned` | Also update explicitly pinned packages. |
| `--update-to-any-depth` | Apply updates to nested dependencies too. |

Without an update flag the pin file is preferred. Core/bootstrapped packages are updated specially (exit 12). Explicit updates remove orphans (backed up first). Incomplete bootstrap directories are archived and re-installed.

## Files (relative to installation root)

| Path | Purpose |
|------|---------|
| `packages.sources` | Declarative package and requirement list. |
| `packages.pinned.sources` | Generated pin file (exact URLs and commits). |
| `package-paths.export` | Generated `__<project>_package_path` variables (with commit/version comments). |
| `bootstrapped.dynamic` | Temporary paths of bootstrapped packages. |
| `FLAG_IS_MISPACKAGED` | Required marker. |
| `archive/downloads/`, `archive/installations/` | Archived ZIPs and previous/incomplete installs. |
| `user/<project>/` | Default location for ordinary packages. |
| `core-packages/` | Bootstrapped core packages. |
| `<package>/packages.control/<name>` | Optional post-install handler. |
| `<package>/installation-settings.markdown` | Per-package metadata. |

## Example packages.sources

```
# name=ex10dead_common_library
# _type=package
https://github.com/freywald/ex10dead-common-library

# _type=requirement
# apt=curl
```

Metadata lines (`# key=value`) can set `tag`, `branch`, `sha256`, `installation_path`, etc.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success. |
| 1 | Failure (missing requirement, download error, conflict, handler failure, …). |
| 12 | Core packages updated; re-source and re-run. |
| 130 / 143 | Interrupted (partial installs cleaned up). |

## Notes

- Primarily targets GitHub. Requires `curl`, `unzip`, `git`, `sha256sum`, `python3`.
- Primary branch is `master`. No concurrent-run locking.
- See `examples/` for a working wrapper and configuration.

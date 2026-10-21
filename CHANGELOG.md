# Changelog

All notable changes to ccdev. Versions follow [Semantic Versioning](https://semver.org/); each release is a git tag (`vX.Y.Z`).

## [1.1.1] - 2026-10-22

### Changed
- `--new` is the documented name again for starting a new Claude conversation; `--fresh` (1.1.0) keeps working as an alias.

## [1.1.0] - 2026-10-22

### Added
- `--ssh` and `[settings] transport = auto|mosh|ssh`: use ssh even when mosh is installed, e.g. when UDP is blocked.
- `--pick`: choose a conversation from Claude's list (`claude --resume`).
- `--dry-run`: resolve target, user, transport, project and Claude command, print them, contact nothing.
- `[settings] screenshot_dir` and `screenshots = on|off`.

### Changed
- Claude continues the last conversation everywhere (`claude --continue`, plain `claude` without history). Starting a single directory used `claude --resume`, which opened Claude's picker.
- `--fresh` (formerly `--new`) now also applies to projects started from the menu and to `--restore`; before it only affected a single directory.
- Renamed, the old names still work: `--new` → `--fresh`, `-s`/`--server` → `--dev`, `--sync-auth` → `--copy-login`.
- `-d`/`--dir` and `-S` are still accepted but no longer documented (use the second argument and `--screenshot-dir`).
- Precedence for the screenshot dir: `--screenshot-dir`, then `$CCDEV_SCREENSHOT_DIR`, then `[settings] screenshot_dir`.

## [1.0.2] - 2026-10-22

### Changed
- README: the Concepts section explains what mosh brings (roaming, surviving sleep, local echo, reconnecting) and how it complements byobu.

## [1.0.1] - 2026-10-21

### Changed
- README and docs/inventory.md use neutral example names instead of the author's own hosts and projects.

## [1.0.0] - 2026-10-21

First tagged release. Earlier development is in the git history.

### Added
- Host inventory: named targets from `static` lists, `~/.ssh/config` aliases, Proxmox containers and VMs with a tag (`proxmox-ssh`) and any `command`; menu when no host is given, names and unique prefixes on the command line, `SOURCE/NAME`.
- `ccdev --init`, an interactive setup for the inventory.
- Inventory cache with background refresh, `--refresh`, `--list [table|tsv|json]`.
- Bash tab completion: `ccdev --completion bash`.
- `--mode MODE` and `[settings] permission_mode` to start Claude with `--permission-mode`.
- Project discovery on the host (`CLAUDE.md` or `.ccdev`), `--restore`, `--base` and `CCDEV_BASE` (relative to the remote `$HOME`).
- `--version`.
- Help in two levels: `-h` (one screen), `--help` (grouped options, examples and "Your setup"), and topics `--help inventory|projects|env`. Unknown options and missing values give a short error instead of the whole help.

### Changed
- All output and prompts are in English. Prompts take `y` (and still `j`).
- Without a user, ssh picks it (`~/.ssh/config`, else the local user) instead of ccdev forcing `$USER`.
- The `/ss` slash command text on the host is in English.
- Dates in the session menu use ISO format (`2026-10-21`).
- `--sync-auth` copies only `~/.claude/.credentials.json` (mode 600), no longer `settings.json`, which overwrote the host's settings.

### Fixed
- mosh is no longer required locally; ccdev falls back to ssh as documented.

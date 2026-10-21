# Changelog

All notable changes to ccdev. Versions follow [Semantic Versioning](https://semver.org/); each release is a git tag (`vX.Y.Z`).

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

# ccdev

Run Claude Code on your servers, not on your laptop, and get back to it with one command.

ccdev connects to a remote host over mosh (or ssh), keeps every project in its own byobu session and starts or resumes Claude Code there. Close the lid, switch networks, reboot your laptop: Claude keeps working on the server, and `ccdev` puts you right back into the session. With a host inventory you pick machines by name (from a list, your `~/.ssh/config`, Proxmox tags or any command) instead of remembering addresses.

```
$ ccdev
Targets:
  proxmox
    [1] api-dev            root@192.168.1.21        CT101 pve1
    [2] web-dev            root@192.168.1.22        CT102 pve1
    [3] ml-box             root@192.168.1.30        VM200 pve2
  servers
    [4] web1               deploy@203.0.113.10      Hetzner
Choice: 2

Running byobu sessions:
──────────────────────────────────────
  [1] shop: 1 windows (created Tue Oct 21 09:12:40 2026)
  [2] blog: 1 windows (created Tue Oct 21 09:12:41 2026)
Saved projects (not started) in /root/projects:
──────────────────────────────────────
  [3] api            /root/projects/api, last used 2026-10-20
  [4] docs           /root/projects/docs, no history yet
──────────────────────────────────────
  [r] Start all saved projects
  [n] New session
  [q] Quit
```

## Contents

- [Quick start](#quick-start)
- [Concepts](#concepts)
- [Install](#install)
- [Guides](#guides): projects and sessions, host inventory, screenshots, dev server, permission mode
- [Reference](#reference): options, environment, files
- [Security notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Requirements](#requirements)

## Quick start

```bash
# 1. Install (see Install for other ways)
sudo curl -fsSL https://raw.githubusercontent.com/efnats/ccdev/v2.0.0/ccdev -o /usr/local/bin/ccdev
sudo chmod +x /usr/local/bin/ccdev

# 2. Connect. ccdev checks the remote and offers to install byobu, mosh and claude if missing.
ccdev root@192.168.1.10

# 3. Work on a project: its own byobu session, Claude resumes where it left off
ccdev root@192.168.1.10 /opt/myproject
```

Optional: `ccdev --init` sets up a host inventory so that `ccdev` alone shows your machines and `ccdev myhost` connects by name.

Help: `ccdev -h` for the essentials, `ccdev --help` for all options and your current setup, `ccdev --help inventory|projects|env` for details.

## Concepts

- **Host**: the machine Claude Code runs on, given as `user@host`, an ssh alias, or a name from the [host inventory](#host-inventory).
- **Connection**: mosh if it is installed locally, otherwise ssh (or always ssh with `--ssh`). mosh is made for unreliable links:
  - **Roaming**: the connection survives a changing IP address, e.g. moving from Wi-Fi to a phone hotspot or VPN.
  - **Sleep**: close the laptop, open it hours later, and the terminal is still connected; there is no ssh timeout to run into.
  - **Bad networks**: keystrokes are echoed locally right away, so typing stays responsive on slow or lossy links (trains, hotel Wi-Fi, mobile data).
  - **No hanging terminal**: if the network drops, mosh says so and reconnects by itself instead of freezing.

  mosh needs `mosh-server` on the host (ccdev offers to install it) and UDP ports 60000-61000 open to it. It has no scrollback of its own; byobu provides that.
- **Session**: a byobu (tmux) session on the host. It survives disconnects; ccdev attaches to it or creates it. mosh keeps the *connection* alive, byobu keeps the *work* alive: even if your laptop reboots or mosh gives up, Claude keeps running in the session and `ccdev` brings you back to it.
- **Project**: a directory on the host with a `CLAUDE.md` or `.ccdev` file. ccdev lists projects that are not running yet and starts each in its own session, named after the directory, resuming the last Claude conversation there.

## Install

```bash
# system-wide
sudo curl -fsSL https://raw.githubusercontent.com/efnats/ccdev/v2.0.0/ccdev -o /usr/local/bin/ccdev
sudo chmod +x /usr/local/bin/ccdev

# or just for you (make sure ~/.local/bin is in your PATH)
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/efnats/ccdev/v2.0.0/ccdev -o ~/.local/bin/ccdev
chmod +x ~/.local/bin/ccdev

ccdev --version
```

Releases are git tags (`v1.0.0`, …), see [CHANGELOG.md](CHANGELOG.md). To **update**, run the same command with the new tag; `master` is the development branch. To **uninstall**, delete the script and, if you used them, `~/.config/ccdev/` and `~/.cache/ccdev/` (see [Files](#files)).

Tab completion (bash; zsh after `autoload -U bashcompinit && bashcompinit`):

```bash
echo 'eval "$(ccdev --completion bash)"' >> ~/.bashrc
```

## Guides

### Connecting

```bash
ccdev root@192.168.1.10                 # session manager on the host
ccdev -u deploy 192.168.1.10            # user given separately
ccdev myalias                           # ssh alias from ~/.ssh/config (User, Port, … apply)
ccdev --new root@192.168.1.10 /opt/app    # new Claude conversation instead of continuing
ccdev --pick root@192.168.1.10 /opt/app   # choose one of the earlier conversations
ccdev --ssh root@192.168.1.10            # ssh even though mosh is installed (UDP blocked)
ccdev --dry-run myhost /opt/app          # show what would happen, connect to nothing
```

Without a user, ssh decides (your `~/.ssh/config`, else your local user name). mosh is used when it is installed locally, otherwise ccdev falls back to ssh.

### Projects and sessions

Without a directory, ccdev shows the session manager: running byobu sessions, then **saved projects**, the subdirectories of the base dir that contain a `CLAUDE.md` or `.ccdev` and have no running session. Picking a project starts a session named after the directory and runs `claude --continue` there (plain `claude` if the project has no history yet).

```bash
ccdev root@192.168.1.10 /opt/myproject     # one project (session "myproject"), see below
ccdev --restore root@192.168.1.10          # after a reboot: start all saved projects, then the menu
ccdev --base /srv/work root@192.168.1.10   # look for projects in another directory
```

The base dir defaults to the remote `$HOME`. `CCDEV_BASE` sets it for all hosts; a relative value is taken from the remote `$HOME`, so `export CCDEV_BASE=projects` means `/root/projects` for root and `/home/alice/projects` for alice.

When you give a directory (`ccdev host /opt/app`) and no session is running on the host yet, ccdev starts a session for it right away. Claude continues the last conversation in that directory (`claude --continue`, plain `claude` without history); `--new` starts a new one, `--pick` lets you choose. The same applies to projects started from the menu and to `--restore`. If sessions are running, it shows the menu; `[n]` creates the session, named after the directory by default.

### Host inventory

Name your machines once and connect by name:

```bash
ccdev --init           # interactive setup, writes ~/.config/ccdev/config
ccdev                  # menu of all targets
ccdev api              # exact name, else unique prefix (api-dev)
ccdev servers/web1     # SOURCE/NAME: only look in one source
ccdev --list json      # for scripts (also: table, tsv)
```

Targets come from sources, queried in parallel and cached for 5 minutes:

| Source | Targets from |
|---|---|
| `static` | a list in the config file |
| `ssh-config` | Host aliases in `~/.ssh/config`, filtered by pattern or a `# ccdev` marker |
| `proxmox-ssh` | Proxmox containers and VMs with a tag, addresses via `pvesh` over SSH |
| `command` | any command printing `NAME<TAB>TARGET[<TAB>STATUS[<TAB>INFO]]` per line |

A host containing `.`, `@` or `:` always goes to ssh unchanged, and so does a name that matches nothing. Config format, all options, caching and examples (including a Tailscale one-liner): **[docs/inventory.md](docs/inventory.md)**.

### Screenshots for Claude

If `inotifywait` is installed locally, ccdev watches `~/Pictures/Screenshots` (or `--screenshot-dir DIR`, `$CCDEV_SCREENSHOT_DIR`, `[settings] screenshot_dir`) while you are connected and copies every new screenshot to the host: into `screenshots/` in the project directory, or `~/screenshots/` without one. It also installs a `/ss` command for Claude there:

```
/ss              read and describe the latest screenshot
/ss 3            read the 3 most recent screenshots
/ss what error?  read the latest screenshot and answer the question
```

`--no-screenshots` turns this off.

### Dev server next to Claude

Put a `.ccdev` file into the project directory on the host:

```bash
# /opt/myproject/.ccdev
SERVER_CMD="npm run dev"
```

New sessions for that project are split: Claude on the left, the dev server on the right. `--dev CMD` does the same from the command line. If `entr` is missing on the host, ccdev offers to install it (handy for restarting servers on file changes).

### Permission mode

`--mode MODE` starts Claude with `--permission-mode MODE`, for example `auto` (Claude works without asking; risky actions are checked and blocked) or `plan` (read and plan only). Set a default in the config file:

```ini
[settings]
permission_mode = auto
```

Without either, Claude uses the default from `~/.claude/settings.json` on the host (`permissions.defaultMode`). The available modes depend on your Claude Code version (`claude --help`).

## Reference

### Options

| Option | Description |
|---|---|
| `-u, --user USER` | SSH user (overrides `user@` in the host) |
| `--ssh` | Use ssh even if mosh is installed (e.g. when UDP is blocked) |
| `--copy-login` | Copy your local Claude login to a host that has none, see [Security notes](#security-notes) |
| `-n, --name NAME` | byobu session name (default: directory name, else `claude`) |
| `--base DIR` | Where to look for projects on the host (relative to the remote `$HOME` unless absolute) |
| `--restore` | Start sessions for all saved projects, then show the menu |
| `--new` | Start a new Claude conversation instead of continuing the last one |
| `--pick` | Choose a conversation from Claude's list (`claude --resume`) |
| `--mode MODE` | Claude permission mode (`auto`, `plan`, …) |
| `--init` | Set up the host inventory interactively |
| `--list [table\|tsv\|json]` | Print the host inventory and exit |
| `--refresh` | Query the inventory live instead of using the cache |
| `--dev CMD` | Dev server command for a split pane (overrides `.ccdev`) |
| `--screenshot-dir DIR` | Local screenshot directory to watch |
| `--no-screenshots` | Don't sync screenshots |
| `--dry-run` | Resolve target, transport and Claude command, print them, connect to nothing |
| `--completion bash` | Print the bash completion script |
| `--version` | Print the version |
| `-h` | Short help |
| `--help [TOPIC]` | All options and your current setup, or a topic: `inventory`, `projects`, `env` |

Removed in 2.0.0: `-d`/`--dir` (use the second argument), `-s`/`--server` (`--dev`), `-S` (`--screenshot-dir`), `--sync-auth` (`--copy-login`) and `--fresh` (`--new`). ccdev names the replacement if you use one of them.

### Environment

| Variable | Meaning |
|---|---|
| `CCDEV_BASE` | Default for `--base` |
| `CCDEV_CONFIG` | Inventory config file (default `~/.config/ccdev/config`) |
| `CCDEV_SCREENSHOT_DIR` | Default for `--screenshot-dir` |
| `CCDEV_PVE`, `CCDEV_PVE_TAG`, `CCDEV_PVE_USER` | Legacy: define one Proxmox source when there is no config file |
| `XDG_CONFIG_HOME`, `XDG_CACHE_HOME` | Respected for the config and cache locations |

### Files

ccdev keeps these files. Everything else only happens after a prompt you answer with yes (installing tools on the host, adding tab completion locally).

| Where | File | Purpose |
|---|---|---|
| local | `~/.config/ccdev/config` | Host inventory and settings (written by `--init` or you) |
| local | `~/.cache/ccdev/inventory.tsv` | Inventory cache |
| local | `~/.bashrc` | Completion line, only if you agree in `--init` |
| host | `<project>/.claude/commands/ss.md` or `~/.claude/commands/ss.md` | `/ss` command, rewritten on every connect |
| host | `<project>/screenshots/` or `~/screenshots/` | Copied screenshots |
| host | `cc-watcher.log` next to `screenshots/` | Log of copied screenshots |
| host | byobu sessions | One per project or name |
| host | `~/.claude/.credentials.json` | Only with `--copy-login` |
| host | `~/.npm-global/`, PATH line in `~/.bashrc` | Only when ccdev installs claude for you |

## Security notes

- **Installing on the host.** When byobu, mosh or claude are missing, ccdev asks once and then runs `sudo apt-get install`, for Node.js the NodeSource setup script (`curl … | sudo bash`), and `npm install -g @anthropic-ai/claude-code` into `~/.npm-global` (adding it to `PATH` in `~/.bashrc`). Answer no and install yourself if you prefer.
- **`--copy-login`** copies your local login, `~/.claude/.credentials.json`, to a host that has none (mode 600). Your Claude account is then usable on that machine; anyone with root there can read it. Settings are never copied.
- **Inventory sources run commands.** A `command` source executes its `run` line locally; only use config files you wrote. `proxmox-ssh` needs SSH to a Proxmox node that can run `pvesh`, usually as root. ccdev leaves authentication to you; consider a dedicated key restricted to that purpose.
- **Permission mode.** `--mode auto` (or `bypassPermissions`) lets Claude act without asking. Use it on hosts where that is acceptable.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Tab completion does nothing | It only applies to shells started after adding it: open a new terminal or run `exec bash`. |
| Inventory shows old state | It comes from the cache (the menu says so); `ccdev --refresh` queries live. |
| A VM shows `(no IP)`, "no guest agent" | Install and enable the QEMU guest agent in the VM and in its Proxmox options. |
| "no node reachable" for Proxmox | ssh to the node must work without a password prompt (`ssh -o BatchMode=yes root@pve1 true`). |
| mosh hangs at "Connecting" | Allow UDP 60000-61000 to the host, and make sure a UTF-8 locale exists on both sides. Or skip mosh: `ccdev --ssh …`, or `transport = ssh` under `[settings]` in the config. |
| `declare: -A: invalid option` (macOS) | ccdev needs bash 4.4 or newer: `brew install bash`. |
| No screenshot sync | `inotifywait` is missing locally (`apt install inotify-tools`) or the screenshot directory does not exist. |

## Requirements

**Local:** bash 4.4+, ssh, scp. Optional: mosh (else ssh), `inotifywait` from inotify-tools (screenshot sync, Linux only), `column` (`--list` tables), bash-completion (adds known ssh hosts to tab completion).

**Host:** byobu, mosh-server and Claude Code; missing ones can be installed by ccdev on Debian/Ubuntu (apt). Proxmox source: `pvesh` and `python3` on the node (standard on Proxmox VE), the QEMU guest agent for VM addresses.

**Tested on** Ubuntu 24.04/26.04 as client and Debian/Ubuntu hosts. macOS and WSL as clients are untested.

## License

MIT, see [LICENSE](LICENSE).

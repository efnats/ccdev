# ccdev

Connect to a remote Claude Code dev session via mosh/ssh with automatic byobu session management.

## Features

- **mosh/ssh connection** with automatic fallback to ssh if mosh is unavailable
- **Byobu session manager** - attach, create, or kill sessions interactively
- **Auto-resume** - reconnects to your last Claude session (`claude --resume`)
- **Remote dependency check** - detects and offers to install missing tools (byobu, mosh-server, claude)
- **Screenshot sync** - watches a local screenshot directory and copies new files to the remote host via scp
- **`/ss` slash command** - automatically creates a Claude custom command on the remote to read the latest screenshot
- **Auth sync** - optionally transfer local Claude credentials to the remote host
- **Dev server split-pane** - automatically splits the terminal when a `.ccdev` config is found in the project directory
- **Host inventory** - pick targets by name from static lists, `~/.ssh/config`, Proxmox tags or any command

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/efnats/ccdev/master/ccdev -o /usr/local/bin/ccdev && chmod +x /usr/local/bin/ccdev
```

## Usage

```
ccdev [OPTIONS] [user@]host [dir]
ccdev [OPTIONS] [name] [dir]      # with a host inventory
```

### Options

| Option | Description |
|---|---|
| `-u, --user USER` | SSH username (overrides user in user@host) |
| `-d, --dir DIR` | Directory to cd into on the remote before starting Claude |
| `-s, --server CMD` | Dev server command (overrides `.ccdev` config on remote) |
| `--no-screenshots` | Disable automatic screenshot sync |
| `--new` | Start a new Claude session (skip --resume) |
| `--sync-auth` | Sync local Claude auth (~/.claude/) to remote host |
| `--refresh` | Query the host inventory live, ignoring the cache |
| `--list [table\|tsv\|json]` | Print the host inventory and exit |
| `--completion bash` | Print a bash completion script |
| `-h, --help` | Show help |

### Examples

```bash
# Basic connection
ccdev root@192.168.1.10

# With explicit user
ccdev -u root 192.168.1.10

# Connect and cd into a project directory
ccdev root@192.168.1.10 /opt/myproject

# Start a fresh Claude session
ccdev --new root@192.168.1.10

# Sync auth credentials to the remote
ccdev --sync-auth root@192.168.1.10

# Override dev server command from CLI
ccdev root@192.168.1.10 /opt/myproject --server "npm run dev"
```

## Host inventory

Instead of remembering addresses, ccdev can read named targets from one or more sources. Without a host it shows all of them to pick from; with a short name it connects directly:

```bash
ccdev                  # menu of all targets, grouped by source
ccdev seedhodler       # exact name, else unique prefix (e.g. seedhodler-dev)
ccdev proxmox/claude   # SOURCE/NAME when two sources share a name
ccdev seedhodler /root/claude/seedhodler   # name plus project dir works as with a host
```

A host containing `.`, `@` or `:` always goes to ssh unchanged, and so does a name that matches nothing. Sources are queried in parallel; a failing source only prints a warning.

Configure the sources in `~/.config/ccdev/config` (or the file named by `$CCDEV_CONFIG`):

```ini
# Fixed hosts
[source servers]
type = static
host = web1 deploy@203.0.113.10 Hetzner
host = lab  root@192.168.1.20

# Aliases you already keep in ~/.ssh/config (User/HostName/Port are honoured by ssh)
[source ssh]
type   = ssh-config
match  = dev-* lab-*       # glob(s) on the Host alias, default *
# marker = ccdev           # alternatively: only Host lines ending in "# ccdev"
# file = ~/.ssh/config

# Proxmox: LXC containers with a tag, IPs looked up via pvesh over SSH
[source proxmox]
type  = proxmox-ssh
nodes = root@pve1 root@pve2   # first reachable node is used (any node of a cluster works)
tag   = claude
user  = root

# Anything else: a command printing NAME<TAB>TARGET[<TAB>STATUS[<TAB>INFO]] per line
[source tailnet]
type    = command
run     = tailscale status --json | jq -r '.Peer[] | select(.Tags // [] | index("tag:dev")) | [.HostName, "root@" + .TailscaleIPs[0], (if .Online then "up" else "off" end), .OS] | @tsv'
timeout = 10
```

Comments start with `#` or `;`, also at the end of a line except in `run`. Targets with status `stopped`, `down`, `off` or `offline` are listed but cannot be selected. Without a config file, the environment variables `CCDEV_PVE` (space-separated PVE node SSH targets), `CCDEV_PVE_TAG` and `CCDEV_PVE_USER` still define a single `proxmox-ssh` source.

### Cache, listing and completion

The inventory is cached in `~/.cache/ccdev/inventory.tsv`. While the cache is younger than `cache_ttl` (and not older than the config file), ccdev uses it right away and refreshes it in the background; otherwise it queries the sources live. If a source fails, its last cached entries are shown marked `(aus Cache)`. `--refresh` forces a live query.

```ini
[settings]
cache_ttl = 300   # seconds, 0 = always query live
```

```bash
ccdev --list            # table
ccdev --list json       # for scripts (also: tsv)
```

Tab completion for names (from the cache, no network) and options, plus known SSH hosts when bash-completion is installed:

```bash
echo 'eval "$(ccdev --completion bash)"' >> ~/.bashrc
```

zsh users can load it after `autoload -U bashcompinit && bashcompinit`.

## How it works

1. Checks local dependencies (mosh, ssh, scp)
2. Verifies remote dependencies (byobu, mosh-server, claude) - offers to install if missing
3. Optionally syncs Claude auth credentials (`--sync-auth`)
4. Starts a screenshot watcher (if `inotifywait` is available)
5. Creates a `/ss` slash command on the remote for screenshot access
6. Connects via mosh (or ssh fallback) and presents the byobu session manager

## Screenshot sync

The screenshot watcher monitors `~/Pictures/Screenshots` on your local machine. When a new screenshot appears, it is copied to the remote host into a `screenshots/` directory next to your project (or `~/screenshots/` if no project dir is set).

ccdev automatically creates a `/ss` slash command on the remote. When a project directory is set, the command is placed in `<project>/.claude/commands/ss.md` (project-specific); otherwise in `~/.claude/commands/ss.md` (global). It is updated on every connect.

### Usage inside Claude

```
/ss              → read and describe the latest screenshot
/ss 3            → read the 3 most recent screenshots
/ss what error?  → read the latest screenshot and answer the question
```

## Dev server split-pane

If your project has a dev server, create a `.ccdev` file in the project root on the remote:

```bash
# /opt/myproject/.ccdev
SERVER_CMD="npm run dev"
```

When ccdev connects and creates a new session, it will automatically:
1. Split the byobu window horizontally (left/right)
2. Start Claude Code in the left pane
3. Run the server command in the right pane

You can also override this from the CLI with `--server CMD`.

## Requirements

**Local (client):**
- mosh (optional, falls back to ssh)
- ssh, scp
- inotifywait (optional, for screenshot sync - `apt install inotify-tools`)

**Remote (server):**
- byobu
- mosh-server
- claude (Claude Code CLI)

Missing remote dependencies are detected automatically and can be installed interactively.

## License

MIT

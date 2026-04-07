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

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/efnats/ccdev/master/ccdev -o /usr/local/bin/ccdev && chmod +x /usr/local/bin/ccdev
```

## Usage

```
ccdev [OPTIONS] [user@]host [dir]
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

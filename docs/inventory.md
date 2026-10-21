# Host inventory

The host inventory lets you connect by name instead of address. ccdev reads named targets from one or more **sources**, queries them in parallel, caches the result and shows it as a menu.

```bash
ccdev --init           # interactive setup
ccdev                  # menu of all targets, grouped by source
ccdev api              # exact name, else unique prefix (api-dev)
ccdev servers/web1     # SOURCE/NAME: only look in one source
ccdev web-dev /srv/shop   # a name works like a host, also with a project dir
```

## Name resolution

- No host: the menu shows all targets.
- A host containing `.`, `@` or `:` (an address, `user@host`, IPv6) always goes to ssh unchanged.
- Any other word is matched against the target names: an exact match first, otherwise names starting with it. One hit connects directly, several show a menu of the hits. `SOURCE/NAME` limits the match to one source.
- No match: the word goes to ssh unchanged, so ssh aliases keep working.
- Targets with status `stopped`, `down`, `off` or `offline` are listed as `(off)` and cannot be selected; running guests without an address show `(no IP)` and the reason.

## Setting it up

`ccdev --init` asks for the sources, checks them right away (Proxmox nodes for SSH access, `pvesh` and `python3`; ssh-config patterns against your aliases; commands with a test run), shows the resulting file before saving, and can add tab completion to `~/.bashrc`. An existing config can be appended to or replaced; the old file is backed up with a timestamp.

You can also write the file yourself: `~/.config/ccdev/config` (or `$CCDEV_CONFIG`).

## Config file

One `[source NAME]` block per source; `NAME` is shown in the menu and used for `SOURCE/NAME`. Comments start with `#` or `;`, also at the end of a line, except in `run`.

```ini
[settings]
cache_ttl       = 300    # seconds the cache is used as is (0 = always query live)
permission_mode = auto   # default for --mode

# Fixed hosts: host = NAME TARGET [INFO]
[source servers]
type = static
host = web1 deploy@203.0.113.10 Hetzner
host = lab  root@192.168.1.20

# Aliases from ~/.ssh/config; User, HostName, Port etc. are applied by ssh
[source ssh]
type   = ssh-config
match  = dev-* lab-*     # glob patterns on the alias (default *)
# marker = ccdev         # or: only Host lines ending in "# ccdev"
# file = ~/.ssh/config   # one or more files

# Proxmox containers and VMs with a tag
[source proxmox]
type  = proxmox-ssh
nodes = root@pve1 root@pve2   # first reachable node is used; any node of a cluster sees all guests
tag   = claude
user  = root                  # login user for the guests
# types = lxc qemu            # default both

# Anything else: a command printing NAME<TAB>TARGET[<TAB>STATUS[<TAB>INFO]] per line
[source tailnet]
type    = command
run     = tailscale status --json | jq -r '.Peer[] | select(.Tags // [] | index("tag:dev")) | [.HostName, "root@" + .TailscaleIPs[0], (if .Online then "up" else "off" end), .OS] | @tsv'
timeout = 10                  # seconds (default 20)
```

### static

`host = NAME TARGET [INFO]`, repeatable. `TARGET` is anything ssh accepts: `user@host`, an address or an ssh alias. Names should not contain `.`, `@`, `:` or spaces, otherwise they cannot be used as a name on the command line.

### ssh-config

Lists the `Host` aliases of `~/.ssh/config` (or the files in `file`), skipping wildcard entries. `match` filters by glob, `marker` by a trailing comment such as `Host lab1 # ccdev`. The target is the alias itself, so ssh applies `User`, `HostName`, `Port`, `ProxyJump` and the rest.

### proxmox-ssh

Runs `pvesh` on the first reachable node over SSH (a small `python3` script, both standard on Proxmox VE) and lists the guests carrying `tag`:

- containers (`lxc`): address from the container's network interfaces
- VMs (`qemu`): address from the QEMU guest agent; the agent must be installed in the VM and enabled in its Proxmox options, otherwise the VM shows `(no IP)`, "no guest agent"

The first IPv4 address that is not loopback is used. The info column shows `CT<id> <node>` or `VM<id> <node>`. Authentication to the node is up to you: ssh must work without a password prompt (`ssh -o BatchMode=yes root@pve1 true`).

### command

Runs `run` with bash and reads one target per line: `NAME<TAB>TARGET`, optionally `<TAB>STATUS` and `<TAB>INFO`. Use this for anything ccdev does not know (cloud CLIs, Incus, libvirt, your own lists). The command runs locally with your permissions, so only use config files you trust.

## Cache

The inventory is cached in `~/.cache/ccdev/inventory.tsv`. While the cache is younger than `cache_ttl` and not older than the config file, ccdev uses it immediately (the menu shows "cached … ago") and refreshes it in the background. Otherwise it queries the sources live.

If a source fails, ccdev prints a warning and shows that source's last cached entries marked `(cached)`; the cache keeps its old timestamp so the next run tries live again. `--refresh` always queries live.

## Listing and completion

```bash
ccdev --list           # table: SOURCE NAME TARGET STATUS INFO
ccdev --list tsv       # tab-separated, one target per line
ccdev --list json      # array of {source, name, target, status, info}
```

`eval "$(ccdev --completion bash)"` in `~/.bashrc` completes target names and `SOURCE/NAME` (from the cache, no network), options, and known ssh hosts when bash-completion is installed.

## Without a config file

For compatibility, `CCDEV_PVE` (space-separated SSH targets of Proxmox nodes), `CCDEV_PVE_TAG` and `CCDEV_PVE_USER` define a single `proxmox-ssh` source named `proxmox` when no config file exists.

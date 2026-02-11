# rsync-bidirectional-sync

Robust bidirectional file synchronization for Linux and WSL using rsync with manifest-based change tracking, conflict resolution, and safe deletion propagation.

## How It Works

Unlike simple "pull then push" approaches, this tool uses a **three-way diff** algorithm:

1. **Manifest tracking** - After each sync, a snapshot of all file metadata (path, mtime, size) is saved
2. **Change detection** - On the next run, both local and remote are compared against the previous manifest
3. **Smart classification** - Each file is classified as: new, modified, deleted, or unchanged on each side
4. **Conflict resolution** - Files modified on both sides are handled according to your configured strategy
5. **Safe deletions** - Deletions are only propagated when the file existed in the previous manifest (so new files are never accidentally deleted)

## Quick Start

```bash
# 1. Install
./install.sh

# 2. Configure
nano ~/.config/rsync-sync/config

# 3. Set up SSH keys
ssh-copy-id user@remote-host

# 4. Test
sync-client --dry-run

# 5. Sync
sync-client
```

## Installation

```bash
git clone https://github.com/INS-JVidal/rsync-bidirectional-sync.git rsync-bidirectional-sync
cd rsync-bidirectional-sync
./install.sh
```

The installer:
- Copies scripts to `~/.local/share/rsync-sync/`
- Creates a `sync-client` symlink in `~/.local/bin/`
- Creates configuration directory at `~/.config/rsync-sync/`
- Sets up bash completion
- Adds `~/.local/bin` to PATH if needed

### Requirements

- Bash 4.0+
- rsync (on both local and remote)
- OpenSSH client (with key-based auth recommended)
- Standard Unix tools: find, stat, sort, md5sum

### Uninstall

```bash
./install.sh --uninstall
```

## Configuration

Default config: `~/.config/rsync-sync/config`

```bash
# Remote connection
REMOTE_USER="user"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT=22

# Sync paths
LOCAL_DIR="/home/user/projects"
REMOTE_DIR="/home/user/projects"

# Conflict resolution: newest, skip, backup, local, remote
CONFLICT_STRATEGY="newest"

# Propagate deletions to the other side
PROPAGATE_DELETES=true

# Back up files before overwriting during conflicts
BACKUP_ON_CONFLICT=true
```

See `config.example` for all available options.

### Profiles

Create named profiles for different sync targets:

```bash
# Create profile
cp ~/.config/rsync-sync/config ~/.config/rsync-sync/profiles/work.conf
nano ~/.config/rsync-sync/profiles/work.conf

# Use profile
sync-client -p work sync
```

Each profile maintains its own sync state, lock file, and logs.

## Usage

```bash
# Basic sync
sync-client

# Check what would change
sync-client status

# Preview without making changes
sync-client --dry-run

# Verbose output
sync-client --verbose

# Use a profile
sync-client --profile work

# Reset sync state (next sync = first sync)
sync-client reset-state

# Combine options
sync-client -p work -v -n
```

### Commands

| Command | Description |
|---------|-------------|
| `sync` | Run bidirectional sync (default) |
| `status` | Show pending changes without syncing |
| `reset-state` | Clear manifest, treat next sync as first sync |

### Options

| Option | Description |
|--------|-------------|
| `-p, --profile NAME` | Use named profile |
| `-n, --dry-run` | No changes, just show what would happen |
| `-v, --verbose` | DEBUG-level logging |
| `-f, --force` | Skip confirmation prompts |
| `-c, --config FILE` | Use specific config file |
| `-h, --help` | Show help |
| `-V, --version` | Show version |

## Conflict Resolution Strategies

| Strategy | Behavior |
|----------|----------|
| `newest` | Keep the version with the most recent mtime (default) |
| `skip` | Leave both versions untouched, report the conflict |
| `backup` | Back up both versions, then apply newest |
| `local` | Always prefer the local version |
| `remote` | Always prefer the remote version |

## Safety Features

- **Lock files** - Prevents concurrent sync runs (with stale lock detection)
- **Signal handling** - Clean shutdown on Ctrl+C or SIGTERM
- **Manifest-based deletions** - Only propagates intentional deletes
- **Backup on conflict** - Optional backup before overwriting
- **Dry-run mode** - Preview all changes safely
- **Partial transfer resume** - rsync `--partial` flag for interrupted transfers
- **State preservation on error** - Manifest only saved after full success
- **Retry logic** - Configurable retries with backoff for network issues

## File Structure

```
~/.config/rsync-sync/
├── config                    # Default configuration
├── profiles/
│   └── work.conf             # Named profile configs
├── state/
│   ├── default.manifest      # Sync state for default profile
│   └── default.lock          # Lock file
└── logs/
    └── sync-default-*.log    # Sync logs
```

## Automation

### Cron

```bash
# Sync every 30 minutes
*/30 * * * * $HOME/.local/bin/sync-client 2>&1 >> /tmp/sync-cron.log
```

### Systemd Timer

See `docs/USAGE.md` for systemd timer setup instructions.

## Documentation

- [Usage Guide](docs/USAGE.md) - Detailed usage examples and automation
- [Troubleshooting](docs/TROUBLESHOOTING.md) - Common issues and solutions

## License

MIT

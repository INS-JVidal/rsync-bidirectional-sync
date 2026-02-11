# Usage Guide

## First-Time Setup

### 1. Install

```bash
cd rsync-bidirectional-sync
./install.sh
```

### 2. Configure Remote Connection

Edit `~/.config/rsync-sync/config`:

```bash
REMOTE_USER="your-username"
REMOTE_HOST="192.168.1.100"    # IP or hostname of remote machine
REMOTE_PORT=22                  # SSH port
LOCAL_DIR="/home/you/sync"      # Local directory to sync
REMOTE_DIR="/home/you/sync"     # Remote directory to sync
```

### 3. Set Up SSH Key Authentication

```bash
# Generate key if you don't have one
ssh-keygen -t ed25519

# Copy to remote
ssh-copy-id -p 22 user@remote-host

# Test connection
ssh user@remote-host "echo ok"
```

### 4. First Sync

```bash
# Preview what will happen
sync-client --dry-run

# Run the sync
sync-client
```

On the first run (no previous state), the tool merges both sides:
- Files only on local -> pushed to remote
- Files only on remote -> pulled to local
- Same file on both sides with identical metadata -> unchanged
- Same file on both sides with different metadata -> conflict (resolved per strategy)

## Common Workflows

### Daily Development Sync

```bash
# Start of day: check what changed overnight
sync-client status

# Sync changes
sync-client

# End of day: sync again before shutting down
sync-client
```

### Preview Before Sync

```bash
# Dry run shows all actions without executing them
sync-client --dry-run

# Verbose dry run for maximum detail
sync-client --verbose --dry-run
```

### Multiple Sync Targets

Create profiles for each target:

```bash
# Work laptop
cp ~/.config/rsync-sync/config ~/.config/rsync-sync/profiles/laptop.conf
nano ~/.config/rsync-sync/profiles/laptop.conf

# Home server
cp ~/.config/rsync-sync/config ~/.config/rsync-sync/profiles/server.conf
nano ~/.config/rsync-sync/profiles/server.conf

# Sync to each
sync-client -p laptop
sync-client -p server
```

### Recovering from Issues

```bash
# Reset sync state (treats next sync as first sync)
sync-client reset-state

# Force first-sync merge
sync-client
```

## Exclusion Patterns

Configured in your config file using rsync pattern syntax:

```bash
EXCLUDE_PATTERNS=(
    ".git/"
    "node_modules/"
    "*.tmp"
    "*.log"
    "build/"
    ".env"
    "*.pyc"
    "__pycache__/"
)
```

## Automation

### Cron Job

```bash
# Edit crontab
crontab -e

# Add sync every 30 minutes
*/30 * * * * $HOME/.local/bin/sync-client 2>&1 >> $HOME/.config/rsync-sync/logs/cron.log

# Add sync every hour during work hours (Mon-Fri 9-18)
0 9-18 * * 1-5 $HOME/.local/bin/sync-client 2>&1 >> $HOME/.config/rsync-sync/logs/cron.log
```

### Systemd Timer

Create `~/.config/systemd/user/sync-client.service`:

```ini
[Unit]
Description=Bidirectional rsync sync
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=%h/.local/bin/sync-client
StandardOutput=journal
StandardError=journal
```

Create `~/.config/systemd/user/sync-client.timer`:

```ini
[Unit]
Description=Run sync every 30 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=30min
Persistent=true

[Install]
WantedBy=timers.target
```

Enable:

```bash
systemctl --user daemon-reload
systemctl --user enable --now sync-client.timer

# Check status
systemctl --user status sync-client.timer
journalctl --user -u sync-client -f
```

### Watch Mode (inotifywait)

For near-real-time sync using filesystem events:

```bash
# Install inotify-tools
sudo apt install inotify-tools

# Watch and sync on changes
while inotifywait -r -e modify,create,delete,move "$HOME/sync"; do
    sync-client
done
```

## WSL-Specific Setup

### On the WSL Machine

1. Install and start SSH server:

```bash
sudo apt install openssh-server
sudo service ssh start
```

2. Auto-start SSH on WSL boot. Create `/etc/wsl.conf`:

```ini
[boot]
command = service ssh start
```

3. Find your WSL IP:

```bash
hostname -I
```

### On the Linux Desktop

Configure `~/.config/rsync-sync/config`:

```bash
REMOTE_USER="wsl-username"
REMOTE_HOST="<wsl-ip>"
REMOTE_PORT=22
LOCAL_DIR="/home/you/projects"
REMOTE_DIR="/home/wsl-user/projects"
```

### Port Forwarding (if WSL IP changes)

If your WSL IP changes on reboot, set up port forwarding on Windows:

```powershell
# Run in PowerShell as admin
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 connectport=22 connectaddress=$(wsl hostname -I)
```

Then use `localhost:2222` as your remote in the config.

## Conflict Resolution Examples

### Scenario: Same File Modified on Both Sides

With `CONFLICT_STRATEGY="newest"`:
```
File: project/main.py
  Local mtime:  2024-01-15 10:30:00
  Remote mtime: 2024-01-15 11:45:00
  Result: Remote version wins (newer), pushed to local
```

With `CONFLICT_STRATEGY="backup"`:
```
File: project/main.py
  -> Backup local:  .sync-backups/project/main.py.20240115_120000
  -> Backup remote: .sync-backups/project/main.py.20240115_120000 (on remote)
  -> Remote version applied (newer)
```

With `CONFLICT_STRATEGY="skip"`:
```
File: project/main.py
  -> Skipped (both versions preserved, reported in summary)
```

## Log Files

Logs are stored in `~/.config/rsync-sync/logs/`:

```bash
# View latest log
ls -t ~/.config/rsync-sync/logs/ | head -1 | xargs -I{} cat ~/.config/rsync-sync/logs/{}

# Follow log in real time (run sync in another terminal)
tail -f ~/.config/rsync-sync/logs/sync-default-*.log
```

Log rotation is automatic: old logs are pruned based on `MAX_LOG_FILES` and `MAX_LOG_SIZE` config values.

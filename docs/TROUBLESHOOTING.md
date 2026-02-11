# Troubleshooting Guide

## Common Issues

### "Another sync is already running"

**Cause:** A lock file exists from a previous run.

**Fix:**
```bash
# Check if a sync process is actually running
ps aux | grep sync-client

# If no sync is running, the lock is stale. Remove it:
rm ~/.config/rsync-sync/state/default.lock

# For a named profile:
rm ~/.config/rsync-sync/state/<profile>.lock
```

The tool auto-detects stale locks (checks if the PID is alive), but if the system crashed, the PID might have been reused.

### SSH Connection Failed

**Symptoms:** "SSH connection failed" or timeout errors.

**Checks:**
```bash
# Test SSH manually
ssh -v -p PORT user@host

# Verify key auth works (should not prompt for password)
ssh -o BatchMode=yes -p PORT user@host "echo ok"

# If key auth fails, set it up:
ssh-copy-id -p PORT user@host
```

**Common causes:**
- SSH server not running on remote: `sudo service ssh start`
- Wrong port: check `REMOTE_PORT` in config
- Firewall blocking: `sudo ufw allow PORT`
- WSL IP changed: check with `wsl hostname -I`
- Key not authorized: run `ssh-copy-id` again

### "rsync is not installed on remote"

**Fix:**
```bash
# On the remote machine (Debian/Ubuntu)
sudo apt install rsync

# On the remote machine (RHEL/CentOS)
sudo yum install rsync
```

### "Local directory does not exist"

**Fix:**
```bash
# Create the directory
mkdir -p /path/to/your/sync/dir

# Or fix the path in config
nano ~/.config/rsync-sync/config
```

### "Configuration file not found"

**Fix:**
```bash
# Run the installer
./install.sh

# Or manually create config
cp config.example ~/.config/rsync-sync/config
nano ~/.config/rsync-sync/config
```

### "Bash 4.0+ required"

**Check version:**
```bash
bash --version
```

**Fix on macOS** (if running there):
```bash
brew install bash
# Use /usr/local/bin/bash instead of /bin/bash
```

On modern Linux distributions, Bash 4+ is standard.

### Sync Seems Stuck

**Possible causes:**
- Large files transferring slowly
- Network congestion
- Remote server under load

**Checks:**
```bash
# Run with verbose mode to see progress
sync-client --verbose

# Check if rsync is running
ps aux | grep rsync

# Check network
ping remote-host
```

**If truly stuck**, press Ctrl+C. The signal handler will clean up the lock file. On the next run, the tool will retry from where it left off (thanks to `--partial`).

### Files Keep Re-syncing Every Run

**Cause:** The manifest isn't being saved, or mtime keeps changing.

**Checks:**
1. Look for errors in the log - manifest is only saved on fully successful sync
2. Check if something (editor, backup tool) is modifying file timestamps
3. Try resetting state: `sync-client reset-state`

**Fix for filesystem clock skew:**
```bash
# Ensure both machines have synchronized time
sudo ntpdate pool.ntp.org
# or
sudo timedatectl set-ntp true
```

### Conflicts on Every Run

**Cause:** Clock skew between machines, or a tool modifying files on both sides.

**Diagnosis:**
```bash
# Check time on both machines
date
ssh user@remote date

# Check if times differ significantly
```

**Fixes:**
- Synchronize clocks on both machines (NTP)
- Use `CHECKSUM_VERIFY=true` in config for content-based comparison
- Use `CONFLICT_STRATEGY="newest"` for automatic resolution

### Permission Errors

**Symptoms:** "Permission denied" during rsync.

**Fixes:**
```bash
# Check local permissions
ls -la /path/to/local/dir

# Check remote permissions
ssh user@host "ls -la /path/to/remote/dir"

# Fix ownership
sudo chown -R $USER:$USER /path/to/sync/dir
```

### Excluded Files Getting Synced

**Check your exclusion patterns:**
```bash
# Test what rsync would transfer
rsync -avn --exclude='.git/' --exclude='node_modules/' /local/dir/ user@host:/remote/dir/
```

**Common mistakes:**
- Missing trailing `/` on directory patterns (`.git/` vs `.git`)
- Pattern not matching nested paths
- Typos in pattern names

### Backups Filling Up Disk

**Fix:** Clean old backups:
```bash
# Check backup size
du -sh ~/sync/.sync-backups/

# Remove backups older than 30 days
find ~/sync/.sync-backups/ -mtime +30 -delete

# Or remove all backups
rm -rf ~/sync/.sync-backups/
```

**Prevent:** Disable backups in config:
```bash
BACKUP_ON_CONFLICT=false
```

## Diagnostic Commands

```bash
# Check sync status without making changes
sync-client status

# Verbose dry run (maximum detail)
sync-client --verbose --dry-run

# View current manifest
cat ~/.config/rsync-sync/state/default.manifest

# View latest log
ls -t ~/.config/rsync-sync/logs/sync-default-*.log | head -1 | xargs cat

# Check lock status
ls -la ~/.config/rsync-sync/state/*.lock 2>/dev/null

# Test SSH connectivity
ssh -o ConnectTimeout=10 -o BatchMode=yes -p PORT user@host "echo ok"

# Test rsync directly
rsync -avn -e "ssh -p PORT" /local/path/ user@host:/remote/path/
```

## Getting Help

1. Run with `--verbose` to get DEBUG-level output
2. Check the log file in `~/.config/rsync-sync/logs/`
3. Test SSH and rsync independently to isolate the issue
4. Try a `reset-state` followed by a `--dry-run` to see what a fresh sync would do

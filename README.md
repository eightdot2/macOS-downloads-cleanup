# macOS-downloads-cleanup
auto deletes files in Mac downloads folder on a schedule
# macOS Downloads Cleanup Automation

**System**: macOS Launch Agent  
**Purpose**: Automated deletion of files older than 10 days in Downloads folder  
**Frequency**: Weekly execution  
**Last Updated**: [Current Date]

---

## Overview

This system automatically removes files from the user's Downloads folder that are older than 10 days. It uses macOS's native Launch Agent system to schedule weekly cleanup operations with comprehensive logging.

## System Components

### 1. Launch Agent Configuration
**File**: `~/Library/LaunchAgents/com.user.downloads-cleanup.plist`
- Controls execution schedule (weekly)
- Defines logging paths
- Manages service lifecycle

### 2. Cleanup Script
**File**: `~/cleanup-downloads.sh`
- Performs actual file deletion
- Implements safety checks
- Provides audit logging

### 3. Log Files
**Standard Output**: `~/Library/Logs/downloads-cleanup.log`
**Error Output**: `~/Library/Logs/downloads-cleanup-error.log`

---

## Configuration Details

### Execution Schedule
- **Frequency**: Every 7 days (604,800 seconds)
- **Timing**: Runs 7 days after Launch Agent is loaded
- **Persistence**: Survives system reboots
- **Dependencies**: Requires user login session

### File Selection Criteria
- **Target Directory**: `~/Downloads`
- **Age Threshold**: Files modified >10 days ago
- **File Types**: All files (directories excluded)
- **Action**: Permanent deletion (not moved to Trash)

---

## Management Commands

### Service Control
```bash
# Load/Start Service
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.downloads-cleanup.plist

# Stop/Unload Service
launchctl bootout gui/$(id -u)/com.user.downloads-cleanup

# Check Service Status
launchctl list | grep downloads-cleanup

# Force Immediate Execution
launchctl kickstart gui/$(id -u)/com.user.downloads-cleanup
```

### Configuration Changes
```bash
# Edit Schedule (modify StartInterval value in seconds)
vi ~/Library/LaunchAgents/com.user.downloads-cleanup.plist

# Edit Cleanup Logic
vi ~/cleanup-downloads.sh

# Reload After plist Changes Only
launchctl bootout gui/$(id -u)/com.user.downloads-cleanup
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.downloads-cleanup.plist
```

### Monitoring
```bash
# View Recent Activity
cat ~/Library/Logs/downloads-cleanup.log

# View Errors
cat ~/Library/Logs/downloads-cleanup-error.log

# Live Monitor (if running)
tail -f ~/Library/Logs/downloads-cleanup.log

# Preview Files That Would Be Deleted
find ~/Downloads -type f -mtime +10 -print
```

---

## Time Intervals Reference

| Frequency | Seconds | Configuration |
|-----------|---------|---------------|
| Daily | 86400 | `<integer>86400</integer>` |
| Every 3 Days | 259200 | `<integer>259200</integer>` |
| Weekly | 604800 | `<integer>604800</integer>` |
| Bi-weekly | 1209600 | `<integer>1209600</integer>` |
| Monthly | 2592000 | `<integer>2592000</integer>` |

---

## File Age Thresholds Reference

| Age Threshold | Configuration | Description |
|---------------|---------------|-------------|
| 7 days | `-mtime +7` | More aggressive cleanup |
| 10 days | `-mtime +10` | **Current setting** |
| 14 days | `-mtime +14` | Conservative cleanup |
| 30 days | `-mtime +30` | Long retention |

---

## Troubleshooting

### Service Not Running
**Check**: `launchctl list | grep downloads-cleanup`
**Solution**: Reload Launch Agent using bootstrap command above

### No Files Being Deleted
**Check**: Run manual test with `~/cleanup-downloads.sh`
**Verify**: Files exist that are >10 days old using `find ~/Downloads -type f -mtime +10`

### Timing Issues
**Behaviour**: StartInterval counts elapsed runtime, not calendar time
**Impact**: Service pauses when system is shut down
**Normal**: Execution time drifts based on usage patterns

### Permission Errors
**Check**: Script executable permissions `ls -la ~/cleanup-downloads.sh`
**Fix**: `chmod +x ~/cleanup-downloads.sh`

---

## Current Operational Profile

### Cleanup Cycle Example
- **Day 1**: File downloaded
- **Day 8**: First weekly check → File is 7 days old → **File kept**
- **Day 11**: File becomes 11 days old
- **Day 15**: Second weekly check → File is 14 days old → **File deleted**

### Retention Window
- **Minimum retention**: 10 days (guaranteed)
- **Maximum retention**: 17 days (worst case if downloaded just after cleanup)
- **Average retention**: ~13.5 days

---

## Security Considerations

### Path Validation
- Script validates Downloads directory exists before execution
- Hard-coded path prevents accidental deletion from wrong locations

### Audit Trail
- All deletions logged with timestamps
- Files are listed before deletion for recovery reference
- Separate error logging for troubleshooting

### Recovery
- Files are permanently deleted (not Trash)
- No automated recovery mechanism
- Restore only possible from backups

---

## Modification Guidelines

### Changing Cleanup Age
**File**: `~/cleanup-downloads.sh`
**Parameter**: `-mtime +10` (change number for different age threshold)
**Note**: Changes take effect immediately, no reload required

### Changing Schedule
**File**: `~/Library/LaunchAgents/com.user.downloads-cleanup.plist`
**Parameter**: `<integer>604800</integer>` (change to desired seconds)
**Note**: Requires Launch Agent reload after changes

### Adding File Type Filters
**Example**: Only delete media files:
```bash
find "$DOWNLOADS_DIR" -type f \( -name "*.mp4" -o -name "*.jpg" -o -name "*.png" \) -mtime +10 -delete
```

---

## System Requirements

- macOS with Launch Agent support
- User-level file system access
- Bash shell availability
- Write permissions to `~/Library/Logs/`

---

**End of Documentation**

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

## Installation

### 1. Create the cleanup script
```bash
vi ~/cleanup-downloads.sh
```

Copy the following content:
```bash
#!/bin/bash

# Downloads cleanup script - Delete files older than 10 days
# Security: Explicit path checking and logging

DOWNLOADS_DIR="$HOME/Downloads"
LOG_FILE="$HOME/Library/Logs/downloads-cleanup.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# Ensure Downloads directory exists and is actually our Downloads folder
if [[ ! -d "$DOWNLOADS_DIR" ]] || [[ "$DOWNLOADS_DIR" != */Downloads ]]; then
    echo "[$DATE] ERROR: Downloads directory not found or path invalid: $DOWNLOADS_DIR" >> "$LOG_FILE"
    exit 1
fi

echo "[$DATE] Starting Downloads cleanup..." >> "$LOG_FILE"

# Count files before deletion
FILES_TO_DELETE=$(find "$DOWNLOADS_DIR" -type f -mtime +10 2>/dev/null | wc -l)

if [[ $FILES_TO_DELETE -eq 0 ]]; then
    echo "[$DATE] No files older than 10 days found" >> "$LOG_FILE"
    exit 0
fi

echo "[$DATE] Found $FILES_TO_DELETE files to delete" >> "$LOG_FILE"

# List files being deleted (for audit trail)
find "$DOWNLOADS_DIR" -type f -mtime +10 -print >> "$LOG_FILE" 2>&1

# Delete the files
find "$DOWNLOADS_DIR" -type f -mtime +10 -delete 2>> "$LOG_FILE"

echo "[$DATE] Cleanup completed" >> "$LOG_FILE"
```

### 2. Make script executable
```bash
chmod +x ~/cleanup-downloads.sh
```

### 3. Create the Launch Agent plist
```bash
vi ~/Library/LaunchAgents/com.user.downloads-cleanup.plist
```

Copy the following content:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.downloads-cleanup</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/[USERNAME]/cleanup-downloads.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>604800</integer>
    <key>StandardOutPath</key>
    <string>/Users/[USERNAME]/Library/Logs/downloads-cleanup.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/[USERNAME]/Library/Logs/downloads-cleanup-error.log</string>
</dict>
</plist>
```

**Note**: Replace `[USERNAME]` with your actual username.

### 4. Create logs directory
```bash
mkdir -p ~/Library/Logs
```

### 5. Load the Launch Agent
```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.downloads-cleanup.plist
```

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

## Customisation Options

### Time Intervals Reference

| Frequency | Seconds | Configuration |
|-----------|---------|---------------|
| Daily | 86400 | `<integer>86400</integer>` |
| Every 3 Days | 259200 | `<integer>259200</integer>` |
| Weekly | 604800 | `<integer>604800</integer>` |
| Bi-weekly | 1209600 | `<integer>1209600</integer>` |
| Monthly | 2592000 | `<integer>2592000</integer>` |

### File Age Thresholds Reference

| Age Threshold | Configuration | Description |
|---------------|---------------|-------------|
| 7 days | `-mtime +7` | More aggressive cleanup |
| 10 days | `-mtime +10` | **Default setting** |
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

## Operational Profile

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
- Files are permanently deleted (not moved to Trash)
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

## License

MIT License - Feel free to modify and distribute as needed.

---

**End of Documentation**

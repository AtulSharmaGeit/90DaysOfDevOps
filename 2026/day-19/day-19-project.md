# SHELL SCRIPTING PROJECT: LOG ROTATION, BACKUP & CRONTAB

## Task 1: Log Rotation Script

`vim log_rotate.sh`

```bash
#!/bin/bash

# Log rotation script
# Usage: ./log_rotate.sh /var/log/myapp

LOG_DIR=$1

if [ ! -d "$LOG_DIR" ]; then
  echo "Error: Directory $LOG_DIR does not exist."
  exit 1
fi

# Compress .log files older than 7 days
COMPRESSED=$(find "$LOG_DIR" -name "*.log" -mtime +7 -exec gzip {} \; -print | wc -l)

# Delete .gz files older than 30 days
DELETED=$(find "$LOG_DIR" -name "*.gz" -mtime +30 -exec rm -f {} \; -print | wc -l)

echo "Compressed $COMPRESSED log files."
echo "Deleted $DELETED old archives."
```
- Shebang
  ```bash
  #!/bin/bash
  ```
  - Tells the system to run this script using the **Bash shell**.
- Usage comment
  ```bash
  # Usage: ./log_rotate.sh /var/log/myapp
  ```
  - A reminder for the user: you must pass a **directory path** as an argument when running the script.
- Capture argument
  ```bash
  LOG_DIR=$1
  ```
  - `$1` is the **first argument** passed to the script.
  - Example: if you run `./log_rotate.sh /var/log/myapp`, then `LOG_DIR=/var/log/myapp`.
- Directory existence check
  ```bash
  if [ ! -d "$LOG_DIR" ]; then
    echo "Error: Directory $LOG_DIR does not exist."
    exit 1
  fi
  ```
  - `-d` checks if the path is a **directory**.
  - If the directory doesn’t exist, the script prints an error and exits with status `1` (failure).
- Compress old `.log` files
  ```bash
  COMPRESSED=$(find "$LOG_DIR" -name "*.log" -mtime +7 -exec gzip {} \; -print | wc -l)
  ```
  - `find` searches inside `$LOG_DIR` for files ending in `.log` that are **older than 7 days** (`-mtime +7`).
  - `-exec gzip {} \;` compresses each file with `gzip`.
  - `-print` outputs the filenames processed.
  - `wc -l` counts how many lines (i.e., how many files).
  - The result is stored in `COMPRESSED`.
- Delete old `.gz` files
  ```bash
  DELETED=$(find "$LOG_DIR" -name "*.gz" -mtime +30 -exec rm -f {} \; -print | wc -l)
  ```
  - Similar logic, but now targeting `.gz` files older than **30 days**.
  - `rm -f` deletes them.
  - Again, `wc -l` counts how many were deleted.
  - Stored in `DELETED`.
- Print summary
  ```bash
  echo "Compressed $COMPRESSED log files."
  echo "Deleted $DELETED old archives."
  ```
  - Displays how many files were compressed and deleted during this run.

![images alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/797edcc78652787d3c84dec4f6370c45be6a9f55/2026/day-19/Screenshots/Screenshot%20(539).png)

## Task 2: Server Backup Script

`vim backup.sh`

```bash
#!/bin/bash

# Backup script
# Usage: ./backup.sh /source/dir /backup/destination

SRC=$1
DEST=$2

if [ ! -d "$SRC" ]; then
  echo "Error: Source directory $SRC does not exist."
  exit 1
fi

if [ ! -d "$DEST" ]; then
  echo "Error: Destination directory $DEST does not exist."
  exit 1
fi

TIMESTAMP=$(date +%Y-%m-%d)
ARCHIVE="$DEST/backup-$TIMESTAMP.tar.gz"

tar -czf "$ARCHIVE" "$SRC"

if [ $? -eq 0 ]; then
  SIZE=$(du -h "$ARCHIVE" | cut -f1)
  echo "Backup created: $ARCHIVE ($SIZE)"
else
  echo "Error: Backup failed."
  exit 1
fi

# Delete backups older than 14 days
OLD=$(find "$DEST" -name "backup-*.tar.gz" -mtime +14 -exec rm -f {} \; -print | wc -l)
echo "Deleted $OLD old backups."
```
- Shebang
  ```bash
  #!/bin/bash
  ```
  - Ensures the script runs with the **Bash shell**.

- Usage comment
  ```bash
  # Usage: ./backup.sh /source/dir /backup/destination
  ```
  - Reminder for the user: you must pass two arguments - the source directory and the backup destination.

- Capture arguments
  ```bash
  SRC=$1
  DEST=$2
  ```
  - `$1` - first argument (source directory).
  - `$2` - second argument (backup destination).

- Validate directories
  ```bash
  if [ ! -d "$SRC" ]; then
    echo "Error: Source directory $SRC does not exist."
    exit 1
  fi

  if [ ! -d "$DEST" ]; then
    echo "Error: Destination directory $DEST does not exist."
    exit 1
  fi
  ```
  - Checks if both source and destination directories exist.
  - If either is missing, prints an error and exits with status `1`.

- Create timestamped archive
  ```bash
  TIMESTAMP=$(date +%Y-%m-%d)
  ARCHIVE="$DEST/backup-$TIMESTAMP.tar.gz"

  tar -czf "$ARCHIVE" "$SRC"
  ```
  - `date +%Y-%m-%d` - generates today’s date (e.g., `2026-03-06`).
  - Archive name becomes `backup-YYYY-MM-DD.tar.gz`.
  - `tar -czf` compresses the source directory into a `.tar.gz` file.

- Verify backup success
  ```bash
  if [ $? -eq 0 ]; then
    SIZE=$(du -h "$ARCHIVE" | cut -f1)
    echo "Backup created: $ARCHIVE ($SIZE)"
  else
    echo "Error: Backup failed."
    exit 1
  fi
  ```
  - `$?` - exit status of the last command (`tar`).
  - If success (`0`), calculate archive size using `du -h`.
  - Print archive name and size.
  - If failure, print error and exit.

- Delete old backups
  ```bash
  OLD=$(find "$DEST" -name "backup-*.tar.gz" -mtime +14 -exec rm -f {} \; -print | wc -l)
  echo "Deleted $OLD old backups."
  ```
  - Finds backups older than **14 days** (`-mtime +14`).
  - Deletes them with `rm -f`.
  - Counts how many were deleted and prints the number.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/797edcc78652787d3c84dec4f6370c45be6a9f55/2026/day-19/Screenshots/Screenshot%20(540).png)

## Task 3: Crontab

`crontab -l`

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/797edcc78652787d3c84dec4f6370c45be6a9f55/2026/day-19/Screenshots/Screenshot%20(542).png)

   ```
   * * * * *   Command
   │ │ │ │ │
   │ │ │ │ └── Day of week (0-7)
   │ │ │ └──── Month (1-12)
   │ │ └────── Day of month (1-31)
   │ └──────── Hour (0-23)
   └────────── Minute (0-59)
   ```

- Minute → exact minute (0–59)

- Hour → exact hour (0–23)

- Day of month → which day (1–31)

- Month → which month (1–12)

- Day of week → which weekday (0–7, Sunday is 0 or 7)
```cron
Run log_rotate.sh every day at 2 AM<br>
0 2 * * * /path/to/log_rotate.sh /var/log/myapp

Run backup.sh every Sunday at 3 AM<br>
0 3 * * 0 /path/to/backup.sh /srv/data /srv/backups

Run health check script every 5 minutes<br>
*/5 * * * * /path/to/health_check.sh
```

## Task 4: Combine — Scheduled Maintenance Script

`vim maintenance.sh`

```bash
#!/bin/bash

# Combined maintenance script
# Calls log rotation and backup, logs output with timestamps

LOGFILE="/var/log/maintenance.log"
LOG_DIR="/var/log/myapp"
SRC="/srv/data"
DEST="/srv/backups"

{
  echo "===== Maintenance run: $(date) ====="
  ./log_rotate.sh "$LOG_DIR"
  ./backup.sh "$SRC" "$DEST"
  echo "===== End of run ====="
} >> "$LOGFILE" 2>&1
```
- Shebang
  ```bash
  #!/bin/bash
  ```
  - Ensures the script runs with the Bash shell.

- Comments
  ```bash
  # Combined maintenance script
  # Calls log rotation and backup, logs output with timestamps
  ```
  - Explains the purpose: this script ties together **log rotation** and **backup**, and records everything in a log file.

- Variables
  ```bash
  LOGFILE="/var/log/maintenance.log"
  LOG_DIR="/var/log/myapp"
  SRC="/srv/data"
  DEST="/srv/backups"
  ```
  - `LOGFILE` - where all output will be stored.
  - `LOG_DIR` - the directory containing logs to rotate.
  - `SRC` - the source directory to back up.
  - `DEST` - the backup destination.

- Grouped commands with logging
  ```bash
  {
    echo "===== Maintenance run: $(date) ====="
    ./log_rotate.sh "$LOG_DIR"
    ./backup.sh "$SRC" "$DEST"
    echo "===== End of run ====="
  } >> "$LOGFILE" 2>&1
  ```
  - `{ ... }` → groups multiple commands together.
  - `echo "===== Maintenance run: $(date) ====="` → prints a header with the current timestamp.
  - `./log_rotate.sh "$LOG_DIR"` - runs your log rotation script.
  - `./backup.sh "$SRC" "$DEST"` - runs your backup script.
  - `echo "===== End of run ====="` - prints a footer marking the end of the run.
  - `>> "$LOGFILE"` - appends all output to /var/log/maintenance.log.
  - `2>&1` - redirects errors (`stderr`) to the same log file, so you capture both normal output and errors.

![image alt](https://github.com/AtulSharmaGeit/90DaysOfDevOps/blob/797edcc78652787d3c84dec4f6370c45be6a9f55/2026/day-19/Screenshots/Screenshot%20(541).png)

## What I Learned
1. Automation with cron — scheduling scripts ensures regular maintenance without manual intervention.

2. Error handling matters — checking directories before running prevents silent failures.

3. Lifecycle management — rotating logs and pruning backups keeps storage under control and systems healthy.
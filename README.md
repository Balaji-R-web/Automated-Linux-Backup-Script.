# Project 5: Automated Linux Backup Script

## 📌 Project Overview

This project demonstrates how to automate file and directory backups on Linux using Bash scripting. It creates compressed archives, records backup details in a log file, verifies archive integrity, and tests restoration of backed-up files.

The project is designed for an Amazon Linux EC2 environment.

## 🎯 Objectives

- Create automated file backups.
- Compress backups using `tar` and `gzip`.
- Generate timestamped backup archives.
- Verify backup archive integrity.
- Restore files from a backup.
- Maintain backup logs.
- Schedule backups using cron.
- Follow basic backup security practices.

## 🛠️ Environment

- Operating System: Amazon Linux
- Platform: AWS EC2
- Shell: Bash
- Utilities: `tar`, `gzip`, `date`, `find`, `cron`

---

## 1. Check the Linux Environment

**Command:**

```bash
whoami
```

**Sample Output:**

```text
ec2-user
```

Check the operating system:

```bash
cat /etc/os-release
```

Check whether the required utilities are installed:

```bash
command -v tar
command -v gzip
command -v find
```

**Explanation:** Confirms the environment and checks for the tools needed to create and verify backups.

## 2. Create the Project Directory

**Command:**

```bash
mkdir -p ~/project5-backup/{source,backups,logs,restore}
cd ~/project5-backup
```

Verify:

```bash
find ~/project5-backup -maxdepth 1 -type d
```

**Sample Output:**

```text
/home/ec2-user/project5-backup
/home/ec2-user/project5-backup/source
/home/ec2-user/project5-backup/backups
/home/ec2-user/project5-backup/logs
/home/ec2-user/project5-backup/restore
```

**Explanation:** Creates separate directories for test data, backup archives, logs, and restored files.

## 3. Create Sample Files for Backup

**Command:**

```bash
mkdir -p ~/project5-backup/source/documents
echo "Linux backup project" > ~/project5-backup/source/readme.txt
echo "Important configuration data" > ~/project5-backup/source/config.txt
echo "Sample report data" > ~/project5-backup/source/documents/report.txt
```

Verify:

```bash
find ~/project5-backup/source -type f
```

Display the contents:

```bash
cat ~/project5-backup/source/readme.txt
cat ~/project5-backup/source/config.txt
```

**Sample Output:**

```text
Linux backup project
Important configuration data
```

**Explanation:** Creates test files to demonstrate backup and restoration without touching production data.

## 4. Create the Backup Script

Create the script:

```bash
nano ~/project5-backup/backup.sh
```

Paste the following code:

```bash
#!/bin/bash

# Automated Linux Backup Script

set -Eeuo pipefail
umask 077

SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"

SOURCE_DIR="${SOURCE_DIR:-$SCRIPT_DIR/source}"
BACKUP_DIR="${BACKUP_DIR:-$SCRIPT_DIR/backups}"
LOG_DIR="${LOG_DIR:-$SCRIPT_DIR/logs}"

TIMESTAMP="$(date '+%Y%m%d_%H%M%S')"
BACKUP_FILE="$BACKUP_DIR/backup_$TIMESTAMP.tar.gz"
LOG_FILE="$LOG_DIR/backup.log"

mkdir -p "$BACKUP_DIR" "$LOG_DIR"

log() {
    printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*" \
        | tee -a "$LOG_FILE"
}

if [[ ! -d "$SOURCE_DIR" ]]; then
    log "ERROR: Source directory does not exist: $SOURCE_DIR"
    exit 1
fi

if [[ "$SOURCE_DIR" -ef "$BACKUP_DIR" ]]; then
    log "ERROR: Source and backup directories must be different."
    exit 1
fi

if [[ "$BACKUP_DIR" == "$SOURCE_DIR/"* ]]; then
    log "ERROR: Backup directory must not be inside the source."
    exit 1
fi

if ! find "$SOURCE_DIR" -type f -print -quit | grep -q .; then
    log "ERROR: No regular files found in source directory."
    exit 1
fi

log "Starting backup"
log "Source: $SOURCE_DIR"

TEMP_FILE="$(mktemp "$BACKUP_DIR/.backup.XXXXXX")"

cleanup() {
    if [[ -n "${TEMP_FILE:-}" ]]; then
        rm -f -- "$TEMP_FILE"
    fi
}

trap cleanup EXIT

if ! tar -czf "$TEMP_FILE" -C "$SOURCE_DIR" .; then
    log "ERROR: Backup creation failed."
    exit 1
fi

if ! tar -tzf "$TEMP_FILE" > /dev/null; then
    log "ERROR: Archive integrity check failed."
    exit 1
fi

mv -- "$TEMP_FILE" "$BACKUP_FILE"
TEMP_FILE=""

chmod 600 "$BACKUP_FILE"

BACKUP_SIZE="$(du -h "$BACKUP_FILE" | awk '{print $1}')"

log "Backup completed successfully"
log "Archive: $BACKUP_FILE"
log "Size: $BACKUP_SIZE"
log "Integrity verification: PASSED"
```

Save the file and exit Nano.

**Explanation:** The script creates a compressed archive of the source directory, verifies that it can be listed, and stores the completed archive with a timestamp. It uses a temporary file so incomplete archives are not published under the final backup name.

## 5. Make the Script Executable

**Command:**

```bash
chmod +x ~/project5-backup/backup.sh
```

Verify:

```bash
ls -l ~/project5-backup/backup.sh
```

**Explanation:** Gives the script execute permission.

## 6. Run the Backup Script

**Command:**

```bash
~/project5-backup/backup.sh
```

**Illustrative Output:**

```text
[2026-10-10 20:00:00] Starting backup
[2026-10-10 20:00:00] Source: /home/ec2-user/project5-backup/source
[2026-10-10 20:00:00] Backup completed successfully
[2026-10-10 20:00:00] Archive: /home/ec2-user/project5-backup/backups/backup_20261010_200000.tar.gz
[2026-10-10 20:00:00] Size: 220
[2026-10-10 20:00:00] Integrity verification: PASSED
```

**Explanation:** The timestamps, archive name, and size depend on the actual execution. The size above is only an illustrative example.

## 7. List the Created Backups

**Command:**

```bash
ls -lh ~/project5-backup/backups/
```

**Sample Output:**

```text
-rw------- 1 ec2-user ec2-user 220 Oct 10 20:00 backup_20261010_200000.tar.gz
```

**Explanation:** Displays the backup archive, size, permissions, owner, and modification time.

## 8. Verify the Archive Contents

Find the latest archive:

```bash
LATEST_BACKUP="$(find ~/project5-backup/backups -maxdepth 1 -type f -name 'backup_*.tar.gz' -printf '%T@ %p\n' | sort -nr | head -n 1 | cut -d' ' -f2-)"
echo "$LATEST_BACKUP"
```

List the archive contents:

```bash
tar -tzf "$LATEST_BACKUP"
```

**Sample Output:**

```text
./
./config.txt
./documents/
./documents/report.txt
./readme.txt
```

**Explanation:** Lists the files stored in the compressed archive without extracting them.

## 9. Verify Archive Integrity

**Command:**

```bash
gzip -t "$LATEST_BACKUP" && echo "GZIP integrity check passed"
```

Then:

```bash
tar -tzf "$LATEST_BACKUP" > /dev/null && echo "TAR archive verification passed"
```

**Sample Output:**

```text
GZIP integrity check passed
TAR archive verification passed
```

**Explanation:** Checks that the gzip stream is valid and that the archive can be read. This does not prove that every restored file matches its original contents.

## 10. Restore the Backup

Create a restore directory:

```bash
RESTORE_DIR="$HOME/project5-backup/restore/test_restore"
mkdir -p "$RESTORE_DIR"
```

Extract the archive:

```bash
tar -xzf "$LATEST_BACKUP" -C "$RESTORE_DIR"
```

**Explanation:** Restores the archived files into a separate test directory rather than overwriting the source.

## 11. Verify the Restored Files

**Command:**

```bash
find "$RESTORE_DIR" -type f -print
```

Display the restored content:

```bash
cat "$RESTORE_DIR/readme.txt"
cat "$RESTORE_DIR/config.txt"
cat "$RESTORE_DIR/documents/report.txt"
```

**Expected Output:**

```text
Linux backup project
Important configuration data
Sample report data
```

**Explanation:** Confirms that the files are present and readable after extraction.

## 12. Compare the Source and Restored Files

**Command:**

```bash
diff -r "$HOME/project5-backup/source" "$RESTORE_DIR"
```

**Expected Output:** No output if the directory contents match.

Check the exit status:

```bash
echo $?
```

**Expected Output:**

```text
0
```

**Explanation:** A zero exit status from `diff -r` means no content differences were detected.

## 13. View Backup Logs

**Command:**

```bash
cat ~/project5-backup/logs/backup.log
```

View the latest entries:

```bash
tail -n 20 ~/project5-backup/logs/backup.log
```

**Explanation:** Records backup start time, completion status, archive path, size, and integrity verification results.

## 14. Test a Second Backup

Add another test file:

```bash
echo "Second backup test" > ~/project5-backup/source/second.txt
```

Run the script again:

```bash
~/project5-backup/backup.sh
```

List the archives:

```bash
ls -lh ~/project5-backup/backups/
```

**Expected Result:** A second timestamped archive should be created, assuming the executions occur in different seconds.

**Explanation:** Demonstrates repeatable backup creation as the source data changes.

## 15. Schedule Automatic Backups with Cron

Open the crontab:

```bash
crontab -e
```

Add this line, replacing the path if your home directory differs:

```cron
0 2 * * * /home/ec2-user/project5-backup/backup.sh
```

**Explanation:** Runs the backup script daily at 2:00 AM according to the server's local time configuration.

Verify the schedule:

```bash
crontab -l
```

**Important:** Ensure the server is running at the scheduled time and that the cron service is available. For production backups, use a durable external destination and configure retention.

## 16. Remove the Cron Schedule

When you no longer need automatic backups, edit the crontab:

```bash
crontab -e
```

Remove the line for this project and save the changes.

Verify:

```bash
crontab -l
```

**Explanation:** Stops the scheduled execution of the lab backup script.

## 📚 Important Commands Learned

| Command | Purpose |
|---|---|
| `tar -czf` | Create a gzip-compressed archive |
| `tar -tzf` | List archive contents |
| `tar -xzf` | Extract an archive |
| `gzip -t` | Test gzip integrity |
| `find` | Locate files |
| `du -h` | Check archive size |
| `diff -r` | Compare directories recursively |
| `chmod` | Configure file permissions |
| `tee -a` | Display and append log messages |
| `crontab -e` | Schedule recurring backups |

## 🔐 Backup Best Practices

- Verify archives and perform restore tests regularly.
- Store production backups outside the source directory.
- Restrict access to backup archives because they may contain sensitive information.
- Configure backup retention and log rotation.
- Use an external destination such as Amazon S3 for durable backups.
- Consider encryption and appropriate IAM permissions for cloud backups.
- Never assume a backup is usable until a restore has been tested.
- Do not commit real backups or sensitive customer data to a public GitHub repository.

## 🚀 Possible Improvements

- Add backup retention and automatic cleanup.
- Upload archives to Amazon S3.
- Enable server-side encryption.
- Send success and failure notifications.
- Add checksums for backup verification.
- Schedule backups through AWS Systems Manager or another managed automation service.

## ✅ Project Outcome

This project demonstrates Linux backup automation, archive compression, integrity verification, file restoration, logging, and cron scheduling using Bash on Amazon Linux.

## 👨‍💻 Technologies Used

- Linux
- Amazon Linux
- Bash Scripting
- `tar` and `gzip`
- Cron
- AWS EC2

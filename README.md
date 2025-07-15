# Daily Server Backup Configuration

This document details the setup of a daily backup solution for specific directories on your server, 
including website files, system configurations, and a standard user's home directory.
Backups will be stored locally in a `/backup` directory and managed by `systemd` for automated execution.


## 1\. Prerequisites

  * `root` or `sudo` access on the server.
  * Basic knowledge of command-line file editing (e.g. `vim`).
  * The `/backup` directory (or another directory of your choice) must exist or be creatable and have sufficient disk space.

## 2\. Directory Structure

The script is designed to back up the following directories:

  * `/var/www/html` (Website files)
  * `/etc` 
  * `/usr/local/bin` (Custom scripts or binaries)
  * `/home/username` (Home directory of the `username` user - **to be adapted**)

Backup archives will be stored in the `/backup` directory.

## 3\. The Backup Script (`backup.sh`)

This Bash script compresses the specified directories into a timestamped `tar.gz` archive.
It also handles the creation of the backup directory, permission checks,
and backup rotation (deleting archives older than 7 days).

**Location:** You can place this script anywhere on your system, for example, in `/usr/local/sbin/backup.sh`.

```bash
#!/bin/bash

# --- Configuration ---
BACKUP_DIR="/backup"
DATE_FORMAT=$(date +%Y%m%d_%H%M%S)
ARCHIVE_NAME="server_backup_${DATE_FORMAT}.tar.gz"

# Define the standard user whose home directory will be backed up and who will own the backups
STANDARD_USER="username"

HOME_DIR_STANDARD_USER="/home/${STANDARD_USER}"

SOURCE_DIRS=(
	"/var/www/html"
	"/etc"
	"/usr/local/bin"
	"${HOME_DIR_STANDARD_USER}" # Include the standard user's home directory
)
LOG_FILE="${BACKUP_DIR}/backup_server.log" # Changed log file location for better standard compliance

# --- Pre-flight Checks ---

# Check if the backup directory exists, create it if not with appropriate permissions
if [ ! -d "${BACKUP_DIR}" ]; then
	echo "$(date): Backup directory '${BACKUP_DIR}' does not exist. Creating it now with permissions 755..." | tee -a "${LOG_FILE}"
	mkdir -p -m 755 "${BACKUP_DIR}" # Create with rwxr-xr-x permissions
	if [ $? -ne 0 ]; then
		echo "$(date): ERROR: Failed to create backup directory '${BACKUP_DIR}'. Script will exit." | tee -a "${LOG_FILE}"
		exit 1
	fi
fi

# Check for write permissions in the backup directory
# This check is primarily for the script's own execution, as ownership will be adjusted later.
if [ ! -w "${BACKUP_DIR}" ]; then
	echo "$(date): ERROR: You do not have write permissions in '${BACKUP_DIR}'. The script must be run as root or with sudo." | tee -a "${LOG_FILE}"
	exit 1
fi

# Check if the standard user's home directory exists
if [ ! -d "${HOME_DIR_STANDARD_USER}" ]; then
    echo "$(date): WARNING: Standard user's home directory '${HOME_DIR_STANDARD_USER}' does not exist. It will be skipped for backup." | tee -a "${LOG_FILE}"
    # Remove it from SOURCE_DIRS if it doesn't exist to avoid tar errors
    SOURCE_DIRS=("${SOURCE_DIRS[@]/${HOME_DIR_STANDARD_USER}/}")
fi


# --- Backup Execution ---

echo "$(date): Starting directory backup..." | tee -a "${LOG_FILE}"
echo "Directories to be backed up: ${SOURCE_DIRS[*]}" | tee -a "${LOG_FILE}"
echo "Archive name: ${ARCHIVE_NAME}" | tee -a "${LOG_FILE}"
echo "Destination directory: ${BACKUP_DIR}" | tee -a "${LOG_FILE}"

# Create the tar.gz archive
# The -P option preserves absolute paths (important for /etc, /var/www, etc.)
# The -c option creates the archive
# The -z option compresses with gzip
# The -f option specifies the output archive file name
tar -Pczf "${BACKUP_DIR}/${ARCHIVE_NAME}" "${SOURCE_DIRS[@]}"

# --- Backup Status Check & Permissions Adjustment ---
if [ $? -eq 0 ]; then
	echo "$(date): Backup completed successfully. Archive created: ${BACKUP_DIR}/${ARCHIVE_NAME}" | tee -a "${LOG_FILE}"
	echo "Archive size: $(du -h "${BACKUP_DIR}/${ARCHIVE_NAME}" | awk '{print $1}')" | tee -a "${LOG_FILE}"

    # Set owner and group of the archive to the standard user
    echo "$(date): Setting ownership of '${ARCHIVE_NAME}' to '${STANDARD_USER}'." | tee -a "${LOG_FILE}"
    chown "${STANDARD_USER}:${STANDARD_USER}" "${BACKUP_DIR}/${ARCHIVE_NAME}"
    if [ $? -ne 0 ]; then
        echo "$(date): WARNING: Failed to change ownership of the archive to '${STANDARD_USER}'. Permissions might be an issue for the standard user." | tee -a "${LOG_FILE}"
    fi

    # Set permissions of the archive to be readable by others (owner:rw, group:r, others:r)
    echo "$(date): Setting permissions of '${ARCHIVE_NAME}' to 644." | tee -a "${LOG_FILE}"
    chmod 644 "${BACKUP_DIR}/${ARCHIVE_NAME}"
    if [ $? -ne 0 ]; then
        echo "$(date): WARNING: Failed to set permissions 644 on the archive." | tee -a "${LOG_FILE}"
    fi

    # Ensure the backup directory itself is owned by the standard user and group, and has correct permissions
    echo "$(date): Setting ownership of backup directory '${BACKUP_DIR}' to '${STANDARD_USER}' and permissions to 755." | tee -a "${LOG_FILE}"
    chown "${STANDARD_USER}:${STANDARD_USER}" "${BACKUP_DIR}"
    chmod 755 "${BACKUP_DIR}"
    if [ $? -ne 0 ]; then
        echo "$(date): WARNING: Failed to set ownership/permissions of the backup directory." | tee -a "${LOG_FILE}"
    fi

else
	echo "$(date): ERROR: Backup failed. Check the messages above for details." | tee -a "${LOG_FILE}"
	exit 1
fi

# --- Cleanup ---
# Remove backups older than 7 days
echo "$(date): Cleaning up old backups (older than 7 days)..." | tee -a "${LOG_FILE}"
find "${BACKUP_DIR}" -type f -name "server_backup_*.tar.gz" -mtime +7 -exec rm {} \;
echo "$(date): Old backups removed." | tee -a "${LOG_FILE}"

echo "$(date): Backup script finished." | tee -a "${LOG_FILE}"
```

**Script Setup Steps:**

1.  **Create the `backup.sh` file**:
    ```bash
    sudo vim /usr/local/sbin/backup.sh
    ```
    Paste the content above.
2.  **Modify the `STANDARD_USER` variable**:
    Replace `"username"` with the actual username (e.g., `david`).
    ```bash
    STANDARD_USER="david" # REPLACE THIS WITH YOUR STANDARD USERNAME
    ```
3.  **Make the script executable**:
    ```bash
    sudo chmod +x /usr/local/sbin/backup.sh
    ```

**How the Script Works:**

  * **Configuration:** Defines the backup directory (`/backup`), date format, archive name, and the list of directories to back up.
  * **User Permissions:** It's configured so that the `/backup` directory and the generated archives are readable by a specified standard user, and owned by that user after the backup. This allows the standard user to access backups without `root` privileges.
  * **Checks:** Verifies the existence and write permissions of the backup directory.
  * **Backup:** Uses `tar -Pczf` to create a compressed archive. The `-P` option is crucial for preserving the absolute paths of files and directories.
  * **Logging:** All output is redirected to `/var/log/backup_server.log` for easy monitoring.
  * **Rotation:** Automatically deletes archives older than 7 days, preventing excessive disk space consumption.

## 4\. Systemd Configuration

To automate the daily execution of the script, we use `systemd` units: a service and a timer.

### Service File (`backup.service`)

This file describes the task to be executed.

**Location:** `/etc/systemd/system/backup.service`

```ini
[Unit]
Description=Daily server backup script
RequiresMountsFor=/backup
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/local/sbin/backup.sh
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Setup Steps:**

1.  **Create the `backup.service` file**:
    ```bash
    sudo vim /etc/systemd/system/backup.service
    ```
    Paste the content above.
2.  **Ensure the path in `ExecStart` matches the location of your `backup.sh` script**.

### Timer File (`backup.timer`)

This file defines when the service should be executed.

**Location:** `/etc/systemd/system/backup.timer`

```ini
[Unit]
Description=Run daily server backup script at midnight
Docs=man:systemd.timer
Requires=backup.service
After=network.target

[Timer]
OnCalendar=*-*-* 00:00:00
Persistent=true
RandomDelaySec=600

[Install]
WantedBy=timers.target
```

**Setup Steps:**

1.  **Create the `backup.timer` file**:
    ```bash
    sudo vim /etc/systemd/system/backup.timer
    ```
    Paste the content above.

## 5\. Activation and Verification

After creating both `systemd` files, you need to activate them:

1.  **Reload systemd configuration:**
    ```bash
    sudo systemctl daemon-reload
    ```
2.  **Enable the timer (for automatic startup on boot):**
    ```bash
    sudo systemctl enable backup.timer
    ```
3.  **Start the timer (to activate it immediately):**
    ```bash
    sudo systemctl start backup.timer
    ```

**Verification of Status:**

  * To check that the timer is active and see the next execution time:
    ```bash
    systemctl status backup.timer
    ```
  * To check the service status (will be `inactive (dead)` except during execution):
    ```bash
    systemctl status backup.service
    ```

## 6\. Logging

Script execution logs are directed to the `systemd` journal and also to the `/var/log/backup_server.log` file.

  * **To view logs via `journalctl`:**
    ```bash
    journalctl -u backup.service -f
    ```
    The `-f` option allows following the logs in real-time.
  * **To view the direct log file:**
    ```bash
    tail -f /backup/backup_server.log
    ```

## 7\. Restoration (Overview)

In case a restoration is needed:

1.  **Identify the archive:** Find the appropriate `.tar.gz` archive in the `/backup` directory based on the date.
2.  **Extract the archive:**
      * To extract the entire archive to a new directory (e.g., `/tmp/restore`):
        ```bash
        sudo tar -Pxvf /backup/server_backup_YYYYMMDD_HHMMSS.tar.gz -C /tmp/restore
        ```
        The `-P` option is crucial for restoring absolute paths (e.g., `/etc/httpd` will be restored as `/tmp/restore/etc/httpd`).
      * To extract a specific file or folder:
        ```bash
        sudo tar -Pxvf /backup/server_backup_YYYYMMDD_HHMMSS.tar.gz -C /tmp/restore /path/within/archive/to/file_or_folder
        ```
        Example: `sudo tar -Pxvf /backup/server_backup_YYYYMMDD_HHMMSS.tar.gz -C /tmp/restore /var/www/html/web.com/index.html`
3.  **Verify and Move:** Once files are extracted to a temporary directory, verify their integrity 
and manually move them to their original location if necessary, taking care of permissions and ownership.

**Be extremely cautious when restoring system files (e.g., `/etc/`)\!**

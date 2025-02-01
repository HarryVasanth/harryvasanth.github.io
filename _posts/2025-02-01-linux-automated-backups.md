---
layout: post
title: "Linux - Automate Backups with rsync and cron (Advanced Guide)"
date: 2025-02-01 15:11:56 +01
categories: linux
tags: backups automation
---

## Intro

This guide demonstrates how to automate backups using `rsync` and `cron`, with advanced techniques for local and remote backups, incremental backups, and retention policies. By the end, you'll have a robust backup solution tailored to your needs.

---

## Step 1: Install rsync

Ensure `rsync` is installed on your system. For Debian-based systems:

```bash
sudo apt update
sudo apt install rsync
```

For Red Hat-based systems:

```bash
sudo yum install rsync
```

---

## Step 2: Set Up SSH for Remote Backups (Optional)

To back up to a remote server, configure SSH key-based authentication:

1. Generate an SSH key pair if you don’t already have one:

   ```bash
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
   ```

2. Copy the public key to the remote server:

   ```bash
   ssh-copy-id username@remote_server_ip
   ```

3. Test the connection:

   ```bash
   ssh username@remote_server_ip
   ```

This ensures passwordless login for secure and automated backups.

---

## Step 3: Create a Backup Script

Create a script to define what to back up, where, and how. This script will include logging, error handling, and optional email notifications.

### Example Script

```bash
#!/bin/bash

# Variables
SOURCE="/home/username/"
DESTINATION="/mnt/backup_drive/"
LOGFILE="/var/log/backup.log"
EMAIL="user@example.com"

# Function to send email notifications (optional)
send_email() {
    mail -s "Backup Status" "$EMAIL" < "$LOGFILE"
}

# Start backup process
echo "Backup started at $(date)" > "$LOGFILE"

rsync -av --delete --exclude="*.tmp" "$SOURCE" "$DESTINATION" >> "$LOGFILE" 2>&1

if [ $? -eq 0 ]; then
    echo "Backup completed successfully at $(date)" >> "$LOGFILE"
    send_email
else
    echo "Backup failed at $(date)" >> "$LOGFILE"
    send_email
    exit 1
fi
```

Make the script executable:

```bash
chmod +x ~/backup.sh
```

---

## Step 4: Schedule Backups with cron

Edit the crontab file:

```bash
crontab -e
```

### Examples of Schedules:

- **Daily at midnight**:
  ```bash
  0 0 * * * /home/username/backup.sh
  ```
- **Every Sunday at 2 AM**:
  ```bash
  0 2 * * 0 /home/username/backup.sh
  ```
- **Every hour**:
  ```bash
  0 * * * * /home/username/backup.sh
  ```

Save and exit. To verify the cron job is active, use:

```bash
crontab -l
```

---

## Step 5: Advanced Backup Features

### Incremental Backups with `--link-dest`

To save space while maintaining multiple backup versions, use incremental backups:

1. Create a directory structure for snapshots:

   ```bash
   mkdir -p /mnt/backup_drive/{current,incremental}
   ```

2. Update the script to use `--link-dest`:

   ```bash
   rsync -av --delete --link-dest=/mnt/backup_drive/current "$SOURCE" /mnt/backup_drive/incremental/$(date +%Y-%m-%d) >> "$LOGFILE" 2>&1

   cp -al /mnt/backup_drive/incremental/$(date +%Y-%m-%d) /mnt/backup_drive/current >> "$LOGFILE" 2>&1
   ```

### Remote Backups with SSH Tunnel:

For remote backups:

```bash
rsync -avz -e "ssh -i ~/.ssh/id_rsa" --delete "$SOURCE" username@remote_server:/path/to/destination >> "$LOGFILE" 2>&1
```

### Retention Policy for Old Backups:

Use `find` to delete old backups automatically:

```bash
find /mnt/backup_drive/incremental/* -mtime +30 -exec rm -rf {} \;
```

This keeps backups from the last 30 days.

---

## Step 6: Verify Backups

Regularly verify that backups are working as expected:

1. Check the log file:

   ```bash
   cat /var/log/backup.log
   ```

2. Test restoring files:
   ```bash
   rsync -av /mnt/backup_drive/current/ /path/to/test_restore/
   ```

---

## Step 7: Troubleshooting

### Common Issues:

- **Cron job not running**: Ensure the script has executable permissions and uses absolute paths.
- **Permission errors**: Run the script as a user with sufficient permissions or use `sudo`.
- **SSH issues**: Verify SSH keys are configured correctly.

### Debugging Tips:

- Add debugging options like `-v` or `--progress` to `rsync`.
- Redirect cron errors to a log file:

  ```bash
  crontab -e

  # Example with error logging:
  0 0 * * * /home/username/backup.sh >> /var/log/cron_backup.log 2>&1
  ```

---

## Conclusion

With this advanced setup, you now have a flexible and reliable backup system using `rsync` and `cron`. Customize it further based on your specific requirements, such as encryption or cloud integration. Always test your backups regularly!

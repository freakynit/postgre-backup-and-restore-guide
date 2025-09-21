## This guide is for Ubuntu and PostgreSQL-17

# Table of Contents

1. [Setup PostgreSQL](#setup-postgresql)

   1. [Update your system](#update-your-system)
   2. [Add the PostgreSQL APT Repository](#add-the-postgresql-apt-repository)
   3. [Add the PostgreSQL repository](#add-the-postgresql-repository)
   4. [Install PostgreSQL 17](#install-postgresql-17)
   5. [Start and Enable PostgreSQL Service](#start-and-enable-postgresql-service)
   6. [Verify installation for PostgreSQL](#verify-installation-for-postgresql)
   7. [Security: Change postgres user password after installation and restart](#security-change-postgres-user-password-after-installation-and-restart)
2. [Setup PGBackRest](#setup-pgbackrest)

   1. [Update your system (skip if already done)](#update-your-system-skip-if-already-done)
   2. [Install pgBackRest](#install-pgbackrest)
   3. [Verify Installation for pgBackRest](#verify-installation-for-pgbackrest)
3. [Configure pgBackRest](#configure-pgbackrest)

   1. [Create Backup Directory](#create-backup-directory)
4. [Configure PostgreSQL for pgBackRest](#configure-postgresql-for-pgbackrest)

   1. [Enable PostgreSQL Archive Mode](#enable-postgresql-archive-mode)
   2. [Add/Update the following lines](#addupdate-the-following-lines)
   3. [Restart PostgreSQL](#restart-postgresql)
5. [Setting Up a Stanza](#setting-up-a-stanza)

   1. [Add stanza to `pgbackrest.conf`](#add-stanza-to-pgbackrestconf)
   2. [Initialise, check and confirm repo state of the stanza](#initialise-check-and-confirm-repo-state-of-the-stanza)
6. [Taking Backup Manually](#taking-backup-manually)

   1. [Full Backups](#full-backups)
   2. [Differential Backups](#differential-backups)
   3. [Incremental Backups](#incremental-backups)
   4. [See latest available backups for repo-2 (s3 in our case)](#see-latest-available-backups-for-repo-2-s3-in-our-case)
7. [Restore Backup](#restore-backup)

   1. [Option 1: Restore with --delta (Recommended for Partial Restore)](#option-1-restore-with---delta-recommended-for-partial-restore)
   2. [Option 2: Perform a Full Clean Restore](#option-2-perform-a-full-clean-restore)
   3. [Option 3: Perform Point-in-Time Restore](#option-3-perform-point-in-time-restore)
   4. [Option 4: Specific Backup Set Restoration](#option-4-specific-backup-set-restoration)
   5. [Check Database Integrity](#check-database-integrity)
8. [Verify Logs After Restore](#verify-logs-after-restore)

   1. [Common log files](#common-log-files)
   2. [Tail the restore log after a restore](#tail-the-restore-log-after-a-restore)
   3. [Tail the backup log after a backup](#tail-the-backup-log-after-a-backup)
9. [Automate Backups with Cron](#automate-backups-with-cron)

10. [References](#references)

## Setup PostgreSQL

### Update your system
```bash
sudo apt update
sudo apt upgrade -y
```

### Add the PostgreSQL APT Repository
```bash
sudo apt install -y curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
```

### Add the PostgreSQL repository
```bash
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'        
```

### Install PostgreSQL 17
```bash
sudo apt update
sudo apt -y install postgresql-17 postgresql-client   
```

### Start and Enable PostgreSQL Service
```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Verify installation for PostgreSQL
```bash
psql --version
```

### Security: Change postgres user password after installation and restart
```bash
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'your_new_password';"
sudo systemctl restart postgresql
```

## Setup PGBackRest

### Update your system

> Skip if already done

```bash
sudo apt update
sudo apt upgrade -y
```

### Install pgBackRest
```bash
sudo apt install pgbackrest -y
```

### Verify Installation for pgBackRest
```bash
pgbackrest --version
```

## Configure pgBackRest

### Create Backup Directory
```bash
# Log Directory
sudo mkdir -p -m 770 /var/log/pgbackrest
sudo chown postgres:postgres /var/log/pgbackrest

# Configuration Directory
sudo mkdir -p /etc/pgbackrest
sudo mkdir -p /etc/pgbackrest/conf.d
sudo touch /etc/pgbackrest/pgbackrest.conf
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
sudo chown postgres:postgres /etc/pgbackrest/pgbackrest.conf

# Backup Directory
sudo mkdir -p /var/lib/pgbackrest
sudo chmod 750 /var/lib/pgbackrest
sudo chown postgres:postgres /var/lib/pgbackrest
```

## Configure PostgreSQL for pgBackRest

### Enable PostgreSQL Archive Mode
```bash
sudo nano /etc/postgresql/17/main/postgresql.conf  # Or use your preferred editor
```

### Add/Update the following lines
```
archive_mode = on
archive_command = '/usr/bin/pgbackrest --stanza=production archive-push "%p"'
archive_timeout = 60
wal_level = replica
max_wal_senders = 3
```

### Restart PostgreSQL
```bash
sudo systemctl restart postgresql
```

## Setting Up a Stanza

### Add stanza to `pgbackrest.conf`
1. `pg1-path` is data directory of postgres database.
2. `repo1-path` is repository path where local backup will be stored.
3. You can skip S3 settings if local backups are enough. **Security note:** In production, avoid storing plain-text credentials; use environment variables or a secrets manager.

```bash
[production]
pg1-path=/var/lib/postgresql/17/main
#pg1-host=<host>
#pg1-socket-path=<custom unix socket or custom PGDATA layout>
pg1-port=5432
pg1-user=postgres

[global]
# Local POSIX repo
repo1-path=/var/lib/pgbackrest
repo1-type=posix
repo1-compress-type=lz4
repo1-retention-full=2
repo1-retention-diff=4

log-level-console=info
log-level-file=debug
log-path=/var/log/pgbackrest

# Remote S3-compatible repo (Using Cloudflare R2 here as an example)
repo2-type=s3
# repo2-path is required syntactically; it is unused for S3 semantics
repo2-path=/pgbackrest
repo2-s3-endpoint=<your-r2-endpoint>
repo2-s3-bucket=<your-bucket-name>
repo2-s3-key=<YOUR_R2_ACCESS_KEY>
repo2-s3-key-secret=<YOUR_R2_SECRET_KEY>
repo2-s3-region=auto
repo2-s3-verify-tls=true
repo2-retention-full=4
repo2-retention-diff=8

# Optional tuning
process-max=4
#compress-type=zstd
#compress-level=3
```

### Initialise, check and confirm repo state of the stanza
```bash
sudo -u postgres pgbackrest --stanza=production --log-level-console=info stanza-create
sudo -u postgres pgbackrest --stanza=production --log-level-console=info check
sudo -u postgres pgbackrest --stanza=production --log-level-console=info info
```

## Taking Backup Manually

### Full Backups

> A full backup is self-contained and does not rely on any previous backups, making it ideal for disaster recovery scenarios.

```bash
sudo -u postgres pgbackrest --stanza=production --type=full backup
```

### Differential Backups

> A differential backup includes only the data that has changed since the last `full backup` (NOT since the `last differential`).

```bash
sudo -u postgres pgbackrest --stanza=production --type=diff backup
```

### Incremental Backups

> After the initial full backup, you can save time and space by running incremental backups. An incremental backup includes only the data that has changed since the last backup of any type (full, diff, or incr).

```bash
sudo -u postgres pgbackrest --stanza=production --type=incr backup
```

### See latest available backups for repo-2 (s3 in our case)
```bash
sudo -u postgres pgbackrest --stanza=production --repo=2 info
```

## Restore Backup

> **Important:** Restores involve downtime. Test in a staging environment first. Use timestamps for backups (e.g., as shown in examples).

### Option 1: Restore with --delta (Recommended for Partial Restore)

#### Stop PostgreSQL
```bash
sudo systemctl stop postgresql
```

#### Perform the Restore with --delta flag

> The `--delta` option allows pgBackRest to restore only the changed files, reducing time and risk.

```bash
sudo -u postgres pgbackrest --stanza=production --delta restore
# You can specify --repo=2 if you want to restore from S3 instead of local backup
```

### Once the restore is complete, restart PostgreSQL and check status
```bash
sudo systemctl start postgresql
sudo systemctl status postgresql
```

### Option 2: Perform a Full Clean Restore

#### Stop PostgreSQL
```bash
sudo systemctl stop postgresql
```

#### Backup Existing Data Directory and Remove Existing Files (Optional but Recommended)
```bash
sudo mv /var/lib/postgresql/17/main /var/lib/postgresql/17/main_backup_$(date +%Y%m%d_%H%M%S)
sudo mkdir /var/lib/postgresql/17/main
sudo chown -R postgres:postgres /var/lib/postgresql/17/main
sudo chmod 700 /var/lib/postgresql/17/main
```

#### Perform the Restore
```bash
sudo -u postgres pgbackrest --stanza=production restore
# You can specify --repo=2 if you want to restore from S3 instead of local backup
```

#### Once the restore is complete, restart PostgreSQL and check status
```bash
sudo systemctl start postgresql
sudo systemctl status postgresql
```

### Option 3: Perform Point-in-Time Restore

#### Stop PostgreSQL
```bash
sudo systemctl stop postgresql
```

#### Backup Existing Data Directory and Remove Existing Files (Optional but Recommended)
```bash
sudo mv /var/lib/postgresql/17/main /var/lib/postgresql/17/main_backup_$(date +%Y%m%d_%H%M%S)
sudo mkdir /var/lib/postgresql/17/main
sudo chown -R postgres:postgres /var/lib/postgresql/17/main
sudo chmod 700 /var/lib/postgresql/17/main
```

#### Perform the Restore
```bash
sudo -u postgres pgbackrest --stanza=production --type=time --target="2025-09-20 14:30:00" --target-action=promote restore
# You can specify --repo=2 if you want to restore from S3 instead of local backup
```

#### Once the restore is complete, restart PostgreSQL and check status
```bash
sudo systemctl start postgresql
sudo systemctl status postgresql
```

### Option 4: Specific Backup Set Restoration

> If you need to restore a specific backup instead of the latest

#### Stop PostgreSQL
```bash
sudo systemctl stop postgresql
```

#### Option 1: Clean Restore (Delete Files First)
```bash
sudo mv /var/lib/postgresql/17/main /var/lib/postgresql/17/main_backup_$(date +%Y%m%d_%H%M%S)
sudo mkdir /var/lib/postgresql/17/main
sudo chown -R postgres:postgres /var/lib/postgresql/17/main
sudo chmod 700 /var/lib/postgresql/17/main
```

#### Option 2: Delta Restore (Keep Existing Files)
```bash
# No action needed here
```

#### Perform the Restore
```bash
# List available backups
sudo -u postgres pgbackrest --stanza=production info

# Option 1: Clean Restore from specific backup set
sudo -u postgres pgbackrest --stanza=production --set=20250921-020001F_20250918-020001D restore

# Option 2: Delta Restore from specific backup set
sudo -u postgres pgbackrest --stanza=production --set=20250921-020001F_20250918-020001D --delta restore

# You can specify --repo=2 if you want to restore from S3 instead of local backup
```

#### Once the restore is complete, restart PostgreSQL and check status
```bash
sudo systemctl start postgresql
sudo systemctl status postgresql
```

### Check Database Integrity
```bash
sudo -u postgres psql -c "SELECT current_timestamp, version();"
```

## Verify Logs After Restore

> pgBackRest writes a separate log file for each operation in `/var/log/pgbackrest/`.

### Common log files
- `production-backup.log` → backup operations
- `production-restore.log` → restore operations
- `production-expire.log` → expired backup cleanup
- `production-stanza-create.log` → stanza creation

### Tail the restore log after a restore
```bash
sudo tail -f /var/log/pgbackrest/production-restore.log
```

### Tail the backup log after a backup
```bash
sudo tail -f /var/log/pgbackrest/production-backup.log
```

## Automate Backups with Cron

1. We'll configure full backups every Sunday and differential backups daily, except Sunday.
2. For this, edit the crontab for postgres user (`sudo crontab -u postgres -e`) and add below jobs

```bash
# Full backup every Sunday at 02:00
0 2 * * 0 /usr/bin/pgbackrest --stanza=production --log-level-console=info --type=full backup >> /var/log/pgbackrest/cron.backup.log 2>&1

# Differential backup Mon-Sat at 02:00
0 2 * * 1-6 /usr/bin/pgbackrest --stanza=production --log-level-console=info --type=diff backup >> /var/log/pgbackrest/cron.backup.log 2>&1
```


### References:
1. https://bootvar.com/guide-to-setup-pgbackrest/
2. https://www.linkedin.com/pulse/install-configure-pgbackrest-postgresql-17-ubuntu-2404-mahto-7ofgf

## This guide is for Ubuntu and PostgreSQL-17

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

### Verify installation
```bash
psql --version
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

### Verify Installation
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

# Bakup Directory
sudo mkdir -p /var/lib/pgbackrest
sudo chmod 750 /var/lib/pgbackrest
sudo chown postgres:postgres /var/lib/pgbackrest
```

## Configure PostgreSQL for pgBackRest

### Enable PosgreSQL Archive Mode
```bash
sudo vi /etc/postgresql/17/main/postgresql.conf
```

### Add/Update the following lines
```toml
archive_mode = on
archive_command = 'pgbackrest --stanza=production archive-push %p'
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
3. You can skip s3 settings if local backups are enough

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

### As the postgres user, initialise, check and confirm repo state of the stanza
```bash
sudo -u postgres pgbackrest --stanza=production --log-level-console=info stanza-create
sudo -u postgres pgbackrest --stanza=production --log-level-console=info check
sudo -u postgres pgbackrest --stanza=production --log-level-console=info info
```

## Taking Backup Mnaually

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

### Check the pgBackRest logs to ensure everything went smoothly
```bash
sudo tail -f /var/log/pgbackrest/pgbackrest.log
```

## Automate Backups with Cron

1. We'll configure full backups every Sunday and differential backups daily, except Sunday.
2. For this, edit the crontab for postgres user (`sudo crontab -u postgres -e`) and add below jobs

```bash
0 2 * * 0 postgres pgbackrest --stanza=production --type=full backup

0 2 * * 1-6 postgres pgbackrest --stanza=production --type=diff backup
```


### References:
1. https://bootvar.com/guide-to-setup-pgbackrest/
2. https://www.linkedin.com/pulse/install-configure-pgbackrest-postgresql-17-ubuntu-2404-mahto-7ofgf

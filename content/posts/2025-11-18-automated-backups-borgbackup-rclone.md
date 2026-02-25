---
title: "Automated Backups: BorgBackup + Rclone"
date: 2025-11-18T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "self-hosted"
---

I lost a database once. [SQLite](https://www.sqlite.org/) file on a $5 VPS. No backups. It was a side project with maybe 200 users, and I spent a weekend apologizing in a [Discord](https://discord.com/) channel while trying to reconstruct data from browser caches people sent me screenshots of. Never again.

That day I went down the backup rabbit hole and came out the other side with a setup that costs me about $1.50/month, runs unattended, and has saved me three times since. Here's the whole thing.

## The 3-2-1 Rule (Actually Matters)

You've probably heard this: 3 copies of your data, on 2 different media types, with 1 offsite. Sounds like enterprise yak shaving, but for indie devs it translates to something simple:

1. **Live data** on your VPS (copy 1)
2. **Local Borg repo** on the same box or an attached volume (copy 2, different format)
3. **[Rclone](https://rclone.org/) sync** to [Backblaze B2](https://www.backblaze.com/cloud-storage) or [Wasabi](https://wasabi.com/) (copy 3, offsite)

That's it. Three layers. If your VPS provider has a datacenter fire ([OVH](https://www.ovhcloud.com/), anyone?), you've still got B2. If you fat-finger a `DROP TABLE`, [Borg](https://www.borgbackup.org/) has your deduped snapshots going back weeks.

## Why BorgBackup

Borg does deduplication and encryption at the block level. That means if you have a 2GB database and only 50MB changed since yesterday, the new backup archive only stores ~50MB of new data. Over time this adds up massively.

Install it:

```bash
# Debian/Ubuntu
sudo apt install borgbackup

# Or grab the standalone binary
wget https://github.com/borgbackup/borg/releases/download/1.2.8/borg-linux64
chmod +x borg-linux64 && sudo mv borg-linux64 /usr/local/bin/borg
```

Initialize a repo:

```bash
# I keep borg repos on a separate volume when possible
export BORG_REPO=/mnt/backups/borg
export BORG_PASSPHRASE='your-strong-passphrase-here'

borg init --encryption=repokey $BORG_REPO
```

Use `repokey` encryption. The key lives in the repo itself (encrypted with your passphrase), but **also export it and store it somewhere safe**:

```bash
borg key export $BORG_REPO /root/borg-key-backup.txt
# Copy this file OFF the server. Put it in your password manager.
```

If you lose the key and the passphrase, your backups are fancy paperweights.

## The Backup Script

Here's my actual backup script, trimmed slightly:

```bash
#!/bin/bash
set -euo pipefail

export BORG_REPO=/mnt/backups/borg
export BORG_PASSPHRASE='your-strong-passphrase-here'

BACKUP_PATHS="/home /var/lib/docker/volumes /etc/nginx /opt/apps"
EXCLUDE="--exclude '*.pyc' --exclude '__pycache__' --exclude 'node_modules' --exclude '.cache'"

# Dump databases before file backup
pg_dump -U postgres myapp > /tmp/myapp-$(date +%Y%m%d).sql
sqlite3 /opt/apps/data/app.db ".backup '/tmp/app-sqlite-$(date +%Y%m%d).db'"

# Create archive with timestamp
borg create \
  --stats --progress \
  --compression zstd,3 \
  $EXCLUDE \
  ::'{hostname}-{now:%Y-%m-%d_%H:%M}' \
  $BACKUP_PATHS /tmp/*.sql /tmp/*.db

# Prune old archives: keep 7 daily, 4 weekly, 6 monthly
borg prune \
  --keep-daily=7 \
  --keep-weekly=4 \
  --keep-monthly=6 \
  --stats

# Compact to reclaim space
borg compact

# Clean up temp dumps
rm -f /tmp/myapp-*.sql /tmp/app-sqlite-*.db

echo "Backup complete: $(date)"
```

Note the database dumps happening *before* the borg create. You don't want borg snapshotting a database mid-write. Dump it to a file first, then back up the file. This is a common footgun that people skip.

## Enter Borgmatic (Optional but Nice)

If you don't want to maintain a shell script, [borgmatic](https://torsion.org/borgmatic/) wraps borg with a YAML config. Install it:

```bash
pip install borgmatic
borgmatic config generate
```

Then edit `/etc/borgmatic/config.yaml`:

```yaml
source_directories:
  - /home
  - /var/lib/docker/volumes
  - /etc/nginx
  - /opt/apps

repositories:
  - path: /mnt/backups/borg
    label: local

exclude_patterns:
  - '*.pyc'
  - 'node_modules'
  - '.cache'

encryption_passphrase: "your-strong-passphrase-here"

compression: zstd,3

retention:
  keep_daily: 7
  keep_weekly: 4
  keep_monthly: 6

hooks:
  before_backup:
    - pg_dump -U postgres myapp > /tmp/myapp-dump.sql
  after_backup:
    - rm -f /tmp/myapp-dump.sql
```

Run it with `borgmatic --verbosity 1`. Same result, cleaner config.

## Rclone: Ship It Offsite

Borg handles local deduped snapshots. Rclone handles getting that repo somewhere else. I use Backblaze B2 because the pricing is absurd: $0.006/GB/month for storage, $0.01/GB for downloads. A 50GB backup repo costs $0.30/month.

```bash
sudo apt install rclone
rclone config
```

Walk through the interactive setup. Pick B2, enter your application key ID and key. Name the remote something like `b2-backups`.

Then sync your borg repo:

```bash
rclone sync /mnt/backups/borg b2-backups:my-borg-backups --transfers 4 --fast-list
```

That's a one-liner that pushes your entire borg repo to B2. Rclone is smart about only transferring changed files, so subsequent syncs are fast.

### Wasabi Alternative

Wasabi is flat $6.99/TB/month with no egress fees. If you're storing more than ~100GB, it's cheaper than B2. The rclone setup is identical -- Wasabi speaks S3, so you pick "S3" as the provider and point the endpoint at `s3.wasabi.com`.

## Cron It

```bash
# /etc/cron.d/backups
# Run borg backup at 3am, then rclone sync at 4am
0 3 * * * root /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
0 4 * * * root rclone sync /mnt/backups/borg b2-backups:my-borg-backups --transfers 4 >> /var/log/rclone-backup.log 2>&1
```

Or with borgmatic:

```bash
0 3 * * * root borgmatic --verbosity 1 --syslog-verbosity 1
0 4 * * * root rclone sync /mnt/backups/borg b2-backups:my-borg-backups --transfers 4
```

## Test Your Restores (Seriously)

Backups you've never tested are just vibes. Do this quarterly:

```bash
# List archives
borg list /mnt/backups/borg

# Extract a specific archive to a temp dir
mkdir /tmp/restore-test
cd /tmp/restore-test
borg extract /mnt/backups/borg::myhost-2025-11-17_03:00

# Verify your database dump loads
psql -U postgres -d testdb < /tmp/restore-test/tmp/myapp-dump.sql

# Clean up
rm -rf /tmp/restore-test
```

You can also restore from the B2 copy. Pull the repo down with rclone, then run borg against it:

```bash
rclone sync b2-backups:my-borg-backups /tmp/borg-restore-repo
borg list /tmp/borg-restore-repo
```

If this works, you're golden. If it doesn't, you found the problem before it mattered.

## Cost Breakdown

Here's what my backup setup costs for a typical indie app (10-30GB of actual data):

| Item | Cost |
|------|------|
| BorgBackup | Free (open source) |
| Rclone | Free (open source) |
| Backblaze B2 (50GB borg repo) | ~$0.30/month |
| Wasabi alternative (50GB) | ~$0.35/month (min $6.99 for 1TB) |
| VPS storage for local borg repo | Usually included in your VPS |

Total: under $2/month for peace of mind. Compare that to managed backup services charging $20-50/month.

## The Paranoid Extra Step

I also dump my borg passphrase and key export into my password manager ([1Password](https://1password.com/)), and I keep a text file in there with the exact restore commands. Future-me, panicking at 2am after a server dies, will appreciate not having to remember the syntax.

## Cheat Sheet

| Tool/Concept | What It Does | Link |
|---|---|---|
| **BorgBackup** | Deduplicating, encrypted backup tool | [borgbackup.org](https://www.borgbackup.org/) |
| **Borgmatic** | YAML config wrapper around Borg | [torsion.org/borgmatic](https://torsion.org/borgmatic/) |
| **Rclone** | Sync files to 50+ cloud storage providers | [rclone.org](https://rclone.org/) |
| **Backblaze B2** | Cheap object storage ($0.006/GB/mo) | [backblaze.com/b2](https://www.backblaze.com/b2/cloud-storage.html) |
| **Wasabi** | Flat-rate S3-compatible storage, no egress fees | [wasabi.com](https://wasabi.com/) |
| `borg init --encryption=repokey` | Create a new encrypted borg repo | -- |
| `borg key export` | Export your encryption key (store it safely!) | -- |
| `borg create --compression zstd,3` | Create a compressed backup archive | -- |
| `borg prune --keep-daily=7` | Clean up old archives per retention policy | -- |
| `rclone sync` | One-way sync local borg repo to cloud storage | -- |
| **3-2-1 Rule** | 3 copies, 2 media types, 1 offsite | -- |
| **Dump DBs before backup** | Avoid backing up databases mid-write | -- |
| **Test restores quarterly** | Backups you've never tested are just vibes | -- |

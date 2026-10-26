# pgBackRest Backup — Production Cheat Sheet

## Environment Used in Examples

```text
PostgreSQL Version : 18
pgBackRest Version : 2.59.x
Stanza              : pg18lab
PGDATA              : /postgres/data/pg18
Repository          : /postgres/backup/pg18
Tablespace          : /postgres/tbs/pg18
```

---

# 1. pgBackRest Backup Concept

pgBackRest performs a **physical backup of the complete PostgreSQL cluster**.

It is not like `pg_dump`, where we can select individual databases.

```text
PostgreSQL Cluster
       |
       +-- postgres database
       +-- edbstore database
       +-- other databases
       +-- system catalogs
       +-- tablespaces
       |
       v
   pgBackRest
       |
       v
 Repository
```

A pgBackRest physical backup includes the PostgreSQL cluster and its tablespaces.

---

# 2. Three Backup Types

pgBackRest supports:

```text
FULL
DIFFERENTIAL
INCREMENTAL
```

Commands:

```bash
--type=full
--type=diff
--type=incr
```

---

# 3. FULL Backup

A full backup copies the entire PostgreSQL cluster.

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  backup
```

Example:

```text
PostgreSQL = 5 TB

FULL Backup
     |
     +---- Copy complete database
     |
     v
Repository
```

A full backup does not depend on an earlier backup.

## Example backup label

```text
20261026-123557F
```

`F` means:

```text
FULL
```

---

# 4. Differential Backup

A differential backup contains data changed since the **last FULL backup**.

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=diff \
  backup
```

Example:

```text
Sunday
FULL
  |
  |---- changes
  |
Monday
DIFF
  |
  |---- changes
  |
Tuesday
DIFF
```

Both Monday and Tuesday differential backups are based on:

```text
Sunday FULL
```

Concept:

```text
FULL
  |
  +-------- DIFF Monday
  |
  +-------- DIFF Tuesday
  |
  +-------- DIFF Wednesday
```

To restore Wednesday's differential:

```text
FULL
 +
Wednesday DIFF
```

are required.

## Example label

```text
20261026-123557F_20261027-020001D
```

`D` means:

```text
DIFFERENTIAL
```

---

# 5. Incremental Backup

An incremental backup contains changes since the **previous successful backup**.

That previous backup may be:

```text
FULL
DIFF
INCR
```

Command:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=incr \
  backup
```

Example:

```text
FULL
  |
  +-- INCR 1
        |
        +-- INCR 2
              |
              +-- INCR 3
```

Another example:

```text
FULL
  |
  +-- DIFF
        |
        +-- INCR
              |
              +-- INCR
```

## Example label

```text
20261026-123557F_20261027-120001I
```

`I` means:

```text
INCREMENTAL
```

---

# 6. Full vs Differential vs Incremental

| Type | Backs up changes since | Backup Size | Restore Dependency |
|---|---|---:|---|
| FULL | Entire cluster | Largest | Independent |
| DIFF | Last FULL | Medium | FULL + DIFF |
| INCR | Last successful backup | Smallest | Backup chain |

Simple memory rule:

```text
FULL = Everything

DIFF = Changes since FULL

INCR = Changes since LAST BACKUP
```

---

# 7. What Happens if `--type` Is Not Specified?

Command:

```bash
pgbackrest \
  --stanza=pg18lab \
  backup
```

The default backup type is:

```text
incremental
```

If pgBackRest cannot find a valid backup on which to base the incremental backup, it will perform a FULL backup instead.

For production and learning, I prefer explicitly specifying the type:

```bash
--type=full
--type=diff
--type=incr
```

so the intention is obvious.

---

# 8. Production FULL Backup Command

A useful production-style command:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  --start-fast \
  --process-max=4 \
  backup
```

Meaning:

```text
--stanza=pg18lab
        |
        +-- PostgreSQL cluster configuration

--type=full
        |
        +-- Full backup

--start-fast
        |
        +-- Force checkpoint and start quickly

--process-max=4
        |
        +-- Use up to 4 worker processes
```

---

# 9. `start-fast`

Normally pgBackRest may wait for PostgreSQL's next checkpoint before beginning backup.

Using:

```bash
--start-fast
```

forces a checkpoint so the backup can begin sooner.

Example:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  --start-fast \
  backup
```

It can also be configured permanently:

```ini
[global]
start-fast=y
```

---

# 10. `process-max`

pgBackRest supports parallel processing.

Example:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  --process-max=4 \
  backup
```

Or configure:

```ini
[global]
process-max=4
```

Concept:

```text
                pgBackRest
                    |
          +---------+---------+
          |         |         |
       Worker1   Worker2   Worker3   Worker4
          |         |         |         |
          +---------+---------+---------+
                    |
                    v
                Repository
```

Do not blindly set very high values.

More processes mean greater:

```text
CPU usage
Disk I/O
Network I/O
Repository I/O
```

---

# 11. Online Backup

Normal pgBackRest backups are online.

PostgreSQL remains running:

```text
Applications
     |
     v
PostgreSQL   <--------- Available
     |
     |
     +------ pgBackRest backup
```

Command:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  backup
```

No database shutdown is required.

---

# 12. Offline Backup

pgBackRest can also perform an offline backup.

First stop PostgreSQL:

```bash
systemctl stop postgresql-18
```

Then:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  --no-online \
  backup
```

Normally production backups are performed online.

---

# 13. WAL During Backup

During an online backup:

```text
Backup starts
     |
     +---- PostgreSQL keeps running
     |
     +---- Users continue transactions
     |
     +---- WAL continues generating
     |
     +---- archive-push sends WAL
     |
     v
Backup completes
```

pgBackRest ensures the WAL needed to make the backup consistent has reached the repository before the online backup completes.

Concept:

```text
        FULL Backup
       /           \
Backup Start      Backup Stop
    |                  |
    +---- Required WAL +
              |
              v
          Repository
```

---

# 14. WAL Archiving

PostgreSQL configuration:

```ini
archive_mode = on
```

Example `archive_command`:

```ini
archive_command = 'pgbackrest --stanza=pg18lab archive-push %p'
```

Flow:

```text
PostgreSQL
   |
   | generates WAL
   v
pg_wal
   |
   | archive_command
   v
pgBackRest archive-push
   |
   v
Repository
```

WAL archiving happens continuously.

It is not necessary to wait for the next FULL backup to archive WAL.

---

# 15. Async WAL Archiving

Production configuration may use:

```ini
archive-async=y
spool-path=/var/spool/pgbackrest
```

Example:

```ini
[global]
archive-async=y
spool-path=/var/spool/pgbackrest

[global:archive-push]
process-max=2
```

Flow:

```text
PostgreSQL
      |
      v
archive-push
      |
      v
Async Archive Process
      |
   +--+--+
   |     |
Worker1 Worker2
   |     |
   +--+--+
      |
      v
Repository
```

---

# 16. Check WAL Archiving

PostgreSQL:

```sql
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

Important columns:

```text
archived_count
last_archived_wal
last_archived_time

failed_count
last_failed_wal
```

---

# 17. Force WAL Switch

Useful when testing backup/PITR:

```sql
SELECT pg_switch_wal();
```

This forces PostgreSQL to move to another WAL segment.

Useful before a recovery test to ensure important WAL gets archived.

---

# 18. Check pgBackRest Before Backup

Run:

```bash
pgbackrest \
  --stanza=pg18lab \
  check
```

This validates important stanza/database/archive configuration.

Production habit:

```text
Configuration
     ↓
stanza check
     ↓
backup
     ↓
backup info
```

---

# 19. Take FULL Backup

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  backup
```

With production options:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  --start-fast \
  --process-max=4 \
  backup
```

---

# 20. Take Differential Backup

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=diff \
  backup
```

Production-style:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=diff \
  --start-fast \
  --process-max=4 \
  backup
```

---

# 21. Take Incremental Backup

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=incr \
  backup
```

Production-style:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=incr \
  --start-fast \
  --process-max=4 \
  backup
```

---

# 22. Backup Information

Check all backups:

```bash
pgbackrest \
  --stanza=pg18lab \
  info
```

Example:

```text
stanza: pg18lab
    status: ok

    db (current)

        full backup: 20261026-123557F
            timestamp start/stop:
            wal start/stop:
            database size:
            repo1 backup size:
```

Important information:

```text
Backup label
Backup type
Start time
Stop time
WAL start
WAL stop
Database size
Backup size
Repository size
```

---

# 23. Backup Label Meaning

FULL:

```text
20261026-123557F
                 ^
                 FULL
```

Differential:

```text
20261026-123557F_20261027-020001D
                                  ^
                                  DIFF
```

Incremental:

```text
20261026-123557F_20261027-120001I
                                  ^
                                  INCR
```

Remember:

```text
F = FULL
D = DIFFERENTIAL
I = INCREMENTAL
```

---

# 24. Check a Specific Backup Set

Example:

```bash
pgbackrest \
  --stanza=pg18lab \
  --set=20261026-123557F \
  info
```

Useful when you want detailed information about one backup set.

---

# 25. Backup Size Meaning

`pgbackrest info` may show:

```text
database size:        4.7GB
database backup size: 4.7GB

repo1:
backup set size:      4.1GB
backup size:          4.1GB
```

For a full backup:

```text
database size
     |
     +-- actual PostgreSQL database size

backup size
     |
     +-- data physically stored in this backup

backup set size
     |
     +-- total repository data needed to restore this backup
```

Compression can make repository size smaller than database size.

---

# 26. Backup Manifest

Every backup has a manifest containing metadata about the backup.

Conceptually:

```text
Backup
   |
   +-- database files
   +-- tablespace information
   +-- file sizes
   +-- checksums
   +-- permissions
   +-- timestamps
   +-- WAL information
   |
   +-- backup.manifest
```

The manifest is important during:

```text
Restore
Delta restore
Backup validation
File comparison
```

Do not manually modify repository backup files or manifests.

---

# 27. Compression

Example configuration:

```ini
compress-type=zst
compress-level=3
```

Concept:

```text
PostgreSQL data
      |
      v
pgBackRest
      |
      v
Compression
      |
      v
Repository
```

Benefits:

```text
Less repository storage
Less network traffic
```

Trade-off:

```text
More CPU usage
```

---

# 28. Backup Resume

pgBackRest supports resuming certain failed backups.

Resume is enabled by default.

Example situation:

```text
5 TB backup

4 TB copied
      |
      X network/storage problem
      |
restart backup
      |
pgBackRest can reuse eligible work
```

To explicitly disable resume:

```bash
--no-resume
```

Normally leave resume enabled.

---

# 29. Archive Timeout

During an online backup, pgBackRest waits for required WAL to be archived.

Example option:

```bash
--archive-timeout=120
```

If required WAL does not reach the archive within the configured timeout, the backup can fail.

Investigate:

```text
archive_command
archive-push
repository connectivity
pg_stat_archiver
spool queue
permissions
```

rather than blindly increasing the timeout.

---

# 30. Backup Retention — Count Based

Example:

```ini
repo1-retention-full=4
```

Meaning:

```text
Keep four FULL backup chains
```

Concept:

```text
FULL1
FULL2
FULL3
FULL4

Take FULL5
     |
     v
FULL1 becomes eligible for expiration
```

When a FULL expires, dependent:

```text
DIFF
INCR
```

backups also expire with it.

---

# 31. Backup Retention — Time Based

For a 14-day policy:

```ini
repo1-retention-full-type=time
repo1-retention-full=14
```

This is useful when the business requirement is expressed as:

```text
Maintain approximately a 14-day recovery window
```

Expiration occurs according to the pgBackRest retention rules after successful backup/expire processing.

---

# 32. Differential Retention

Optional setting:

```ini
repo1-retention-diff=4
```

This controls differential backup retention.

If it is not configured, differential backups can remain until the FULL backup on which they depend expires.

---

# 33. WAL Retention

By default, let pgBackRest retain WAL according to retained backups.

Recommended approach:

```ini
repo1-retention-full-type=time
repo1-retention-full=14
```

and initially do NOT aggressively configure:

```ini
repo1-retention-archive
repo1-retention-archive-type
```

because removing WAL too aggressively can break PITR.

Concept:

```text
FULL
 |
 +-- WAL
 +-- WAL
 +-- WAL
 +-- WAL
 |
 +---- PITR possible
```

If required WAL is removed:

```text
FULL
 |
 +-- WAL
 +-- WAL
 +-- X missing WAL
 +-- WAL
```

PITR beyond the gap can fail.

---

# 34. Expire Old Backups

Normally pgBackRest automatically runs expiration after successful backups.

Manual command:

```bash
pgbackrest \
  --stanza=pg18lab \
  expire
```

---

# 35. Dry-Run Expiration

Before deleting anything, see what pgBackRest intends to expire:

```bash
pgbackrest \
  --stanza=pg18lab \
  --dry-run \
  expire
```

Excellent production habit.

```text
dry-run
   |
   +-- show what would be removed
   |
   +-- nothing actually removed
```

---

# 36. Never Manually Delete Repository WAL

Do NOT:

```bash
rm -rf /postgres/backup/pg18/archive/...
```

Do NOT manually delete backup directories.

Always let:

```text
pgBackRest expire
```

manage repository retention.

Otherwise:

```text
Backup appears available
        +
Required WAL manually deleted
        =
PITR failure
```

---

# 37. PostgreSQL `pg_wal` vs Repository WAL

These are separate.

```text
PGDATA/pg_wal
      |
      | PostgreSQL active/recent WAL
      |
      v
archive-push
      |
      v
pgBackRest Repository
```

PostgreSQL manages/recycles WAL in:

```text
$PGDATA/pg_wal
```

pgBackRest manages archived WAL in:

```text
repository/archive/
```

Never manually delete files from `pg_wal`.

---

# 38. Oracle RMAN Comparison

Oracle:

```rman
BACKUP DATABASE PLUS ARCHIVELOG DELETE INPUT;
```

Concept:

```text
Database backup
       +
Archived redo backup
       +
Delete source archived redo
```

PostgreSQL/pgBackRest:

```text
PostgreSQL backup
       |
       +--------------------+
       |                    |
       v                    v
Physical Backup         archive-push
                            |
                            v
                       Archived WAL
```

Cleanup:

```text
Local pg_wal
    |
    +-- PostgreSQL manages/recycles

Repository backups/WAL
    |
    +-- pgBackRest expire manages
```

There is normally no need for an RMAN-style:

```text
DELETE INPUT
```

command.

---

# 39. Suggested Production Backup Schedule

Example:

```text
Sunday
02:00 FULL

Monday
02:00 DIFF

Tuesday
02:00 DIFF

Wednesday
02:00 DIFF

Thursday
02:00 DIFF

Friday
02:00 DIFF

Saturday
02:00 DIFF

WAL
Continuous archive-push
```

---

# 40. Cron Example

Sunday FULL:

```cron
0 2 * * 0 pgbackrest --stanza=pg18lab --type=full backup
```

Monday-Saturday DIFF:

```cron
0 2 * * 1-6 pgbackrest --stanza=pg18lab --type=diff backup
```

WAL archiving is NOT scheduled with cron.

It is continuous through PostgreSQL:

```ini
archive_command='pgbackrest --stanza=pg18lab archive-push %p'
```

---

# 41. FULL + DIFF + INCR Schedule Example

Another possible strategy:

```text
Sunday 02:00
FULL

Monday 02:00
DIFF

Monday 12:00
INCR

Tuesday 02:00
DIFF

Tuesday 12:00
INCR

...
```

Whether this makes sense depends on:

```text
Database size
Change rate
Backup window
Restore-time objective
Repository capacity
Network bandwidth
```

---

# 42. Backup From Standby

In a primary/standby environment pgBackRest can be configured to perform much of the backup work from a standby.

Concept:

```text
             PRIMARY
                |
                | streaming replication
                v
             STANDBY
                |
                | pgBackRest backup
                v
            Repository
```

This can reduce backup read workload on the primary.

Typical configuration option:

```ini
backup-standby=y
```

This requires a correctly configured multi-host pgBackRest environment.

---

# 43. Multiple Repositories

pgBackRest can be configured with multiple repositories.

Example:

```ini
repo1-path=/backup/local
repo2-path=/backup/remote
```

Concept:

```text
PostgreSQL
    |
    +------ Repo1
    |
    +------ Repo2
```

A command can target a repository using the configured repo selection option when appropriate.

Example:

```bash
pgbackrest \
  --stanza=pg18lab \
  --repo=1 \
  --type=full \
  backup
```

---

# 44. Useful Monitoring Commands

Check pgBackRest:

```bash
pgbackrest \
  --stanza=pg18lab \
  check
```

Check backups:

```bash
pgbackrest \
  --stanza=pg18lab \
  info
```

Check running processes:

```bash
ps -ef | grep pgbackrest
```

Check backup logs:

```bash
ls -ltr /var/log/pgbackrest/
```

Follow log:

```bash
tail -f /var/log/pgbackrest/pg18lab-backup.log
```

Check repository size:

```bash
du -sh /postgres/backup/pg18
```

Check backup area:

```bash
du -sh /postgres/backup/pg18/backup/pg18lab
```

Check WAL archive:

```bash
du -sh /postgres/backup/pg18/archive/pg18lab
```

---

# 45. PostgreSQL Checks Before Backup

Check database status:

```sql
SELECT pg_is_in_recovery();
```

On primary:

```text
f
```

Check archiver:

```sql
SELECT *
FROM pg_stat_archiver;
```

Check current WAL:

```sql
SELECT pg_current_wal_lsn();
```

Check current WAL filename:

```sql
SELECT pg_walfile_name(pg_current_wal_lsn());
```

---

# 46. Check Disk Space

Before a large backup:

```bash
df -h /postgres/data/pg18
df -h /postgres/tbs/pg18
df -h /postgres/backup/pg18
```

Also:

```bash
du -sh /postgres/data/pg18
du -sh /postgres/tbs/pg18
du -sh /postgres/backup/pg18
```

Repository capacity must account for:

```text
FULL backups
+
DIFF backups
+
INCR backups
+
Archived WAL
+
Retention window
+
Growth
```

---

# 47. Backup Success Verification

After backup:

```bash
pgbackrest \
  --stanza=pg18lab \
  info
```

Look for:

```text
status: ok

full backup:
or
diff backup:
or
incr backup:

timestamp start/stop
wal start/stop
database size
backup size
```

Also check the command ended with:

```text
backup command end: completed successfully
```

---

# 48. Backup Is Not Enough — Test Restore

A successful backup command is only one part of backup management.

Production mindset:

```text
Backup
   |
   +-- Did backup complete?
   |
   +-- Is required WAL available?
   |
   +-- Is repository healthy?
   |
   +-- Can we actually restore?
   |
   +-- Can PITR succeed?
```

Regular restore testing is essential.

---

# 49. FULL Backup Example for Our Lab

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  backup
```

Check:

```bash
pgbackrest \
  --stanza=pg18lab \
  info
```

---

# 50. Differential Backup Example

Generate some changes:

```sql
INSERT INTO pitr_test
VALUES (10,'DIFF BACKUP TEST',clock_timestamp());

COMMIT;
```

Then:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=diff \
  backup
```

Check:

```bash
pgbackrest \
  --stanza=pg18lab \
  info
```

---

# 51. Incremental Backup Example

Generate more changes:

```sql
INSERT INTO pitr_test
VALUES (20,'INCREMENTAL TEST',clock_timestamp());

COMMIT;
```

Then:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=incr \
  backup
```

Check:

```bash
pgbackrest \
  --stanza=pg18lab \
  info
```

---

# 52. Backup Dependency Example

Suppose:

```text
FULL
20261026-020000F

DIFF
20261026-020000F_20261027-020000D

INCR
20261026-020000F_20261027-120000I
```

Conceptually:

```text
FULL
  |
  +---- DIFF
  |
  +---- INCR / references required backup data
```

pgBackRest automatically understands backup dependencies during restore.

You do not manually restore each backup one by one.

For example, you select:

```bash
--set=<backup-label>
```

and pgBackRest retrieves the required referenced backup files.

---

# 53. Important Difference From Oracle RMAN Incremental Restore

Do NOT think:

```text
Restore FULL
then restore DIFF
then restore INCR manually
```

pgBackRest handles the backup references itself.

You simply request the desired backup set:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --set=<backup-label>
```

pgBackRest determines which underlying backup files are required.

---

# 54. Most Important Commands

## Check

```bash
pgbackrest --stanza=pg18lab check
```

## FULL

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  backup
```

## Differential

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=diff \
  backup
```

## Incremental

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=incr \
  backup
```

## Backup Information

```bash
pgbackrest \
  --stanza=pg18lab \
  info
```

## Expire Dry Run

```bash
pgbackrest \
  --stanza=pg18lab \
  --dry-run \
  expire
```

## Expire

```bash
pgbackrest \
  --stanza=pg18lab \
  expire
```

---

# 55. Interview Questions — Quick Answers

## What is FULL backup?

Copies the complete PostgreSQL cluster and does not depend on a previous backup.

---

## What is Differential backup?

Copies data changed since the most recent FULL backup.

```text
FULL → DIFF
```

---

## What is Incremental backup?

Copies data changed since the previous successful backup.

```text
FULL → DIFF → INCR
```

or:

```text
FULL → INCR → INCR
```

---

## Does pgBackRest backup WAL?

WAL is continuously archived separately through:

```text
archive_command
        ↓
archive-push
        ↓
repository
```

The backup process also ensures that WAL required to make the backup consistent is present in the archive.

---

## Does pgBackRest require PostgreSQL shutdown?

No.

Normal pgBackRest backups are:

```text
ONLINE
```

---

## Can we backup only one PostgreSQL database with pgBackRest?

No.

pgBackRest performs a physical cluster-level backup.

For individual databases use logical tools such as:

```text
pg_dump
pg_restore
```

---

## Does pgBackRest backup tablespaces?

Yes.

Tablespaces belonging to the PostgreSQL cluster are part of the physical backup.

---

## Who deletes old backups?

```text
pgBackRest expire
```

according to retention policy.

---

## Who deletes/recycles WAL from `pg_wal`?

PostgreSQL manages the local `pg_wal` area.

Do not manually delete WAL from `pg_wal`.

---

## Who deletes archived WAL from pgBackRest repository?

```text
pgBackRest expire
```

according to repository retention requirements.

---

## Does `--delta` affect backup?

`--delta` is primarily a RESTORE optimization.

It is not a backup type.

Backup types are:

```text
full
diff
incr
```

---

# 56. Production Backup Flow

```text
             PostgreSQL
                 |
        +--------+--------+
        |                 |
        |                 |
   Physical Backup       WAL
        |                 |
        |            archive-push
        |                 |
        +--------+--------+
                 |
                 v
           pgBackRest Repo
                 |
        +--------+--------+
        |                 |
     backup/            archive/
        |                 |
     FULL/DIFF/INCR      WAL
        |                 |
        +--------+--------+
                 |
                 v
              expire
                 |
                 v
       Retention Management
```

---

# 57. Golden Rules

Remember:

```text
FULL
= Entire cluster

DIFF
= Changes since last FULL

INCR
= Changes since last successful backup
```

And:

```text
Backup
    +
Required WAL
    =
Recoverable PostgreSQL database
```

Also:

```text
archive-push
    =
Put WAL into repository

backup
    =
Take physical database backup

expire
    =
Remove backups/WAL according to retention

info
    =
Show backup/repository information

check
    =
Validate pgBackRest configuration/archive operation

restore
    =
Recover the database
```

Most important production rule:

```text
A backup is useful only when it can be successfully restored.
```

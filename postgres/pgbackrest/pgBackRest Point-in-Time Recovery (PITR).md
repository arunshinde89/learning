# pgBackRest Point-in-Time Recovery (PITR)

## Environment Example

```text
PostgreSQL Version : 18
pgBackRest Stanza  : pg18lab
PGDATA             : /postgres/data/pg18
Repository         : /postgres/backup/pg18
```

---

# 1. Basic PITR Concept

A pgBackRest recovery has three separate concepts:

```text
--set
    |
    +--> Which backup should be restored?

--type + --target
    |
    +--> Where should WAL replay stop?

--delta
    |
    +--> How should the physical files be restored?
```

Remember:

```text
--set    = Backup selection
--target = Recovery stopping point
--delta  = Restore optimization
```

`--delta` does NOT mean PITR.

---

# 2. Standard Recovery Workflow

Always stop PostgreSQL first:

```bash
systemctl stop postgresql-18
```

Check available backups:

```bash
pgbackrest --stanza=pg18lab info
```

Perform the required restore.

Then start PostgreSQL:

```bash
systemctl start postgresql-18
```

Monitor recovery:

```bash
journalctl -u postgresql-18.service -f
```

After recovery:

```sql
SELECT pg_is_in_recovery();
```

After promotion, expected:

```text
f
```

---

# 3. Time-Based PITR

Use this when you know the time before a problem occurred.

Example:

```text
14:00:00  Database OK

14:05:00  <-- Recovery target

14:10:00  Bad INSERT

14:15:00  DROP TABLE
```

Restore:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 14:05:00+01" \
  --target-action=promote \
  --delta
```

Recommended format:

```text
YYYY-MM-DD HH:MI:SS+TIMEZONE
```

Example:

```text
2026-10-26 14:05:00+01
```

To capture time before making a test change:

```sql
SELECT clock_timestamp();
```

---

# 4. LSN-Based PITR

Use this when you know the exact PostgreSQL WAL LSN.

Capture LSN:

```sql
SELECT pg_current_wal_lsn();
```

Example:

```text
2/2D000158
```

Restore:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=lsn \
  --target="2/2D000158" \
  --target-action=promote \
  --delta
```

Important:

This is an LSN:

```text
2/2D000158
```

This is a WAL filename:

```text
00000001000000020000002D
```

Do NOT use a WAL filename with:

```text
--type=lsn
```

Wrong:

```bash
--type=lsn --target=00000001000000020000002D
```

Correct:

```bash
--type=lsn --target="2/2D000158"
```

---

# 5. Named Restore Point PITR

This is one of the cleanest methods for planned activities.

Before a risky change:

```sql
SELECT pg_create_restore_point('before_table_drop');
```

Perform the change:

```sql
DROP TABLE backup_test_5gb;
```

Force WAL switching so the required WAL can be archived:

```sql
SELECT pg_switch_wal();
```

Restore to the named point:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=name \
  --target="before_table_drop" \
  --target-action=promote \
  --delta
```

Concept:

```text
Normal activity
      |
      v
before_table_drop    <-- Restore point
      |
      v
DROP TABLE
      |
      v
More transactions
```

Recovery stops at:

```text
before_table_drop
```

The DROP is not replayed.

---

# 6. Transaction ID / XID-Based PITR

You can also recover based on a PostgreSQL transaction ID.

Find current transaction ID:

```sql
SELECT txid_current();
```

Example:

```text
123456
```

Restore:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=xid \
  --target="123456" \
  --target-action=promote \
  --delta
```

Use this carefully.

Transaction IDs are allocated when transactions begin, but transactions do not necessarily commit in exactly the same order.

For normal DBA recovery, time, restore point, or LSN recovery is often easier to reason about.

---

# 7. Immediate Recovery

Immediate recovery means:

> Recover only until the restored backup reaches a consistent state.

Example:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --set=20261026-123557F \
  --type=immediate \
  --target-action=promote \
  --delta
```

Concept:

```text
Backup START
     |
     |
Backup STOP
     |
     +---- database becomes consistent
               ^
               |
              STOP
```

It does NOT replay all WAL up to the latest available point.

This is useful when you want a state approximately corresponding to the completion of a particular backup.

---

# 8. Default / Full Recovery

If you run:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --set=20261026-123557F \
  --delta
```

there is no PITR target.

PostgreSQL will normally replay WAL toward the end of the available archive stream.

Example:

```text
Backup
  |
  +--- WAL
  |
  +--- INSERT
  |
  +--- WAL
  |
  +--- DROP TABLE
  |
  +--- WAL
  |
  +--- Latest available WAL
```

Even though the table existed in the backup, recovery may replay:

```sql
DROP TABLE ...
```

and the table will disappear again.

Therefore:

```text
--set alone != PITR
```

`--set` only tells pgBackRest which backup to restore.

---

# 9. Restore a Specific Backup + Time PITR

This is useful when you want complete control.

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --set=20261026-123557F \
  --type=time \
  --target="2026-10-26 13:20:45+01" \
  --target-action=promote \
  --delta
```

Meaning:

```text
--set
    ↓
Restore this backup

--type=time
    ↓
Perform time-based recovery

--target
    ↓
Stop WAL replay here
```

---

# 10. Restore Specific Backup + LSN

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --set=20261026-123557F \
  --type=lsn \
  --target="2/2D000158" \
  --target-action=promote \
  --delta
```

---

# 11. Restore Specific Backup + Restore Point

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --set=20261026-123557F \
  --type=name \
  --target="before_table_drop" \
  --target-action=promote \
  --delta
```

---

# 12. Inclusive vs Exclusive Recovery

By default, PostgreSQL recovery targets are inclusive where applicable.

Meaning:

```text
--target="14:10:00"

Transaction committed exactly at 14:10:00
may be included.
```

To stop BEFORE the target:

```bash
--target-exclusive
```

Example:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 14:10:00+01" \
  --target-exclusive \
  --target-action=promote \
  --delta
```

Supported with:

```text
time
lsn
xid
```

Think:

```text
Normal/default:

--------- Transaction X --------|
                                TARGET
                                  |
                                  INCLUDED


--target-exclusive:

--------- Transaction X --------|
                                TARGET
                                  |
                                EXCLUDED
```

---

# 13. Target Action

When PostgreSQL reaches the requested PITR target, you decide what should happen.

## Promote

Most common for our lab:

```bash
--target-action=promote
```

Example:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 14:00:00+01" \
  --target-action=promote \
  --delta
```

PostgreSQL reaches target and becomes a normal writable database.

---

## Pause

```bash
--target-action=pause
```

PostgreSQL pauses when target is reached.

Useful when you want to inspect recovery state before promotion.

---

## Shutdown

```bash
--target-action=shutdown
```

PostgreSQL shuts down after reaching the target.

Useful in controlled recovery workflows.

---

# 14. Timeline Recovery

After PITR + promotion, PostgreSQL creates a new timeline.

Example:

```text
Timeline 1
------------------------------------------------>
                         |
                         | PITR
                         v
                    Timeline 2
                    ----------------------------->
```

WAL names show timeline in their first 8 characters.

Example:

```text
00000001000000020000002D
^^^^^^^^
Timeline 1
```

Example:

```text
00000002000000020000004F
^^^^^^^^
Timeline 2
```

---

# 15. Recover Using Latest Timeline

PostgreSQL normally uses the latest available timeline.

Explicit example:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 14:00:00+01" \
  --target-timeline=latest \
  --target-action=promote \
  --delta
```

Use when recovery needs to follow PITR-created timelines.

---

# 16. Recover Using Current Backup Timeline

```bash
--target-timeline=current
```

Example:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --set=20261026-123557F \
  --type=time \
  --target="2026-10-26 13:00:00+01" \
  --target-timeline=current \
  --target-action=promote \
  --delta
```

Meaning:

> Stay on the timeline that was current when the backup was created.

Useful in complex re-recovery cases.

Do not use this blindly after previous PITR operations.

---

# 17. Recover to Specific Timeline

You can explicitly specify a timeline:

```bash
--target-timeline=2
```

Example:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 13:30:00+01" \
  --target-timeline=2 \
  --target-action=promote \
  --delta
```

Mostly used in advanced recovery scenarios.

---

# 18. Delta Restore

Normal restore:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 14:00:00+01" \
  --target-action=promote
```

Delta restore:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 14:00:00+01" \
  --target-action=promote \
  --delta
```

Both recover to the SAME logical PITR point.

Difference:

```text
Without --delta

Backup
   |
   +---- Restore entire PGDATA
   |
   +---- WAL replay
   |
   +---- PITR target


With --delta

Existing PGDATA
   |
   +---- Compare files
   |
   +---- Reuse matching files
   |
   +---- Restore changed/missing files
   |
   +---- WAL replay
   |
   +---- Same PITR target
```

For large databases, delta restore can significantly reduce physical restore time.

---

# 19. Delete PGDATA vs Delta Restore

## Full physical restore

```bash
systemctl stop postgresql-18
```

Move/remove old PGDATA according to operational procedure.

Then:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 14:00:00+01" \
  --target-action=promote
```

Entire backup must be restored.

---

## Delta restore

```bash
systemctl stop postgresql-18
```

Then:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 14:00:00+01" \
  --target-action=promote \
  --delta
```

Existing matching files can be reused.

---

# 20. Critical WAL Requirement

PITR requires:

```text
Base Backup
     +
All required WAL
     =
Successful PITR
```

Example:

```text
Backup
   |
   +--- WAL 4D
   +--- WAL 4E
   +--- WAL 4F
   +--- WAL 50
            |
          Target
```

If PostgreSQL requires:

```text
WAL 50
```

but the pgBackRest repository only has:

```text
WAL 4F
```

recovery may fail with:

```text
recovery ended before configured recovery target was reached
```

You cannot reconstruct a missing WAL segment simply by changing the restore command.

---

# 21. Force Current WAL to Archive Before PITR Testing

After creating your unwanted data/change:

```sql
SELECT pg_switch_wal();
```

Check archiving:

```sql
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal
FROM pg_stat_archiver;
```

Check pgBackRest:

```bash
pgbackrest --stanza=pg18lab check
```

Check archive range:

```bash
pgbackrest --stanza=pg18lab info
```

---

# 22. Recommended PITR Practice Test

Create table:

```sql
CREATE TABLE pitr_test
(
    id int,
    message text,
    created_at timestamptz DEFAULT clock_timestamp()
);
```

Insert good data:

```sql
INSERT INTO pitr_test
VALUES
(1,'GOOD DATA',clock_timestamp());

COMMIT;
```

Take backup:

```bash
pgbackrest \
  --stanza=pg18lab \
  --type=full \
  backup
```

Create restore point:

```sql
SELECT pg_create_restore_point('before_bad_change');
```

Insert bad data:

```sql
INSERT INTO pitr_test
VALUES
(2,'BAD DATA',clock_timestamp());

COMMIT;
```

Drop table:

```sql
DROP TABLE pitr_test;
```

Force WAL switch:

```sql
SELECT pg_switch_wal();
```

Check archive:

```bash
pgbackrest --stanza=pg18lab check
```

Stop PostgreSQL:

```bash
systemctl stop postgresql-18
```

Recover:

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=name \
  --target="before_bad_change" \
  --target-action=promote \
  --delta
```

Start PostgreSQL:

```bash
systemctl start postgresql-18
```

Verify:

```sql
SELECT *
FROM pitr_test;
```

Expected:

```text
1 | GOOD DATA
```

The bad INSERT and DROP should not exist.

---

# 23. PITR Target Types — Quick Reference

| Recovery Method | pgBackRest Option | Target Example |
|---|---|---|
| Time | `--type=time` | `2026-10-26 14:10:00+01` |
| LSN | `--type=lsn` | `2/2D000158` |
| Restore Point | `--type=name` | `before_table_drop` |
| Transaction ID | `--type=xid` | `123456` |
| Backup consistency point | `--type=immediate` | No `--target` |
| Latest available WAL | Default | No target |

---

# 24. Most Useful Commands

## Time PITR

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=time \
  --target="2026-10-26 14:00:00+01" \
  --target-action=promote \
  --delta
```

## LSN PITR

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=lsn \
  --target="2/2D000158" \
  --target-action=promote \
  --delta
```

## Restore Point PITR

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=name \
  --target="before_table_drop" \
  --target-action=promote \
  --delta
```

## XID PITR

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --type=xid \
  --target="123456" \
  --target-action=promote \
  --delta
```

## Immediate Recovery

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --set=20261026-123557F \
  --type=immediate \
  --target-action=promote \
  --delta
```

## Full Recovery to Latest WAL

```bash
pgbackrest restore \
  --stanza=pg18lab \
  --set=20261026-123557F \
  --delta
```

---

# 25. Interview-Level Summary

## `--set`

Selects the backup used as the recovery starting point.

```text
--set=20261026-123557F
```

---

## `--type`

Defines the recovery method.

```text
time
lsn
name
xid
immediate
default
```

---

## `--target`

Defines where WAL replay stops.

Examples:

```text
Time:
2026-10-26 14:00:00+01

LSN:
2/2D000158

Restore point:
before_table_drop

XID:
123456
```

---

## `--target-action`

Defines what PostgreSQL does when the target is reached.

```text
promote
pause
shutdown
```

---

## `--target-exclusive`

Stops before the specified target.

Applicable to:

```text
time
lsn
xid
```

---

## `--target-timeline`

Controls which PostgreSQL timeline recovery follows.

Examples:

```text
latest
current
2
3
```

---

## `--delta`

Optimizes physical restore.

It does NOT define the PITR point.

---

# Golden Rule

```text
          BACKUP
             |
             |  --set
             v
      Physical Restore
             |
             |  --delta controls HOW
             v
         WAL Replay
             |
             |  --type
             |  --target
             v
        PITR TARGET
             |
             | --target-action
             v
        PROMOTE / PAUSE
```

The most important formula to remember:

```text
Backup + Required WAL + Recovery Target = PITR
```

And:

```text
--set    = WHERE RECOVERY STARTS
--target = WHERE RECOVERY STOPS
--delta  = HOW FILES ARE RESTORED
```

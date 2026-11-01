# PostgreSQL 18 Physical Streaming Replication Setup

## 1. Architecture

This lab configures PostgreSQL **physical streaming replication** between one primary and one standby.

```text
                    PostgreSQL Streaming Replication

              PRIMARY                        STANDBY
              as-alpha                       as-beta
           192.168.136.10                 192.168.136.11
                 |                              |
                 | WAL Sender                   | WAL Receiver
                 |                              |
                 +---------- WAL Stream -------->
                                                |
                                                | WAL Replay
                                                v
                                          Read-Only Standby
```

### Environment

| Component | Primary | Standby |
|---|---|---|
| Hostname | `as-alpha` | `as-beta` |
| IP Address | `192.168.136.10` | `192.168.136.11` |
| PostgreSQL | 18 | 18 |
| Port | `5432` | `5432` |
| PGDATA | `/u01/18/data` | `/u01/18/data` |
| Role | Primary | Physical Standby |

---

# 2. How Physical Streaming Replication Works

PostgreSQL physical replication is based on **WAL — Write Ahead Log**.

```text
Application transaction
        |
        v
PRIMARY DATABASE
        |
        v
WAL generated
        |
        v
pg_wal
        |
        v
WAL Sender
        |
        | TCP/IP
        v
WAL Receiver
        |
        v
Standby pg_wal
        |
        v
WAL Replay
        |
        v
STANDBY DATABASE
```

The primary does not send individual SQL statements such as:

```sql
INSERT ...
UPDATE ...
DELETE ...
```

Instead, it sends WAL records representing the physical database changes.

---

# 3. Primary Server Configuration

Server:

```text
Hostname : as-alpha
IP       : 192.168.136.10
PGDATA   : /u01/18/data
Port     : 5432
```

## 3.1 Create Dedicated Replication User

Connect as a PostgreSQL superuser:

```bash
psql
```

Create a dedicated replication role:

```sql
CREATE ROLE replication
WITH
    LOGIN
    REPLICATION
    PASSWORD '<REPL_PASSWORD>';
```

Check:

```sql
\du
```

Example:

```text
 Role name   | Attributes
-------------+-------------------------------------------------------
 postgres    | Superuser, Create role, Create DB, Replication
 replication | Replication
```

A dedicated replication user is preferred instead of using the `postgres` superuser.

---

# 4. Configure `postgresql.conf`

Edit:

```bash
vi /u01/18/data/postgresql.conf
```

Configure:

```conf
listen_addresses = '*'

wal_level = replica

max_wal_senders = 10

wal_keep_size = '1GB'

wal_log_hints = on
```

## Parameter Explanation

### `listen_addresses`

```conf
listen_addresses = '*'
```

Allows PostgreSQL to listen on network interfaces.

Access is still controlled by `pg_hba.conf` and network/firewall rules.

---

### `wal_level`

```conf
wal_level = replica
```

Controls how much information PostgreSQL writes into WAL.

`replica` provides enough WAL information for:

- Physical streaming replication
- WAL archiving
- PITR
- Hot standby

`replica` is the default in PostgreSQL 18, but it is useful to explicitly understand this requirement.

---

### `max_wal_senders`

```conf
max_wal_senders = 10
```

Maximum number of simultaneous WAL sender connections.

WAL sender processes are used by:

- Standby servers
- `pg_basebackup`
- Other physical replication clients

---

### `wal_keep_size`

```conf
wal_keep_size = '1GB'
```

Requests PostgreSQL to retain at least approximately 1 GB of recent WAL so a temporarily delayed standby has a better chance of catching up.

This is **not a guarantee that a standby can stay disconnected indefinitely**.

For stronger WAL retention control, physical replication slots can be used.

---

### `wal_log_hints`

```conf
wal_log_hints = on
```

This parameter is **not required for streaming replication**.

It is useful when learning `pg_rewind`.

`pg_rewind` requires either:

```text
data checksums enabled

OR

wal_log_hints = on
```

Therefore it is useful to enable in this HA lab because later we will test failover and `pg_rewind`.

---

# 5. Configure `pg_hba.conf`

Edit:

```bash
vi /u01/18/data/pg_hba.conf
```

## Lab Configuration

For a simple isolated lab:

```conf
# TYPE  DATABASE       USER           ADDRESS              METHOD

host    replication    replication    192.168.136.11/32    trust
```

This single rule is enough for physical replication.

We do **not** need:

```conf
host all         postgres      192.168.136.11/32 trust
host replication postgres      192.168.136.11/32 trust
host all         replication   192.168.136.11/32 trust
```

unless normal SQL connections are also specifically required.

The important replication entry is:

```conf
host replication replication 192.168.136.11/32 trust
```

Meaning:

```text
host
 |
 +-- TCP connection

replication
 |
 +-- physical replication connection

replication
 |
 +-- PostgreSQL replication user

192.168.136.11/32
 |
 +-- only standby IP is allowed

trust
 |
 +-- no password authentication
```

---

# 6. Production Authentication

`trust` is acceptable for this isolated lab but should generally **not** be used for production TCP replication.

Production example:

```conf
host    replication    replication    192.168.136.11/32    scram-sha-256
```

On the standby, a `.pgpass` file can be used instead of putting the password directly inside `primary_conninfo`.

Example:

```bash
vi /var/lib/pgsql/.pgpass
```

```text
192.168.136.10:5432:replication:replication:<REPL_PASSWORD>
```

Permissions must be:

```bash
chmod 600 /var/lib/pgsql/.pgpass
chown postgres:postgres /var/lib/pgsql/.pgpass
```

---

# 7. Restart Primary PostgreSQL

Because some replication parameters require PostgreSQL startup/restart:

```bash
pg_ctl -D /u01/18/data restart
```

Check status:

```bash
pg_ctl -D /u01/18/data status
```

Validate parameters:

```sql
SHOW listen_addresses;
SHOW wal_level;
SHOW max_wal_senders;
SHOW wal_keep_size;
SHOW wal_log_hints;
```

Example:

```text
listen_addresses | *
wal_level         | replica
max_wal_senders   | 10
wal_keep_size     | 1GB
wal_log_hints     | on
```

---

# 8. Validate `pg_hba.conf`

Optional but useful:

```sql
SELECT
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules;
```

Verify there are no errors.

---

# 9. Prepare Standby Server

Server:

```text
Hostname : as-beta
IP       : 192.168.136.11
PGDATA   : /u01/18/data
Port     : 5432
```

Make sure:

- PostgreSQL 18 is installed.
- PostgreSQL is stopped.
- `PGDATA` is empty.
- PostgreSQL OS user owns the directory.
- Standby can reach primary port `5432`.

Test network connectivity if required:

```bash
nc -zv 192.168.136.10 5432
```

or:

```bash
telnet 192.168.136.10 5432
```

---

# 10. Prepare Standby PGDATA

Stop PostgreSQL if running:

```bash
pg_ctl -D /u01/18/data stop
```

Create the data directory if required:

```bash
mkdir -p /u01/18/data
chown postgres:postgres /u01/18/data
chmod 700 /u01/18/data
```

The directory must be empty before running `pg_basebackup`.

Be very careful before deleting anything from an existing data directory.

---

# 11. Create Standby Using `pg_basebackup`

Run this command **on `as-beta`** as the PostgreSQL OS user:

```bash
/usr/pgsql-18/bin/pg_basebackup \
    -h 192.168.136.10 \
    -p 5432 \
    -U replication \
    -D /u01/18/data \
    -Fp \
    -X stream \
    -P \
    -R
```

Example:

```text
40136/40136 kB (100%), 1/1 tablespace
```

---

# 12. `pg_basebackup` Options Explained

## `-h`

```text
-h 192.168.136.10
```

Primary database hostname/IP.

---

## `-p`

```text
-p 5432
```

Primary PostgreSQL port.

---

## `-U`

```text
-U replication
```

Replication user.

The user must have:

```text
LOGIN
REPLICATION
```

privileges.

---

## `-D`

```text
-D /u01/18/data
```

Destination PGDATA directory on the standby.

---

## `-Fp`

Same as:

```text
--format=plain
```

Creates the standby in normal PostgreSQL data-directory format.

This means the resulting directory can directly be used as `PGDATA`.

---

## `-X stream`

Same as:

```text
--wal-method=stream
```

WAL generated while the backup is running is streamed simultaneously from the primary.

Conceptually:

```text
                PRIMARY

             Base Backup
                 |
                 +-----------> Standby PGDATA
                 |
             WAL generated
                 |
                 +-----------> Standby pg_wal
```

`-X stream` is preferred for this lab.

It uses a second replication connection for WAL streaming.

---

# 13. `-X fetch` vs `-X stream`

The original test used:

```bash
-X fetch
```

Example:

```bash
pg_basebackup \
-D /u01/18/data \
-X fetch \
-P \
-h 192.168.136.10 \
-U replication \
-R
```

This works.

However, `fetch` collects the required WAL **at the end of the backup**.

Therefore the primary must retain all required WAL until the backup finishes.

If WAL is recycled before the backup completes, the backup can fail.

For this reason I prefer:

```text
-X stream
```

for normal standby creation.

Comparison:

| Option | Behavior |
|---|---|
| `-X fetch` | Collect required WAL after base backup finishes |
| `-X stream` | Stream WAL while base backup is running |
| Recommended for this lab | `-X stream` |

`stream` is also the default WAL method in current PostgreSQL.

---

# 14. `-P`

```text
-P
```

Displays backup progress.

Example:

```text
40136/40136 kB (100%)
```

---

# 15. `-R` — Very Important

```text
-R
```

Same as:

```text
--write-recovery-conf
```

This performs two important operations.

### 1. Creates:

```text
$PGDATA/standby.signal
```

### 2. Adds replication connection information to:

```text
$PGDATA/postgresql.auto.conf
```

This makes the copied PostgreSQL cluster start as a standby.

---

# 16. Check Files After `pg_basebackup`

After the backup completes:

```bash
cd /u01/18/data
```

Check:

```bash
ls -l standby.signal
```

You should see:

```text
standby.signal
```

Check:

```bash
cat postgresql.auto.conf
```

Example:

```conf
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.

primary_conninfo = 'user=replication host=192.168.136.10 port=5432'
```

Depending on the authentication configuration, additional parameters may appear.

---

# 17. Meaning of `primary_conninfo`

`primary_conninfo` tells the standby:

> Which upstream PostgreSQL server should I connect to for streaming WAL?

For example:

```conf
primary_conninfo =
'user=replication host=192.168.136.10 port=5432'
```

Architecture:

```text
as-beta
Standby
   |
   | primary_conninfo
   |
   +------------------------>
                              as-alpha
                              192.168.136.10
                              port 5432
```

---

# 18. Meaning of `standby.signal`

PostgreSQL checks for:

```text
$PGDATA/standby.signal
```

during startup.

If the file exists, PostgreSQL starts in **standby/recovery mode**.

Therefore:

```text
standby.signal exists
        |
        v
PostgreSQL starts
        |
        v
Standby mode
        |
        v
Receive/replay WAL
```

---

# 19. Start the Standby

On `as-beta`:

```bash
pg_ctl -D /u01/18/data start
```

Example:

```text
waiting for server to start....

LOG: redirecting log output to logging collector process

done
server started
```

---

# 20. OS-Level Validation

## Primary

Check WAL sender:

```bash
ps -ef | grep '[w]alsender'
```

Example:

```text
postgres  8310  ... postgres: walsender replication 192.168.136.11 streaming
```

This confirms the primary has a WAL sender process connected to the standby.

Architecture:

```text
PRIMARY
   |
   +-- walsender
          |
          | WAL
          v
```

---

## Standby

Check WAL receiver:

```bash
ps -ef | grep '[w]alreceiver'
```

Example:

```text
postgres  7788 ... postgres: walreceiver streaming
```

This confirms the standby has established a streaming connection.

```text
                 WAL
                  |
                  v
STANDBY
   |
   +-- walreceiver
```

---

# 21. Database-Level Validation on Primary

Connect to primary:

```bash
psql
```

Instead of using:

```sql
SELECT * FROM pg_stat_replication;
```

a more useful DBA query is:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

Example:

```text
pid              | 8310
usename          | replication
application_name | walreceiver
client_addr      | 192.168.136.11
state            | streaming
sync_state       | async
sent_lsn         | 0/3000168
write_lsn        | 0/3000168
flush_lsn        | 0/3000168
replay_lsn       | 0/3000168
```

---

# 22. Understanding the Four Important LSN Values

These four columns are extremely important when troubleshooting replication.

```text
sent_lsn
write_lsn
flush_lsn
replay_lsn
```

## `sent_lsn`

```text
sent_lsn = 0/3000168
```

Last WAL location that the **primary WAL sender has sent** to the standby.

---

## `write_lsn`

```text
write_lsn = 0/3000168
```

Last WAL location that the **standby has received and written**.

Think:

```text
Network
   |
   v
Standby WAL receiver
   |
   v
write
```

---

## `flush_lsn`

```text
flush_lsn = 0/3000168
```

Last WAL location that the standby has **flushed durably to storage**.

Therefore:

```text
write_lsn
    |
    v
OS write
    |
    v
flush_lsn
    |
    v
durable storage
```

---

## `replay_lsn`

```text
replay_lsn = 0/3000168
```

Last WAL location that the standby has **replayed/applied to the standby database**.

```text
WAL received
     |
     v
write_lsn
     |
     v
flush_lsn
     |
     v
WAL replay
     |
     v
replay_lsn
     |
     v
Changes visible on standby
```

---

# 23. Replication Flow Using LSNs

Complete flow:

```text
PRIMARY
                          STANDBY

WAL generated
     |
     v
sent_lsn
     |
     | Network
     +------------------------>
                              |
                              v
                          write_lsn
                              |
                              v
                          flush_lsn
                              |
                              v
                          WAL replay
                              |
                              v
                          replay_lsn
```

If:

```text
sent_lsn   = 0/3000168
write_lsn  = 0/3000168
flush_lsn  = 0/3000168
replay_lsn = 0/3000168
```

then, at that instant, the standby has replayed everything that the WAL sender reported as sent.

---

# 24. Better Replication Lag Query

On the primary:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(sent_lsn, replay_lsn)
    ) AS sent_to_replay_lag
FROM pg_stat_replication;
```

Example:

```text
application_name    | walreceiver
client_addr         | 192.168.136.11
state               | streaming
sync_state          | async
sent_to_replay_lag  | 0 bytes
```

`0 bytes` means all WAL sent to the standby has already been replayed.

---

# 25. Asynchronous Replication

Current output:

```text
sync_state = async
```

means this standby is configured for **asynchronous replication**.

In asynchronous replication:

```text
Application
     |
     v
PRIMARY commit
     |
     +----> client can receive COMMIT success
     |
     +----> WAL travels to standby
```

The primary does not normally wait for the standby before acknowledging every commit.

Later, synchronous replication can be configured using:

```text
synchronous_standby_names
```

---

# 26. Validate Standby Is in Recovery

On `as-beta`:

```sql
SELECT pg_is_in_recovery();
```

Expected:

```text
 pg_is_in_recovery
-------------------
 t
```

Meaning:

```text
t = standby/recovery mode

f = primary/read-write mode
```

This is one of the quickest commands for identifying whether a PostgreSQL instance is currently a standby.

---

# 27. Validate WAL Receiver

On standby:

```sql
SELECT
    pid,
    status,
    receive_start_lsn,
    written_lsn,
    flushed_lsn,
    latest_end_lsn,
    latest_end_time,
    slot_name,
    sender_host,
    sender_port
FROM pg_stat_wal_receiver;
```

Expected:

```text
status      | streaming
sender_host | 192.168.136.10
sender_port | 5432
```

This tells us that:

```text
as-beta
    |
walreceiver
    |
    +-------------> as-alpha:5432
```

is active.

---

# 28. Standby Receive vs Replay Lag

On standby:

```sql
SELECT
    pg_last_wal_receive_lsn() AS receive_lsn,
    pg_last_wal_replay_lsn() AS replay_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_last_wal_receive_lsn(),
            pg_last_wal_replay_lsn()
        )
    ) AS receive_replay_lag;
```

Example:

```text
 receive_lsn | replay_lsn | receive_replay_lag
-------------+------------+--------------------
 0/3000168   | 0/3000168  | 0 bytes
```

This tells us whether WAL is being received faster than PostgreSQL can replay it.

---

# 29. Do Not Rely Only on Timestamp Lag

This query is useful:

```sql
SELECT
    now(),
    pg_last_xact_replay_timestamp(),
    now() - pg_last_xact_replay_timestamp();
```

But it can be misleading when the primary is idle.

For example:

```text
Last transaction replayed : 15:00
Current time              : 15:40
```

It may show:

```text
40 minutes
```

even when the standby is completely caught up.

Why?

Because no new transaction may have happened since `15:00`.

Therefore always combine timestamp monitoring with:

```text
sent_lsn
write_lsn
flush_lsn
replay_lsn
```

and WAL byte differences.

---

# 30. Test Actual Replication

On primary:

```sql
CREATE TABLE replication_test
(
    id         BIGSERIAL PRIMARY KEY,
    message    VARCHAR(200),
    created_at TIMESTAMPTZ DEFAULT clock_timestamp()
);
```

Insert:

```sql
INSERT INTO replication_test(message)
VALUES ('Streaming replication test');
```

Check:

```sql
SELECT * FROM replication_test;
```

---

# 31. Check Data on Standby

On standby:

```sql
SELECT * FROM replication_test;
```

The row should appear automatically.

Try writing on the standby:

```sql
INSERT INTO replication_test(message)
VALUES ('standby test');
```

It should fail because a physical standby is read-only while recovery is active.

---

# 32. Generate Larger Replication Load

On primary:

```sql
INSERT INTO replication_test(message)
SELECT
    'Replication load test row ' || generate_series(1,1000000);
```

While this runs, monitor:

```sql
SELECT
    application_name,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(sent_lsn,replay_lsn)
    ) AS replay_lag
FROM pg_stat_replication;
```

This is useful for seeing how the standby behaves under heavy WAL generation.

---

# 33. Important Note About `wal_keep_size`

`wal_keep_size` helps keep recent WAL in:

```text
$PGDATA/pg_wal
```

but it should not be considered unlimited protection.

For example:

```text
Standby stopped
      |
      v
Primary continues generating WAL
      |
      v
Old WAL eventually recycled
```

If the standby needs WAL that is no longer available, streaming replication alone may not be able to continue.

Production solutions normally consider:

```text
Replication slot

and/or

WAL archive
```

---

# 34. Physical Replication Slot — Next Improvement

A physical replication slot can tell the primary:

> Do not remove WAL that this standby still requires.

Example:

```sql
SELECT pg_create_physical_replication_slot('as_beta_slot');
```

Then the standby can use:

```conf
primary_slot_name = 'as_beta_slot'
```

Check slots:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn
FROM pg_replication_slots;
```

Important:

A disconnected standby using a replication slot can cause `pg_wal` to grow significantly.

Therefore replication slots **must be monitored**.

---

# 35. WAL Archive + Streaming Replication

A stronger architecture can combine streaming replication with WAL archiving:

```text
                         PRIMARY
                            |
                         pg_wal
                            |
               +------------+------------+
               |                         |
               v                         v
          WAL Sender                 archive-push
               |                         |
               v                         v
           STANDBY                 pgBackRest Repo
               ^                         |
               |                         |
               +------ archive-get ------+
```

Normal operation:

```text
Streaming replication
```

If required WAL is not available through streaming and archive recovery is configured, the standby can obtain older WAL from the archive.

---

# 36. Tablespaces

If a tablespace already exists before creating the standby, `pg_basebackup` can map its location using:

```text
-T old_path=new_path
```

Example:

```bash
pg_basebackup \
    -h 192.168.136.10 \
    -U replication \
    -D /u01/18/data \
    -X stream \
    -P \
    -R \
    -T /u01/app_ts=/u02/app_ts
```

However, if streaming replication is already running and a **new tablespace** is later created on the primary, the tablespace path should normally be available at the same logical pathname on the standby.

For HA systems it is therefore easier to keep logical mount paths identical:

```text
PRIMARY                     STANDBY

/u01/app_ts                 /u01/app_ts
     |                           |
Storage A                   Storage B
```

The underlying storage devices do not need to be the same.

---

# 37. Important Production Recommendations

For a production-style physical replication design:

```text
Dedicated replication user
        +
SCRAM-SHA-256 authentication
        +
Restricted /32 HBA rule
        +
Streaming replication
        +
Physical replication slot
        +
WAL archive
        +
Replication monitoring
        +
Failover procedure
        +
pg_rewind procedure
```

Do not rely only on:

```text
trust authentication
+
wal_keep_size
```

for a production HA environment.

---

# 38. Useful DBA Commands

## Am I Primary or Standby?

```sql
SELECT pg_is_in_recovery();
```

```text
false = primary
true  = standby
```

---

## Current Primary WAL Position

```sql
SELECT pg_current_wal_lsn();
```

Run on primary.

---

## Last WAL Received on Standby

```sql
SELECT pg_last_wal_receive_lsn();
```

---

## Last WAL Replayed on Standby

```sql
SELECT pg_last_wal_replay_lsn();
```

---

## Primary Replication Status

```sql
SELECT * FROM pg_stat_replication;
```

---

## Standby WAL Receiver

```sql
SELECT * FROM pg_stat_wal_receiver;
```

---

## Replication Slots

```sql
SELECT * FROM pg_replication_slots;
```

---

## Check Replication User

```sql
SELECT
    rolname,
    rolcanlogin,
    rolreplication
FROM pg_roles
WHERE rolname = 'replication';
```

---

# 39. Troubleshooting Flow

If replication stops:

```text
Is standby running?
        |
        v
SELECT pg_is_in_recovery();
        |
        v
Check pg_stat_wal_receiver
        |
        v
Is WAL receiver streaming?
        |
       YES
        |
        v
Check receive_lsn vs replay_lsn
```

If WAL receiver is missing:

```text
Check primary_conninfo
        |
        v
Check network
        |
        v
Check port 5432
        |
        v
Check pg_hba.conf
        |
        v
Check replication user
        |
        v
Check primary pg_stat_replication
        |
        v
Check PostgreSQL logs
```

---

# 40. Final Architecture

```text
                        as-alpha
                         PRIMARY
                    192.168.136.10
                           |
                    PostgreSQL WAL
                           |
                       pg_wal
                           |
                      WAL Sender
                           |
                           |
                  Streaming Replication
                           |
                           v
                      WAL Receiver
                           |
                       pg_wal
                           |
                      WAL Replay
                           |
                           v
                        as-beta
                        STANDBY
                    192.168.136.11
```

---

# 41. Key Points to Remember

1. PostgreSQL physical streaming replication works using **WAL**.

2. A dedicated user with `LOGIN` + `REPLICATION` privilege should be used.

3. Physical replication access is controlled through the `replication` database keyword in `pg_hba.conf`.

4. `pg_basebackup` creates the initial physical copy of the primary.

5. `-R` creates `standby.signal` and adds connection information to `postgresql.auto.conf`.

6. `primary_conninfo` tells the standby where its upstream primary is.

7. `standby.signal` tells PostgreSQL to start in standby/recovery mode.

8. `walsender` runs on the primary.

9. `walreceiver` runs on the standby.

10. Important replication positions are:

```text
sent_lsn
   ↓
write_lsn
   ↓
flush_lsn
   ↓
replay_lsn
```

11. `state = streaming` means the WAL sender is actively streaming after the standby has caught up.

12. `sync_state = async` means asynchronous replication.

13. `pg_is_in_recovery() = true` indicates a standby.

14. `wal_log_hints` is not required for streaming replication but is useful for future `pg_rewind` testing.

15. `wal_keep_size` provides limited WAL retention; replication slots and WAL archives provide stronger protection against standby outages.

16. Streaming replication provides the replication engine. Tools such as Patroni/repmgr can later add automated failover, switchover and cluster management.

---

# Next Lab

After mastering basic streaming replication:

```text
1. Physical replication slots
        ↓
2. WAL archive + pgBackRest archive-get
        ↓
3. Replication under pgbench load
        ↓
4. Manual standby promotion
        ↓
5. Failover
        ↓
6. pg_rewind old primary
        ↓
7. Planned switchover
        ↓
8. Synchronous replication
        ↓
9. Patroni + etcd
        ↓
10. HAProxy / client routing
```

This progression helps understand what PostgreSQL itself provides before introducing automatic HA tooling.

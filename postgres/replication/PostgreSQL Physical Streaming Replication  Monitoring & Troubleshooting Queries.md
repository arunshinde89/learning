# PostgreSQL Physical Streaming Replication
# Monitoring & Troubleshooting Queries

Environment used in this lab:

```text
PRIMARY
Hostname : as-alpha
IP       : 192.168.136.10
PGDATA   : /u01/18/data
Port     : 5432

STANDBY
Hostname : as-beta
IP       : 192.168.136.11
PGDATA   : /u01/18/data
Port     : 5432
```

---

# 1. Quick Reference — Where to Run What?

| Check | Run On |
|---|---|
| Is this primary or standby? | BOTH |
| Overall replication status | PRIMARY |
| Total replication lag | PRIMARY |
| Sent/write/flush/replay lag | PRIMARY |
| WAL receiver status | STANDBY |
| Last WAL received | STANDBY |
| Last WAL applied/replayed | STANDBY |
| Receive vs replay lag | STANDBY |
| Is WAL replay paused? | STANDBY |
| Replication slots | PRIMARY |
| Replication parameters | PRIMARY |
| `primary_conninfo` | STANDBY |
| `primary_slot_name` | STANDBY |
| WAL archive status | PRIMARY |
| WAL sender OS process | PRIMARY |
| WAL receiver OS process | STANDBY |
| `pg_hba.conf` validation | PRIMARY |
| Network connectivity | STANDBY → PRIMARY |

---

# 2. Identify Primary or Standby

## Run on: BOTH

```sql
SELECT pg_is_in_recovery();
```

Interpretation:

```text
false / f = PRIMARY
true  / t = STANDBY
```

Better formatted query:

```sql
SELECT
    CASE
        WHEN pg_is_in_recovery()
        THEN 'STANDBY'
        ELSE 'PRIMARY'
    END AS database_role;
```

Example:

```text
 database_role
---------------
 STANDBY
```

This should be one of the first commands during troubleshooting.

---

# 3. PRIMARY — Quick Replication Health Check

## Run on: PRIMARY

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

Healthy example:

```text
 application_name | client_addr    | state     | sync_state | sent_lsn  | write_lsn | flush_lsn | replay_lsn
------------------+----------------+-----------+------------+-----------+-----------+-----------+-----------
 walreceiver       | 192.168.136.11 | streaming | async      | 0/5000168 | 0/5000168 | 0/5000168 | 0/5000168
```

Important values:

```text
state = streaming
```

means the standby is actively streaming WAL.

```text
sync_state = async
```

means asynchronous replication.

It does NOT mean replication is unhealthy.

If all LSN values are equal:

```text
sent_lsn
write_lsn
flush_lsn
replay_lsn
```

then the standby has replayed all WAL reported as sent by the primary.

---

# 4. PRIMARY — Best Overall Replication Lag Query

## Run on: PRIMARY

This is one of the most useful DBA queries.

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,

    pg_current_wal_lsn() AS primary_current_lsn,

    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,

    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            replay_lsn
        )
    ) AS total_replication_lag

FROM pg_stat_replication;
```

Example:

```text
application_name       | walreceiver
client_addr            | 192.168.136.11
state                  | streaming
sync_state             | async
primary_current_lsn    | 0/5000168
sent_lsn               | 0/5000168
write_lsn              | 0/5000168
flush_lsn              | 0/5000168
replay_lsn             | 0/5000168
total_replication_lag  | 0 bytes
```

Ideal result:

```text
state = streaming
total_replication_lag = 0 bytes
```

---

# 5. PRIMARY — Detailed Lag Breakdown

## Run on: PRIMARY

This query helps identify exactly **where replication is slow**.

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,

    pg_current_wal_lsn() AS current_lsn,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,

    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            sent_lsn
        )
    ) AS primary_send_lag,

    pg_size_pretty(
        pg_wal_lsn_diff(
            sent_lsn,
            write_lsn
        )
    ) AS send_write_lag,

    pg_size_pretty(
        pg_wal_lsn_diff(
            write_lsn,
            flush_lsn
        )
    ) AS write_flush_lag,

    pg_size_pretty(
        pg_wal_lsn_diff(
            flush_lsn,
            replay_lsn
        )
    ) AS flush_replay_lag,

    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            replay_lsn
        )
    ) AS total_lag

FROM pg_stat_replication;
```

Interpretation:

```text
PRIMARY
   |
   | WAL generated
   v
current_lsn
   |
   | WAL sender
   v
sent_lsn
   |
   | Network
   v
write_lsn
   |
   | Disk flush
   v
flush_lsn
   |
   | WAL apply
   v
replay_lsn
```

---

# 6. How to Interpret Lag Location

## Large `current_lsn → sent_lsn`

Possible issue:

```text
Primary generating WAL faster
than WAL sender can send it
```

Check:

- Primary CPU
- Disk I/O
- WAL sender
- Heavy transaction workload

---

## Large `sent_lsn → write_lsn`

Possible issue:

```text
Network / standby receive problem
```

Check:

- Network latency
- Packet loss
- Standby CPU
- WAL receiver

---

## Large `write_lsn → flush_lsn`

Possible issue:

```text
Standby storage is slow
```

Check:

- Disk latency
- Filesystem
- Storage IOPS
- `pg_wal` disk performance

---

## Large `flush_lsn → replay_lsn`

Possible issue:

```text
Standby receives WAL quickly
but cannot replay it quickly enough
```

Check:

- Standby disk I/O
- Long-running queries on hot standby
- Recovery conflicts
- Replay paused
- Heavy WAL generation

---

# 7. PRIMARY — PostgreSQL Lag Time Columns

## Run on: PRIMARY

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

These values can help, but for troubleshooting I prefer combining them with the actual LSN byte differences.

---

# 8. PRIMARY — Check How Many Standbys Are Connected

## Run on: PRIMARY

```sql
SELECT count(*) AS connected_standbys
FROM pg_stat_replication;
```

For our lab:

```text
connected_standbys
------------------
1
```

If expected:

```text
1
```

but result is:

```text
0
```

the standby is not connected to the primary.

---

# 9. PRIMARY — Show All Connected Standbys

## Run on: PRIMARY

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    backend_start,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication
ORDER BY application_name;
```

Useful when you have:

```text
Primary
   |
   +---- Standby1
   |
   +---- Standby2
   |
   +---- Standby3
```

---

# 10. STANDBY — Quick Health Check

## Run on: STANDBY

This is the query we should keep as our main standby query.

```sql
SELECT
    pg_is_in_recovery() AS is_standby,

    pg_last_wal_receive_lsn() AS received_lsn,

    pg_last_wal_replay_lsn() AS replayed_lsn,

    pg_last_xact_replay_timestamp() AS last_replay_time,

    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_last_wal_receive_lsn(),
            pg_last_wal_replay_lsn()
        )
    ) AS receive_replay_lag;
```

Example:

```text
is_standby         | t
received_lsn       | 0/5000168
replayed_lsn       | 0/5000168
last_replay_time   | 2026-11-01 17:30:25+00
receive_replay_lag | 0 bytes
```

Healthy:

```text
is_standby = true

received_lsn = replayed_lsn

receive_replay_lag = 0 bytes
```

---

# 11. STANDBY — Last WAL Received

## Run on: STANDBY

```sql
SELECT pg_last_wal_receive_lsn();
```

Meaning:

```text
Last WAL position received and synced
from streaming replication.
```

---

# 12. STANDBY — Last WAL Applied

## Run on: STANDBY

Very important command:

```sql
SELECT pg_last_wal_replay_lsn();
```

Meaning:

```text
Last WAL position actually replayed/applied
on the standby.
```

Interview question:

> How do you check the last applied WAL on PostgreSQL standby?

Answer:

```sql
SELECT pg_last_wal_replay_lsn();
```

---

# 13. STANDBY — Received vs Applied WAL

## Run on: STANDBY

```sql
SELECT
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn()  AS replayed_lsn;
```

Example:

```text
received_lsn | replayed_lsn
-------------+-------------
0/5000168    | 0/5000168
```

If they are equal:

```text
Everything received has been replayed.
```

However, remember:

This only compares what the standby has **received** with what it has **replayed**.

The primary might have generated newer WAL that has not reached the standby yet.

Therefore the primary-side `pg_stat_replication` query gives a more complete view.

PostgreSQL documentation specifically recommends comparing primary and standby WAL locations when monitoring replication lag.

---

# 14. STANDBY — Receive vs Replay Lag in Bytes

## Run on: STANDBY

```sql
SELECT
    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_last_wal_receive_lsn(),
            pg_last_wal_replay_lsn()
        )
    ) AS receive_replay_lag;
```

Example:

```text
receive_replay_lag
------------------
0 bytes
```

Healthy.

Example:

```text
receive_replay_lag
------------------
500 MB
```

Meaning:

```text
WAL has already reached standby,
but approximately 500 MB is waiting to be replayed.
```

---

# 15. STANDBY — WAL Receiver Status

## Run on: STANDBY

```sql
SELECT
    pid,
    status,
    receive_start_lsn,
    written_lsn,
    flushed_lsn,
    received_tli,
    latest_end_lsn,
    latest_end_time,
    slot_name,
    sender_host,
    sender_port
FROM pg_stat_wal_receiver;
```

Healthy example:

```text
status      = streaming
sender_host = 192.168.136.10
sender_port = 5432
```

`pg_stat_wal_receiver` contains the state of the WAL receiver and the upstream server to which the standby is connected.

---

# 16. STANDBY — Check Upstream Primary

## Run on: STANDBY

```sql
SELECT
    sender_host,
    sender_port,
    status,
    slot_name
FROM pg_stat_wal_receiver;
```

Expected:

```text
sender_host | 192.168.136.10
sender_port | 5432
status      | streaming
```

Very useful after failover/switchover to confirm which primary the standby is following.

---

# 17. STANDBY — Check `primary_conninfo`

## Run on: STANDBY

```sql
SHOW primary_conninfo;
```

Example:

```text
user=replication
host=192.168.136.10
port=5432
```

This tells the standby where to connect for streaming replication.

Do not copy passwords from this output into GitHub.

---

# 18. STANDBY — Check Replication Slot Configuration

## Run on: STANDBY

```sql
SHOW primary_slot_name;
```

Example:

```text
as_beta_slot
```

If empty:

```text
No permanent physical replication slot configured.
```

---

# 19. STANDBY — Is WAL Replay Paused?

## Run on: STANDBY

This is a very useful troubleshooting check.

```sql
SELECT pg_is_wal_replay_paused();
```

Better in PostgreSQL 18:

```sql
SELECT pg_get_wal_replay_pause_state();
```

Expected:

```text
not paused
```

Possible values:

```text
not paused
pause requested
paused
```

If replay is paused:

```text
WAL Receiver
     |
     | continues receiving WAL
     v
pg_wal
     |
     X
WAL Replay stopped
```

This can create large replay lag and potentially increase standby disk usage. PostgreSQL documents that WAL can continue arriving while replay is paused.

---

# 20. STANDBY — Resume WAL Replay

## Run on: STANDBY

Only if replay was intentionally or accidentally paused:

```sql
SELECT pg_wal_replay_resume();
```

Then verify:

```sql
SELECT pg_get_wal_replay_pause_state();
```

Expected:

```text
not paused
```

---

# 21. PRIMARY — Check Replication Slot

## Run on: PRIMARY

```sql
SELECT
    slot_name,
    slot_type,
    active,
    active_pid,
    restart_lsn,
    wal_status,
    safe_wal_size,
    inactive_since,
    invalidation_reason
FROM pg_replication_slots;
```

Physical slot healthy example:

```text
slot_name           | as_beta_slot
slot_type           | physical
active              | t
restart_lsn         | 0/5000000
invalidation_reason |
```

PostgreSQL 18 exposes slot state, retained WAL position and invalidation information through `pg_replication_slots`.

---

# 22. PRIMARY — Check Inactive Replication Slots

## Run on: PRIMARY

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    inactive_since
FROM pg_replication_slots
WHERE active = false;
```

An inactive physical slot can be dangerous because WAL may continue accumulating.

---

# 23. PRIMARY — How Much WAL Is a Slot Retaining?

## Run on: PRIMARY

```sql
SELECT
    slot_name,
    active,
    restart_lsn,

    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            restart_lsn
        )
    ) AS retained_wal

FROM pg_replication_slots
WHERE slot_type = 'physical';
```

Example:

```text
slot_name    | active | retained_wal
-------------+--------+-------------
as_beta_slot | f      | 20 GB
```

This should immediately get DBA attention.

---

# 24. PRIMARY — Check Important Replication Parameters

## Run on: PRIMARY

```sql
SELECT
    name,
    setting,
    unit,
    source
FROM pg_settings
WHERE name IN
(
    'wal_level',
    'max_wal_senders',
    'max_replication_slots',
    'wal_keep_size',
    'wal_log_hints',
    'synchronous_standby_names',
    'synchronous_commit',
    'archive_mode',
    'archive_command'
)
ORDER BY name;
```

Important parameters include:

```text
wal_level
max_wal_senders
wal_keep_size
max_replication_slots
synchronous_standby_names
archive_mode
archive_command
```

---

# 25. STANDBY — Check Important Standby Parameters

## Run on: STANDBY

```sql
SELECT
    name,
    setting,
    unit,
    source
FROM pg_settings
WHERE name IN
(
    'primary_conninfo',
    'primary_slot_name',
    'restore_command',
    'hot_standby',
    'hot_standby_feedback',
    'max_standby_streaming_delay',
    'max_standby_archive_delay',
    'recovery_min_apply_delay'
)
ORDER BY name;
```

Useful when replication behavior is unexpected.

---

# 26. PRIMARY — Check `pg_hba.conf`

## Run on: PRIMARY

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

To focus on replication:

```sql
SELECT
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules
WHERE 'replication' = ANY(database)
   OR error IS NOT NULL;
```

Example expected rule:

```text
host replication replication 192.168.136.11/32 scram-sha-256
```

For your lab it may currently be:

```text
host replication replication 192.168.136.11/32 trust
```

---

# 27. PRIMARY — Check Replication User

## Run on: PRIMARY

```sql
SELECT
    rolname,
    rolcanlogin,
    rolreplication,
    rolsuper
FROM pg_roles
WHERE rolname = 'replication';
```

Expected:

```text
rolcanlogin      = true
rolreplication   = true
```

Superuser is not required.

---

# 28. PRIMARY — Check WAL Archive Status

## Run on: PRIMARY

If WAL archiving is configured:

```sql
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time,
    stats_reset
FROM pg_stat_archiver;
```

Important indicators:

```text
archived_count increasing
failed_count not continuously increasing
last_archived_time recent
```

If using pgBackRest, also use:

```bash
pgbackrest --stanza=pg18lab check
```

and:

```bash
pgbackrest --stanza=pg18lab info
```

---

# 29. STANDBY — Check `restore_command`

## Run on: STANDBY

If using pgBackRest archive recovery:

```sql
SHOW restore_command;
```

Example:

```text
pgbackrest --stanza=pg18lab archive-get %f "%p"
```

A standby can obtain WAL from archive, local `pg_wal`, and streaming replication during recovery.

---

# 30. PRIMARY — OS-Level WAL Sender Check

## Run on: PRIMARY OS

```bash
ps -ef | grep '[w]alsender'
```

Expected:

```text
postgres ... postgres: walsender replication 192.168.136.11 streaming
```

No WAL sender when a standby should be connected usually means:

```text
connection/authentication/network/standby problem
```

---

# 31. STANDBY — OS-Level WAL Receiver Check

## Run on: STANDBY OS

```bash
ps -ef | grep '[w]alreceiver'
```

Expected:

```text
postgres ... postgres: walreceiver streaming
```

---

# 32. STANDBY — Check Recovery Process

## Run on: STANDBY OS

```bash
ps -ef | grep '[p]ostgres'
```

You should normally see processes associated with recovery, including the startup/recovery process and WAL receiver when streaming.

---

# 33. STANDBY → PRIMARY Network Test

## Run on: STANDBY OS

```bash
nc -zv 192.168.136.10 5432
```

Expected:

```text
Connection succeeded
```

If this fails:

```text
Check firewall
Check PostgreSQL listener
Check IP
Check port
Check routing
```

---

# 34. PRIMARY — Check PostgreSQL Listener

## Run on: PRIMARY

```sql
SHOW listen_addresses;
SHOW port;
```

OS:

```bash
ss -lntp | grep 5432
```

---

# 35. STANDBY — Check Last Replayed Transaction Time

## Run on: STANDBY

```sql
SELECT
    now() AS current_time,
    pg_last_xact_replay_timestamp() AS last_replay_time,
    now() - pg_last_xact_replay_timestamp() AS time_since_last_replay;
```

Important:

Do NOT automatically interpret:

```text
time_since_last_replay = 40 minutes
```

as:

```text
replication lag = 40 minutes
```

If the primary has been idle for 40 minutes, the standby may still be completely caught up.

Always check LSNs as well.

---

# 36. PRIMARY — Best One-Query Health Check

## Run on: PRIMARY

If I could run only **one query on the primary**, I would use:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,

    pg_current_wal_lsn() AS primary_lsn,

    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,

    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            replay_lsn
        )
    ) AS total_lag,

    write_lag,
    flush_lag,
    replay_lag

FROM pg_stat_replication;
```

Healthy result:

```text
state     = streaming
total_lag = 0 bytes
```

---

# 37. STANDBY — Best One-Query Health Check

## Run on: STANDBY

If I could run only **one query on the standby**, I would use:

```sql
SELECT
    pg_is_in_recovery() AS is_standby,

    pg_last_wal_receive_lsn() AS received_lsn,

    pg_last_wal_replay_lsn() AS replayed_lsn,

    pg_last_xact_replay_timestamp() AS last_replay_time,

    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_last_wal_receive_lsn(),
            pg_last_wal_replay_lsn()
        )
    ) AS receive_replay_lag;
```

---

# 38. Troubleshooting — No Row in `pg_stat_replication`

## PRIMARY shows:

```sql
SELECT * FROM pg_stat_replication;
```

Result:

```text
0 rows
```

Check in this order:

```text
1. Is standby PostgreSQL running?

2. Is standby in recovery?
   SELECT pg_is_in_recovery();

3. Is primary_conninfo correct?
   SHOW primary_conninfo;

4. Can standby reach primary port 5432?

5. Is replication user correct?

6. Does user have REPLICATION privilege?

7. Is pg_hba.conf allowing standby IP?

8. Is primary listening on network?

9. Check primary PostgreSQL logs.

10. Check standby PostgreSQL logs.
```

---

# 39. Troubleshooting — State Is `catchup`

Example:

```text
state = catchup
```

Usually means:

```text
Standby is behind
and WAL sender is sending historical WAL
to help it catch up.
```

Once caught up, normally:

```text
state = streaming
```

---

# 40. Troubleshooting — Large `sent_lsn → write_lsn`

Likely:

```text
network / standby receive bottleneck
```

Check:

```bash
ping 192.168.136.11
```

Check network throughput and WAL receiver.

---

# 41. Troubleshooting — Large `flush_lsn → replay_lsn`

Likely:

```text
WAL is already on standby disk,
but WAL replay is slow.
```

Check standby:

```sql
SELECT pg_get_wal_replay_pause_state();
```

Then:

```sql
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();
```

Also check long-running queries and storage I/O.

---

# 42. Troubleshooting — Standby Stopped for Long Time

Check slot:

## PRIMARY

```sql
SELECT
    slot_name,
    active,
    restart_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            restart_lsn
        )
    ) AS retained_wal
FROM pg_replication_slots;
```

Potential issue:

```text
Standby DOWN
      +
Physical slot active = false
      +
Primary generating WAL
      =
pg_wal grows
```

Monitor disk space.

---

# 43. Troubleshooting — Required WAL Removed

You may see errors similar to:

```text
requested WAL segment has already been removed
```

Possible recovery methods:

```text
If WAL archive available
        ↓
standby may retrieve missing WAL
using restore_command/archive-get

If required WAL unavailable everywhere
        ↓
standby may need to be rebuilt
```

---

# 44. Troubleshooting — Check PostgreSQL Logs

## PRIMARY

```bash
tail -100 $PGDATA/log/*.log
```

or:

```bash
journalctl -u postgresql-18 -n 100
```

Look for:

```text
authentication failure
replication connection failure
requested WAL removed
could not send data
connection reset
```

---

## STANDBY

```bash
tail -100 $PGDATA/log/*.log
```

Look for:

```text
started streaming WAL
waiting for WAL
requested WAL segment
could not connect to primary
archive recovery
recovery ended
WAL replay
```

---

# 45. Understanding the Four Main LSNs

```text
PRIMARY                            STANDBY

WAL generated
     |
     v
current WAL LSN

     |
     v

sent_lsn
     |
     | Network
     v

write_lsn
     |
     | Written on standby
     v

flush_lsn
     |
     | Durable on standby storage
     v

replay_lsn
     |
     | WAL applied
     v

Standby database
```

Remember:

```text
sent
  ↓
write
  ↓
flush
  ↓
replay
```

---

# 46. What Does "Primary and Standby Are in Sync" Mean?

For practical DBA monitoring, a standby is caught up when:

```text
state = streaming
```

and:

```text
primary current WAL position
≈
standby replay position
```

and preferably:

```text
total_replication_lag = 0 bytes
```

or a very small acceptable value under active workload.

Do not confuse this with:

```text
sync_state = async
```

`async` means the replication **commit mode** is asynchronous.

It does not mean the standby is currently lagging.

---

# 47. My Recommended Three Daily Queries

## Query 1 — PRIMARY

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    pg_current_wal_lsn() AS primary_lsn,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            replay_lsn
        )
    ) AS total_lag
FROM pg_stat_replication;
```

---

## Query 2 — STANDBY

```sql
SELECT
    pg_is_in_recovery() AS is_standby,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn,
    pg_last_xact_replay_timestamp() AS last_replay_time,
    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_last_wal_receive_lsn(),
            pg_last_wal_replay_lsn()
        )
    ) AS receive_replay_lag;
```

---

## Query 3 — PRIMARY

If using replication slots:

```sql
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(
            pg_current_wal_lsn(),
            restart_lsn
        )
    ) AS retained_wal,
    inactive_since,
    invalidation_reason
FROM pg_replication_slots
WHERE slot_type = 'physical';
```

These three queries will answer most first-level replication health questions.

---

# 48. Senior DBA Troubleshooting Flow

```text
Replication problem reported
          |
          v
PRIMARY or STANDBY?
SELECT pg_is_in_recovery();
          |
          v
Check PRIMARY
pg_stat_replication
          |
          +---- No row?
          |       |
          |       v
          |   Connection problem
          |   HBA / Network / User /
          |   primary_conninfo
          |
          +---- state != streaming?
          |       |
          |       v
          |   Check sender / logs
          |
          v
Check LSN differences
          |
          +---- current → sent large
          |       Primary sender/load issue
          |
          +---- sent → write large
          |       Network/receiver issue
          |
          +---- write → flush large
          |       Standby storage issue
          |
          +---- flush → replay large
                  Standby replay issue
                          |
                          v
                 Check replay paused
                 Long queries
                 Disk I/O
                 Recovery conflicts
```

---

# 49. Important Functions to Remember

```text
PRIMARY

pg_current_wal_lsn()

pg_stat_replication

pg_replication_slots

pg_stat_archiver
```

```text
STANDBY

pg_is_in_recovery()

pg_last_wal_receive_lsn()

pg_last_wal_replay_lsn()

pg_last_xact_replay_timestamp()

pg_stat_wal_receiver

pg_get_wal_replay_pause_state()
```

---

# 50. One-Line Interview Answers

**How do you identify a standby?**

```sql
SELECT pg_is_in_recovery();
```

---

**How do you check the last WAL received on standby?**

```sql
SELECT pg_last_wal_receive_lsn();
```

---

**How do you check the last WAL applied on standby?**

```sql
SELECT pg_last_wal_replay_lsn();
```

---

**How do you check streaming replication from primary?**

```sql
SELECT * FROM pg_stat_replication;
```

---

**How do you check WAL receiver on standby?**

```sql
SELECT * FROM pg_stat_wal_receiver;
```

---

**How do you check replication slots?**

```sql
SELECT * FROM pg_replication_slots;
```

---

**How do you check whether WAL replay is paused?**

```sql
SELECT pg_get_wal_replay_pause_state();
```

---

# 51. Golden Rule

When troubleshooting PostgreSQL physical replication, remember:

```text
PRIMARY

current WAL
    ↓
sent_lsn
    ↓
NETWORK
    ↓
write_lsn
    ↓
flush_lsn
    ↓
replay_lsn

STANDBY
```

Find **between which two positions the gap is increasing**.

That tells you where to investigate.

```text
current → sent
    Primary/WAL sender

sent → write
    Network/WAL receiver

write → flush
    Standby storage

flush → replay
    Standby WAL replay
```

This is the core method for troubleshooting physical streaming replication.

# Patroni Basic DBA Administration — PostgreSQL 18 HA Lab

> **Purpose:** A practical, repeatable GitHub runbook for daily Patroni administration, configuration management, monitoring, safe restarts, and HA operations.  
> **Environment:** Rocky Linux / PostgreSQL 18 / Patroni / etcd  
> **Cluster:** `pg18-ha-lab`  
> **Last reviewed:** 18 November 2026

## 1. Lab topology

| Server | IP address | PostgreSQL role (at last check) |
|---|---|---|
| `as-pg1` | `192.168.136.10` | Leader / primary |
| `as-pg2` | `192.168.136.11` | Replica / standby |
| `as-pg3` | `192.168.136.12` | Replica / standby |

- Patroni executable: `/opt/patroni/venv/bin/patronictl`
- Patroni configuration **example used by this lab**: `/etc/patroni/patroni.yml`
- Cluster scope: `pg18-ha-lab`
- PostgreSQL data directory previously used in this lab: `/u01/18/data` — **verify on each node**.
- **Roles are dynamic**: never assume `as-pg1` is still the primary without checking `patronictl list`.

**Architecture:** etcd stores cluster coordination and dynamic configuration; Patroni manages PostgreSQL HA; PostgreSQL manages data, queries and WAL replication. The etcd leader and PostgreSQL primary can be on different servers.

## 2. Create a permanent `pctl` function in `.bash_profile`

Run the following as the **`postgres` operating-system user**, on each node where you want the shortcut.

### Step 1 — Open `.bash_profile`

```bash
whoami
vi ~/.bash_profile
```

Add the following at the **end** of the file (only once):

```bash
# Patroni command shortcut (pg18-ha-lab)
pctl() {
    /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml "$@"
}
```

Why `"$@"`? It passes every supplied argument to `patronictl` correctly, preserving quoting. Example: `pctl list -e` becomes `patronictl -c ... list -e`.

### Step 2 — Load the updated profile without logging out

```bash
source ~/.bash_profile
type pctl
pctl list -e
```

The function is persistent for future **Bash login shells**. On Rocky/RHEL, an existing `.bash_profile` commonly sources `.bashrc`; if you prefer non-login interactive shells to have `pctl` too, define the function in `.bashrc` instead and keep the existing source line in `.bash_profile`.

**Notes:** The function belongs to the account whose profile you edited. Do not loosen permissions on `patroni.yml` merely to make the shortcut work: it may contain passwords. If your actual config path is different, adjust the function.

## 3. Daily cluster monitoring — READ ONLY

```bash
pctl list pg18-ha-lab              # Cluster member roles and state
pctl list -e                      # Extended WAL LSN/lag and restart columns
pctl list -e -w 2                 # Refresh status every 2 seconds (Ctrl+C to stop)
pctl topology pg18-ha-lab         # Tree view of current replication topology
pctl history pg18-ha-lab          # Recorded PostgreSQL timeline/leader changes
pctl version pg18-ha-lab          # CLI and member versions
pctl dsn pg18-ha-lab -r primary   # Current primary connection information
pctl dsn pg18-ha-lab -m as-pg2   # Connection information for a particular member
```

### Read `pctl list -e`

| Column | Meaning |
|---|---|
| `Leader` / `Replica` | Current primary / standby role |
| `running` / `streaming` | PostgreSQL running / standby streaming WAL |
| `TL` | PostgreSQL timeline; usually increments after promotion |
| `Receive LSN` | Last WAL position received and synced to standby storage |
| `Replay LSN` | Last WAL position replayed on standby |
| Receive/Replay `Lag` | Lag relative to upstream, displayed in MB (rounded) |
| `Pending restart` | Restart needed for one or more changed parameters |
| `Scheduled restart` | Restart planned through Patroni |

**Our last healthy lab snapshot:** `as-pg1` leader, `as-pg2` and `as-pg3` streaming, all TL `7`, both replicas at `0/5490230`, displayed lag `0 MB`. This is only a snapshot, not a guarantee of current state or synchronous replication.

## 4. View current Patroni configuration

```bash
pctl show-config pg18-ha-lab
```

This displays the **dynamic configuration stored in etcd (DCS)**, not every effective local setting.

Our verified DCS configuration included:

```yaml
loop_wait: 10
maximum_lag_on_failover: 1048576
retry_timeout: 10
ttl: 30
postgresql:
  parameters:
    hot_standby: 'on'
    max_replication_slots: 10
    max_wal_senders: 10
    password_encryption: scram-sha-256
    wal_level: replica
    wal_log_hints: 'on'
    work_mem: 2MB
  pg_hba:
    - local all all peer
    - host all all 127.0.0.1/32 scram-sha-256
    - host replication replicator 192.168.136.0/24 scram-sha-256
    - host all all 192.168.136.0/24 scram-sha-256
  use_pg_rewind: true
```

### Config ownership: where to make changes

| Change type | Correct management method |
|---|---|
| Shared PostgreSQL GUCs, HA timers, shared `pg_hba` | `pctl edit-config pg18-ha-lab` (DCS/etcd) |
| Node address, data path, REST API, local-only settings | Edit local `patroni.yml`, then `pctl reload pg18-ha-lab <member>` |
| Actual applied PostgreSQL value | Inspect `pg_settings` on the relevant node |

**Avoid directly editing Patroni-generated `postgresql.conf` or `pg_hba.conf`.** Local Patroni settings often override DCS settings, with important exceptions (e.g., replication-critical GUCs are restricted to global configuration). `bootstrap.dcs` in the local YAML is only used to initialize an unbootstrapped cluster; editing it later will **not** update the running DCS configuration.

## 5. Change a PostgreSQL parameter cluster-wide — CONFIG CHANGE

Example: Set `work_mem` from `2MB` to `4MB` on all three nodes.

```bash
pctl show-config pg18-ha-lab
pctl edit-config pg18-ha-lab --pg work_mem=4MB
# Review displayed diff; confirm
pctl show-config pg18-ha-lab
```

On each PostgreSQL node, verify **effective** value:

```sql
SELECT name, setting, unit, source, pending_restart
FROM pg_settings
WHERE name = 'work_mem';
```

Expected `setting = 4096`, `unit = kB`, assuming there is no local or session-level override.

`work_mem` is reloadable: **no restart required**. Patroni generally propagates dynamic changes during its HA cycle. To revert to the lab's original DCS value:

```bash
pctl edit-config pg18-ha-lab --pg work_mem=2MB
```

To open an interactive editor for multiple shared settings:

```bash
pctl edit-config pg18-ha-lab
```

Review the diff carefully and do not remove unrelated settings.

## 6. Manage `pg_hba.conf` through Patroni — CONFIG CHANGE

Your current lab has a subnet-wide rule: `host all all 192.168.136.0/24 scram-sha-256`. This already covers `192.168.136.50`; adding a `/32` rule for that address would be redundant unless it changes access restrictions and is ordered appropriately.

To add **a genuinely new, approved client** such as `192.168.137.50`:

```bash
pctl edit-config pg18-ha-lab
```

Under the existing `postgresql.pg_hba` list, add an appropriately scoped entry, for example:

```yaml
- host appdb appuser 192.168.137.50/32 scram-sha-256
```

Do **not** replace existing authentication lines. PostgreSQL evaluates HBA entries in order; prefer narrow database/user/network rules over blanket access. Patroni regenerates the managed HBA file and reloads PostgreSQL; a PostgreSQL restart is normally unnecessary. This assumes the default HBA location is Patroni-managed (no custom `hba_file`).

Verify on a node:

```sql
SHOW hba_file;
SELECT line_number, type, database, user_name, address, auth_method, error
FROM pg_hba_file_rules
ORDER BY line_number;
```

Check `error IS NULL` on all intended lines and test the application connection.

## 7. Reload vs. restart PostgreSQL

### Reload node-local Patroni config — usually non-disruptive

After changing local `/etc/patroni/patroni.yml` **on `as-pg2` only**:

```bash
pctl reload pg18-ha-lab as-pg2
```

This tells Patroni on `as-pg2` to reread local configuration and triggers PostgreSQL config reload. Reloadable parameters change without database restart.

### Restart a selected PostgreSQL instance — DISRUPTIVE

```bash
pctl list -e
pctl restart pg18-ha-lab as-pg2
pctl list -e
```

This requests a **PostgreSQL restart through Patroni**, not a Patroni service restart. `as-pg2` must currently be the intended target role before using this example. For production, restart standbys one at a time, verifying streaming after each; handle primary restart in a change window with application impact understood.

### Restart only when a parameter is pending

```bash
pctl list -e
pctl restart pg18-ha-lab as-pg2 --pending
```

For a restart-required parameter such as `shared_buffers` (set cluster-wide using `edit-config`), Patroni marks affected members as **Pending restart**. **Caution:** For shared-memory-related changes, follow Patroni's recommended node restart order and compatibility rules; do not blindly restart the primary first.

**Do not use `systemctl restart patroni` as a substitute for a PostgreSQL restart.** Restarting the Patroni service on the current primary can trigger failover.

## 8. Planned switchover — DISRUPTIVE

Before switching the leader, confirm `pctl list -e` and that the candidate is healthy/streaming and caught up. If **`as-pg1` is the current leader**, move primary to `as-pg2`:

```bash
pctl list -e
pctl switchover pg18-ha-lab --leader as-pg1 --candidate as-pg2
pctl list -e
pctl history pg18-ha-lab
```

Patroni coordinates demotion, leadership and promotion. This changes the writable endpoint; applications must reconnect through an HA-aware route (e.g., HAProxy). **Never assume a fixed host remains primary.** Use switchover for a healthy cluster.

## 9. Manual failover — HIGH RISK

Use only for an unhealthy cluster, after diagnosing primary availability and replication safety:

```bash
pctl list -e
pctl failover pg18-ha-lab --candidate as-pg2
```

**Warning:** Forced promotion of an asynchronous standby can lose transactions not received by that standby. Check leader state, lag and fencing/split-brain risks. Do not use manual failover as a routine switchover substitute.

## 10. Pause / resume automatic failover — USE CAREFULLY

```bash
pctl pause pg18-ha-lab --wait     # Enter maintenance mode
pctl list
pctl resume pg18-ha-lab --wait    # Restore automatic failover
pctl list
```

`pause` does **not** stop PostgreSQL. It changes Patroni's HA behavior, notably disabling automatic failover; plan operational coverage during maintenance. Always verify that normal HA management is resumed.

## 11. Rebuild a broken standby — DESTRUCTIVE

```bash
pctl list -e
pctl reinit pg18-ha-lab as-pg3 --wait
pctl list -e
```

This **reinitializes the existing PostgreSQL standby data directory**. Only execute against a confirmed replica after checking backup/recovery requirements; it is not a harmless restart. Do not target the primary.

## 12. Service and log checks — READ ONLY

```bash
systemctl status patroni --no-pager
systemctl status etcd --no-pager
journalctl -u patroni -n 100 --no-pager
journalctl -u patroni -f -o short-iso
journalctl -u etcd -n 100 --no-pager
journalctl -u etcd -f -o short-iso
```

Logs show HA decisions and some operations, **not every command** Patroni executes. Enable local Patroni `log.level: DEBUG` temporarily if deeper troubleshooting is required; reload the local config and return to INFO afterward. PostgreSQL logs provide recovery, WAL and timeline details.

## 13. Patroni REST API — READ ONLY checks

Patroni commonly listens on port `8008`. Verify your actual `restapi.listen` and authentication/TLS settings. Run locally if plain HTTP is enabled:

```bash
curl -s http://127.0.0.1:8008/patroni  # This node's status
curl -s http://127.0.0.1:8008/cluster  # Cluster membership
curl -s http://127.0.0.1:8008/config   # Dynamic configuration
curl -i http://127.0.0.1:8008/primary  # HTTP 200 only for eligible current primary
curl -i http://127.0.0.1:8008/replica  # HTTP 200 for healthy eligible replica
```

For production, restrict REST API network access and use the configured security model rather than assuming unauthenticated HTTP.

## 14. PostgreSQL health SQL (run on the correct node)

**Any node — verify primary/standby role:**

```sql
SELECT inet_server_addr() AS server_ip, pg_is_in_recovery() AS is_standby;
```

**On primary — inspect replication clients:**

```sql
SELECT application_name, client_addr, state, sync_state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;
```

**On standby — inspect WAL receive/replay:**

```sql
SELECT pg_last_wal_receive_lsn() AS receive_lsn,
       pg_last_wal_replay_lsn() AS replay_lsn,
       pg_last_xact_replay_timestamp() AS last_replay_time;
```

**On primary — check replication slots:**

```sql
SELECT slot_name, slot_type, active, restart_lsn,
       wal_status, safe_wal_size
FROM pg_replication_slots;
```

Your DCS setting `max_replication_slots: 10` is **capacity**, not proof that a replication slot was created. `postgresql.use_slots` defaults to `true` in supported current versions, but verify actual slots and `primary_slot_name` during your lab.

**On any node — identify pending parameter restart:**

```sql
SELECT name, setting, unit, source, pending_restart
FROM pg_settings
WHERE pending_restart;
```

## 15. HA parameters to know

| Parameter | Lab value | DBA meaning |
|---|---|---|
| `loop_wait` | `10` | Normal time between HA loop iterations (seconds) |
| `ttl` | `30` | Leader-lock lease TTL (seconds) |
| `retry_timeout` | `10` | Retry budget for some DCS/PostgreSQL operations (seconds) |
| `maximum_lag_on_failover` | `1048576` | 1 MiB maximum lag threshold for candidate eligibility |
| `postgresql.use_pg_rewind` | `true` | Attempt rewind for a former primary when appropriate |
| `postgresql.use_slots` | Not explicitly shown | Current Patroni default generally `true`; verify real slots |

Important constraint: `loop_wait + 2 × retry_timeout <= ttl`. Our values satisfy `10 + 2×10 = 30`.

**Important:** Healthy asynchronous streaming and a displayed `0 MB` lag do not automatically guarantee zero data loss on unexpected failure.

## 16. Everyday DBA checklist

- [ ] Run `pctl list -e` and confirm the intended primary and healthy standbys.
- [ ] Check timeline and receive/replay lag; investigate unexpected lag.
- [ ] Check `Pending restart` before and after configuration changes.
- [ ] Use `pctl show-config` for shared DCS configuration.
- [ ] Verify actual PostgreSQL values using `pg_settings` on each node.
- [ ] Check `pg_stat_replication` on primary and `pg_replication_slots` where applicable.
- [ ] Monitor Patroni and PostgreSQL logs.
- [ ] For changes, choose `edit-config` vs local `patroni.yml` correctly.
- [ ] Never use a hardcoded hostname to infer the primary after a failover.
- [ ] Before disruptive operations, confirm role, replica health, backup and application impact.

## 17. Quick command cheat sheet

| Task | Command | Risk |
|---|---|---|
| Cluster health | `pctl list -e` | Read only |
| Continuous watch | `pctl list -e -w 2` | Read only |
| Replication tree | `pctl topology` | Read only |
| Timeline history | `pctl history` | Read only |
| Dynamic config | `pctl show-config` | Read only |
| Current primary DSN | `pctl dsn -r primary` | Read only |
| Set `work_mem` cluster-wide | `pctl edit-config pg18-ha-lab --pg work_mem=4MB` | Configuration change |
| Edit shared HBA | `pctl edit-config pg18-ha-lab` | Configuration change |
| Reload node | `pctl reload pg18-ha-lab as-pg2` | Generally non-disruptive |
| Restart a replica | `pctl restart pg18-ha-lab as-pg2` | Disruptive |
| Planned switchover | `pctl switchover pg18-ha-lab --leader as-pg1 --candidate as-pg2` | Disruptive |
| Emergency manual failover | `pctl failover pg18-ha-lab --candidate as-pg2` | High risk |
| Maintenance mode | `pctl pause pg18-ha-lab --wait` | Alters failover behavior |
| End maintenance mode | `pctl resume pg18-ha-lab --wait` | Restores HA management |
| Rebuild standby | `pctl reinit pg18-ha-lab as-pg3 --wait` | Destructive |

## References (official Patroni documentation)

- [Patronictl command reference](https://patroni.readthedocs.io/en/latest/patronictl.html)
- [Patroni configuration model and priority](https://patroni.readthedocs.io/en/latest/patroni_configuration.html)
- [Dynamic DCS configuration](https://patroni.readthedocs.io/en/latest/dynamic_configuration.html)
- [Local YAML configuration](https://patroni.readthedocs.io/en/latest/yaml_configuration.html)
- [Pause/resume maintenance mode](https://patroni.readthedocs.io/en/latest/pause.html)

---

**Runbook rule:** Monitor first → confirm the *current* role → change through Patroni → verify on every relevant PostgreSQL instance → record the result.

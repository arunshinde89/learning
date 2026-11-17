# PostgreSQL 18 High Availability with etcd + Patroni

**A complete, hands-on lab: installation → three-node HA → streaming replication → controlled switchover**

![Status](https://img.shields.io/badge/lab-3--node%20HA%20operational-success) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-blue) ![Patroni](https://img.shields.io/badge/Patroni-4.1.5-blue) ![etcd](https://img.shields.io/badge/etcd-3.7.3-blue)

> **Lab date:** 17 November 2026. This guide documents an actual VMware/Rocky Linux hands-on build. Three PostgreSQL members, two streaming standbys, zero *reported* lag in the captured snapshots, and a write/read replication test were demonstrated. A controlled switchover from `as-pg1` to `as-pg2` was **reported successful by the operator**; the post-switchover CLI output was **not preserved**, so target roles below are labeled expected until verified. This is not a production-hardened configuration.

## Contents

1. [Architecture and key concepts](#1-architecture-and-key-concepts)
2. [Lab inventory](#2-lab-inventory)
3. [Install etcd on all nodes](#3-install-etcd-on-all-nodes)
4. [Configure the three-member etcd cluster](#4-configure-the-three-member-etcd-cluster)
5. [Verify etcd, quorum, and recovery](#5-verify-etcd-quorum-and-recovery)
6. [Install PostgreSQL 18 and Patroni](#6-install-postgresql-18-and-patroni)
7. [Configure Patroni on all three nodes](#7-configure-patroni-on-all-three-nodes)
8. [Create the Patroni systemd service](#8-create-the-patroni-systemd-service)
9. [Bootstrap the PostgreSQL cluster](#9-bootstrap-the-postgresql-cluster)
10. [Verify replication and application data](#10-verify-replication-and-application-data)
11. [Controlled switchover: as-pg1 to as-pg2](#11-controlled-switchover-as-pg1-to-as-pg2)
12. [Troubleshooting cheat sheet](#12-troubleshooting-cheat-sheet)
13. [Production readiness checklist](#13-production-readiness-checklist)
14. [Learning outcomes and references](#14-learning-outcomes-and-references)

---

## 1. Architecture and key concepts

```text
                         Clients / applications
                       (HAProxy not installed yet)
                                   |
                        connect to current primary
                                   |
   +----------------------- PostgreSQL HA -----------------------+
   |                                                             |
   |  as-pg1 (192.168.136.10)  ← initial PRIMARY / Leader         |
   |       |                         \                           |
   |       | PostgreSQL WAL           \ PostgreSQL WAL             |
   |       v                           v                         |
   |  as-pg2 (192.168.136.11)     as-pg3 (192.168.136.12)         |
   |       REPLICA                     REPLICA                     |
   +-------------------------------------------------------------+
           |                         |                     |
           +------- Patroni agents on all three VMs -------+
                                   |
                     etcd v3 Distributed Config Store
                                   |
             +---------------------+-------------------+
             |                     |                   |
       etcd as-pg1:2379     etcd as-pg2:2379   etcd as-pg3:2379
             \___________________ quorum __________________/
                    2 of 3 voting members required
```

- **PostgreSQL** holds tables, writes WAL, streams physical changes, and replays changes on standby servers.
- **Patroni** starts and manages PostgreSQL, registers the active leader, decides which eligible standby can be promoted, and helps old primaries rejoin.
- **etcd** stores small coordination/configuration keys. Raft consensus lets Patroni agree on the active leader. **It does not contain PostgreSQL application data or WAL.**
- **systemd** supervises the local `patroni` and `etcd` processes; it does not elect the PostgreSQL leader.

> **Important:** The elected *etcd Raft leader* and the *PostgreSQL/Patroni leader* are different concepts. The etcd quorum of three members tolerates **one** member failure, but quorum alone does not guarantee a zero-data-loss PostgreSQL failover.

**Oracle DBA analogy:** PostgreSQL primary + physical standbys resembles an Oracle Data Guard topology; Patroni provides role/HA orchestration; etcd is its coordination store. This is a functional analogy, **not** a claim of feature equivalence with Data Guard Broker, Oracle Clusterware, or RAC.

## 2. Lab inventory

| Item | Specification |
|---|---|
| Hypervisor | VMware Workstation + Vagrant |
| OS | Rocky Linux 10.1 |
| Nodes | `as-pg1`, `as-pg2`, `as-pg3` |
| Private IPs | `192.168.136.10`, `.11`, `.12` |
| Per-VM resources | 2 vCPU, 4 GB RAM |
| PostgreSQL | PGDG PostgreSQL 18 (18.7 observed on as-pg1) |
| Patroni | 4.1.5 (observed on as-pg1; used across running cluster) |
| etcd | 3.7.3 |
| etcd client/peer ports | 2379 / 2380 |
| Patroni REST API | 8008 |
| PostgreSQL | 5432 |
| PGDATA | `/var/lib/pgsql/18/data` |
| PostgreSQL binaries | `/usr/pgsql-18/bin` |
| Patroni virtual environment | `/opt/patroni/venv` |
| Patroni configuration | `/etc/patroni/patroni.yml` |
| etcd configuration | `/etc/etcd/etcd.yml` |
| etcd data | `/var/lib/etcd` |

**Scope:** Lab-only, isolated VMware host-only network. The following setup deliberately uses **unencrypted etcd HTTP** and a basic trusted private network. Never expose these services on an untrusted network or reuse this exact configuration for a bank/production deployment.

## 3. Install etcd on all nodes

Run the following on **all three VMs as root**, with empty etcd data directories before *first-time* bootstrap.

### 3.1 Service account and directories

```bash
getent passwd etcd >/dev/null || \
  useradd --system --user-group --home-dir /var/lib/etcd --shell /sbin/nologin etcd

install -d -o etcd -g etcd -m 0700 /var/lib/etcd
install -d -o root -g etcd -m 0750 /etc/etcd
id etcd
ls -ld /var/lib/etcd /etc/etcd
```

**Why:** etcd should own its storage, just as the `postgres` user owns PGDATA; it should not run as `root`.

### 3.2 Download and install etcd binaries

```bash
ETCD_VER=v3.7.3
cd /tmp
curl -fL --retry 3 \
  "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz" \
  -o "etcd-${ETCD_VER}-linux-amd64.tar.gz"
curl -fL --retry 3 \
  "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/SHA256SUMS" \
  -o etcd-SHA256SUMS

grep -F "etcd-${ETCD_VER}-linux-amd64.tar.gz" etcd-SHA256SUMS | sha256sum -c -

mkdir -p /tmp/etcd-install
tar -xzf "/tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz" \
  -C /tmp/etcd-install --strip-components=1 --no-same-owner
install -o root -g root -m 0755 \
  /tmp/etcd-install/{etcd,etcdctl,etcdutl} /usr/local/bin/

etcd --version
etcdctl version
etcdutl version
```

**Why `install -m 0755`?** It copies the binaries into `/usr/local/bin` and makes them executable, while setting root ownership. It does **not** start the etcd service. The lab validated the downloaded tarball checksum on `as-pg1`.

## 4. Configure the three-member etcd cluster

### 4.1 Configuration on `as-pg1`

```bash
cat > /etc/etcd/etcd.yml <<'EOF_ETCD'
name: as-pg1
data-dir: /var/lib/etcd
listen-client-urls: http://192.168.136.10:2379,http://127.0.0.1:2379
advertise-client-urls: http://192.168.136.10:2379
listen-peer-urls: http://192.168.136.10:2380
initial-advertise-peer-urls: http://192.168.136.10:2380
initial-cluster: as-pg1=http://192.168.136.10:2380,as-pg2=http://192.168.136.11:2380,as-pg3=http://192.168.136.12:2380
initial-cluster-state: new
initial-cluster-token: pg-ha-etcd-lab-20261117
EOF_ETCD
chown root:etcd /etc/etcd/etcd.yml
chmod 0640 /etc/etcd/etcd.yml
```

**Config meaning:** port `2379` is the client API for Patroni/etcdctl; `2380` is for peer consensus. Every node has a unique `name` and URLs; **all nodes share the same `initial-cluster` map**. `initial-cluster-state: new` is a one-time *new-cluster bootstrap setting*, not a recipe for restarting an existing cluster from scratch.

### 4.2 Copy and adjust `as-pg2` / `as-pg3`

Run on `as-pg2` with `NODE=as-pg2`, `IP=192.168.136.11`; on `as-pg3` with `NODE=as-pg3`, `IP=192.168.136.12`:

```bash
# Set these TWO variables correctly for the current VM.
NODE=as-pg2
IP=192.168.136.11

scp root@192.168.136.10:/etc/etcd/etcd.yml /etc/etcd/etcd.yml
sed -i \
  -e "s/^name: as-pg1$/name: ${NODE}/" \
  -e "s|^listen-client-urls:.*|listen-client-urls: http://${IP}:2379,http://127.0.0.1:2379|" \
  -e "s|^advertise-client-urls:.*|advertise-client-urls: http://${IP}:2379|" \
  -e "s|^listen-peer-urls:.*|listen-peer-urls: http://${IP}:2380|" \
  -e "s|^initial-advertise-peer-urls:.*|initial-advertise-peer-urls: http://${IP}:2380|" \
  /etc/etcd/etcd.yml
chown root:etcd /etc/etcd/etcd.yml
chmod 0640 /etc/etcd/etcd.yml

grep -E '^(name|listen-client-urls|advertise-client-urls|listen-peer-urls|initial-advertise-peer-urls|initial-cluster):' /etc/etcd/etcd.yml
```

**For `as-pg3`, change both variables before running these commands.** Never globally replace `.10` everywhere: that would corrupt the shared `initial-cluster` membership list.

### 4.3 `etcd.service` on each node

```bash
cat > /etc/systemd/system/etcd.service <<'EOF_SERVICE'
[Unit]
Description=etcd distributed key-value store
Documentation=https://etcd.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd
WorkingDirectory=/var/lib/etcd
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.yml
Restart=on-failure
RestartSec=5
TimeoutStartSec=180
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF_SERVICE

systemctl daemon-reload
systemd-analyze verify /etc/systemd/system/etcd.service
systemctl enable etcd
systemctl start etcd
systemctl status etcd --no-pager -l
```

Run the start command on **all three nodes** to form the first quorum. Startup on a single node may wait for peers.

## 5. Verify etcd, quorum, and recovery

Run on any node:

```bash
export ETCDCTL_ENDPOINTS='http://192.168.136.10:2379,http://192.168.136.11:2379,http://192.168.136.12:2379'

etcdctl endpoint status -w table
etcdctl endpoint health -w table
etcdctl --endpoints=http://192.168.136.10:2379 member list -w table
```

**Observed:** three voting, started etcd members; exactly one Raft leader; all endpoints healthy; raft term `2`, with raft index and applied index `9` in a captured snapshot. etcd Raft leader was `as-pg1` at that moment. A later election may legitimately select a different etcd leader.

### Failure drill actually performed

1. `systemctl stop etcd` on `as-pg3`.
2. From the surviving nodes, health checks showed `as-pg1` and `as-pg2` still healthy (**2 of 3 quorum**).
3. `systemctl start etcd` on `as-pg3`.
4. All three endpoints were healthy again.

**Important:** A three-member etcd cluster cannot tolerate the loss of **two** voters. Stopping etcd members does not simulate a PostgreSQL primary crash.

Useful operations:

```bash
journalctl -u etcd -n 100 --no-pager
journalctl -u etcd -f
etcdctl alarm list
ss -lntp | grep -E ':(2379|2380)[[:space:]]'
```

**Optional backup (not confirmed completed in this lab):**

```bash
mkdir -p /root/etcd-backups
etcdctl --endpoints=http://192.168.136.10:2379 \
  snapshot save /root/etcd-backups/etcd-snapshot.db
etcdutl snapshot status /root/etcd-backups/etcd-snapshot.db -w table
```

etcd snapshots save **HA metadata**, not PostgreSQL tables, datafiles, or WAL. Keep separate PostgreSQL backups using pgBackRest or another supported tool.

## 6. Install PostgreSQL 18 and Patroni

On **all three nodes**, as root:

```bash
dnf install -y \
  https://download.postgresql.org/pub/repos/yum/reporpms/EL-10-x86_64/pgdg-redhat-repo-latest.noarch.rpm

dnf install -y postgresql18 postgresql18-server postgresql18-contrib
/usr/pgsql-18/bin/postgres --version
```

The lab observed PostgreSQL `18.7` on `as-pg1`.

Install Patroni in an isolated Python virtual environment:

```bash
dnf install -y python3 python3-pip
python3 -m venv /opt/patroni/venv
/opt/patroni/venv/bin/python -m pip install 'patroni[psycopg3,etcd3]==4.1.5'
/opt/patroni/venv/bin/patroni --version
```

**Why:** Patroni requires a PostgreSQL driver (`psycopg3`) and an etcd v3 client (`etcd3`). The virtual environment avoids mixing its Python dependencies with the OS package manager.

**Critical:** Do **not** run `postgresql-18-setup initdb` or start `postgresql-18.service`; Patroni will initialize and manage the instance.

```bash
systemctl is-active postgresql-18 || true
test -f /var/lib/pgsql/18/data/PG_VERSION && echo 'Already initialized' || echo 'Not initialized'
```

Before the first Patroni start, the native PostgreSQL service was `inactive` and PGDATA uninitialized on the checked nodes.

## 7. Configure Patroni on all three nodes

### 7.1 `as-pg1` configuration template

Create `/etc/patroni/patroni.yml` on `as-pg1` (replace each password placeholder **locally** before startup):

```yaml
scope: pg18-ha-lab
namespace: /service/
name: as-pg1

restapi:
  listen: 192.168.136.10:8008
  connect_address: 192.168.136.10:8008

etcd3:
  hosts: 192.168.136.10:2379,192.168.136.11:2379,192.168.136.12:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        wal_level: replica
        hot_standby: 'on'
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: 'on'
        password_encryption: scram-sha-256
      pg_hba:
        - local all all peer
        - host all all 127.0.0.1/32 scram-sha-256
        - host replication replicator 192.168.136.0/24 scram-sha-256
        - host all all 192.168.136.0/24 scram-sha-256
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 192.168.136.10:5432
  connect_address: 192.168.136.10:5432
  data_dir: /var/lib/pgsql/18/data
  bin_dir: /usr/pgsql-18/bin
  pgpass: /var/lib/pgsql/.pgpass_patroni
  authentication:
    superuser:
      username: postgres
      password: 'REPLACE_WITH_PRIVATE_PG_LAB_PASSWORD'
    replication:
      username: replicator
      password: 'REPLACE_WITH_PRIVATE_REPLICATION_LAB_PASSWORD'
    rewind:
      username: rewind_user
      password: 'REPLACE_WITH_PRIVATE_REWIND_LAB_PASSWORD'
```

Do **not** commit the live file with passwords. Publish only a sanitized template. Keep the same password for each corresponding account across the three nodes.

**Key concepts:** `scope` = shared Patroni cluster ID; `name` = unique node ID; `restapi` = local node management/health; `etcd3.hosts` = shared DCS endpoints; `bootstrap.dcs` = cluster-wide settings applied on initial bootstrap; `postgresql` = local database paths/network settings. **After bootstrap, cluster-wide dynamic settings are changed using `patronictl edit-config`, not simply by modifying `bootstrap.dcs` on an existing node.**

### 7.2 Configure as-pg2 and as-pg3

Copy the *private* as-pg1 config securely to each node, then change only the local node name and its REST/PostgreSQL addresses. Example on **as-pg2**:

```bash
install -d -o root -g postgres -m 0750 /etc/patroni
scp root@192.168.136.10:/etc/patroni/patroni.yml /etc/patroni/patroni.yml
sed -i \
  -e 's/^name: as-pg1$/name: as-pg2/' \
  -e 's|^  listen: 192.168.136.10:8008$|  listen: 192.168.136.11:8008|' \
  -e 's|^  connect_address: 192.168.136.10:8008$|  connect_address: 192.168.136.11:8008|' \
  -e 's|^  listen: 192.168.136.10:5432$|  listen: 192.168.136.11:5432|' \
  -e 's|^  connect_address: 192.168.136.10:5432$|  connect_address: 192.168.136.11:5432|' \
  /etc/patroni/patroni.yml
```

On **as-pg3** repeat with `name: as-pg3` and local IP `192.168.136.12`, replacing the five corresponding lines that refer to `as-pg1` / `.10`.

**Do not change** `scope`, `namespace`, `etcd3.hosts`, or the shared PostgreSQL role passwords. Do not replace the `.10` address inside the `etcd3.hosts` list.

### 7.3 Permissions and validation

On **each node**:

```bash
install -d -o root -g postgres -m 0750 /etc/patroni
chown root:postgres /etc/patroni/patroni.yml
chmod 0640 /etc/patroni/patroni.yml

runuser -u postgres -- \
  /opt/patroni/venv/bin/patroni --validate-config /etc/patroni/patroni.yml
echo "Validation exit code: $?"

grep -E '^(scope|namespace|name):|^  (listen|connect_address):' /etc/patroni/patroni.yml
```

**Observed:** as-pg1 validated successfully, including when run as `postgres`; as-pg3 returned exit code `0`; as-pg2 supplied correct node-specific settings and successfully joined as a streaming replica, but its actual validation exit code was not separately captured. No secrets are printed by the suggested `grep`.

## 8. Create the Patroni systemd service

Run on **all three nodes**:

```bash
cat > /etc/systemd/system/patroni.service <<'EOF_PATRONI'
[Unit]
Description=Patroni PostgreSQL HA Manager
Wants=network-online.target
After=network-online.target

[Service]
Type=notify
User=postgres
Group=postgres
WorkingDirectory=/var/lib/pgsql
Environment=PATH=/usr/pgsql-18/bin:/opt/patroni/venv/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=/opt/patroni/venv/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=5
TimeoutStartSec=0
TimeoutStopSec=120
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF_PATRONI

systemctl daemon-reload
systemd-analyze verify /etc/systemd/system/patroni.service
systemctl disable postgresql-18
systemctl enable patroni
```

**Why:** `systemd → Patroni → PostgreSQL`. Do not separately start/stop PostgreSQL using `postgresql-18.service` once Patroni owns the database. The service runs as the `postgres` OS user; credentials are only in protected local configuration.

**Observed on as-pg1:** `systemd-analyze verify` printed no errors; Patroni configuration validation as the `postgres` user exited `0`; native PostgreSQL service was inactive.

## 9. Bootstrap the PostgreSQL cluster

### 9.1 Start primary first (as-pg1 only)

Ensure placeholders are gone without displaying the passwords:

```bash
grep -q 'REPLACE_WITH_PRIVATE_' /etc/patroni/patroni.yml \
  && echo 'STOP: replace sample passwords' \
  || echo 'No password placeholders found'
```

Then start:

```bash
systemctl start --no-block patroni
systemctl status patroni --no-pager -l
journalctl -u patroni -n 60 --no-pager
/opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```

**Observed initial bootstrap**:

```text
Cluster: pg18-ha-lab (7709168589345127344)
Member   Host             Role     State    TL
as-pg1   192.168.136.10   Leader   running  1
```

Patroni coordinated startup through etcd, initialized PGDATA and started PostgreSQL as the first primary. **Do not** run `initdb` manually.

### 9.2 Add standbys one at a time

On `as-pg2` (after config and service verification), then `as-pg3`:

```bash
systemctl start --no-block patroni
systemctl status patroni --no-pager -l
```

Patroni uses PostgreSQL `pg_basebackup` by default to initialize replicas when no alternative replica creation method is configured; it then starts WAL streaming. **The DBA did not have to run `pg_basebackup` manually.**

**Actual `patronictl list` result after both joined:**

```text
Cluster: pg18-ha-lab (7709168589345127344)
Member   Host             Role     State      TL   Receive LSN   Replay LSN   Lag
as-pg1   192.168.136.10   Leader   running     1      -             -         -
as-pg2   192.168.136.11   Replica  streaming   1   0/5000060     0/5000060     0
as-pg3   192.168.136.12   Replica  streaming   1   0/5000060     0/5000060     0
```

**Interpretation:** both standby members were active and had replayed received WAL through `0/5000060` at the time of that check. These are point-in-time metrics, not a guarantee of continuously zero lag.

## 10. Verify replication and application data

### 10.1 Check from the primary

On `as-pg1` (before switchover):

```bash
runuser -u postgres -- /usr/pgsql-18/bin/psql -d postgres -c "
SELECT application_name, client_addr, state, sync_state, sent_lsn, replay_lsn
FROM pg_stat_replication
ORDER BY application_name;"
```

**Observed output:**

```text
 application_name | client_addr     | state     | sync_state | sent_lsn  | replay_lsn
------------------+-----------------+-----------+------------+-----------+-----------
 as-pg2           | 192.168.136.11  | streaming | async      | 0/5000060 | 0/5000060
 as-pg3           | 192.168.136.12  | streaming | async      | 0/5000060 | 0/5000060
```

**Why `async` matters:** These are **asynchronous** standbys. PostgreSQL can acknowledge a commit before a standby receives it; a hard primary failure can therefore lose recent committed transactions. `0` observed lag does **not** establish synchronous or zero-RPO protection.

### 10.2 Write on primary

On `as-pg1`:

```bash
runuser -u postgres -- /usr/pgsql-18/bin/psql \
  -d postgres -v ON_ERROR_STOP=1 -c "
CREATE TABLE IF NOT EXISTS public.ha_replication_test (
  id integer PRIMARY KEY,
  note text NOT NULL
);
INSERT INTO public.ha_replication_test (id, note)
VALUES (1, 'Inserted on as-pg1')
ON CONFLICT (id) DO UPDATE SET note = EXCLUDED.note;
SELECT * FROM public.ha_replication_test;"
```

### 10.3 Read on both standbys

On **as-pg2 and as-pg3**:

```bash
runuser -u postgres -- /usr/pgsql-18/bin/psql -d postgres -c "
SELECT pg_is_in_recovery() AS is_standby, id, note
FROM public.ha_replication_test;"
```

**Observed on each standby:**

```text
 is_standby | id | note
------------+----+--------------------
 t          |  1 | Inserted on as-pg1
```

The test validates actual end-to-end data replay on both standby nodes. A standby is read-only while `pg_is_in_recovery() = true`.

## 11. Controlled switchover: as-pg1 to as-pg2

**Status:** Operator reported the switchover worked. A complete post-switchover `patronictl list` result was **not supplied**, so the commands below document the procedure and the expected topology; they do not present an invented verified result.

### 11.1 Understand the role change

| Before | After a successful switchover (expected) |
|---|---|
| `as-pg1` = Leader / primary | `as-pg1` = Replica / standby |
| `as-pg2` = Replica / standby | `as-pg2` = Leader / primary |
| `as-pg3` = Replica / standby | `as-pg3` = Replica / standby |
| Timeline 1 | Usually a new timeline (often 2); verify |

Unlike an unplanned failover, a **switchover** is an orderly role change from a healthy primary to a healthy standby. In Oracle terminology, this resembles a Data Guard *switchover*, not a forced failover. A brief client disconnect can still occur.

### 11.2 Prechecks

On `as-pg1` or any Patroni node:

```bash
/opt/patroni/venv/bin/patronictl \
  -c /etc/patroni/patroni.yml list
```

Verify a healthy leader, both replicas streaming, acceptable lag, and no application write workload during this teaching exercise. In production, also coordinate application drain, client routing, connection pools, and a change window.

### 11.3 Run the switchover

```bash
/opt/patroni/venv/bin/patronictl \
  -c /etc/patroni/patroni.yml \
  switchover pg18-ha-lab \
  --leader as-pg1 \
  --candidate as-pg2
```

Choose immediate execution (`now`) if asked, and review/confirm the prompt. Do not add `--force` for your first exercise. Do not manually `pg_ctl promote` or run `pg_promote()` while Patroni manages the cluster.

### 11.4 Post-switchover verification (capture these outputs next)

1. On any Patroni node:

   ```bash
   /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
   ```

   Expected: `as-pg2 Leader running`; `as-pg1` and `as-pg3 Replica streaming`, usually with a higher timeline.

2. On **each node**:

   ```bash
   runuser -u postgres -- /usr/pgsql-18/bin/psql \
     -d postgres -Atc 'SELECT pg_is_in_recovery();'
   ```

   Expected: `as-pg2 = f` (new primary); `as-pg1 = t` and `as-pg3 = t` (standbys).

3. On the **new primary as-pg2**:

   ```bash
   runuser -u postgres -- /usr/pgsql-18/bin/psql -d postgres -c "
   SELECT application_name, client_addr, state, sync_state, sent_lsn, replay_lsn
   FROM pg_stat_replication ORDER BY application_name;"
   ```

   Expected: replication connections from `as-pg1` and `as-pg3`.

4. Verify application data and then run an insert **on as-pg2**, not on the demoted `as-pg1`:

   ```bash
   runuser -u postgres -- /usr/pgsql-18/bin/psql -d postgres -c \
     'SELECT * FROM public.ha_replication_test;'
   ```

   Check that the existing record is still visible on the new primary and the two replicas. **Capture actual outputs** before marking these postchecks passed in this README.

> **Client-routing gap:** We have not installed HAProxy, PgBouncer, VIP, or a cluster-aware connection endpoint. Applications hard-coded to `192.168.136.10:5432` will **not** automatically follow the new primary `192.168.136.11:5432`. This needs addressing before a production-style failover test.

## 12. Troubleshooting cheat sheet

| Symptom | First check | Why |
|---|---|---|
| `patronictl list` cannot connect | `etcdctl endpoint health -w table` | Confirm the shared DCS can form quorum |
| `patroni.service` failed | `journalctl -u patroni -n 100 --no-pager` | Locate parse, permission, PostgreSQL or bootstrap errors |
| Standby not streaming | `pg_stat_replication` on current primary | See whether the standby has connected |
| Replica cloning failed | Patroni logs on standby | Check `pg_basebackup`, credentials and `pg_hba` |
| Primary unexpectedly read-only | `SELECT pg_is_in_recovery();` | Determine whether it was demoted or failed over |
| etcd member unavailable | `journalctl -u etcd -n 100 --no-pager` | Distinguish service, port and quorum failures |
| Unexpected role/timeline | `patronictl list` and Patroni logs | Observe switchover/failover transitions |
| PostgreSQL config differs from file edits | `patronictl show-config` | Cluster-wide dynamic settings are held in DCS |

Quick commands:

```bash
# PostgreSQL HA topology — any Patroni node
/opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list

# DCS health — any node with etcdctl
etcdctl --endpoints=http://192.168.136.10:2379,http://192.168.136.11:2379,http://192.168.136.12:2379 endpoint health -w table

# Local process and logs
systemctl status patroni --no-pager -l
journalctl -u patroni -n 100 --no-pager
journalctl -u etcd -n 100 --no-pager

# Current node's database role
runuser -u postgres -- /usr/pgsql-18/bin/psql -d postgres \
  -c 'SELECT pg_is_in_recovery();'

# Cluster-level dynamic configuration
/opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml show-config

# Patroni REST checks (plain HTTP, private LAB only)
curl -i http://192.168.136.10:8008/primary
curl -i http://192.168.136.11:8008/primary
curl -i http://192.168.136.12:8008/primary
```

An HTTP `200` response to `/primary` indicates the **current** Patroni primary on that node; after switchover it should move to the new primary. A `503` on a replica's `/primary` is normal, not proof of total service failure.

## 13. Production readiness checklist

This laboratory **is not a banking/finance production reference architecture yet**. Before that, plan and test:

- [ ] etcd TLS/mTLS on both client and peer networks; authentication and least privilege; segregated network, firewall policy and certificate lifecycle.
- [ ] Store PostgreSQL and Patroni credentials in a suitable secrets-management workflow; no real passwords in GitHub, logs, or copied templates.
- [ ] Independent failure domains and sufficient etcd quorum availability; do not place all voters on one host/storage fault domain.
- [ ] PostgreSQL backup/PITR with pgBackRest, archive retention and restore tests; etcd snapshots with restore drills.
- [ ] HAProxy or another managed writable endpoint for application reconnects, connection pooling and reliable health checks.
- [ ] Define RPO/RTO, understand async data-loss exposure, and evaluate synchronous replication if required.
- [ ] Test planned switchover, unplanned primary crash, network partition, DCS quorum loss, standby lag, and old-primary rejoin/`pg_rewind`.
- [ ] Prometheus/Grafana or comparable monitoring for PostgreSQL, Patroni, etcd, replication lag, disk, WAL and cluster changes.
- [ ] OS hardening, controlled upgrades, capacity planning, reliable time sync and documented operational runbooks.

## 14. Learning outcomes and references

**Verified in the supplied lab output:**

- Three-member etcd cluster formed and passed healthy-member checks; survived one etcd member stopping and rejoining.
- Patroni initial PostgreSQL primary `as-pg1` bootstrapped successfully (cluster ID `7709168589345127344`).
- `as-pg2` and `as-pg3` joined as streaming standbys, timeline `1`, matching receive/replay LSN in the captured snapshot.
- `pg_stat_replication` returned both replicas as `streaming`, `sync_state = async`.
- A test row inserted on `as-pg1` was read from **both** standby databases, and both returned `pg_is_in_recovery() = true`.
- The operator subsequently **reported a successful controlled switchover** to `as-pg2`; the actual after-switchover role/LSN/timeline output has yet to be saved.

**Next practical topics:** verify post-switchover roles and timeline; simulate an unplanned primary outage; recover/rejoin the old primary; configure HAProxy; harden TLS and monitor the stack.

**Documentation:**

- [Patroni documentation](https://patroni.readthedocs.io/en/latest/)
- [Patroni `patronictl` switchover commands](https://patroni.readthedocs.io/en/latest/patronictl.html)
- [Patroni REST API](https://patroni.readthedocs.io/en/latest/rest_api.html)
- [etcd documentation](https://etcd.io/docs/)
- [PostgreSQL physical replication](https://www.postgresql.org/docs/18/warm-standby.html)
- [PostgreSQL streaming replication settings](https://www.postgresql.org/docs/18/runtime-config-replication.html)

---

**Maintainer note:** Keep this README free of live passwords. When rerunning from scratch, first verify whether etcd has existing membership data and whether PostgreSQL PGDATA already contains a cluster. **Never delete production data directories to “reset” HA.**

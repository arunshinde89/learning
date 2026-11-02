# OEM 24ai – Pre-Patch Backup and Restore Procedure

## Purpose

This document describes the pre-patch backup and recovery procedure for Oracle Enterprise Manager 24ai OMS.

Use this procedure before applying:

- OEM Release Updates
- OMSPatcher patches
- OPatch patches
- WebLogic-related patches
- Middleware patches

The goal is to have a complete rollback option in case the patch fails.

---

# Environment Example

OEM base directory:

```bash
/oracle/oms_base/em
```

Directory structure:

```text
/oracle/oms_base/em/
├── gc_inst
├── agent
├── middleware
└── wallets
```

Backup location example:

```bash
/backup
```

---

# Backup Strategy

Before patching, protect the following:

```text
1. OEM Repository Database
2. OMS configuration
3. OEM filesystem
4. Oracle Inventory
5. Current OPatch inventory
6. Current OMSPatcher inventory
```

A filesystem backup alone may not be sufficient because some OEM patches also modify the repository database.

---

# 1. Check Current OMS Status

Run as the Oracle software owner:

```bash
$OMS_HOME/bin/emctl status oms -details
```

Check the agent:

```bash
$AGENT_HOME/bin/emctl status agent
```

---

# 2. Export OMS Configuration

This is an important OEM recovery backup.

Create a backup directory:

```bash
mkdir -p /oracle/oms_base/oem24ai_config_backup
```

Run:

```bash
$OMS_HOME/bin/emctl exportconfig oms \
-dir /oracle/oms_base/oem24ai_config_backup
```

Verify:

```bash
ls -lh /oracle/oms_base/oem24ai_config_backup
```

This backup protects important OMS configuration information including configuration required for OMS recovery.

---

# 3. Save Current OPatch Inventory

```bash
$OMS_HOME/OPatch/opatch lsinventory -detail \
> /oracle/oms_base/opatch_lsinventory_before_patch.txt
```

Verify:

```bash
cat /oracle/oms_base/opatch_lsinventory_before_patch.txt
```

---

# 4. Save Current OMSPatcher Inventory

```bash
$OMS_HOME/OMSPatcher/omspatcher lspatches \
> /oracle/oms_base/omspatcher_lspatches_before_patch.txt
```

Verify:

```bash
cat /oracle/oms_base/omspatcher_lspatches_before_patch.txt
```

---

# 5. Create Repository Database Guaranteed Restore Point

Connect to the OEM repository database as SYSDBA.

Check existing restore points:

```sql
SET LINES 200

COL NAME FORMAT A40
COL GUARANTEE_FLASHBACK_DATABASE FORMAT A10
COL TIME FORMAT A35

SELECT
    name,
    guarantee_flashback_database,
    time,
    scn,
    storage_size
FROM v$restore_point
ORDER BY time;
```

If an old patch restore point exists and is no longer required:

```sql
DROP RESTORE POINT old_restore_point_name;
```

Create a new guaranteed restore point:

```sql
CREATE RESTORE POINT OEM24AI_BEFORE_PATCH
GUARANTEE FLASHBACK DATABASE;
```

Verify:

```sql
SELECT
    name,
    guarantee_flashback_database,
    time,
    scn
FROM v$restore_point;
```

Expected:

```text
OEM24AI_BEFORE_PATCH    YES
```

Also check FRA usage:

```sql
SELECT
    name,
    space_limit/1024/1024/1024 AS limit_gb,
    space_used/1024/1024/1024 AS used_gb,
    space_reclaimable/1024/1024/1024 AS reclaimable_gb
FROM v$recovery_file_dest;
```

Guaranteed restore points may cause flashback logs to accumulate, so FRA usage must be monitored.

---

# 6. Stop OEM Before Filesystem Backup

Stop OMS:

```bash
$OMS_HOME/bin/emctl stop oms -all
```

Stop Agent:

```bash
$AGENT_HOME/bin/emctl stop agent
```

Verify that OEM processes are stopped:

```bash
ps -ef | grep /oracle/oms_base/em | grep -v grep
```

---

# 7. Take Complete OEM Filesystem Backup

Run the filesystem backup as `root`.

Create backup directory:

```bash
mkdir -p /backup
```

Go to the parent directory:

```bash
cd /oracle/oms_base
```

Take the complete backup:

```bash
tar --acls --xattrs --selinux --numeric-owner \
-czpf /backup/OEM24AI_PREPATCH.tar.gz \
em
```

This includes:

```text
/oracle/oms_base/em/gc_inst
/oracle/oms_base/em/agent
/oracle/oms_base/em/middleware
/oracle/oms_base/em/wallets
```

Important:

```text
Do NOT create the tar file inside /oracle/oms_base/em.
```

Keep it outside the source directory.

---

# 8. Verify Filesystem Backup

Check archive integrity:

```bash
gzip -t /backup/OEM24AI_PREPATCH.tar.gz
```

If there is no output, the gzip archive passed the integrity check.

Check archive contents:

```bash
tar -tzf /backup/OEM24AI_PREPATCH.tar.gz | head -50
```

Check backup size:

```bash
ls -lh /backup/OEM24AI_PREPATCH.tar.gz
```

Compare source size:

```bash
du -sh /oracle/oms_base/em
```

Backup size:

```bash
du -sh /backup/OEM24AI_PREPATCH.tar.gz
```

The sizes will normally differ because gzip compression is used.

---

# 9. Backup Oracle Inventory

Find Oracle Inventory:

```bash
cat /etc/oraInst.loc
```

Example:

```text
inventory_loc=/oracle/oraInventory
inst_group=dba
```

If Oracle Inventory is outside `/oracle/oms_base/em`, back it up separately.

Run as `root`:

```bash
cd /oracle
```

```bash
tar --acls --xattrs --selinux --numeric-owner \
-czpf /backup/OEM24AI_oraInventory_PREPATCH.tar.gz \
oraInventory
```

Verify:

```bash
gzip -t /backup/OEM24AI_oraInventory_PREPATCH.tar.gz
```

Check:

```bash
tar -tzf /backup/OEM24AI_oraInventory_PREPATCH.tar.gz | head
```

---

# Final Pre-Patch Backup Checklist

Before starting the patch, verify that the following exist:

```text
Repository DB:
    Guaranteed Restore Point
    OEM24AI_BEFORE_PATCH

Filesystem:
    /backup/OEM24AI_PREPATCH.tar.gz

Oracle Inventory:
    /backup/OEM24AI_oraInventory_PREPATCH.tar.gz

OMS Configuration:
    emctl exportconfig oms backup

Inventory Files:
    opatch_lsinventory_before_patch.txt
    omspatcher_lspatches_before_patch.txt
```

---

# RESTORE PROCEDURE

Use this only if patch recovery or rollback is required.

Important:

```text
Filesystem restoration should be performed as root.
```

The backup was taken as `root` using:

```bash
--acls
--xattrs
--selinux
--numeric-owner
```

Restoring as `root` allows original ownership, UID/GID, ACLs and filesystem attributes to be restored correctly.

---

# 1. Stop OEM

If OEM components are running:

```bash
su - oracle
```

Stop OMS:

```bash
$OMS_HOME/bin/emctl stop oms -all
```

Stop agent:

```bash
$AGENT_HOME/bin/emctl stop agent
```

Return to root:

```bash
exit
```

Verify:

```bash
ps -ef | grep /oracle/oms_base/em | grep -v grep
```

---

# 2. Preserve Failed/Patched Installation

Do not immediately delete the failed installation.

Run as root:

```bash
cd /oracle/oms_base
```

Rename it:

```bash
mv em em_failed_patch
```

Verify:

```bash
ls -ld em*
```

Example:

```text
em_failed_patch
```

---

# 3. Restore OEM Filesystem

Run as root:

```bash
cd /oracle/oms_base
```

Restore:

```bash
tar --acls --xattrs --selinux --numeric-owner \
-xzpf /backup/OEM24AI_PREPATCH.tar.gz
```

Because the archive contains:

```text
em/
```

the restore recreates:

```text
/oracle/oms_base/em
```

including:

```text
gc_inst
agent
middleware
wallets
```

---

# 4. Verify Ownership and Permissions

Check top-level directories:

```bash
ls -ld /oracle/oms_base/em
```

```bash
ls -ld /oracle/oms_base/em/*
```

Expected ownership should normally resemble:

```text
oracle dba
```

Check deeper:

```bash
find /oracle/oms_base/em -maxdepth 2 \
-printf '%u:%g %p\n' | head -30
```

Do not automatically run:

```bash
chown -R oracle:dba /oracle/oms_base/em
```

unless ownership is actually incorrect.

Some files may intentionally have different ownership or permissions.

---

# 5. Restore Oracle Inventory If Required

Check current inventory:

```bash
cat /etc/oraInst.loc
```

Example:

```text
inventory_loc=/oracle/oraInventory
```

Preserve the failed inventory:

```bash
cd /oracle
```

```bash
mv oraInventory oraInventory_failed_patch
```

Restore:

```bash
tar --acls --xattrs --selinux --numeric-owner \
-xzpf /backup/OEM24AI_oraInventory_PREPATCH.tar.gz
```

Verify:

```bash
ls -ld /oracle/oraInventory
```

---

# 6. Repository Database Recovery

Important:

Restoring only the OEM filesystem may not be sufficient if OMSPatcher modified the OEM repository schema.

The filesystem and repository database should represent the same pre-patch state.

Conceptually:

```text
       OEM 24ai Rollback

Repository Database
        |
        |
Guaranteed Restore Point
        |
        |
        +----------------------+
                               |
                               v
                    OEM Filesystem Backup
                    /oracle/oms_base/em
```

Repository rollback procedures should therefore be coordinated with the OEM filesystem recovery.

Do not independently restore the filesystem while leaving the repository at a newer patch level.

---

# 7. Start OEM After Recovery

Switch to the Oracle software owner:

```bash
su - oracle
```

Verify environment:

```bash
echo $OMS_HOME
echo $AGENT_HOME
echo $ORACLE_HOME
```

Start OMS:

```bash
$OMS_HOME/bin/emctl start oms
```

Check OMS:

```bash
$OMS_HOME/bin/emctl status oms -details
```

Start Agent:

```bash
$AGENT_HOME/bin/emctl start agent
```

Check:

```bash
$AGENT_HOME/bin/emctl status agent
```

---

# 8. Verify Patch Inventory After Recovery

Check OPatch:

```bash
$OMS_HOME/OPatch/opatch lsinventory
```

Check OMSPatcher:

```bash
$OMS_HOME/OMSPatcher/omspatcher lspatches
```

Compare these with the files saved before patching:

```text
opatch_lsinventory_before_patch.txt

omspatcher_lspatches_before_patch.txt
```

---

# 9. After Successful Patch

Once the patch has been tested and confirmed stable, remove the guaranteed restore point.

Check:

```sql
SELECT
    name,
    guarantee_flashback_database,
    time
FROM v$restore_point;
```

Drop:

```sql
DROP RESTORE POINT OEM24AI_BEFORE_PATCH;
```

Verify:

```sql
SELECT name
FROM v$restore_point;
```

This is important because keeping a guaranteed restore point for too long may cause FRA usage to increase.

---

# Quick Backup Commands

```bash
# Export OMS configuration
$OMS_HOME/bin/emctl exportconfig oms \
-dir /oracle/oms_base/oem24ai_config_backup

# Save patch inventory
$OMS_HOME/OPatch/opatch lsinventory -detail \
> /oracle/oms_base/opatch_lsinventory_before_patch.txt

$OMS_HOME/OMSPatcher/omspatcher lspatches \
> /oracle/oms_base/omspatcher_lspatches_before_patch.txt

# Stop OEM
$OMS_HOME/bin/emctl stop oms -all
$AGENT_HOME/bin/emctl stop agent
```

Run as root:

```bash
cd /oracle/oms_base

tar --acls --xattrs --selinux --numeric-owner \
-czpf /backup/OEM24AI_PREPATCH.tar.gz \
em
```

Verify:

```bash
gzip -t /backup/OEM24AI_PREPATCH.tar.gz

tar -tzf /backup/OEM24AI_PREPATCH.tar.gz | head
```

---

# Quick Restore Commands

Run as root:

```bash
cd /oracle/oms_base

mv em em_failed_patch
```

Restore:

```bash
tar --acls --xattrs --selinux --numeric-owner \
-xzpf /backup/OEM24AI_PREPATCH.tar.gz
```

Verify:

```bash
ls -ld /oracle/oms_base/em
ls -ld /oracle/oms_base/em/*
```

Start OEM as Oracle:

```bash
su - oracle
```

```bash
$OMS_HOME/bin/emctl start oms
$OMS_HOME/bin/emctl status oms -details

$AGENT_HOME/bin/emctl start agent
$AGENT_HOME/bin/emctl status agent
```

---

# Important Points to Remember

```text
1. Take filesystem backup as root.

2. Restore filesystem backup as root.

3. Stop OMS and Agent before taking the filesystem backup.

4. Always run emctl exportconfig oms.

5. Backup Oracle Inventory separately if it is outside the OEM base.

6. Save OPatch and OMSPatcher inventories before patching.

7. Create a Guaranteed Restore Point in the OEM repository DB.

8. Filesystem restore alone may not be enough if the repository schema
   was changed by the patch.

9. Filesystem and repository DB should be restored to a consistent
   pre-patch state.

10. Remove the Guaranteed Restore Point after the patch is confirmed
    successful.
```

## Recommended Rollback Protection

```text
Repository DB Restore Point
          +
emctl exportconfig oms
          +
Complete /oracle/oms_base/em TAR backup
          +
Oracle Inventory backup
          +
Pre-patch OPatch/OMSPatcher inventory
```

This combination provides a much safer rollback position before applying OEM 24ai patches.

# ============================================================
# pgBackRest Configuration - Example
# PostgreSQL Version : 18
# Stanza             : pg18lab
# Purpose            : Production-style reference
# ============================================================


# ============================================================
# STANZA CONFIGURATION
# ============================================================

[pg18lab]

# PostgreSQL data directory
pg1-path=/postgres/data/pg18

# PostgreSQL port
pg1-port=5432


# ============================================================
# GLOBAL CONFIGURATION
# ============================================================

[global]


# ------------------------------------------------------------
# Repository
# ------------------------------------------------------------

# pgBackRest repository location
repo1-path=/postgres/backup/pg18


# ------------------------------------------------------------
# Retention
# ------------------------------------------------------------

# Time-based retention policy
repo1-retention-full-type=time

# Keep approximately 14 days of full-backup recovery history
repo1-retention-full=14

# Keep backup history metadata for reporting/reference
repo1-retention-history=365


# ------------------------------------------------------------
# Compression
# ------------------------------------------------------------

# Use Zstandard compression
compress-type=zst

# Balanced compression level
compress-level=3


# ------------------------------------------------------------
# Performance
# ------------------------------------------------------------

# Maximum parallel worker processes for backup/restore
process-max=4

# Force a checkpoint so backup can start quickly
start-fast=y


# ------------------------------------------------------------
# Logging
# ------------------------------------------------------------

# Console logging
log-level-console=info

# Detailed file logging
log-level-file=detail

# pgBackRest log directory
log-path=/var/log/pgbackrest


# ------------------------------------------------------------
# Timeouts
# ------------------------------------------------------------

# Maximum time allowed for database operations
db-timeout=1800

# Maximum protocol timeout
protocol-timeout=3600


# ------------------------------------------------------------
# WAL Archiving
# ------------------------------------------------------------

# Enable asynchronous WAL archive processing
archive-async=y

# Queue/spool directory used by async archive-push/archive-get
spool-path=/var/spool/pgbackrest


# ============================================================
# ARCHIVE-PUSH SETTINGS
# ============================================================

[global:archive-push]

# Parallel workers for WAL archive-push
process-max=2

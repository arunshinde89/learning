# ============================================================
# pgBackRest Configuration
# Environment : PostgreSQL 18
# Stanza      : pg18lab
# Purpose     : Production-style baseline
# ============================================================


# ============================================================
# STANZA CONFIGURATION
# ============================================================

[pg18lab]

# PostgreSQL data directory
pg1-path=/pgdata/18/data

# PostgreSQL port
pg1-port=5432


# ============================================================
# GLOBAL CONFIGURATION
# ============================================================

[global]


# ------------------------------------------------------------
# Repository
# ------------------------------------------------------------

# Backup repository location
repo1-path=/pgbackup/pgbackrest


# ------------------------------------------------------------
# Retention
# ------------------------------------------------------------

# Number of FULL backup chains to retain
repo1-retention-full=4

# Keep backup history metadata
repo1-retention-history=365


# ------------------------------------------------------------
# Compression
# ------------------------------------------------------------

# Zstandard compression
compress-type=zst

# Compression level
compress-level=3


# ------------------------------------------------------------
# Performance
# ------------------------------------------------------------

# Number of parallel processes
process-max=4

# Force fast checkpoint before backup starts
start-fast=y


# ------------------------------------------------------------
# Logging
# ------------------------------------------------------------

# Console logging level
log-level-console=info

# File logging level
log-level-file=detail

# Log directory
log-path=/var/log/pgbackrest


# ------------------------------------------------------------
# Timeouts
# ------------------------------------------------------------

# Database timeout
db-timeout=1800

# Protocol timeout
protocol-timeout=3600


# ------------------------------------------------------------
# WAL Archiving
# ------------------------------------------------------------

# Enable asynchronous WAL archiving
archive-async=y

# Spool location used for async archiving
spool-path=/var/spool/pgbackrest


# ============================================================
# ARCHIVE-PUSH SPECIFIC SETTINGS
# ============================================================

[global:archive-push]

# Parallel processes for WAL archive-push
process-max=2

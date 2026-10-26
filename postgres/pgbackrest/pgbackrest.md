[pg18lab]
pg1-path=/pgdata/18/data
pg1-port=5432

[global]

# =====================================================
# REPOSITORY
# =====================================================
repo1-path=/pgbackup/pgbackrest

# Keep 4 full backup chains
# Example: weekly FULL = approximately 4 weeks
repo1-retention-full=4

# Keep backup history metadata for reporting/trending
repo1-retention-history=365


# =====================================================
# COMPRESSION
# =====================================================
compress-type=zst
compress-level=3


# =====================================================
# PERFORMANCE
# =====================================================
process-max=4

# Start backup immediately instead of waiting
# for the next normal checkpoint
start-fast=y


# =====================================================
# LOGGING
# =====================================================
log-level-console=info
log-level-file=detail
log-path=/var/log/pgbackrest


# =====================================================
# TIMEOUTS
# =====================================================
db-timeout=1800
protocol-timeout=3600


# =====================================================
# ASYNC WAL ARCHIVING
# =====================================================
archive-async=y
spool-path=/var/spool/pgbackrest


[global:archive-push]
process-max=2

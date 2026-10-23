How to Enable Postgres repository 
=====================================


  #dnf repolist --all
  #dnf config-manager --set-enabled Company_name_PostgreSQL_PGSQL18_RHEL9
  #dnf repolist --all


# DNF Package Management Notes

## 1. Repository Management

### List Enabled Repositories

```bash
dnf repolist
```

### List All Repositories

```bash
dnf repolist --all
```

### Enable a Repository

```bash
dnf config-manager --set-enabled <repository_name>
```

### Disable a Repository

```bash
dnf config-manager --set-disabled <repository_name>
```

---

## 2. RPM / YUM vs DNF Commands

| Purpose | Old RPM/YUM Command | DNF Command |
|---|---|---|
| List installed packages | `rpm -qa` | `dnf list installed` |
| Find installed PostgreSQL packages | `rpm -qa \| grep postgres` | `dnf list installed '*postgres*'` |
| Remove a package | `rpm -e <package>` | `dnf remove <package>` |
| Install a package | `yum install <package>` | `dnf install <package>` |
| Remove a package | `yum remove <package>` | `dnf remove <package>` |
| Search repositories | `yum search postgres` | `dnf search postgres` |
| Show enabled repositories | `yum repolist` | `dnf repolist` |
| Show all repositories | `yum repolist all` | `dnf repolist --all` |
| Get package information | `yum info <package>` | `dnf info <package>` |
| Check for updates | `yum check-update` | `dnf check-update` |
| Update packages | `yum update` | `dnf upgrade` |

---

## 3. Check Whether a Package Is Available

### Search for a Package

```bash
dnf search postgres
```

### List Available PostgreSQL Packages

```bash
dnf list available '*postgres*'
```

### Get Package Information

```bash
dnf info <package_name>
```

Example:

```bash
dnf info postgresql18-server
```

---

## 4. Check Installed Packages

### List All Installed Packages

```bash
dnf list installed
```

### Check PostgreSQL Packages

```bash
dnf list installed '*postgres*'
```

Alternative RPM command:

```bash
rpm -qa | grep -i postgres
```

---

## 5. Install and Remove Packages

### Install

```bash
dnf install <package_name>
```

Example:

```bash
dnf install postgresql18-server
```

### Remove

```bash
dnf remove <package_name>
```

DNF is generally preferred over:

```bash
rpm -e <package_name>
```

because DNF handles package dependencies.

---

## 6. Useful DNF Commands

| Task | Command |
|---|---|
| Show enabled repositories | `dnf repolist` |
| Show all repositories | `dnf repolist --all` |
| Search for package | `dnf search <package>` |
| Package information | `dnf info <package>` |
| List installed packages | `dnf list installed` |
| List available packages | `dnf list available` |
| Install package | `dnf install <package>` |
| Remove package | `dnf remove <package>` |
| Check updates | `dnf check-update` |
| Upgrade packages | `dnf upgrade` |
| Clean cached metadata/packages | `dnf clean all` |
| Show DNF history | `dnf history` |

---

## 7. DBA Quick Reference

```bash
# Check repositories
dnf repolist

# Check all repositories
dnf repolist --all

# Search PostgreSQL packages
dnf search postgres

# Check installed PostgreSQL packages
dnf list installed '*postgres*'

# Check available PostgreSQL packages
dnf list available '*postgres*'

# Get package details
dnf info <package_name>

# Install package
dnf install <package_name>

# Remove package
dnf remove <package_name>

# Check transaction history
dnf history
```

---

## Key Point

> **RPM** is the lower-level package management tool, while **DNF** is a higher-level package manager that resolves dependencies and works with configured repositories.

For normal package installation and removal on modern RHEL/Rocky Linux systems, prefer **DNF**.

  
  
  

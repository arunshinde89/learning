How to Enable Postgres repository 
=====================================


  #dnf repolist --all
  #dnf config-manager --set-enabled Company_name_PostgreSQL_PGSQL18_RHEL9
  #dnf repolist --all


+-------------------------------+--------------------------------------+----------------------------------+
| Old RPM/YUM style             | DNF equivalent                       | Purpose                          |
+-------------------------------+--------------------------------------+----------------------------------+
| rpm -qa                       | dnf list installed                   | List installed packages          |
| rpm -qa | grep postgres       | dnf list installed '*postgres*'      | Find installed PostgreSQL pkgs   |
| rpm -e package                | dnf remove package                   | Remove a package                 |
| yum install package           | dnf install package                  | Install package                  |
| yum remove package            | dnf remove package                   | Remove package                   |
| yum search postgres           | dnf search postgres                  | Search repositories              |
| yum repolist                  | dnf repolist                         | Show enabled repositories        |
+-------------------------------+--------------------------------------+----------------------------------+

  
  
  

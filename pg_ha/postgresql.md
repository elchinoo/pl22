# PostgreSQL 

We are using the packages and instructions from PostgreSQL project here https://www.postgresql.org/download/linux/redhat/

```bash
# We install the epel package
yum install -y epel-release

# Install the repository RPM:
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Install PostgreSQL:
yum install -y postgresql14-server

# We will initialize the database but not enable automatic start:
/usr/pgsql-14/bin/postgresql-14-setup initdb
systemctl disable postgresql-14
systemctl stop postgresql-14


# Firewalld
## We disable it for this tutorial
sudo systemctl stop firewalld
sudo systemctl disable firewalld

## But we should allow the needed ports in production instead:
# for service_port in (5432 6432 2380 2376 8008 7000) 
# do
#     sudo firewall-cmd --permanent --zone=public --add-port=${service_port}/tcp
# done

```
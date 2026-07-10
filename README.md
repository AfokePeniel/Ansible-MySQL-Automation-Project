# Ansible + MySQL Automation Project

Automated provisioning of a production-ready MySQL database server on RHEL 9
using Ansible, following enterprise practices: least-privilege access, security
hardening, and automated backups.

## The Scenario

A tech org's database team manually installs and configures MySQL on every new
server, which is slow, inconsistent, and error prone. This project automates the
full setup so any new database server gets an identical, production-ready MySQL
install in minutes: installed, secured, with an application database, a
least-privilege app user, and scheduled backups.

## Environment

| Node | Role | OS |
|---|---|---|
| controlnode | Ansible control node, runs all playbooks | RHEL 9 (VMware) |
| node1 | Managed host, becomes the MySQL database server | RHEL 9 (VMware) |

## Prerequisites Completed

- Hostname resolution: node1's IP and hostname added to /etc/hosts on the control node
- Dedicated `ansible` service user created on both nodes with passwordless sudo
- Passwordless SSH configured from control node to node1 for the ansible user
- ansible-core installed on the control node

## Project Structure

mysql-automation/    # root project folder
ansible.cfg          # Project config: inventory path, privilege escalation, collections/roles paths
inventory.ini        # Defines the dbservers group containing node1
requirements.yml     # Collection dependencies: community.mysql, ansible.posix
vars.yml             # Variables: passwords, database name, app user
mysql-setup.yml      # Main playbook, full MySQL provisioning end to end
collections/         # Project-scoped collection installs
roles/               # Project-scoped roles 


## Collections Used

- **community.mysql**: provides mysql_user, mysql_db, and related modules for managing MySQL
- **ansible.posix**: provides the firewalld module for firewall configuration

Installed via: `ansible-galaxy collection install -r requirements.yml`

Note: MySQL modules require python3-PyMySQL on the managed node, which the playbook installs.

## What the Playbook Does

| # | Task | Purpose |
|---|---|---|
| 1 | Install MySQL server and dependencies | Installs mysql-server and python3-PyMySQL |
| 2 | Start and enable MySQL service | Service runs now and survives reboots |
| 3 | Open MySQL port in firewalld | Allows remote client connections on 3306 |
| 4 | Set MySQL root password | Secures root via Unix socket auth |
| 5 | Create /root/.my.cnf | Root credentials file (0600) so later tasks authenticate automatically |
| 6 | Remove anonymous MySQL users | Closes default passwordless access hole |
| 7 | Remove MySQL test database | Removes the default open-access database |
| 8 | Create application database | Creates appdb for the application |
| 9 | Create app user with least privilege | appuser gets only SELECT, INSERT, UPDATE, DELETE on appdb.* |
| 10 | Create backup directory | /var/backups/mysql with restricted permissions |
| 11 | Schedule nightly mysqldump backup | Cron job at 2:00 AM, compressed date-stamped dumps |

Tasks 4-7 are the automated equivalent of running `mysql_secure_installation` manually.

## Running the Playbook

```bash
cd ~/mysql-automation
ansible-playbook mysql-setup.yml
```

The playbook is idempotent: running it repeatedly changes nothing once the
desired state is reached.

## Verification Performed

- `systemctl status mysqld` on node1: active (running), MySQL 8.0.46
- `mysql -u appuser -p appdb`: app user authenticates and lands in appdb
- `SHOW GRANTS;`: confirms only USAGE plus the four DML privileges on appdb.*
- `sudo crontab -l`: nightly backup cron present, managed by Ansible marker



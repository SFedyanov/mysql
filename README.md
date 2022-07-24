# Infrastructure
## My test servers:
### Ubuntu 22.04

# Task
```
Написать ansible роль для установки mariadb
роль должна уметь:
- Установка mariadb заданной версии , иметь возможность обновления версии
- Применение начальных параметров безопасности mysql_secure_installation
- Иметь возможность задавать произвольный набор параметров в конфиге для различных нод (кол-во параметров может отличаться)
- Возможность обновлять динамические параметры конфига в runtime
- Настройка master-slave репликации
- Добавление произвольного списка пользователей , с различным уровнем прав доступа
- Добавление произвольного списка cron задач (или events) , чтоб иметь возможность выполнять дополнительное обслуживание
```
# How to use
## inventory/mariadb-servers.yaml
Add correct information about your infrastructure

Example:
```yaml
all:
  hosts:
    mysql:
  children:
    dbservers:
      hosts:
        mysql-1:
          ansible_host: 84.201.166.234
          master: true
        mysql-2:
          ansible_host: 84.252.140.154
          master: false
      vars:
        master_ip: 10.129.0.81
        slave_ip: 10.129.0.82
```
`ansible_host:` - Server external IP

`master_ip`     - Cluster master IP

`slave_ip`      - Cluster slave IP

`master`        - true or false

## ansible.cfg
Change remote_user to your user.
This user has to be able to connect to servers by SSH using SSH key.


# Roles

## linux-config
Prepare linux to install mariadb
### defaults/main.yaml
Variables:

```yaml
skip_task: true             ### skip task for speed up development. Should be 'false' to make it work
work_dir: /opt/mariadb      ### working directory for scripts
mariadb_version: 10.8       ### Mariadb version (10.8 and 10.9 was tested)
reinstall_mariadb: false    ### If you change mariadb_version value set this variable to true for one run. 
mariadb_pkgs:               ### Mariadb packages list 
  - mariadb-server
  - galera-4
  - mariadb-client
  - libmariadb3
  - mariadb-backup
  - mariadb-common
```

## mariadb-config
Configure mariadb service
### defaults/main.yaml 
Variables:
```yaml
skip_task: true                     ### skip task for speed up development. Should be 'false' to make it work
mariadb_root_password : [password]  ### Mariadb root password
mariadb_slave_password: [password]  ### Mariadb slave password
create_claster: true                ### create cluster true or false
```

### mysql_secure_installation
1. sets the root password
2. removes anonymous users
3. removes root remote access
4. removes the test database

### configuration

1. Add variable to:
`templates/90-ansible.cnf.j2`

Example:
```
{% if variable is defined and variable|length %}
max_allowed_packet      = {{ max_allowed_packet }}
{% endif %}
```

2. Add varable to:
`defaults/cnf.yaml`

Example:
```
max_allowed_packet: '32M'
```

3. Check parameter:
```sql
SHOW GLOBAL VARIABLES LIKE 'max_allowed_packet';
```
This variable will be default for all servers. 

4. If require to add specific value to server create server varables file in host_vars folder. As example server name is mysql-1:

`host_vars/mysql-1.yaml`

5. Same for group of servers:

`grop_vars/group_name.yaml`

6. To add variable to all servers exept one just create empty variable for this server:

`max_allowed_packet: ''`

### runtime variables

1. Add varable to `runtime_settings` block in `defaults/runtime_settings.yaml`:

Example:
```
runtime_settings:
  - variable: read_only
    value: 'OFF'
  - variable: session_track_transaction_info
    value: 'OFF'
```
This variable will be default for all servers. 

2. If require to add specific value to server create server varables file in host_vars folder. As example server name is mysql-1:

`host_vars/mysql-1.yaml`

3. Same for group of servers:

`grop_vars/group_name.yaml`

### Master-slave replication

Add to inventory:

`master_ip`     - Cluster master IP

`slave_ip`      - Cluster slave IP

`master`        - true or false

Add to `defaults/main.yaml`
```
create_claster: true   
```

Manual way:
```bash
#### On master

vim /etc/mysql/mariadb.conf.d/50-server.cnf

bind-address           = 0.0.0.0
server-id              = 1
log_bin                = /var/log/mysql/mysql-bin.log

systemctl restart mariadb.service

CREATE USER 'slave'@'172.31.1.82' IDENTIFIED BY 'slave4!';
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'172.31.1.82';
FLUSH PRIVILEGES;
FLUSH TABLES WITH READ LOCK;

SHOW MASTER STATUS;


mysqldump --all-databases --master-data > data.sql

scp  data.sql sfedyanov@172.31.1.82:/tmp/

UNLOCK TABLES;

#### On slave
vim /etc/mysql/mariadb.conf.d/50-server.cnf

bind-address           = 0.0.0.0
server-id              = 2
log_bin                = /var/log/mysql/mysql-bin.log

systemctl restart mariadb.service

mysql < data.sql


CHANGE MASTER TO MASTER_HOST='172.31.1.81', MASTER_USER='slave', MASTER_PASSWORD='slave4!', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=787;

STOP SLAVE;
START SLAVE;

on master:
CREATE DATABASE sampledb;
on slave: 
SHOW DATABASES;
```

### User management:

1. Add to `defaults\users.yaml`

```
---
users:
  - user: 'sfedynov'
    password: 'passwd1234!'
    host: 'localhost'
    state: 'present'
    priv: '*.*:ALL,GRANT'
  - user: 'jack'
    password: 'passwd1234!'
    host: 'localhost'
    state: 'present'
    priv: '*.*:ALL,GRANT'
...
```
2. To remove user just chang `state:` to `'absent'`

3. This users wii be added to all servers. 

4. If require to add user to certain server - create server varables file in host_vars folder. As example server name is mysql-1:

`host_vars/mysql-1.yaml`

5. Same for group of servers:

`grop_vars/group_name.yaml`

### Cron tasks management:

1. Add to `defaults\cron_tasks.yaml`

```
cron_tasks:
  - name: "check dirs"
    minute: "0"
    hour: "5,2"
    day: "*"
    month: "*"
    job: "ls -alh > /dev/null"
    state: "present"
  - name: "check dirs 2"
    minute: "0"
    hour: "5,2"
    day: "*"
    month: "*"
    job: "ls -alh > /dev/null"
    state: "absent"
```
2. To remove task just chang `state:` to `'absent'`

3. This tasks wii be added to all servers. 

4. If require to add task to certain server - create server varables file in host_vars folder. As example server name is mysql-1:

`host_vars/mysql-1.yaml`

5. Same for group of servers:

`grop_vars/group_name.yaml`

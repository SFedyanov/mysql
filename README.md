# Infrastructure
## My test servers:
mysql-1 89.207.12.134 # Ubuntu 22.04

# Task

Написать ansible роль для установки mariadb
роль должна уметь:
- Установка mariadb заданной версии , иметь возможность обновления версии
- Применение начальных параметров безопасности mysql_secure_installation
- Иметь возможность задавать произвольный набор параметров в конфиге для различных нод (кол-во параметров может отличаться)
- Возможность обновлять динамические параметры конфига в runtime
- Настройка master-slave репликации
- Добавление произвольного списка пользователей , с различным уровнем прав доступа
- Добавление произвольного списка cron задач (или events) , чтоб иметь возможность выполнять дополнительное обслуживание

# How to use
## inventory/mariadb-servers.yaml
Add correct information about your infrastructure
## ansible.cfg
Change remote_user to your user.
This user has to be able to connect to servers by SSH using SSH key.


# Roles

## linux-config
Prepare linux to install mariadb
### defaults/main.yaml
Variables:

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


## mariadb-config
Configure mariadb service
### defaults/main.yaml 
Variables:

skip_task: true                     ### skip task for speed up development. Should be 'false' to make it work
mariadb_root_password : [password]  ### Mariadb password

### mysql_secure_installation
1. sets the root password
2. removes anonymous users
3. removes root remote access
4. removes the test database
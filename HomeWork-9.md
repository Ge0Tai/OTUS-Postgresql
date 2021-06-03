

1. Установим 13-ю версию PostgreSQL:
 `sudo apt update`  
 `sudo apt -y upgrade`  
 `sudo apt -y install vim bash-completion wget`  
 `wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -`
 `echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list`  
 `sudo apt update`  
 `sudo apt install postgresql-13 postgresql-client-13`  
 `sudo pg_lsclusters //видим 2 запущенных кластера (12 и 13 версии)`  
 `sudo pg_dropcluster 13 main --stop`  
 `sudo pg_upgradecluster 12 main`  
 `sudo apt-get purge postgresql-12 postgresql-client-12`
 
 ![](pics/dz9/0_upgrade_PSQL_13.PNG)
 
2. Устанавливаем <b>sysbench-tpcc</b>:

 `curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash sudo apt -y install sysbench`
 
3. Подготовим данные для теста:

 `CREATE USER sbtest WITH PASSWORD 'password';`  
 `CREATE DATABASE sbtest;`  
 `GRANT ALL PRIVILEGES ON DATABASE sbtest TO sbtest;`  
 
 ![](pics/dz9/0_create_testDB.PNG)
 
 ![](pics/dz9/0_create_testDB_1.PNG)
 
4. Инициализируем БД <b>sysbench</b> с помощью скрипта:

 `sudo sysbench \`  
 `--db-driver=pgsql \`  
 `--oltp-table-size=100000 \`  
 `--oltp-tables-count=24 \`  
 `--threads=1 \`  
 `--pgsql-host=127.0.0.1 \`  
 `--pgsql-port=5432 \`  
 `--pgsql-user=sbtest \`  
 `--pgsql-password=password \`  
 `--pgsql-db=sbtest \`  
 `/usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua \`  
 `run`
 
 ![](pics/dz9/0_create_testDB_2.PNG)
 
 ![](pics/dz9/0_create_testDB_3.PNG)
 
 

## Домашнее задание № 7 (Репликация PostgreSQL )


Подготовим стенд для выполнения ДЗ со следующими характеристиками:

* otus-node1 (Ubuntu 20.04, PostgreSQL 13)  
* otus-node2 (Ubuntu 20.04, PostgreSQL 13)
* otus-node3 (Ubuntu 20.04, PostgreSQL 13)
* otus-repl (Ubuntu 20.04, PostgreSQL 13)

 
1. Настроим логическую репликацию. Создадим БД и таблицы в <b>nodedb1</b> и <b>nodedb2</b>:

* На <b>otus-node1</b>:

 `alter system set wal_level = logical;  //включаем логическую репликацию`
 
 `sudo pg_ctlcluster 13 main restart  //не забываем рестартовать кластер для применения изменений`
 
 `create database nodedb1;`
 
 `\c nodedb1`  
 
 `create table t_node1(id integer, mesg varchar(50));` 
 
 `create table t_node2(id integer, mesg varchar(50));`
 
  ![](pics/dz10/1_prepare_node1.PNG)
  
* На <b>otus-node2</b>:

 ![](pics/dz10/1_prepare_node2.PNG)
 
2. Сделаем перекрёстную публикацию/подписку на эти таблиц:

* На <b>otus-node1</b>:

Опубликуем таблицу <b>t_node1</b>:

 `CREATE PUBLICATION t_node1_pub FOR TABLE t_node1;`
 
 `\dRp+  //проверим, что наша публикация состоялась`
 
 `\password //назначим пароль для нашей логической репликации` - <b>otus123</b>
 
![](pics/dz10/2_public_t_node1.PNG) 
 
* На <b>otus-node2</b>:

Проделаем тоже самое с таблицей <b>t_node2</b>:

![](pics/dz10/2_public_t_node2.PNG) 
 
3. Таблицы опубликованы. Теперь оформим подписку с кластера <b>otus-node1</b> на таблицу <b>t_node2</b> на кластере <b>otus-node2</b>:

Для начала пропишем правила в <b>firewall</b> для трёх портов (чтобы два раза не бегать):

 `gcloud compute firewall-rules create replica --allow tcp:5432 --source-ranges=0.0.0.0/0 --description="postgresql"`  
 `gcloud compute firewall-rules create replica1 --allow tcp:5433 --source-ranges=0.0.0.0/0 --description="postgresql1"`  
 `gcloud compute firewall-rules create replica2 --allow tcp:5434 --source-ranges=0.0.0.0/0 --description="postgresql2"`

![](pics/dz10/3_open_port.PNG)

Изменим параметр в файле <b>postgres.conf</b>, чтобы кластер слушал все адреса:

`listen_addresses = '*'`

Разрешим удалённое подключение всем пользователям с любых адресов к кластеру по паролю:

`host    all     all    0.0.0.0/0     md5`  
`host    all     all         ::/0     md5`

Подпишемся:

 `CREATE SUBSCRIPTION t_node2_sub CONNECTION 'host=10.128.0.6 port=5432 user=postgres password=otus123 dbname=nodedb2' PUBLICATION t_node2_pub WITH (copy_data = false);`
 
 Проверим статус:

 `SELECT * FROM pg_stat_subscription \gx`
 
 ![](pics/dz10/3_create_sub_1.PNG)
 
 
4. Теперь оформим подписку с кластера <b>otus-node2</b> на таблицу <b>t_node1</b> на кластере <b>otus-node1</b>:

`CREATE SUBSCRIPTION t_node1_sub CONNECTION 'host=10.128.0.5 port=5432 user=postgres password=otus123 dbname=nodedb1' PUBLICATION t_node1_pub WITH (copy_data = false);`  

`SELECT * FROM pg_stat_subscription \gx`

![](pics/dz10/4_create_sub_2.PNG)

На каждом узле вставим по одной строке в таблицы:


| <b>node1</b>                                                     | <b>node2</b>                                                     | 
|+-----------------------------------------------------------------|+-----------------------------------------------------------------|
| `insert into t_node1 values (1, 'Строка в таблице t_node1 - 1');`| `insert into t_node2 values (1, 'Строка в таблице t_node2 - 1');`|

#### Ссылки:  
https://serverfault.com/questions/1042838/how-to-connect-datagrip-to-postgres-on-google-compute-engine  //настраиваем доступ

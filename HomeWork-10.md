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

| <b>node1</b>          | <b>node2</b>  |
| :------------ |:---------------|
| `insert into t_node1 values (1, 'Строка в таблице t_node1 - 1');`         |  `insert into t_node2 values (1, 'Строка в таблице t_node2 - 1');`        |

Сделаем запрос к обоим таблицам и убедимся, что данные, доступные на двух узлах, идентичные:

`select * from t_node1 a, t_node2 b`  
`where`  
`a.id = b.id;`

![](pics/dz10/4_1insert.PNG)

На первом узле (<b>node1</b>) вставим ещё несколько строк только в таблицу <b>t_node1</b> и обратимся к ней на втором узле (<b>node2</b>):

 `insert into t_node1 values (generate_series(2,10),md5(random()::text));`
 
 ![](pics/dz10/4_2insert.PNG)
 
Теперь, на втором узле (<b>node2</b>), вставим несколько строк в таблицу <b>t_node2</b> и обратимся к ней с первого узла (<b>node1</b>):

`insert into t_node2 values (generate_series(2,10),(random()::text));`

![](pics/dz10/4_3insert.PNG)

### Репликационные идентификаторы

>Весь процесс логической репликации в принципе строится на идее репликационных идентификаторов. Поэтому дальнейшая подготовка состоит в проверке наличия во всех реплицируемых таблицах либо первичного ключа, либо индекса, соответствующего некоторым минимальным требованиям и задействованного в REPLICA IDENTITY USING INDEX, либо назначении REPLICA IDENTITY FULL. То есть проверка наличия в таблицах репликационных идентификаторов. Они нужны для однозначной идентификации изменяемых или удаляемых строк при репликации команд UPDATE и DELETE и передаются на реплику в специальном поле для каждой записи.  
Репликационные идентификаторы можно не настраивать, или даже отключить, если планируется реплицировать только команды INSERT. Главное не забыть правильно создать публикацию — исключить из неё команды UPDATE и DELETE. Но если вам на реплике нужны актуальные данные из активно изменяющихся таблиц, а первичные ключи или уникальные NOT NULL индексы в таблицах отсутствуют, то репликационные идентификаторы придётся настраивать с нуля. Не выполнив это условие, можно добиться того, что UPDATE и DELETE будут приводить к отмене транзакций на мастере, малоприятный факт на рабочей базе.  

Чтобы потом не запутаться создадим индексы (пока поле <b>id</b> уникально):

`create unique index on t_node1 (id); //На node1`  
`create unique index on t_node2 (id); //На node2`

![](pics/dz10/4_create_index.PNG)

Как видим, логическая (перекрёстная) репликация между <b>node1</b> и <b>node2</b> работает.  
Теперь переходим к третьему узлу - <b>node3</b>.

5. Прежде чем подписаться, необходимо создать объекты - логическая репликация не работает с <b>DDL</b>. Пробуем подписаться без создания таблиц:

 `CREATE SUBSCRIPTION t_node1_3_sub CONNECTION 'host=10.128.0.5 port=5432 user=postgres password=otus123 dbname=nodedb1' PUBLICATION t_node1_pub WITH (copy_data = false);`
 
Как видим, получаем ошибку, что таблица (<b>relation</b>) не существует:

![](pics/dz10/5_not_relation.PNG)

Создадим таблицы и оформим подписку:

`create table t_node1(id integer, mesg varchar(50));` 
 
`create table t_node2(id integer, mesg varchar(50));`

`CREATE SUBSCRIPTION t_node1_3_sub CONNECTION 'host=10.128.0.5 port=5432 user=postgres password=otus123 dbname=nodedb1' PUBLICATION t_node1_pub WITH (copy_data = false);`

`CREATE SUBSCRIPTION t_node2_3_sub CONNECTION 'host=10.128.0.6 port=5432 user=postgres password=otus123 dbname=nodedb2' PUBLICATION t_node2_pub WITH (copy_data = false);`

![](pics/dz10/5_create_2subscription_n3.PNG)

Сделаем выборку данных из таблиц на третьей ноде (<b>node3</b>):

`select * from t_node1;`  
`select * from t_node2;`

И... ничего не получаем - таблицы пустые!

Почему так произошло? Причина - параметр <b>copy_data = false</b>:

>copy_data (boolean)  
Определяет, должны ли копироваться существующие данные в публикациях, на которые оформляется подписка, сразу после начала репликации. Значение по умолчанию — true.

<i>Т.е. этот параметр сообщает - нужно ли копировать существующие данные (те, которые были в таблицах до создания подписки)</i>

Пересоздадим наши подписки с условием получения всех данных:

`drop subscription t_node1_3_sub;`  
`drop subscription t_node2_3_sub;`   

`CREATE SUBSCRIPTION t_node1_3_sub CONNECTION 'host=10.128.0.5 port=5432 user=postgres password=otus123 dbname=nodedb1' PUBLICATION t_node1_pub WITH (copy_data = true);`  
`CREATE SUBSCRIPTION t_node2_3_sub CONNECTION 'host=10.128.0.6 port=5432 user=postgres password=otus123 dbname=nodedb2' PUBLICATION t_node2_pub WITH (copy_data = true);`  

![](pics/dz10/5_recreate_subs.PNG)

#### Ссылки:  
https://serverfault.com/questions/1042838/how-to-connect-datagrip-to-postgres-on-google-compute-engine  //настраиваем доступ  
https://habr.com/ru/company/postgrespro/blog/489308/  //репликационные идентификаторы  
https://postgrespro.ru/docs/postgresql/12/sql-createsubscription  //create subscription  
https://eax.me/postgresql-replication/ //потоковая репликация  


## Домашнее задание № 8 (Работа с большим объемом реальных данных)

1. Подготовим VM - ssd 200Gb, PostgreSQL 13:

![](pics/dz11/1_createVM.PNG)

2. Экспортируем данные в Google Cloud Storage (разбив на несколько частей):

![](pics/dz11/2_export_to_csv.PNG)

3. С помощью встроенной (есть по умолчанию в GMC) утилиты <b>gsutil</b> загрузим данные на нашу виртуальную машину:

`gsutil -m cp -R gs://otus_dz11_bigdata /home/bucket`

![](pics/dz11/3_upload_bucket_to_VM.PNG)

Используем данные из <i>chicago_crime</i>:

`gsutil -m cp -R gs://chic_crime_bucket /home/bucket`

Скачаем данные в <b>csv</b>, создадим БД <b>chic_crime</b> и подсоеденимся к ней:

![](pics/dz11/3_create_DB_PG.PNG)

(сразу же закачаем данные на вторую машину - <b>otus-bigdata</b>) - <i>не пошло (мало места)</i>

![](pics/dz11/3_download_second_server.PNG)

Создадим таблицу <b>crime</b> в БД <b>chic_crime</b>:

>create table crime (  
	unique_key integer,  
	case_number text,  
	date timestamp,  
	block text,  
	iucr text,  
	primary_type text,  
	description text,  
	location_description text,  
	arrest boolean,  
	domestic boolean,  
	beat integer,  
	district integer,  
	ward integer,  
	community_area integer,  
	fbi_code text,  
	x_coordinate double precision,  
	y_coordinate double precision,  
	year integer,  
	updated_on timestamp,  
	latitude double precision,  
	longitude double precision,  
	location text  
	); 
	
![](pics/dz11/3_create_table_crime_PG.PNG)

Теперь загрузим данные с помощью <b>SQL COPY</b>:

>COPY crime(unique_key,  
	case_number,  
	date,  
	block,  
	iucr,  
	primary_type,  
	description,  
	location_description,  
	arrest,  
	domestic,  
	beat,  
	district,  
	ward,  
	community_area,  
	fbi_code,  
	x_coordinate,  
	y_coordinate,  
	year,  
	updated_on,  
	latitude,  
	longitude,  
	location)  
FROM PROGRAM 'awk FNR-1 /home/bucket/chic_crime_bucket*.csv | cat' DELIMITER ',' CSV HEADER;  

![](pics/dz11/3_download_data_PG.PNG)

### Неудачный пример - мало данных. Возвращаемся на таблицу такси.

`gsutil -m cp -R gs://otus_taxi_bigdata /home/bucket`

Создаём БД <b>chic_taxi</b> и в ней таблицу <b>taxi_trips</b>:

`create database chic_taxi;`  

>create table taxi_trips (  
	unique_key text,   
	taxi_id text,   
	trip_start_timestamp TIMESTAMP,   
	trip_end_timestamp TIMESTAMP,   
	trip_seconds bigint,   
	trip_miles numeric,   
	pickup_census_tract bigint,   
	dropoff_census_tract bigint,   
	pickup_community_area bigint,   
	dropoff_community_area bigint,   
	fare numeric,   
	tips numeric,   
	tolls numeric,   
	extras numeric,   
	trip_total numeric,   
	payment_type text,   
	company text,   
	pickup_latitude numeric,   
	pickup_longitude numeric,  
	pickup_location text,  
	dropoff_latitude numeric,   
	dropoff_longitude numeric,   
	dropoff_location text  
);  

Загружаем полученные <b>*.csv</b> файлы:

>COPY taxi_trips(  
	unique_key,   
	taxi_id,   
	trip_start_timestamp,   
	trip_end_timestamp,   
	trip_seconds,   
	trip_miles,   
	pickup_census_tract,   
	dropoff_census_tract,   
	pickup_community_area,   
	dropoff_community_area,   
	fare,   
	tips,   
	tolls,   
	extras,   
	trip_total,   
	payment_type,   
	company,   
	pickup_latitude,   
	pickup_longitude,   
	pickup_location,   
	dropoff_latitude,   
	dropoff_longitude,   
	dropoff_location)  
FROM PROGRAM 'awk FNR-1 /home/bucket/otus_taxi_bigdata/taxi_trips*.csv | cat' DELIMITER ',' CSV HEADER;  

![](pics/dz11/3_load_taxi_PG.PNG)

4. Подготовим аналогичный по конфигурации сервер и установим на него <b>MySQL</b>:

`sudo apt update`  

`sudo apt install mysql-server mysql-client`

`sudo mysql_secure_installation`

<i>Пароль <b>root</b> - <b>postgres</b></i>

`create database chic_taxi;`

`use chic_taxi`

![](pics/dz11/4_create_mysql.PNG)

Создаём таблицу:

![](pics/dz11/4_create_mysql_tbl.PNG)

Загружаем данные таблицы:

`gsutil -m cp -R gs://otus_taxi_bigdata /home/bucket`

Для загрузки данных в БД создадим скрипт (<b>load.sh</b>):

`#!/usr/bin/env bash`  
`cd /home/bucket/otus_taxi_bigdata/`  
`for f in *.csv`  
`do`  
   `mysql -uroot --local-infile chic_taxi -e "LOAD DATA LOCAL INFILE '"$f"'INTO TABLE taxi_trips FIELDS TERMINATED BY ',' IGNORE 1 ROWS;"`  
`done` 

Настроим <b>MySQL</b>:

Т.к. среда тестовая - не забываем отключить бинлоги для экономии места:

![](pics/dz11/4_set_mysql.png)

`mysql> SET GLOBAL local_infile=1;`

`mysql> SHOW GLOBAL VARIABLES LIKE 'local_infile';  //Проверим`

![](pics/dz11/4_set_mysql1.png)

Запускаем скрипт загрузки:

`# ./load.sh`

После завершения загрузки проверим количество записей:

![](pics/dz11/4_download_mysql.png)

Теперь запустим запрос на БД <b>MySQL</b>:

`mysql> set profiling=1;`

`SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c`  
`FROM taxi_trips`   
`group by payment_type`  
`order by 3;`

![](pics/dz11/4_select_mysql.png)

#### Проблема:

Этот запрос (с <b>group by</b>) выполнить так и не удалось. Возможная [причина](https://bugs.mysql.com/bug.php?id=96497) этой ошибки - <b><i>[ERROR] [MY-012263] [InnoDB] The Auto-extending innodb_system data file './ibdata1' is of a different size 768 pages (rounded down to MB) than specified in the .cnf file: initial 4096 pages, max 4096000 (relevan t if non-zero) pages!</i></b>. И хотя параметр <b>innodb_data_file_path</b> не ограничен (<i>ibdata1:10M:autoextend</i>):

![](pics/dz11/5_param_mysql.PNG)

от ошибки <b>error 1114 (hy000)</b> избавиться не удалось:

![](pics/dz11/6_error_select_mysql.PNG)

Возможное решение проблемы описано [здесь](https://docs.oracle.com/cd/E17952_01/mysql-5.6-en/innodb-system-tablespace.html). Но не было времени проверить. В <b>п.6</b> попробуем запустить для сравнение запрос с <b>count</b>.

5. Подключимся к <b>PostgreSQL</b> и запустим запрос:

![](pics/dz11/5_prepare_postgres.PNG)

Время выполнения <b>17 минут 47 секунд</b>:

![](pics/dz11/5_result_psql.PNG)

Внесём изменения в [конфигурацию](http://pgconfigurator.cybertec.at/) из расчёта того, что имеем практически не обновляемую БД с 90% транзакций, которые просто читают (не изменяют данные):

![](pics/dz11/5_set_psql.conf.PNG)

Применяем параметры, перегружаем сервер и запускаем запрос повторно:

![](pics/dz11/5_result2_psql.PNG)

Время выполнения <b>13 минут 40 секунд</b>

Разница в скорости выполнения запроса около <b>25%</b>.

6. Выполним запрос на обоих серверах:

`select count(unique_key) from taxi_trips;`

На <b>MySQL</b> время выполнения - <b>13 минут 31 секунда</b>:

![](pics/dz11/6_select_mysql_2.PNG)

На <b>PostgreSQL</b> - <b>13 минут 48 секунд</b>:

![](pics/dz11/6_select_psql.PNG)

### Ссылки
http://pgconfigurator.cybertec.at/ - продвинутый конфигуратор  
https://pgtune.leopard.in.ua/#/ - конфигуратор

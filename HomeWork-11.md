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

(сразу же закачаем данные на вторую машину - <b>otus-bigdata</b>)

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



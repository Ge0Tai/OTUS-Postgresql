### Домашнее задание № 7 (Журналы PostgreSQL)

1. Подготовим наш кластер к экспериментам: установим количество буферов 200 (для удобства наблюдения),
   создадим БД (<b>otus_dz7</b>) и установим расширение <b>pg_buffercache</b> (устанавливается только на текущую БД):

	`show shared_buffers;`  
	`1600 kb` // 1600/8 = 200 (200 буферов по 8 kb каждый)    
	`create database otus_dz7;`  
	`\c otus_dz7;`  
	`create extension pg_buffercache;`  
	`\dx+` 

	![](pics/dz7/1_create_ext.PNG)   

	![](pics/dz7/1_set_DB.PNG)

2. Создадим таблицу (<b>t_test</b>) в нашей БД и создадим представление (<b>v_buffercache</b>) для наблюдения за буферным кэшем:  

	`create table t_test(i int);`  

	`create or replace view v_buffercache as`  
	`SELECT bufferid,`  
		`(SELECT c.relname FROM pg_class c WHERE  pg_relation_filenode(c.oid) = b.relfilenode ) relname,`  
		`CASE relforknumber`  
			`WHEN 0 THEN 'main'`  
			`WHEN 1 THEN 'fsm'`  
			`WHEN 2 THEN 'vm'`  
		`END relfork,`  
		`relblocknumber,`  
		`isdirty,`  
		`usagecount`  
	`FROM   pg_buffercache b`  
	`WHERE  b.relDATABASE IN (0, (SELECT oid FROM pg_DATABASE WHERE datname = current_database()) )`  
	`AND    b.usagecount is not null;`  

   Убедимся, что в буферном кэше нет никаких объектов, относящейся к нашей пустой таблице t_test:
  
	`select * from v_buffercache WHERE relname='t_test';`  

	![](pics/dz7/2_cr_view.PNG)

3. Вставим одну строку в нашу таблицу. Что должно произойти? В буферном кэше должен появится "грязный" (требует записи на диск) буфер с счётчиком обращений
   (<b>usage count</b>) равным 1:

	`insert into t_test values(1);`  
	`SELECT * FROM v_buffercache WHERE relname='t_test';`

	![](pics/dz7/3_ins_one_row.PNG)

4. Вставим в таблицу (<b>t_test</b>) ещё 2 строки. Видим, что значения буфера (bufferid 186) меняются: сначала, по прошествии некоторого времени после вставки первой строки, мы
 видим, что изменения были записаны на диск и буфер перестал быть "грязным" (<b>isdirty f</b>). После очередной вставки буфер опять выставляет значение <b>isdirty</b> в  значение <b>t(rue)</b> и увеличивает счётчик обращений (<b>usage count</b>) на 1:

	`insert into t_test values(2);`  
	`insert into t_test values(3);`  

	![](pics/dz7/3_ins_2_rows.PNG)
	
   Ещё стоит обратить внимание, что при обращении к нашей таблице счётчик обращений (<b>usage count</b>) также увеличивается:

	`select * from t_test;`

	![](pics/dz7/3_sel_t_test.PNG)

5. Выполним очистку и посмотрим что изменится в буферном кэше:

	`VACUUM t_test;`

   Очистка создала карту видимости (одна страница) и карту свободного пространства (три страницы — минимальный размер этой карты):

	![](pics/dz7/3_vacuum_t_test.PNG) 
	


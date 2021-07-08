## Домашнее задание № 12 (Сбор и использование статистики)

1. Создадим БД <b>dbt</b> с таблицей <b>orders</b> и заполним её данными:

`insert into orders(id, user_id, order_date, status, some_text)`  
`select generate_series, (random() * 70), date'2019-01-01' + (random() * 300)::int as order_date`  
        `, (array['returned', 'completed', 'placed', 'shipped'])[(random() * 4)::int]`  
        `, concat_ws(' ', (array['go', 'space', 'sun', 'London'])[(random() * 5)::int]`  
            `, (array['the', 'capital', 'of', 'Great', 'Britain'])[(random() * 6)::int]`  
            `, (array['some', 'another', 'example', 'with', 'words'])[(random() * 6)::int]`  
            `)`  
`from generate_series(100001, 1000000);`

Посмотрим план выполнения запроса к этой таблице в отсутствии индекса:

`explain select * from orders where id < 120000;`

Вывод следующий:

> QUERY PLAN                                   
> ---------------------------------------------------------------------------  
> Gather  (cost=1000.00..14882.00 rows=19835 width=34)  
   Workers Planned: 2  
   ->  Parallel Seq Scan on orders  (cost=0.00..11898.50 rows=8265 width=34)    
         Filter: (id < 120000)   
(4 rows)  

Время выполнения запроса:

`otusGEO:5432 postgres@dbt=# select * from orders where id < 120000;`  
`Time: 91.637 ms`

Построим индекс по полю <b>id</b>:

`create index idx_ord_id on orders(id);`

Повторим запрос:

`explain select * from orders where id < 120000;`

> QUERY PLAN                                
> --------------------------------------------------------------------------  
> Index Scan using idx_ord_id on orders  (cost=0.42..729.54 rows=19835 width=34)  
   Index Cond: (id < 120000)  
(2 rows)  

Время выполнения запроса:

`otusGEO:5432 postgres@dbt=# select * from orders where id < 120000;`  
`Time: 21.736 ms`

![](pics/dz12/1_explain_noind_ind.PNG)

2. #### Полнотекстовый поиск (FTS)

>Как гласит документация, для полнотекстового поиска требуется использовать типы tsvector и tsquery. 
>Первый хранит текст документа в оптимизированном для поиска виде, второй — хранит полнотекстовый запрос.
>Для поиска в PostgreSQL есть функции to_tsvector, plainto_tsquery, to_tsquery. Для ранжирования результатов есть ts_rank.

Выполним запрос к нашей таблице - найти значения поля <b>some_text</b>, где встречается слово <b>London</b>:

`SELECT some_text, to_tsvector("some_text") @@ to_tsquery('London') FROM orders;`

![](pics/dz12/2_tsvector_noind_nocolumn.PNG)

`otusGEO:5432 postgres@dbt=# SELECT some_text, to_tsvector("some_text") @@ to_tsquery('London') FROM orders;`  
`Time: 4228.042 ms (00:04.228)`   

`otusGEO:5432 postgres@dbt=# explain select some_text, to_tsvector("some_text") @@ to_tsquery('London') FROM orders;`  
                                   `QUERY PLAN`                                    
`---------------------------------------------------------------------------------`  
 `Gather  (cost=1000.00..290398.50 rows=900000 width=15)`  
   `Workers Planned: 2`  
   `->  Parallel Seq Scan on orders  (cost=0.00..199398.50 rows=375000 width=15)`  
 `JIT:`  
   `Functions: 2`  
   `Options: Inlining false, Optimization false, Expressions true, Deforming true`  
`(6 rows)`  

Время выполнения запроса около 4,2 сек. Видим, что запрос сканирует всю таблицу (т.к. нет индекса) и функция <b>to_tsvector</b> вызывается для каждой строки таблицы.

Построим индекс:

>Для полнотекстового поиска PostgreSQL предлагает два индекса на выбор:  
>
> * GIN — быстро ищет, но не слишком быстро обновляется. Отлично работает, если вы сравнительно редко меняете данные, по которым ищите;
> * GiST — ищет медленнее GIN, зато очень быстро обновляется. Может лучше подходить для поиска по очень часто обновляемым данным;
>Если сомневаетесь, берите GIN. Обычно все используют именно его. GiST же имеет смысл использовать при довольно экзотической нагрузке. Узнать чуть больше о кишочках работы >полнотекстового поиска поверх GIN и GiST можно здесь.

3. Подготовим [тренировочную БД](https://postgrespro.com/docs/postgrespro/13/demodb-bookings-installation):
  - скачиваем  
  `wget https://edu.postgrespro.com/demo-medium-en.zip`
  
  - создаём ДБ <b>demo</b>

  ![](pics/dz12/1_create_DB.PNG)
  
  - разворачиваем  (немного отредактируем файл дампа и запустим на выполнение)   
  
  ![](pics/dz12/1_edit_dump.PNG)
  
  `psql --set ON_ERROR_STOP=on demo < /home/dump/demo-medium-en-20170815.sql`
  
  ![](pics/dz12/1_start_dump.PNG)


#### Ссылки:  
https://postgrespro.com/education/demodb - демо БД  
https://eax.me/postgresql-full-text-search/ - основы FTS
https://habr.com/ru/post/442170/ - хабр FTS


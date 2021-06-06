## Домашнее задание № 6 (MVCC, vacuum и autovacuum)

1. Применим параметры к кластеру, указанные в задании:

 ![](pics/dz6/1_set_config_dwh.png)
 
2. Создаём объекты <b>pgbench</b> для тестирования:
 
 `pgbench -i postgres -U postgres`
 
 ![](pics/dz6/1_create_schema_pgbench.png)
 
3. Прогон теста (без настроек Vacuum):

 `pgbench -c8 -P 60 -T 3600 -U postgres postgres`
 
![](pics/dz6/2_first_test.png) 



### Ссылки:

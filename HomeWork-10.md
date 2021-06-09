## Домашнее задание № 7 (Репликация PostgreSQL )


Подготовим стенд для выполнения ДЗ со следующими характеристиками:

* otus-node1 (Ubuntu 20.04, PostgreSQL 13)  
* otus-node2 (Ubuntu 20.04, PostgreSQL 13)
* otus-node3 (Ubuntu 20.04, PostgreSQL 13)
* otus-repl (Ubuntu 20.04, PostgreSQL 13)

 
1. Создадим БД и таблицы в <b>nodedb1</b> и <b>nodedb2</b>. При этом сделаем перекрёстную публикацию/подписку на эти таблицы (<b>логическую репликацию</b>):

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
 
 
 
 

## Домашнее задание № 12 (Сбор и использование статистики)


1. Подготовим [тренировочную БД](https://postgrespro.com/docs/postgrespro/13/demodb-bookings-installation):
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

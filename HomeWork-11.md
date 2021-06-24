## Домашнее задание № 8 (Работа с большим объемом реальных данных)

1. Подготовим VM - ssd 200Gb, PostgreSQL 13:

![](pics/dz11/1_createVM.PNG)

2. Экспортируем данные в Google Cloud Storage (разбив на несколько частей):

![](pics/dz11/2_export_to_csv.PNG)

3. С помощью встроенной (есть по умолчанию в GMC) утилиты <b>gsutil</b> загрузим данные на нашу виртуальную машину:

`gsutil -m cp -R gs://otus_dz11_bigdata /home/bucket`

![](pics/dz11/3_upload_bucket_to_VM.PNG)


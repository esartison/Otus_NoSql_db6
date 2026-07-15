# Домашнее задание Сартисона Евгения N6 #

Описание/Пошаговая инструкция выполнения домашнего задания:
Необходимо, используя туториал https://clickhouse.tech/docs/en/getting-started/tutorial/

развернуть БД;
выполнить импорт тестовой БД;
выполнить несколько запросов и оценить скорость выполнения.
развернуть дополнительно одну из тестовых БД https://clickhouse.com/docs/en/getting-started/example-datasets , протестировать скорость запросов
развернуть Кликхаус в кластерном исполнении, создать распределенную таблицу, заполнить данными и протестировать скорость по сравнению с 1 инстансом
Дз сдается в виде миниотчета.



## развернуть БД (Некластерный режим) ## 
с помощью след манифест файл поднял в докере образ 
```
student:~/clickhouse/clickhouse-demo-main$ cat docker-compose.yml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:24.8
    container_name: clickhouse-demo
    ports:
      - "8123:8123"   # HTTP-интерфейс (JDBC-драйвер, curl, веб-UI)
      - "9000:9000"   # нативный TCP-протокол (clickhouse-client)
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    environment:
      CLICKHOUSE_DB: user_service
      CLICKHOUSE_USER: demo
      CLICKHOUSE_PASSWORD: demo
      # Разрешаем пользователю demo управлять доступом (для наглядности в демо)
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: 1
    volumes:
      - clickhouse-data:/var/lib/clickhouse
      - ./docker/clickhouse/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8123/ping"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  clickhouse-data:
```

Запустить БД
```
docker compose -f docker-compose.yml up -d
```
<img width="1892" height="230" alt="image" src="https://github.com/user-attachments/assets/ba17c91b-109f-4f68-8421-18b1c9dd4b73" />


выполнить тестовое подключение
```
docker exec -it clickhouse-demo clickhouse-client -u demo --password demo  -q "SELECT version()"
```
<img width="1431" height="48" alt="image" src="https://github.com/user-attachments/assets/b79318bb-8f2c-4676-8e88-80e327ae40b6" />



## выполнить импорт тестовой БД ## 

создал базу esartison и создал таблицу synthetic_test с синтетическими данными
```
docker exec -it clickhouse-demo clickhouse-client -u demo --password demo
CREATE DATABASE IF NOT EXISTS esartison;
USE esartison;

SELECT * FROM generateRandom('trip_id UInt32, price Float32, user String', 1, 10, 2) LIMIT 1000000;

CREATE TABLE synthetic_test
ENGINE = MergeTree
ORDER BY tuple()
AS SELECT *
FROM generateRandom('trip_id UInt32, price Float32, user String', 1, 10, 2)
LIMIT 1000000

Query id: 948ee40b-c278-4722-ba4c-118b1b313587

Ok.

0 rows in set. Elapsed: 0.132 sec. Processed 1.19 million rows, 26.08 MB (8.97 million rows/s., 197.35 MB/s.)
Peak memory usage: 37.51 MiB.
```

## выполнить несколько запросов и оценить скорость выполнения. ## 

выполнил простые запросы, скорость очень хорошая
```
52331f8a59e7 :) select count(0) from synthetic_test;

SELECT count(0)
FROM synthetic_test

Query id: b6a335f3-f4b4-4da4-a373-1777d1420f09

   ┌─count(0)─┐
1. │  1000000 │
   └──────────┘

1 row in set. Elapsed: 0.009 sec. 



52331f8a59e7 :) select distinct user from synthetic_test;

SELECT DISTINCT user
FROM synthetic_test

Query id: 8ced408b-15fa-4080-b1a9-96f36767a1f5

       ┌─user───────┐
    1. │ 1j:a       │
    2. │ xY$Pwpj}z] │

  Showed first 10000.

731560 rows in set. Elapsed: 0.625 sec. Processed 1.00 million rows, 14.00 MB (1.60 million rows/s., 22.41 MB/s.)
Peak memory usage: 264.49 MiB.
```


## развернуть дополнительно одну из тестовых БД https://clickhouse.com/docs/en/getting-started/example-datasets , протестировать скорость запросов ## 

Используем dataset [COVID-19 open data](https://clickhouse.com/docs/getting-started/example-datasets/covid19) для загрузки данных

создаем таблицу и загружаем данные
```
52331f8a59e7 :) CREATE TABLE covid19 (
    date Date,
    location_key LowCardinality(String),
    new_confirmed Int32,
    new_deceased Int32,
    new_recovered Int32,
    new_tested Int32,
    cumulative_confirmed Int32,
    cumulative_deceased Int32,
    cumulative_recovered Int32,
    cumulative_tested Int32
)
ENGINE = MergeTree
ORDER BY (location_key, date);

CREATE TABLE covid19
(
    `date` Date,
    `location_key` LowCardinality(String),
    `new_confirmed` Int32,
    `new_deceased` Int32,
    `new_recovered` Int32,
    `new_tested` Int32,
    `cumulative_confirmed` Int32,
    `cumulative_deceased` Int32,
    `cumulative_recovered` Int32,
    `cumulative_tested` Int32
)
ENGINE = MergeTree
ORDER BY (location_key, date)

Query id: 0fb57ad1-1ad5-46c3-aaf0-f0ce59a5ce4a

Ok.

0 rows in set. Elapsed: 0.019 sec. 

52331f8a59e7 :) INSERT INTO covid19
   SELECT *
   FROM
      url(
        'https://storage.googleapis.com/covid19-open-data/v3/epidemiology.csv',
        CSVWithNames,
        'date Date,
        location_key LowCardinality(String),
        new_confirmed Int32,
        new_deceased Int32,
        new_recovered Int32,
        new_tested Int32,
        cumulative_confirmed Int32,
        cumulative_deceased Int32,
        cumulative_recovered Int32,
        cumulative_tested Int32'
    );

INSERT INTO covid19 SELECT *
FROM url('https://storage.googleapis.com/covid19-open-data/v3/epidemiology.csv', CSVWithNames, 'date Date,\n        location_key LowCardinality(String),\n        new_confirmed Int32,\n        new_deceased Int32,\n        new_recovered Int32,\n        new_tested Int32,\n        cumulative_confirmed Int32,\n        cumulative_deceased Int32,\n        cumulative_recovered Int32,\n        cumulative_tested Int32')

Query id: b90cc4ac-1c03-411c-a27e-0808aaddd2d5

Ok.

0 rows in set. Elapsed: 60.529 sec. Processed 12.53 million rows, 451.11 MB (206.94 thousand rows/s., 7.45 MB/s.)
Peak memory usage: 205.66 MiB.

52331f8a59e7 :) commit;

COMMIT

Query id: 47f595f6-9cd8-430b-9d4d-db361bdfd364


Elapsed: 0.003 sec. 

```


выполним несколько запросов и сохраним время выполнения
```
52331f8a59e7 :) SELECT formatReadableQuantity(count())
FROM covid19;

SELECT formatReadableQuantity(count())
FROM covid19

Query id: 522eae49-914d-4942-ba23-5e7bc212acec

   ┌─formatReadableQuantity(count())─┐
1. │ 12.53 million                   │
   └─────────────────────────────────┘

1 row in set. Elapsed: 0.008 sec. 

52331f8a59e7 :) SELECT formatReadableQuantity(sum(new_confirmed))
FROM covid19;

SELECT formatReadableQuantity(sum(new_confirmed))
FROM covid19

Query id: b3ebf758-1f22-49f7-9fb4-ec172f509e3a

   ┌─formatReadableQuantity(sum(new_confirmed))─┐
1. │ 1.39 billion                               │
   └────────────────────────────────────────────┘

1 row in set. Elapsed: 0.034 sec. Processed 12.53 million rows, 50.10 MB (372.30 million rows/s., 1.49 GB/s.)
Peak memory usage: 405.22 KiB.

52331f8a59e7 :) SELECT
   AVG(new_confirmed) OVER (PARTITION BY location_key ORDER BY date ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING) AS cases_smoothed,
   new_confirmed,
   location_key,
   date
FROM covid19;

SELECT
    AVG(new_confirmed) OVER (PARTITION BY location_key ORDER BY date ASC ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING) AS cases_smoothed,
    new_confirmed,
    location_key,
    date
FROM covid19

Query id: fcef48bd-e01e-4f59-8d8d-091262e64873

       ┌─────cases_smoothed─┬─new_confirmed─┬─location_key─┬───────date─┐
    1. │                  0 │             0 │ AD           │ 2020-01-01 │
    2. │                  0 │             0 │ AD           │ 2020-01-02 │
    3. │                  0 │             0 │ AD           │ 2020-01-03 │
    4. │                  0 │             0 │ AD           │ 2020-01-04 │
....
 9998. │               51.2 │            94 │ AF_KDZ       │ 2021-06-28 │
 9999. │               62.8 │           101 │ AF_KDZ       │ 2021-06-29 │
10000. │               82.2 │            35 │ AF_KDZ       │ 2021-07-01 │
       └─────cases_smoothed─┴─new_confirmed─┴─location_key─┴───────date─┘
  Showed first 10000.

12525825 rows in set. Elapsed: 2.191 sec. Processed 12.53 million rows, 98.70 MB (5.72 million rows/s., 45.05 MB/s.)
Peak memory usage: 93.16 MiB.



52331f8a59e7 :) WITH latest_deaths_data AS
   ( SELECT location_key,
            date,
            new_deceased,
            new_confirmed,
            ROW_NUMBER() OVER (PARTITION BY location_key ORDER BY date DESC) AS rn
     FROM covid19)
SELECT location_key,
       date,
       new_deceased,
       new_confirmed,
       rn
FROM latest_deaths_data
WHERE rn=1;

WITH latest_deaths_data AS
    (
        SELECT
            location_key,
            date,
            new_deceased,
            new_confirmed,
            ROW_NUMBER() OVER (PARTITION BY location_key ORDER BY date DESC) AS rn
        FROM covid19
    )
SELECT
    location_key,
    date,
    new_deceased,
    new_confirmed,
    rn
FROM latest_deaths_data
WHERE rn = 1

Query id: ab0805a7-95bb-4b1c-a19f-2947b0552e59

     ┌─location_key─┬───────date─┬─new_deceased─┬─new_confirmed─┬─rn─┐
  1. │ AD           │ 2022-09-13 │            0 │            34 │  1 │
  2. │ AE           │ 2022-09-13 │            0 │           402 │  1 │


9998. │ GB_ENG_E09000025 │ 2022-09-14 │            0 │             7 │  1 │
 9999. │ GB_ENG_E09000026 │ 2022-09-14 │            0 │            12 │  1 │
10000. │ GB_ENG_E09000027 │ 2022-09-14 │            0 │             6 │  1 │
       └─location_key─────┴───────date─┴─new_deceased─┴─new_confirmed─┴─rn─┘
  Showed first 10000.

20906 rows in set. Elapsed: 4.455 sec. Processed 12.53 million rows, 148.81 MB (2.81 million rows/s., 33.40 MB/s.)
Peak memory usage: 119.81 MiB.
```

## развернуть Кликхаус в кластерном исполнении, создать распределенную таблицу, заполнить данными и протестировать скорость по сравнению с 1 инстансом ## 

Запустить БД
```
docker compose -f docker-compose.cluster.yml up -d
```
<img width="1890" height="390" alt="image" src="https://github.com/user-attachments/assets/bd131208-6903-4678-b22d-72b775a6678d" />

проверить состав кластера
```
student:~/clickhouse/clickhouse-demo-main$ docker exec clickhouse-01 clickhouse-client -q \
  "SELECT shard_num, replica_num, host_name FROM system.clusters WHERE cluster='demo_cluster'"
1	1	clickhouse-01
2	1	clickhouse-02
```

```
student:~/clickhouse/clickhouse-demo-main$ docker exec -i clickhouse-01 clickhouse-client --multiquery < create_table.sql
clickhouse-01	9000	0		1	0
clickhouse-02	9000	0		0	0
```

Создать распределенную таблицу с ключем секционирования location_key и  загрузить данные в таблицу covid19
```
student:~/clickhouse/clickhouse-demo-main$ docker exec -i clickhouse-01 clickhouse-client --multiquery < docker/cluster/create_dist_table.sql
clickhouse-01	9000	0		1	0
clickhouse-02	9000	0		0	0
clickhouse-02	9000	0		1	0
clickhouse-01	9000	0		0	0
clickhouse-01	9000	0		1	0
clickhouse-02	9000	0		0	0

student:~/clickhouse/clickhouse-demo-main$ cat docker/cluster/create_dist_table.sql

CREATE DATABASE IF NOT EXISTS esartison ON CLUSTER demo_cluster;
use esartison;

CREATE TABLE IF NOT EXISTS covid19_local ON CLUSTER demo_cluster
(
    date Date,
    location_key LowCardinality(String),
    new_confirmed Int32,
    new_deceased Int32,
    new_recovered Int32,
    new_tested Int32,
    cumulative_confirmed Int32,
    cumulative_deceased Int32,
    cumulative_recovered Int32,
    cumulative_tested Int32
)
ENGINE = MergeTree()
ORDER BY (location_key, date);


CREATE TABLE covid19 ON CLUSTER demo_cluster
(
    date Date,
    location_key LowCardinality(String),
    new_confirmed Int32,
    new_deceased Int32,
    new_recovered Int32,
    new_tested Int32,
    cumulative_confirmed Int32,
    cumulative_deceased Int32,
    cumulative_recovered Int32,
    cumulative_tested Int32
)
ENGINE = Distributed(
    'demo_cluster',      -- Name of your 2-node cluster
    'esartison',               -- Database name where the local table lives
    'covid19_local',         -- Target physical local table
    cityHash64(location_key) -- Sharding key expression
);

INSERT INTO covid19
   SELECT *
   FROM
      url(
        'https://storage.googleapis.com/covid19-open-data/v3/epidemiology.csv',
        CSVWithNames,
        'date Date,
        location_key LowCardinality(String),
        new_confirmed Int32,
        new_deceased Int32,
        new_recovered Int32,
        new_tested Int32,
        cumulative_confirmed Int32,
        cumulative_deceased Int32,
        cumulative_recovered Int32,
        cumulative_tested Int32'
    );
commit;

```


Выполнили те же запросы, время выполнения выросло, данные тестовые и ключе секционирования не отлаживал, ожидаемо что время возросло
```
student:~/clickhouse/clickhouse-demo-main$ docker exec clickhouse-01 clickhouse-client -q \
  "SELECT _shard_num AS shard, count() FROM esartison.covid19 GROUP BY shard ORDER BY shard"
1	6266643
2	6259182
```

выполним несколько запросов и сохраним время выполнения
```
52331f8a59e7 :) SELECT formatReadableQuantity(count()) FROM esartison.covid19;
####1 row in set. Elapsed: 0.008 sec. 
1 row in set. Elapsed: 0.015 sec. 


52331f8a59e7 :) SELECT formatReadableQuantity(sum(new_confirmed)) FROM esartison.covid19;
###1 row in set. Elapsed: 0.034 sec. Processed 12.53 million rows, 50.10 MB (372.30 million rows/s., 1.49 GB/s.)
1 row in set. Elapsed: 0.099 sec. Processed 12.53 million rows, 50.10 MB (126.00 million rows/s., 504.01 MB/s.)



52331f8a59e7 :) SELECT
   AVG(new_confirmed) OVER (PARTITION BY location_key ORDER BY date ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING) AS cases_smoothed,
   new_confirmed,
   location_key,
   date
FROM esartison.covid19;

###12525825 rows in set. Elapsed: 2.191 sec. Processed 12.53 million rows, 98.70 MB (5.72 million rows/s., 45.05 MB/s.)
12525825 rows in set. Elapsed: 3.927 sec. Processed 12.53 million rows, 98.77 MB (3.19 million rows/s., 25.15 MB/s.)



52331f8a59e7 :) WITH latest_deaths_data AS
   ( SELECT location_key,
            date,
            new_deceased,
            new_confirmed,
            ROW_NUMBER() OVER (PARTITION BY location_key ORDER BY date DESC) AS rn
     FROM esartison.covid19)
SELECT location_key,
       date,
       new_deceased,
       new_confirmed,
       rn
FROM latest_deaths_data
WHERE rn=1;


#### 20906 rows in set. Elapsed: 4.455 sec. Processed 12.53 million rows, 148.81 MB (2.81 million rows/s., 33.40 MB/s.)
20906 rows in set. Elapsed: 5.635 sec. Processed 12.53 million rows, 148.87 MB (2.22 million rows/s., 26.42 MB/s.)
```

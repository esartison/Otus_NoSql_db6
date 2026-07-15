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
<img width="1892" height="230" alt="image" src="https://github.com/user-attachments/assets/3a3d2e9e-f181-481e-b8f9-d2f62ff0d813" />

выполнить тестовое подключение
<img width="1890" height="102" alt="image" src="https://github.com/user-attachments/assets/bb49e5b7-cb97-4aa3-8bfe-e46d44a5d6de" />


## выполнить импорт тестовой БД ## 
Одна из таблиц должна иметь составной Partition key, как минимум одно поле - clustering key, как минимум одно поле не входящее в primiry key.
Заполнить данными обе таблицы.

создать keyspace esartison
```
CREATE KEYSPACE esartison
WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 1
};
```

Создать таблицу transaction_history и заполнить данными
```
Partition Key: (user_id, transaction_month)
Clustering Key: transaction_time

CREATE TABLE esartison.transaction_history (
    user_id uuid,
    transaction_month text,
    transaction_time timestamp,
    amount decimal,
    merchant text,
    PRIMARY KEY ((user_id, transaction_month), transaction_time)
) WITH CLUSTERING ORDER BY (transaction_time DESC);

--- 1-я партиция
INSERT INTO  esartison.transaction_history (user_id, transaction_month, transaction_time, amount, merchant)
VALUES (
    6a2f7df3-ba3a-4be7-9b7e-9730722956cf, 
    '2026-07', 
    '2026-07-01 10:15:30+0000', 
    45.50, 
    'Supermarket Magnit'
);
INSERT INTO  esartison.transaction_history (user_id, transaction_month, transaction_time, amount, merchant)
VALUES (
    6a2f7df3-ba3a-4be7-9b7e-9730722956cf, 
    '2026-07', 
    '2026-07-03 14:22:00+0000', 
    12.99, 
    'Netflix'
);
INSERT INTO  esartison.transaction_history (user_id, transaction_month, transaction_time, amount, merchant)
VALUES (
    6a2f7df3-ba3a-4be7-9b7e-9730722956cf, 
    '2026-07', 
    '2026-07-03 19:45:15+0000', 
    8.20, 
    'Coffee House'
);

--- 2-я партиция
INSERT INTO  esartison.transaction_history (user_id, transaction_month, transaction_time, amount, merchant)
VALUES (
    6a2f7df3-ba3a-4be7-9b7e-9730722956cf, 
    '2026-08', 
    '2026-08-01 09:00:00+0000', 
    120.00, 
    'Gas Station'
);
```

Создать таблицу users_by_country и заполнить данными
```
CREATE TABLE IF NOT EXISTS esartison.users_by_country (
    country text,
    user_id uuid,
    first_name text,
    last_name text,
    email text,
    joined_date date,
    PRIMARY KEY (country, user_id)
);

INSERT INTO esartison.users_by_country (country, user_id, first_name, last_name, email, joined_date) 
VALUES ('US', 123e4567-e89b-12d3-a456-426614174000, 'Alice', 'Smith', 'alice@example.com', '2026-01-15');

INSERT INTO esartison.users_by_country (country, user_id, first_name, last_name, email, joined_date) 
VALUES ('US', 223e4567-e89b-12d3-a456-426614174001, 'Bob', 'Jones', 'bob@example.com', '2026-03-22');

INSERT INTO esartison.users_by_country (country, user_id, first_name, last_name, email, joined_date) 
VALUES ('UK', 323e4567-e89b-12d3-a456-426614174002, 'Charlie', 'Brown', 'charlie@example.com', '2026-05-10');
```

проверить созданные таблицы в keyspace esartison
<img width="529" height="156" alt="image" src="https://github.com/user-attachments/assets/0b0e816d-4a91-4b29-9e88-63c453462688" />


## Выполнить 2-3 варианта запроса использую WHERE ## 
таблица users_by_country
<img width="1125" height="286" alt="image" src="https://github.com/user-attachments/assets/48e989cc-61c7-498d-a916-dbb39ca66549" />

таблица transaction_history
<img width="1178" height="344" alt="image" src="https://github.com/user-attachments/assets/93d5f86c-a93b-4059-99fa-d4b4e337fee3" />


## Создать вторичный индекс на поле, не входящее в primiry key. ## 
создал индекс на таблицу esartison.users_by_country по полю first_name
<img width="989" height="44" alt="image" src="https://github.com/user-attachments/assets/ecd4699c-5cce-455d-8f81-fbdca2c7f643" />

проверить что индекс используется
```
cqlsh:esartison> tracing on
Now Tracing is enabled
cqlsh:esartison> select * from users_by_country where first_name='Bob' ;

 country | user_id                              | email           | first_name | joined_date | last_name
---------+--------------------------------------+-----------------+------------+-------------+-----------
      US | 223e4567-e89b-12d3-a456-426614174001 | bob@example.com |        Bob |  2026-03-22 |     Jones

(1 rows)

Tracing session: f28c3fb0-802c-11f1-80a7-c9ccd4d00bc5

 activity                                                                                                                        | timestamp                  | source     | source_elapsed | client
---------------------------------------------------------------------------------------------------------------------------------+----------------------------+------------+----------------+-----------
                                                                                                              Execute CQL3 query | 2026-07-15 09:09:56.141000 | 172.22.0.2 |              0 | 127.0.0.1
                                   Parsing select * from users_by_country where first_name='Bob' ; [Native-Transport-Requests-1] | 2026-07-15 09:09:56.144000 | 172.22.0.2 |           3958 | 127.0.0.1
                                                                               Preparing statement [Native-Transport-Requests-1] | 2026-07-15 09:09:56.145000 | 172.22.0.2 |           4578 | 127.0.0.1
        Index mean cardinalities are idx_tr_his_first_name:1. Scanning with idx_tr_his_first_name. [Native-Transport-Requests-1] | 2026-07-15 09:09:56.147000 | 172.22.0.2 |           6548 | 127.0.0.1
                                                                         Computing ranges to query [Native-Transport-Requests-1] | 2026-07-15 09:09:56.149000 | 172.22.0.2 |           8185 | 127.0.0.1
 Submitting range requests on 17 ranges with a concurrency of 17 (0.05625 rows per range expected) [Native-Transport-Requests-1] | 2026-07-15 09:09:56.149000 | 172.22.0.2 |           8774 | 127.0.0.1
                                                             Submitted 1 concurrent range requests [Native-Transport-Requests-1] | 2026-07-15 09:09:56.151000 | 172.22.0.2 |          10745 | 127.0.0.1
                                    Executing read on esartison.users_by_country using index idx_tr_his_first_name [ReadStage-3] | 2026-07-15 09:09:56.158000 | 172.22.0.2 |          17585 | 127.0.0.1
                                        Executing single-partition query on users_by_country.idx_tr_his_first_name [ReadStage-3] | 2026-07-15 09:09:56.158000 | 172.22.0.2 |          17882 | 127.0.0.1
                                                                                      Acquiring sstable references [ReadStage-3] | 2026-07-15 09:09:56.159000 | 172.22.0.2 |          18678 | 127.0.0.1
                                         Skipped 0/1 non-slice-intersecting sstables, included 0 due to tombstones [ReadStage-3] | 2026-07-15 09:09:56.159000 | 172.22.0.2 |          18841 | 127.0.0.1
                                                                Partition index with 0 entries found for sstable 1 [ReadStage-3] | 2026-07-15 09:09:56.160000 | 172.22.0.2 |          19343 | 127.0.0.1
                                                              Executing single-partition query on users_by_country [ReadStage-3] | 2026-07-15 09:09:56.166000 | 172.22.0.2 |          25997 | 127.0.0.1
                                                                                      Acquiring sstable references [ReadStage-3] | 2026-07-15 09:09:56.167000 | 172.22.0.2 |          26373 | 127.0.0.1
                                                                                         Merging memtable contents [ReadStage-3] | 2026-07-15 09:09:56.167000 | 172.22.0.2 |          26523 | 127.0.0.1
                                                                                       Key cache hit for sstable 1 [ReadStage-3] | 2026-07-15 09:09:56.167000 | 172.22.0.2 |          26834 | 127.0.0.1
                                                                            Read 1 live rows and 0 tombstone cells [ReadStage-3] | 2026-07-15 09:09:56.171000 | 172.22.0.2 |          30320 | 127.0.0.1
                                                                         Merged data from memtables and 1 sstables [ReadStage-3] | 2026-07-15 09:09:56.171000 | 172.22.0.2 |          30568 | 127.0.0.1
                                                                                                                Request complete | 2026-07-15 09:09:56.173179 | 172.22.0.2 |          32179 | 127.0.0.1
```
Видим что индекс используется...


## (*) нагрузить кластер при помощи Cassandra Stress Tool (используя "How to use Apache Cassandra Stress Tool.pdf" из материалов). ## 
пропустил шаг, чтобы успеть сделать другие ДЗ.


  

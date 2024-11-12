1. Сгенерировать таблицу с 1 млн JSONB документов

```sql
drop table tests.t;


CREATE TABLE tests.t
(
    id INT PRIMARY KEY,
    js JSONB
);
    
INSERT INTO tests.t(id, js) ---в чанки
SELECT i AS id, 
       (SELECT jsonb_object_agg(j, j) FROM generate_series(1, 1000) j) js
FROM generate_series(1, 1000) i; ---1,000 rows affected in 1 s 149 ms

INSERT INTO tests.t(id, js) --- поместится в таблицу
SELECT i AS id,
       (SELECT jsonb_object_agg(j, j) FROM generate_series(1, 10) j) js
FROM generate_series(1001, 1000000) i; ---999,000 rows affected in 2 s 357 ms


SELECT COUNT(1) FROM tests.t; --- 1000000

```

```sql
SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class 
WHERE relname = 't' 
  AND relnamespace = 'tests'::regnamespace;
```

| heap\_rel | heap\_rel\_size | toast\_rel                  | toast\_rel\_size |
|:----------|:----------------|:----------------------------|:-----------------|
| tests.t   | 205 MB          | pg\_toast.pg\_toast\_554441 | 8000 kB          |



2. Создать индекс

```sql
CREATE INDEX ix_test_t ON tests.t USING gin (js); 
```

```sql
SELECT
    indexname AS index_name,
    indexdef AS index_definition
FROM
    pg_indexes
WHERE
    tablename = 't'
  AND schemaname = 'tests';
```

| index\_name | index\_definition                                         |
|:------------|:----------------------------------------------------------|
| t\_pkey     | CREATE UNIQUE INDEX t\_pkey ON tests.t USING btree \(id\) |
| ix\_test\_t | CREATE INDEX ix\_test\_t ON tests.t USING gin \(js\)      |


3. Обновить 1 из полей в json

```sql
SELECT pg_current_wal_lsn(); ---C/DF21B950
UPDATE tests.t SET js = js::jsonb || '{"a":1}';  --- 1,000,000 rows affected in 21 s 723 ms
SELECT pg_current_wal_lsn(); -->  D/29595100
SELECT pg_size_pretty(pg_wal_lsn_diff('D/29595100','C/DF21B950')) AS wal_size; --- 1187 MB
```

4. Убедиться в блоатинге TOAST

```sql
SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class 
WHERE relname = 't' 
  AND relnamespace = 'tests'::regnamespace;
```

| heap\_rel | heap\_rel\_size | toast\_rel                  | toast\_rel\_size |
|:----------|:----------------|:----------------------------|:-----------------|
| tests.t   | 428 MB          | pg\_toast.pg\_toast\_554441 | 16 MB            |




```sql
SELECT chunk_id, chunk_seq, length(chunk_data) 
FROM pg_toast.pg_toast_554441;
```

Часть таблицы:

| chunk\_id | chunk\_seq | length |
|:----------|:-----------|:-------|
| 555449    | 0          | 1996   |
| 555449    | 1          | 1996   |
| 555449    | 2          | 1996   |
| 555449    | 3          | 55     |



5. Придумать методы избавится от него и проверить на практике

1) Чистим все мертвые строки

```sql
VACUUM FULL tests.t; ----completed in 17 s 753 ms
VACUUM FULL pg_toast.pg_toast_554441; --- completed in 109 ms
```

```sql
SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class 
WHERE relname = 't' 
  AND relnamespace = 'tests'::regnamespace;
```

Распухание не исчезло:

| heap\_rel | heap\_rel\_size | toast\_rel                  | toast\_rel\_size |
|:----------|:----------------|:----------------------------|:-----------------|
| tests.t   | 422 MB          | pg\_toast.pg\_toast\_554441 | 16 MB            |


```sql
SELECT
    relname AS table_name,
    n_dead_tup AS dead_row_count,
    last_autovacuum,
    last_autoanalyze,
    NOW() now_time
FROM
    pg_stat_user_tables
WHERE
    relname = 't'
  AND schemaname = 'tests';
```

| table\_name | dead\_row\_count | last\_autovacuum                  | last\_autoanalyze                 | now\_time                         |
|:------------|:-----------------|:----------------------------------|:----------------------------------|:----------------------------------|
| t           | 1000000          | 2024-11-12 18:10:16.300131 +00:00 | 2024-11-12 17:50:21.195677 +00:00 | 2024-11-12 18:10:50.264391 +00:00 |

в логах увидела:

```sql
2024-11-12 18:27:23.593 UTC [6235] LOG:  automatic vacuum of table "wbpgdb.tests.t": index scans: 0
2024-11-12 23:27:23     pages: 0 removed, 54013 remain, 54013 scanned (100.00% of total)
2024-11-12 23:27:23     tuples: 0 removed, 2000000 remain, 1000000 are dead but not yet removable
2024-11-12 23:27:23     removable cutoff: 910, which was 152 XIDs old when operation ended
2024-11-12 23:27:23     frozen: 0 pages from table (0.00% of total) had 0 tuples frozen
2024-11-12 23:27:23     index scan not needed: 0 pages from table (0.00% of total) had 0 dead item identifiers removed
2024-11-12 23:27:23     index "ix_test_t": pages: 6149 in total, 0 newly deleted, 0 currently deleted, 0 reusable
2024-11-12 23:27:23     avg read rate: 200.447 MB/s, avg write rate: 0.000 MB/s
2024-11-12 23:27:23     buffer usage: 60771 hits, 53469 misses, 0 dirtied
2024-11-12 23:27:23     WAL usage: 1 records, 0 full page images, 134 bytes
2024-11-12 23:27:23     system usage: CPU: user: 0.27 s, system: 0.11 s, elapsed: 2.08 s
```

- активных транзакций нет

```sql
SELECT pid, backend_start, xact_start, query
FROM pg_stat_activity
WHERE state = 'active' AND xact_start IS NOT NULL;
```

| pid   | backend\_start                    | xact\_start                       | query                                                                                                                             |
|:------|:----------------------------------|:----------------------------------|:----------------------------------------------------------------------------------------------------------------------------------|
| 5823  | 2024-11-12 16:23:10.038162 +00:00 | 2024-11-12 18:31:37.318356 +00:00 | SELECT pid, backend\_start, xact\_start, query<br/>FROM pg\_stat\_activity<br/>WHERE state = 'active' AND xact\_start IS NOT NULL |

- в блокировках пусто
```sql
SELECT * FROM pg_locks WHERE relation = (SELECT oid FROM pg_class WHERE relname = 't'); --- пусто
```

> Не понимаю почему мертвые строки не удаляются

Решила вообще убить все транзакции
```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid <> pg_backend_pid();
```

```sql
2024-11-12 23:37:06 2024-11-12 18:37:06.894 UTC [5823] WARNING:  PID 27 is not a PostgreSQL backend process
2024-11-12 23:37:06 2024-11-12 18:37:06.894 UTC [5823] WARNING:  PID 28 is not a PostgreSQL backend process
2024-11-12 23:37:06 2024-11-12 18:37:06.894 UTC [5823] WARNING:  PID 30 is not a PostgreSQL backend process
2024-11-12 23:37:06 2024-11-12 18:37:06.895 UTC [4218] FATAL:  terminating connection due to administrator command
2024-11-12 23:37:06 2024-11-12 18:37:06.896 UTC [4236] FATAL:  terminating connection due to administrator command
2024-11-12 23:37:06 2024-11-12 18:37:06.896 UTC [4207] FATAL:  terminating connection due to administrator command
2024-11-12 23:37:06 2024-11-12 18:37:06.897 UTC [4158] FATAL:  terminating connection due to administrator command
2024-11-12 23:37:06 2024-11-12 18:37:06.897 UTC [4145] FATAL:  terminating connection due to administrator command
2024-11-12 23:37:06 2024-11-12 18:37:06.898 UTC [4144] FATAL:  terminating connection due to administrator command
2024-11-12 23:37:06 2024-11-12 18:37:06.897 UTC [4196] FATAL:  terminating connection due to administrator command
2024-11-12 23:37:06 2024-11-12 18:37:06.898 UTC [5843] FATAL:  terminating connection due to administrator command
2024-11-12 23:37:06 2024-11-12 18:37:06.905 UTC [1] LOG:  background worker "logical replication launcher" (PID 32) exited with exit code 1


LOG:  automatic vacuum of table "wbpgdb.pg_catalog.pg_statistic": index scans: 1
2024-11-12 23:37:48     pages: 0 removed, 33 remain, 33 scanned (100.00% of total)
2024-11-12 23:37:48     tuples: 306 removed, 418 remain, 0 are dead but not yet removable
2024-11-12 23:37:48     removable cutoff: 1064, which was 0 XIDs old when operation ended
2024-11-12 23:37:48     new relfrozenxid: 1051, which is 150 XIDs ahead of previous value
2024-11-12 23:37:48     frozen: 7 pages from table (21.21% of total) had 54 tuples frozen
2024-11-12 23:37:48     index scan needed: 21 pages from table (63.64% of total) had 294 dead item identifiers removed
2024-11-12 23:37:48     index "pg_statistic_relid_att_inh_index": pages: 5 in total, 0 newly deleted, 0 currently deleted, 0 reusable
2024-11-12 23:37:48     avg read rate: 14.426 MB/s, avg write rate: 30.655 MB/s
2024-11-12 23:37:48     buffer usage: 102 hits, 16 misses, 34 dirtied
2024-11-12 23:37:48     WAL usage: 73 records, 27 full page images, 102883 bytes
2024-11-12 23:37:48     system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
```

>Так и не поняла какая транзакция держала записи, если в таблице не было активных транзакций....

Зато, наконец, вакуум справился

```sql
SELECT
    relname AS table_name,
    n_dead_tup AS dead_row_count,
    last_autovacuum,
    last_autoanalyze,
    NOW() now_time
FROM
    pg_stat_user_tables
WHERE
    relname = 't'
  AND schemaname = 'tests';
```

| table\_name | dead\_row\_count | last\_autovacuum | last\_autoanalyze | now\_time |
| :--- | :--- | :--- | :--- | :--- |
| t | 0 | 2024-11-12 18:38:24.449050 +00:00 | 2024-11-12 17:50:21.195677 +00:00 | 2024-11-12 18:39:31.098103 +00:00 |

6. Не забываем про блоатинг индексов*

> Но распухание так и не исчезло, пока не почистила индексы

```sql
REINDEX TABLE tests.t; --- completed in 15 s 310 ms
REINDEX TABLE pg_toast.pg_toast_554441; ---  completed in 118 ms
```
Победа:

| heap\_rel | heap\_rel\_size | toast\_rel                  | toast\_rel\_size |
|:----------|:----------------|:----------------------------|:-----------------|
| tests.t   | 223 MB          | pg\_toast.pg\_toast\_554441 | 8000 kB          |



2) Изменить стратегию TOAST с EXTENDED на MAIN для уменьшения сжатия и создания меньшего количества чанков:

Снова обновим:
```sql
ALTER TABLE tests.t ALTER COLUMN js SET STORAGE MAIN;
```
```sql
SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class 
WHERE relname = 't' 
  AND relnamespace = 'tests'::regnamespace;
```

| heap\_rel | heap\_rel\_size | toast\_rel                  | toast\_rel\_size |
|:----------|:----------------|:----------------------------|:-----------------|
| tests.t   | 223 MB          | pg\_toast.pg\_toast\_554441 | 8000 kB          |

```sql
UPDATE tests.t SET js = js::jsonb || '{"a":1}';   --- 1,000,000 rows affected in 22 s 350 ms
```

Как мы видим результат хуже чем 422 + 16 мб, как было с наличием TOAST

| heap\_rel | heap\_rel\_size | toast\_rel                  | toast\_rel\_size |
|:----------|:----------------|:----------------------------|:-----------------|
| tests.t   | 452 MB          | pg\_toast.pg\_toast\_554441 | 8000 kB          |


Перестраиваем таблицу:

```sql
VACUUM FULL tests.t;
```

| heap\_rel | heap\_rel\_size | toast\_rel                  | toast\_rel\_size |
|:----------|:----------------|:----------------------------|:-----------------|
| tests.t   | 229 MB          | pg\_toast.pg\_toast\_554441 | 0 bytes          |


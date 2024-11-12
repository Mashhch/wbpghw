1. Развернуть ВМ (Linux) с PostgreSQL
2. Залить Тайские перевозки
   https://github.com/aeuge/postgres16book/tree/main/database

- первые пункты можно посмотреть в hw1

3. Проверить скорость выполнения сложного запроса (приложен в конце файла скриптов)

```sql
EXPLAIN ANALYSE 
WITH all_place AS (SELECT count(s.id) as all_place, 
                          s.fkbus as fkbus
                   FROM book.seat s
                   group by s.fkbus),
     order_place AS (SELECT count(t.id) as order_place, t.fkride
                     FROM book.tickets t
                     group by t.fkride)
SELECT r.id,
       r.startdate                as depart_date,
       bs.city || ', ' || bs.name as busstation,
       t.order_place,
       st.all_place
FROM book.ride r
         JOIN book.schedule as s
              on r.fkschedule = s.id
         JOIN book.busroute br
              on s.fkroute = br.id
         JOIN book.busstation bs
              on br.fkbusstationfrom = bs.id
         JOIN order_place t
              on t.fkride = r.id
         JOIN all_place st
              on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place, st.all_place
ORDER BY r.startdate
limit 10;
```


```sql
| QUERY PLAN |
| :--- |
| Limit  \(cost=330525.96..330525.98 rows=10 width=56\) \(actual time=844.924..845.004 rows=10 loops=1\) |
|   -&gt;  Sort  \(cost=330525.96..330891.84 rows=146353 width=56\) \(actual time=829.700..829.779 rows=10 loops=1\) |
|         Sort Key: r.startdate |
|         Sort Method: top-N heapsort  Memory: 25kB |
|         -&gt;  Group  \(cost=324802.15..327363.32 rows=146353 width=56\) \(actual time=788.853..814.673 rows=144000 loops=1\) |
|               Group Key: r.id, \(\(\(bs.city \|\| ', '::text\) \|\| bs.name\)\), \(count\(t.id\)\), \(count\(s\_1.id\)\) |
|               -&gt;  Sort  \(cost=324802.15..325168.03 rows=146353 width=56\) \(actual time=788.824..797.865 rows=144000 loops=1\) |
|                     Sort Key: r.id, \(\(\(bs.city \|\| ', '::text\) \|\| bs.name\)\), \(count\(t.id\)\), \(count\(s\_1.id\)\) |
|                     Sort Method: external merge  Disk: 7864kB |
|                     -&gt;  Hash Join  \(cost=257696.87..307240.72 rows=146353 width=56\) \(actual time=546.776..764.635 rows=144000 loops=1\) |
|                           Hash Cond: \(r.fkbus = s\_1.fkbus\) |
|                           -&gt;  Nested Loop  \(cost=257691.75..305794.03 rows=146353 width=84\) \(actual time=546.698..741.846 rows=144000 loops=1\) |
|                                 -&gt;  Hash Join  \(cost=257691.61..302311.12 rows=146353 width=24\) \(actual time=546.658..706.699 rows=144000 loops=1\) |
|                                       Hash Cond: \(s.fkroute = br.id\) |
|                                       -&gt;  Hash Join  \(cost=257689.26..301897.46 rows=146353 width=24\) \(actual time=546.636..689.117 rows=144000 loops=1\) |
|                                             Hash Cond: \(r.fkschedule = s.id\) |
|                                             -&gt;  Merge Join  \(cost=257645.86..301468.75 rows=146353 width=24\) \(actual time=546.432..672.971 rows=144000 loops=1\) |
|                                                   Merge Cond: \(r.id = t.fkride\) |
|                                                   -&gt;  Index Scan using ride\_pkey on ride r  \(cost=0.42..4555.42 rows=144000 width=16\) \(actual time=0.014..12.552 rows=144000 loops=1\) |
|                                                   -&gt;  Finalize GroupAggregate  \(cost=257645.44..294723.92 rows=146353 width=12\) \(actual time=546.410..634.457 rows=144000 loops=1\) |
|                                                         Group Key: t.fkride |
|                                                         -&gt;  Gather Merge  \(cost=257645.44..291796.86 rows=292706 width=12\) \(actual time=546.361..603.302 rows=432000 loops=1\) |
|                                                               Workers Planned: 2 |
|                                                               Workers Launched: 2 |
|                                                               -&gt;  Sort  \(cost=256645.41..257011.30 rows=146353 width=12\) \(actual time=528.106..540.103 rows=144000 loops=3\) |
|                                                                     Sort Key: t.fkride |
|                                                                     Sort Method: external merge  Disk: 3672kB |
|                                                                     Worker 0:  Sort Method: external merge  Disk: 3672kB |
|                                                                     Worker 1:  Sort Method: external merge  Disk: 3672kB |
|                                                                     -&gt;  Partial HashAggregate  \(cost=219020.10..241586.49 rows=146353 width=12\) \(actual time=371.881..481.386 rows=144000 loops=3\) |
|                                                                           Group Key: t.fkride |
|                                                                           Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27624kB |
|                                                                           Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27624kB |
|                                                                           Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27904kB |
|                                                                           Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27560kB |
|                                                                           -&gt;  Parallel Seq Scan on tickets t  \(cost=0.00..80585.33 rows=2160933 width=12\) \(actual time=0.053..87.518 rows=1728502 loops=3\) |
|                                             -&gt;  Hash  \(cost=25.40..25.40 rows=1440 width=8\) \(actual time=0.194..0.195 rows=1440 loops=1\) |
|                                                   Buckets: 2048  Batches: 1  Memory Usage: 73kB |
|                                                   -&gt;  Seq Scan on schedule s  \(cost=0.00..25.40 rows=1440 width=8\) \(actual time=0.004..0.087 rows=1440 loops=1\) |
|                                       -&gt;  Hash  \(cost=1.60..1.60 rows=60 width=8\) \(actual time=0.014..0.015 rows=60 loops=1\) |
|                                             Buckets: 1024  Batches: 1  Memory Usage: 11kB |
|                                             -&gt;  Seq Scan on busroute br  \(cost=0.00..1.60 rows=60 width=8\) \(actual time=0.005..0.008 rows=60 loops=1\) |
|                                 -&gt;  Memoize  \(cost=0.15..0.36 rows=1 width=68\) \(actual time=0.000..0.000 rows=1 loops=144000\) |
|                                       Cache Key: br.fkbusstationfrom |
|                                       Cache Mode: logical |
|                                       Hits: 143990  Misses: 10  Evictions: 0  Overflows: 0  Memory Usage: 2kB |
|                                       -&gt;  Index Scan using busstation\_pkey on busstation bs  \(cost=0.14..0.35 rows=1 width=68\) \(actual time=0.003..0.003 rows=1 loops=10\) |
|                                             Index Cond: \(id = br.fkbusstationfrom\) |
|                           -&gt;  Hash  \(cost=5.05..5.05 rows=5 width=12\) \(actual time=0.065..0.066 rows=5 loops=1\) |
|                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB |
|                                 -&gt;  HashAggregate  \(cost=5.00..5.05 rows=5 width=12\) \(actual time=0.061..0.063 rows=5 loops=1\) |
|                                       Group Key: s\_1.fkbus |
|                                       Batches: 1  Memory Usage: 24kB |
|                                       -&gt;  Seq Scan on seat s\_1  \(cost=0.00..4.00 rows=200 width=8\) \(actual time=0.022..0.029 rows=200 loops=1\) |
| Planning Time: 1.158 ms |
| JIT: |
|   Functions: 84 |
|   Options: Inlining false, Optimization false, Expressions true, Deforming true |
|   Timing: Generation 3.169 ms \(Deform 1.129 ms\), Inlining 0.000 ms, Optimization 1.806 ms, Emission 29.414 ms, Total 34.389 ms |
| Execution Time: 850.182 ms |
```

4. Навесить индексы на внешние ключи

- навесим индексы на внешние ключи + для полей с группировкой сделаем составные, чтобы не ходить лишний раз в таблицу при группировке:

```sql
CREATE INDEX ix_ride_fkschedule ON book.ride(fkschedule);
CREATE INDEX ix_schedule_fkroute ON book.schedule(fkroute);
CREATE INDEX ix_busroute_fkbusstationfrom ON book.busroute(fkbusstationfrom);
CREATE INDEX ix_ride_fkschedule_id ON book.ride(fkschedule, id);
CREATE INDEX ix_tickets_fkride_id ON book.tickets(fkride, id);
CREATE INDEX ix_seat_fkbus_fkride ON book.seat(fkbus, id);
```

```sql
SELECT
    n.nspname AS "Schema",
    t.relname AS "Table",
    c.relname AS "Index"
FROM
    pg_catalog.pg_class c
        JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
        JOIN pg_catalog.pg_index i ON i.indexrelid = c.oid
        JOIN pg_catalog.pg_class t ON i.indrelid = t.oid
WHERE
    c.relkind = 'i'
  AND n.nspname NOT IN ('pg_catalog', 'pg_toast')  -- исключаем системные схемы
ORDER BY
    n.nspname,
    t.relname,
    c.relname;
```

| Schema | Table        | Index                          |
|:-------|:-------------|:-------------------------------|
| book   | bus          | bus\_pkey                      |
| book   | busroute     | busroute\_pkey                 |
| book   | busroute     | ix\_busroute\_fkbusstationfrom |
| book   | busstation   | busstation\_pkey               |
| book   | ride         | ix\_ride\_fkschedule           |
| book   | ride         | ix\_ride\_fkschedule\_id       |
| book   | ride         | ride\_pkey                     |
| book   | schedule     | ix\_schedule\_fkroute          |
| book   | schedule     | schedule\_pkey                 |
| book   | seat         | ix\_busroute\_seat\_fkbus      |
| book   | seat         | ix\_seat\_fkbus\_fkride        |
| book   | seat         | seat\_pkey                     |
| book   | seatcategory | seatcategory\_pkey             |
| book   | tickets      | ix\_busroute\_tickets\_fkride  |
| book   | tickets      | ix\_tickets\_fkride\_id        |
| book   | tickets      | tickets\_pkey                  |



5. Проверить, помогли ли индексы на внешние ключи ускориться

Удалим историю, перезагрузив кэш планов

```sql
show config_file; --- /var/lib/postgresql/data/postgresql.conf
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Запишем экстеншн в конфиги
```sql
root@f6881221607e:/var/lib/postgresql/data/base/16384# echo "shared_preload_libraries = 'pg_stat_statements' " >> /var/lib/postgresql/data/postgresql.conf
root@f6881221607e:/var/lib/postgresql/data/base/16384# docker restart wbpg
```
```sql
SELECT pg_stat_statements_reset(); ---2024-11-12 19:49:55.275656 +00:00
```

```sql
EXPLAIN ANALYSE 
WITH all_place AS (SELECT count(s.id) as all_place, 
                          s.fkbus as fkbus
                   FROM book.seat s
                   group by s.fkbus),
     order_place AS (SELECT count(t.id) as order_place, 
                            t.fkride
                     FROM book.tickets t
                     group by t.fkride)
SELECT r.id,
       r.startdate                as depart_date,
       bs.city || ', ' || bs.name as busstation,
       t.order_place,
       st.all_place
FROM book.ride r
         JOIN book.schedule as s
              on r.fkschedule = s.id
         JOIN book.busroute br
              on s.fkroute = br.id
         JOIN book.busstation bs
              on br.fkbusstationfrom = bs.id
         JOIN order_place t
              on t.fkride = r.id
         JOIN all_place st
              on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place, st.all_place
ORDER BY r.startdate
limit 10;
```

```sql
| QUERY PLAN |
| :--- |
| Limit  \(cost=210452.86..210452.89 rows=10 width=56\) \(actual time=463.790..463.857 rows=10 loops=1\) |
|   -&gt;  Sort  \(cost=210452.86..210818.74 rows=146353 width=56\) \(actual time=447.774..447.840 rows=10 loops=1\) |
|         Sort Key: r.startdate |
|         Sort Method: top-N heapsort  Memory: 25kB |
|         -&gt;  Group  \(cost=204729.05..207290.22 rows=146353 width=56\) \(actual time=409.552..433.778 rows=144000 loops=1\) |
|               Group Key: r.id, \(\(\(bs.city \|\| ', '::text\) \|\| bs.name\)\), \(count\(t.id\)\), \(count\(s\_1.id\)\) |
|               -&gt;  Sort  \(cost=204729.05..205094.93 rows=146353 width=56\) \(actual time=409.525..417.991 rows=144000 loops=1\) |
|                     Sort Key: r.id, \(\(\(bs.city \|\| ', '::text\) \|\| bs.name\)\), \(count\(t.id\)\), \(count\(s\_1.id\)\) |
|                     Sort Method: external merge  Disk: 7864kB |
|                     -&gt;  Hash Join  \(cost=1052.96..187167.62 rows=146353 width=56\) \(actual time=32.661..378.690 rows=144000 loops=1\) |
|                           Hash Cond: \(r.fkbus = s\_1.fkbus\) |
|                           -&gt;  Hash Join  \(cost=1047.85..185720.93 rows=146353 width=84\) \(actual time=32.532..348.710 rows=144000 loops=1\) |
|                                 Hash Cond: \(br.fkbusstationfrom = bs.id\) |
|                                 -&gt;  Hash Join  \(cost=1046.63..185172.71 rows=146353 width=24\) \(actual time=32.499..328.891 rows=144000 loops=1\) |
|                                       Hash Cond: \(s.fkroute = br.id\) |
|                                       -&gt;  Hash Join  \(cost=1044.28..184759.05 rows=146353 width=24\) \(actual time=32.454..309.187 rows=144000 loops=1\) |
|                                             Hash Cond: \(r.fkschedule = s.id\) |
|                                             -&gt;  Merge Join  \(cost=1000.88..184330.34 rows=146353 width=24\) \(actual time=31.979..287.311 rows=144000 loops=1\) |
|                                                   Merge Cond: \(r.id = t.fkride\) |
|                                                   -&gt;  Index Scan using ride\_pkey on ride r  \(cost=0.42..4555.42 rows=144000 width=16\) \(actual time=0.021..17.948 rows=144000 loops=1\) |
|                                                   -&gt;  Finalize GroupAggregate  \(cost=1000.46..177585.51 rows=146353 width=12\) \(actual time=31.937..238.848 rows=144000 loops=1\) |
|                                                         Group Key: t.fkride |
|                                                         -&gt;  Gather Merge  \(cost=1000.46..174658.45 rows=292706 width=12\) \(actual time=31.832..212.508 rows=161979 loops=1\) |
|                                                               Workers Planned: 2 |
|                                                               Workers Launched: 2 |
|                                                               -&gt;  Partial GroupAggregate  \(cost=0.43..139872.89 rows=146353 width=12\) \(actual time=2.141..253.846 rows=53993 loops=3\) |
|                                                                     Group Key: t.fkride |
|                                                                     -&gt;  Parallel Index Only Scan using ix\_tickets\_fkride\_id on tickets t  \(cost=0.43..127606.23 rows=2160627 width=12\) \(actual time=0.064..175.629 rows=1728502 loops=3\) |
|                                                                           Heap Fetches: 0 |
|                                             -&gt;  Hash  \(cost=25.40..25.40 rows=1440 width=8\) \(actual time=0.454..0.455 rows=1440 loops=1\) |
|                                                   Buckets: 2048  Batches: 1  Memory Usage: 73kB |
|                                                   -&gt;  Seq Scan on schedule s  \(cost=0.00..25.40 rows=1440 width=8\) \(actual time=0.007..0.215 rows=1440 loops=1\) |
|                                       -&gt;  Hash  \(cost=1.60..1.60 rows=60 width=8\) \(actual time=0.024..0.025 rows=60 loops=1\) |
|                                             Buckets: 1024  Batches: 1  Memory Usage: 11kB |
|                                             -&gt;  Seq Scan on busroute br  \(cost=0.00..1.60 rows=60 width=8\) \(actual time=0.006..0.012 rows=60 loops=1\) |
|                                 -&gt;  Hash  \(cost=1.10..1.10 rows=10 width=68\) \(actual time=0.015..0.016 rows=10 loops=1\) |
|                                       Buckets: 1024  Batches: 1  Memory Usage: 9kB |
|                                       -&gt;  Seq Scan on busstation bs  \(cost=0.00..1.10 rows=10 width=68\) \(actual time=0.008..0.009 rows=10 loops=1\) |
|                           -&gt;  Hash  \(cost=5.05..5.05 rows=5 width=12\) \(actual time=0.103..0.104 rows=5 loops=1\) |
|                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB |
|                                 -&gt;  HashAggregate  \(cost=5.00..5.05 rows=5 width=12\) \(actual time=0.098..0.099 rows=5 loops=1\) |
|                                       Group Key: s\_1.fkbus |
|                                       Batches: 1  Memory Usage: 24kB |
|                                       -&gt;  Seq Scan on seat s\_1  \(cost=0.00..4.00 rows=200 width=8\) \(actual time=0.026..0.041 rows=200 loops=1\) |
| Planning Time: 1.419 ms |
| JIT: |
|   Functions: 58 |
|   Options: Inlining false, Optimization false, Expressions true, Deforming true |
|   Timing: Generation 1.658 ms \(Deform 0.347 ms\), Inlining 0.000 ms, Optimization 0.732 ms, Emission 21.599 ms, Total 23.989 ms |
| Execution Time: 465.853 ms |
```

Как мы видим время запроса упало почти в 2 раза.
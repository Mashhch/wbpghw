1. Создать таблицу с продажами.

```sql
CREATE TABLE tests.sales
(
    sale_id   SERIAL PRIMARY KEY NOT NULL,
    sale_date DATE               NULL,
    amount    NUMERIC(10, 2)     NOT NULL
);

INSERT INTO tests.sales (sale_date, amount)
VALUES ('2024-01-15', 100.00),
       ('2024-05-20', 200.00),
       ('2024-09-11', 150.00),
       (NULL, 150.00),
       ('2024-11-05', 300.00);
```

2. Реализовать функцию выбор трети года (1-4 месяц - первая треть, 5-8 - вторая и тд)
   а. через case

```sql
CREATE OR REPLACE FUNCTION get_13(sale_date DATE)
RETURNS INT AS $$
BEGIN
    RETURN CASE
        WHEN EXTRACT(MONTH FROM sale_date) BETWEEN 1 AND 4 THEN 1
        WHEN EXTRACT(MONTH FROM sale_date) BETWEEN 5 AND 8 THEN 2
        WHEN EXTRACT(MONTH FROM sale_date) BETWEEN 9 AND 12 THEN 3
    END;
END;
$$ LANGUAGE plpgsql;
```

   b. * (бонуса в виде зачета дз не будет) используя математическую операцию (лучше 2+ варианта)

```sql
CREATE OR REPLACE FUNCTION get_13_v2(sale_date DATE)
RETURNS INT AS $$
BEGIN
   RETURN CEIL(EXTRACT(MONTH FROM sale_date) / 4);
END;
$$ LANGUAGE plpgsql;

```
3. Вызвать эту функцию в SELECT из таблицы с продажами, уведиться, что всё отработало

```sql
SELECT
    sale_id,
    sale_date,
    amount,
    get_13(sale_date),
    get_13_v2(sale_date)
FROM tests.sales;
```

Результат:

| sale\_id | sale\_date | amount | get\_13 | get\_13\_v2 |
|:---------|:-----------|:-------|:--------|:------------|
| 1        | 2024-01-15 | 100.00 | 1       | 1           |
| 2        | 2024-05-20 | 200.00 | 2       | 2           |
| 3        | 2024-09-11 | 150.00 | 3       | 3           |
| 4        | null       | 150.00 | null    | null        |
| 5        | 2024-11-05 | 300.00 | 3       | 3           |

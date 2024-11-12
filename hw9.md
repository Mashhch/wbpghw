Проанализировать данные о зарплатах сотрудников с использованием оконных функций.
а) На сколько было увеличение с предыдущей зарплатой
б) если это первая зарплата - вместо NULL вывести 0

со * - вывести текущую должность

Вставка данных (добавила еще одно промежуточное повышение для fk_employee = 1):

```sql
CREATE TYPE employee_gender AS ENUM (
    'M',
    'F'
    );

CREATE TABLE grade
(
    id    SERIAL PRIMARY KEY,
    value text NOT NULL
);

insert into grade (value)
values ('junior'),
       ('middle'),
       ('senoir'),
       ('lead');

CREATE TABLE employee
(
    id         SERIAL PRIMARY KEY,
    birth_date date            NOT NULL,
    first_name varchar(255)    NOT NULL,
    last_name  varchar(255)    NOT NULL,
    gender     employee_gender NOT NULL,
    hire_date  date            NOT NULL,
    doc_id     text            NOT NULL default '' -- passport N
);

insert into employee (birth_date, first_name, last_name, gender, hire_date, doc_id)
values ('19800101', 'Eugene', 'Aristov', 'M', '20240101', '2700123456'),
       ('19790901', 'Ivan', 'Ivanov', 'M', '20230101', '2400123456'),
       ('19810505', 'Petr', 'Petrov', 'M', '20240301', '4700123456');

CREATE TABLE salary
(
    fk_employee int  NOT NULL,
    amount      int  NOT NULL,
    from_date   date NOT NULL,
    to_date     date NOT NULL,
    fk_grade    int  NOT NULL references grade (id),
    CONSTRAINT pk_primary PRIMARY KEY (fk_employee, from_date),
    CONSTRAINT salaries_fk FOREIGN KEY (fk_employee) REFERENCES employee (id) ON UPDATE RESTRICT ON DELETE RESTRICT
);

insert into salary (fk_employee, amount, from_date, to_date, fk_grade)
values (1, 100000, '20240101', '20240131', 1),
       (1, 150000, '20240201', '20240202', 1),
       (1, 200000, '20240203', '20240229', 2),
       (1, 300000, '20240301', '20991231', 3),
       (2, 200000, '20230101', '20240131', 2),
       (3, 200000, '20240301', '20240131', 2);

CREATE TABLE title_name
(
    id          SERIAL PRIMARY KEY,
    title       varchar(255) NOT NULL,
    amount_from int          NOT NULL,
    amount_to   int          NOT NULL
);

insert into title_name(title, amount_from, amount_to)
values ('manager', 300000, 1000000),
       ('teamlead', 250000, 800000),
       ('python developer', 100000, 500000),
       ('vice president', 300000, 5000000);

CREATE TABLE title
(
    id           SERIAL PRIMARY KEY,
    fk_employee  int  NOT NULL,
    fk_titlename int  NOT NULL references title_name (id),
    from_date    date NOT NULL,
    to_date      date,
    CONSTRAINT pk_title UNIQUE (fk_employee, fk_titlename, from_date),
    CONSTRAINT titles_fk FOREIGN KEY (fk_employee) REFERENCES employee (id) ON UPDATE RESTRICT ON DELETE CASCADE
);

insert into title(fk_employee, fk_titlename, from_date)
values (1, 1, '20240101'),
       (1, 4, '20240301'),
       (2, 2, '20230101'),
       (3, 3, '20240301');

create table command
(
    fk_title  int references title (id), -- boss
    fk_title2 int references title (id), -- team
    CONSTRAINT pk_command PRIMARY KEY (fk_title, fk_title2)
);

insert into command
values (1, 2),
       (1, 3);
```

Оконная функция:

- Выбираем в латерал последнюю профессию, с датой меньше или равной для текущей зарплаты:

```sql
SELECT s.fk_employee,
       e.first_name || ' ' || e.last_name                                                           AS employee_name,
       s.amount,
       s.from_date,
       tn.title,
       COALESCE(s.amount - LAG(s.amount) OVER (PARTITION BY s.fk_employee ORDER BY s.from_date), 0) AS diff
FROM salary s
         INNER JOIN employee e
                    ON e.id = s.fk_employee
         LEFT JOIN LATERAL ( SELECT t.fk_titlename
                             FROM title t
                             WHERE t.fk_employee = s.fk_employee
                               AND t.from_date <= s.from_date
                             ORDER BY t.from_date DESC
                             LIMIT 1) AS l
                   ON TRUE
         LEFT JOIN title_name tn
                   ON l.fk_titlename = tn.id
ORDER BY s.fk_employee, s.from_date ASC;
```
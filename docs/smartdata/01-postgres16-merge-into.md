# Ядро PostgreSQL 16

> Карточка: `01-postgres16-merge-into.pdf`

## Проблема

Greenplum 6 живёт на ядре PostgreSQL 9.4 — релизе 2014 года. Это не только
«старые фичи»: это отдельная вселенная знаний. Любой вопрос про SQL приходится
переводить с современного Postgres на диалект двенадцатилетней давности, а ответы
со Stack Overflow — проверять на «а было ли это уже в 9.4».

## Что сделали

Мы обновили ядро Cloudberry до PostgreSQL 16. Полный список изменений занимает
несколько страниц — он собран в блоге проекта:
<https://cloudberry.apache.org/blog/postgresql16-for-apache-cloudberry-202606>.

Но ключевое изменение не в списке фич, а в том, что **Cloudberry стал понятен
обычному пользователю Postgres**. Больше не нужно вспоминать, как это делалось
в Postgres 12 лет назад: гуглите свой вопрос — и с большой вероятностью ответ
подойдёт и для Cloudberry.

## Пример: `MERGE`

Классический сценарий загрузки — есть целевая таблица и staging-таблица,
нужно обновить существующие строки и вставить новые.

```sql
CREATE TABLE users (
    id         BIGINT PRIMARY KEY,
    name       TEXT NOT NULL,
    email      TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ,
    is_active  BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE users_staging (
    id           BIGINT NOT NULL,
    name         TEXT,
    email        TEXT,
    status       TEXT,
    last_seen_at TIMESTAMPTZ
);
```

**Cloudberry, ядро PostgreSQL 16:**

```sql
postgres=# MERGE INTO users AS u
USING users_staging AS s ON u.id = s.id
WHEN MATCHED THEN UPDATE SET email = s.email, name = s.name
WHEN NOT MATCHED THEN INSERT (id, email, name, created_at)
VALUES (s.id, s.email, s.name, NOW());
MERGE 5
```

**Greenplum 6, ядро PostgreSQL 9.4:**

```sql
gp=> MERGE INTO users AS u
USING users_staging AS s ON u.id = s.id
WHEN MATCHED THEN UPDATE SET email = s.email, name = s.name
WHEN NOT MATCHED THEN INSERT (id, email, name, created_at)
VALUES (s.id, s.email, s.name, NOW());
ERROR:  syntax error at or near "MERGE"
LINE 1: MERGE INTO users AS u
```

`MERGE` в PostgreSQL появился только в 15-й версии — в 9.4 он числился
в [списке неподдержанных возможностей стандарта SQL](https://www.postgresql.org/docs/9.4/unsupported-features-sql-standard.html).
Вместо одного оператора приходилось писать связку `UPDATE ... FROM` +
`INSERT ... WHERE NOT EXISTS` в одной транзакции и самостоятельно следить
за согласованностью.

## Вывод

`MERGE` — это один пример из многих. Обновление ядра — это не «плюс N фич
в чейнджлоге», а снятие целого класса проблем: чужой опыт, документация,
инструменты и библиотеки современного Postgres теперь применимы к Cloudberry
напрямую.

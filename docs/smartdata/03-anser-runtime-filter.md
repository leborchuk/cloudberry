# Anser — runtime-фильтры для распределённых join'ов

> Карточка: `03-anser-runtime-filter.pdf`

## Проблема

Типичная картина в распределённом плане: большая таблица соединяется с маленькой,
и до результата доживает лишь малая доля строк большой таблицы. При этом
сканируется она целиком — сотни тысяч кортежей читаются с диска, прогоняются
через Motion и отбрасываются уже в `Hash Join`.

## Идея

В распределённом плане с Motion и Hash Join заранее отфильтруем большую таблицу
значениями из меньшей.

## Реализация

Автоматически строим Bloom-фильтры по результатам сканирования меньшей таблицы
и применяем их при сканировании большей. В плане появляются два новых
Custom Scan-узла:

- **Anser Bloom Producer** — над сканом build-стороны: пропускает строки дальше
  и попутно набивает Bloom-фильтр;
- **Anser Bloom Consumer** — над сканом probe-стороны: получает готовый фильтр
  и отбрасывает строки до join'а.

Фильтры собираются на сегментах, сводятся в один на координаторе
и раздаются обратно на сегменты. Включается через `gp_anser_runtime_filter`.

Это MVP по мотивам статьи [Predicate Transfer (VLDB, p3636-wu)](https://vldb.org/pvldb/vol16/p3636-wu.pdf);
здесь покрыт только сценарий с добавлением Bloom-фильтров в план выполнения.
Реализация — [Cloudberry PR #1942](https://github.com/apache/cloudberry/pull/1942).

## Стенд: создаём таблицы

Широкая «анкетная» таблица и таблица дополнительных полей к ней — обе
column-oriented append-only, распределены по разным ключам, так что join
неизбежно требует Motion.

```sql
CREATE TABLE applications
(
   id                    bigint,
   first_name            character varying(32),
   last_name             character varying(32),
   patronymic            character varying(60),
   birth_dt              timestamp(0) without time zone,
   phone_mobile          character varying(16),
   phone_work            character varying(16),
   additional_work_phone character varying(16),
   phone_home            character varying(16),
   additional_phone_home character varying(16),
   email                 character varying(50),
   region                character varying(255),
   product_category      character varying(50),
   dt_created            timestamp(0) without time zone,
   some_foreign_key_1    character varying(255),
   some_foreign_key_2    character varying(255),
   some_foreign_key_3    character varying(255),
   some_foreign_key_4    character varying(255),
   some_foreign_key_5    character varying(255),
   some_foreign_key_6    character varying(255),
   some_foreign_key_7    character varying(255),
   some_foreign_key_8    character varying(255),
   some_foreign_key_9    character varying(255),
   some_foreign_key_10   character varying(255),
   some_foreign_key_11   character varying(255),
   some_foreign_key_12   character varying(255),
   some_foreign_key_13   character varying(255),
   some_foreign_key_14   character varying(255),
   some_foreign_key_15   character varying(255),
   some_foreign_key_16   character varying(255),
   some_foreign_key_17   character varying(255)
)
WITH (APPENDONLY=true, ORIENTATION=column)
DISTRIBUTED BY (id);

CREATE TABLE applications_extra_fields
(
  id    bigint,
  name  character varying(100),
  value character varying(255)
)
WITH (APPENDONLY=true, ORIENTATION=column)
DISTRIBUTED BY (name);
```

Миллион анкет:

```sql
INSERT INTO applications (
    id, first_name, last_name, patronymic, birth_dt,
    phone_mobile, phone_work, additional_work_phone,
    phone_home, additional_phone_home, email,
    region, product_category, dt_created,
    some_foreign_key_1,  some_foreign_key_2,  some_foreign_key_3,
    some_foreign_key_4,  some_foreign_key_5,  some_foreign_key_6,
    some_foreign_key_7,  some_foreign_key_8,  some_foreign_key_9,
    some_foreign_key_10, some_foreign_key_11, some_foreign_key_12,
    some_foreign_key_13, some_foreign_key_14, some_foreign_key_15,
    some_foreign_key_16, some_foreign_key_17
)
SELECT
    gs AS id,
    'FirstName'  || gs::text,
    'LastName'   || gs::text,
    'Patronymic' || gs::text,
    (DATE '1980-01-01' + (gs % 15000))::timestamp,
    '+7900' || lpad((gs % 10000000)::text, 7, '0'),
    '+7495' || lpad((gs % 10000000)::text, 7, '0'),
    '+7496' || lpad((gs % 10000000)::text, 7, '0'),
    '+7499' || lpad((gs % 10000000)::text, 7, '0'),
    '+7498' || lpad((gs % 10000000)::text, 7, '0'),
    'user' || gs::text || '@example.com',
    (ARRAY['Moscow','Saint Petersburg','Novosibirsk','Yekaterinburg','Kazan'])[1 + (gs % 5)],
    (ARRAY['Loan','Credit Card','Mortgage','Deposit','Insurance'])[1 + (gs % 5)],
    (TIMESTAMP '2024-01-01' + (gs || ' seconds')::interval),
    'fk1_'  || gs::text, 'fk2_'  || gs::text, 'fk3_'  || gs::text,
    'fk4_'  || gs::text, 'fk5_'  || gs::text, 'fk6_'  || gs::text,
    'fk7_'  || gs::text, 'fk8_'  || gs::text, 'fk9_'  || gs::text,
    'fk10_' || gs::text, 'fk11_' || gs::text, 'fk12_' || gs::text,
    'fk13_' || gs::text, 'fk14_' || gs::text, 'fk15_' || gs::text,
    'fk16_' || gs::text, 'fk17_' || gs::text
FROM generate_series(1, 1000000) AS gs;
```

Дополнительные поля — по десять атрибутов на анкету. Здесь два режима, и
разница между ними — весь смысл демонстрации:

```sql
-- Селективный вариант: 1000 строк, покрывают ~100 анкет из миллиона.
-- Bloom-фильтр отбрасывает почти всю большую таблицу.
INSERT INTO applications_extra_fields (id, name, value)
SELECT
    1 + ((gs - 1) / 10) AS id,
    (ARRAY['passport_number','inn','snils','employment_status','monthly_income',
           'employer_name','education_level','marital_status','dependents_count',
           'address_registration'])[1 + ((gs - 1) % 10)] AS name,
    'value_' || gs::text AS value
FROM generate_series(1, 1000) AS gs;

-- Неселективный вариант: 1 000 000 строк, покрывают всю большую таблицу.
-- Фильтру нечего отбрасывать — видно чистый оверхед (см. раздел ниже).
-- INSERT ... FROM generate_series(1, 1000000) AS gs;
```

```sql
ANALYZE applications;
ANALYZE applications_extra_fields;
```

Сам запрос — обычный `LEFT JOIN`, никаких подсказок оптимизатору:

```sql
SELECT aef.name, aef.value, aef.id, a.*
FROM applications_extra_fields AS aef
LEFT JOIN applications AS a ON aef.id = a.id;
```

Переключатель — один:

```sql
SET gp_anser_runtime_filter = on;   -- или off
```

## Как это выглядит

```
postgres=# explain analyze select aef.name, aef.value, aef.id, a.*
from applications_extra_fields as aef
left join applications as a on aef.id = a.id;

 Gather Motion 3:1  (slice1; segments: 3)  (actual time=45.030..244.790 rows=10000 loops=1)
   ->  Hash Right Join  (actual time=46.043..233.784 rows=3400 loops=1)
         Hash Cond: (a.id = aef.id)
         ->  Custom Scan (Anser Bloom Consumer)  (actual time=33.550..218.237 rows=340 loops=1)
               Bloom Filter Size: 1048576 bytes
               Bloom Filter Stats: memory=1024kB checked=334042 rejected=333702
               Rows Removed by Bloom Filter: 333702
               ->  Seq Scan on applications a  (actual time=1.921..150.350 rows=334042 loops=1)
         ->  Hash  (actual time=10.895..10.896 rows=3400 loops=1)
               ->  Redistribute Motion 3:3  (slice2; segments: 3)  (actual time=0.768..10.359 rows=3400 loops=1)
                     Hash Key: aef.id
                     ->  Custom Scan (Anser Bloom Producer)  (actual time=0.591..9.212 rows=5000 loops=1)
                           Bloom Filter Size: 1048576 bytes
                           ->  Seq Scan on applications_extra_fields aef  (actual time=0.589..1.247 rows=5000 loops=1)
 Optimizer: GPORCA
 Execution Time: 252.013 ms
```

Из 334 042 прочитанных строк в join ушло 340 — фильтр отбросил 333 702, то есть
99,9 %.

## Цифры

Таблицы `applications` (1 млн строк, AO column) и `applications_extra_fields`,
кластер из 3 сегментов, запрос — `LEFT JOIN` по `id`:

| Сценарий | Оптимизатор | `gp_anser_runtime_filter` | Время |
|---|---|---|---|
| Селективный join | GPORCA | off | 970 ms |
| Селективный join | GPORCA | on | 589 ms |
| Селективный join | Postgres planner | off | 687 ms |
| Селективный join | Postgres planner | on | 602 ms |

Bloom-фильтр в 8 КБ убирает 333 129 строк из 334 042.

## Когда фильтр не помогает

Bloom-фильтр — не бесплатный: его нужно построить, свести и разослать. На
запросе, где в join проходит почти всё (1 млн × 1 млн, ~334 тыс. строк после
`Hash Right Join`), включённый фильтр дал 2442 ms против 2167 ms без него —
чистый оверхед.

Поэтому важна стоимостная оценка: фильтр окупается ровно тогда, когда build-сторона
отбрасывает существенную долю probe-стороны. Тайминг сбора фильтра на трёх
сегментах — порядка 34 ms от старта producer'ов до получения фильтра всеми
consumer'ами (1 МБ payload).

## Что дальше

- векторные операции над Bloom-фильтром;
- оптимизация пересылки: bucket sorting и кодирование Голомба;
- полный Robust Predicate Transfer (RPT+) — не только Bloom-фильтры на скане.

## Вывод

Anser переносит часть работы join'а на скан: вместо «прочитать всё и выбросить
лишнее в Hash Join» — «не читать лишнее вовсе». На селективных join'ах это
полтора-два раза по времени выполнения, без изменения запроса.

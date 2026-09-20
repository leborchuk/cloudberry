# Parallel join — два уровня параллелизма

> Карточка: `04-parallel-join.pdf`

## Идея

У MPP-движка и у PostgreSQL 16 параллелизм устроен по-разному — и эти два
уровня перемножаются:

- **MPP-уровень**: движок выполняет запрос на каждом сегменте;
- **внутрисегментный уровень**: PG16 запускает несколько worker-процессов
  для каждого процесса сегмента.

3 сегмента × 2 worker'а = 6 параллельных исполнителей, и `Gather Motion 6:1`
на координаторе вместо `3:1`.

![Два уровня параллелизма](images/parallel-two-levels.png)

Отдельно важен сам PG16: в нём сняли запрет на параллельные `FULL` и `RIGHT`
hash join'ы (и добавили `JOIN_RIGHT_ANTI`). До этого такие join'ы всегда
выполнялись одним процессом на сегмент.

## Цифры

Одинаковый `FULL OUTER JOIN`, 10 млн × 10 млн строк:

| | `enable_parallel = off` | `enable_parallel = on` |
|---|---|---|
| План | `Hash Full Join` · 3:1 | `Parallel Hash Full Join` · 6:1 |
| Время | **6.30 s** | **3.60 s** |

Ускорение — **~1.7×**.

## Ещё пример

```sql
CREATE TABLE p_probe (id int, pad text) DISTRIBUTED BY (id);
CREATE TABLE p_build (id int, v   text) DISTRIBUTED BY (id);
INSERT INTO p_probe SELECT g, repeat('x', 200) FROM generate_series(1, 2000000) g;
INSERT INTO p_build SELECT g, 'b' || g        FROM generate_series(1, 5000) g;
ANALYZE p_probe; ANALYZE p_build;

SELECT b.v, p.id FROM p_probe p JOIN p_build b ON p.id = b.id;
```

| Настройка | План | Execution Time |
|---|---|---|
| `enable_parallel = off` | `Gather Motion 2:1`, `Seq Scan` | 551 ms |
| `enable_parallel = on`, Postgres planner | `Gather Motion 8:1`, `Parallel Seq Scan` | 242 ms |
| `enable_parallel = on`, GPORCA | `Gather Motion 16:1`, `Parallel Seq Scan` | 167 ms |

Ключевая строчка в плане — `Parallel Seq Scan` под `Hash Join` и увеличившийся
знаменатель у `Gather Motion`:

```
 Gather Motion 16:1  (slice1; segments: 16)  (actual time=11.965..142.310 rows=5000 loops=1)
   ->  Hash Join  (actual time=11.072..133.499 rows=246 loops=1)
         Hash Cond: (p.id = b.id)
         ->  Parallel Seq Scan on p_probe p  (actual time=0.074..57.005 rows=144486 loops=1)
         ->  Hash  (actual time=2.126..2.127 rows=5000 loops=1)
               ->  Broadcast Motion 16:16  (slice2; segments: 16)  (actual time=0.047..0.915 rows=5000 loops=1)
                     ->  Parallel Seq Scan on p_build b
 Optimizer: GPORCA
 Execution Time: 166.964 ms
```

Включается так:

```sql
SET enable_parallel = on;
SET max_parallel_workers_per_gather = 8;
```

## Что уже есть, а что в планах

![PG16 Parallel Full/Right joins: live vs roadmap](images/parallel-join-matrix.png)

| Возможность | Статус в Apache Cloudberry main | Оптимизатор |
|---|---|---|
| Parallel Hash Full Join | ✅ ~1.7× (6-way: 3 seg × 2 workers) | Postgres planner |
| Parallel Hash Right Join | ✅ та же механика | Postgres planner |
| Right Anti Join (`JOIN_RIGHT_ANTI`) | ✅ из upstream PG16 | Postgres planner |
| FULL join под ORCA (по умолчанию) | ⭕ пока не параллельный — откат на однопроцессный `Hash Full Join` | GPORCA |
| ORCA parallel Full / Right Semi / Right Anti | ⭕ roadmap, готовим контрибьют в Apache | GPORCA |

Стенд: Apache Cloudberry 3.0.0-devel, PostgreSQL 16.9, 3 сегмента.
Чтобы получить параллельный `FULL`/`RIGHT` сегодня: `SET optimizer = off;
SET enable_parallel = on;`.

Подробнее о фиксе в PG16 — [разбор Max Yang](https://www.linkedin.com/pulse/postgresql-16s-parallel-hash-full-join-apache-cloudberry-max-yang-7ftbc/).

## Вывод

MPP-параллелизм по сегментам упирается в количество сегментов. Внутрисегментный
параллелизм PG16 умножает его на число worker'ов — та же железка, тот же запрос,
почти вдвое меньше времени. Под ORCA параллельные Full/Right join'ы пока в работе,
и мы несём их в апстрим.

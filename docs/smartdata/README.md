# Посмотрите на будущее Greenplum

Greenplum 6 живёт на ядре PostgreSQL 9.4 — релизе 2014 года. Всё, что появилось
в Postgres за эти двенадцать лет, проходило мимо: новый синтаксис, новый
планировщик, параллельное выполнение, инфраструктура расширений. Развитие
продолжилось в Apache Cloudberry — и ниже четыре фичи, которые показывают,
куда этот путь ведёт.

Это код из development-ветки Cloudberry. Он **уже работает**: планы, цифры
и скриншоты ниже сняты на живом стенде, а не нарисованы для слайда. Сейчас мы
активно готовим новый релиз, и всё перечисленное войдёт в него.

| № | Фича | Карточка | О чём |
|---|---|---|---|
| 01 | [Ядро PostgreSQL 16](01-postgres16-merge-into.md) | [PDF](cards/01-postgres16-merge-into.pdf) | Двенадцать лет апстрима разом — на примере `MERGE` |
| 02 | [YAGPCC 2.0](02-yagpcc-2.0.md) | [PDF](cards/02-yagpcc-2.0.pdf) | Дашборды кластера и live-план выполнения запроса |
| 03 | [Anser](03-anser-runtime-filter.md) | [PDF](cards/03-anser-runtime-filter.pdf) | Runtime Bloom-фильтры для распределённых join'ов |
| 04 | [Parallel join](04-parallel-join.md) | [PDF](cards/04-parallel-join.pdf) | Внутрисегментный параллелизм PG16 поверх MPP |

Порядок здесь не случаен. Обновлённое ядро — это фундамент: Parallel join
приходит прямо из PG16, Anser и YAGPCC строятся на его инфраструктуре
исполнителя. Одно обновление открывает дорогу остальным трём.

## Попробуйте прямо здесь

Рядом стоит ноутбук с развёрнутым Cloudberry. В каждом описании есть раздел
**«Стенд: создаём таблицы»** — полный скрипт: DDL, наполнение данными, запрос
и переключатель фичи. Всё копируется в `psql` целиком.

Быстрый маршрут, если времени мало:

1. [**01 — Ядро PostgreSQL 16**](01-postgres16-merge-into.md) — тридцать секунд,
   пять строк в staging, один `MERGE`.
2. [**03 — Anser**](03-anser-runtime-filter.md) — `SET gp_anser_runtime_filter = on/off`
   на готовых таблицах и сравнить `EXPLAIN ANALYZE`.
3. [**04 — Parallel join**](04-parallel-join.md) — `SET enable_parallel = on/off`,
   посмотреть на знаменатель у `Gather Motion`.
4. [**02 — YAGPCC 2.0**](02-yagpcc-2.0.md) — запустить долгий запрос и открыть
   его live-план в YAGPCC.

Наполнение данными для 03 и 04 занимает несколько минут — если таблицы уже
созданы на стенде, начните сразу с запроса.

---

<details>
<summary><b>🥚 А что если я хочу нормально побенчмаркать?</b></summary>

<br>

Синтетические таблицы на две колонки — это, конечно, не бенчмарк. На стенде
уже установлено расширение [`cbdb_tpcds`](https://github.com/avamingli/cbdb_tpcds):
полный TPC-DS — генерация данных, загрузка, 99 запросов и отчёт — не выходя
из `psql`.

```sql
CREATE EXTENSION tpcds;

-- весь путь одной командой: SF=1, 1 воркер на сегмент, AOCS-хранение
CALL tpcds.run(scale := 1, parallel := 1, storage_type := 'aocs');
```

На четырёх сегментах SF=1 проходит целиком примерно за пару минут. Дальше —
отчёт:

```sql
SELECT tpcds.report();
SELECT * FROM tpcds.bench_summary ORDER BY duration_ms DESC LIMIT 10;
SELECT tpcds.gen_chart();          -- PNG с картинкой + CSV
```

Если хочется по шагам, а не одной кнопкой:

```sql
SELECT tpcds.gen_schema('aocs');   -- 25 таблиц TPC-DS
SELECT tpcds.gen_data(1, 1);       -- dsdgen прямо на сегментах
SELECT tpcds.load_data(16);        -- загрузка через gpfdist
SELECT tpcds.gen_query();          -- 99 запросов из dsqgen
SELECT tpcds.bench();              -- прогон
```

### Здесь-то фичи выше и встречаются

TPC-DS — это 99 настоящих аналитических запросов, и на них интересно
покрутить ровно те переключатели, о которых шла речь:

```sql
SELECT tpcds.show(14);                          -- посмотреть текст запроса
SELECT tpcds.explain(14, 'ANALYZE, BUFFERS');   -- и его план

SET enable_parallel = on;                       -- фича 04
SET gp_anser_runtime_filter = on;               -- фича 03
SELECT tpcds.exec(14);                          -- один запрос + тайминг
```

Запрос 14 — самый тяжёлый в наборе, его удобно запустить и открыть
live-план в YAGPCC (фича 02), пока он идёт. А `tpcds.bench(optimizer := 'orca')`
против `tpcds.bench(optimizer := 'postgres')` — готовое сравнение двух
оптимизаторов на одинаковых данных.

### Мелкий шрифт

- Выше SF=1 нужна память: SF=100 — это 64 ГБ RAM и 8–16 сегментов, около
  11 минут прогона. Самый чувствительный параметр — `statement_mem`.
- `.dat`-файлы после загрузки не удаляются сами: `SELECT tpcds.clean_data();`.
- Это **не официальные результаты TPC-DS** — для них нужен полный аудит
  по спецификации.

</details>

---

Иллюстрации — в [`images/`](images/), карточки презентации — в [`cards/`](cards/).

# Полезные функции и команды

## Работа с блокировками

Матрица блокировок отношений:

![Matrix of locks.png](art/Matrix%20of%20locks.png)

Получить информацию о блокировках, взятых текущим процессом:
```sql
SELECT locktype, virtualxid, mode, granted
FROM pg_locks WHERE pid = (SELECT pg_backend_pid());
```
Представление для просмотра информации о блокировках
```sql
CREATE VIEW locks AS
SELECT pid,
locktype,
CASE locktype
WHEN 'relation' THEN relation::regclass::text
WHEN 'transactionid' THEN transactionid::text
WHEN 'virtualxid' THEN virtualxid
END AS lockid,
mode,
granted
FROM pg_locks
ORDER BY 1, 2, 3;
```
Показать номера процессов, которые стоят в очереди перед указанным и либо удерживают, либо
запрашивают несовместимую блокировку:

```sql
SELECT pid,
       pg_blocking_pids(pid),
       wait_event_type,
       state,
    left(query,50) AS query
FROM pg_stat_activity
WHERE pid IN (...)
```

## Работа с WAL

### Выбрать текущий LSN и LSN для следующей вставки
```sql
SELECT pg_current_wal_lsn(), pg_current_wal_insert_lsn();
```

### Заглянуть в заголовки журнальных записей можно либо утиллитой pg_waldump либо с помощью расширения pg_walinspect
```sql
CREATE EXTENSION pg_walinspect;
```

## Расшрение для просмотра информации о страницах
```sql
CREATE EXTENSION pageinspect;
```

## Расширение для просмотра информации о плотности распределения данных по страницам таблицы
```sql
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('TABLE_NAME');
SELECT * FROM pgstatindex('INDEX_NAME');
```

## Внутрь буферного кеша позволяет заглянуть расширение
```sql
CREATE EXTENSION pg_buffercache;
```

### Функция, позволяющая получить буферный кеши, относящиеся к таблице
```sql
CREATE FUNCTION buffercache(rel regclass)
RETURNS TABLE(
bufferid integer, relfork text, relblk bigint,
isdirty boolean, usagecount smallint, pins integer
) AS $$
SELECT bufferid,
CASE relforknumber
WHEN 0 THEN 'main'
WHEN 1 THEN 'fsm'
WHEN 2 THEN 'vm'
END,
relblocknumber,
isdirty,
usagecount,
pinning_backends
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode(rel)
AND relforknumber = 0 -- только main
ORDER BY relforknumber, relblocknumber;
$$ LANGUAGE sql;
```

## Запрос, чтобы посмотреть, какая часть таблицы закеширована в буфере и насколько активно используются эти данные:

```sql
SELECT c.relname,
count(*) blocks,
round( 100.0 * 8192 * count(*) /
pg_table_size(c.oid) ) AS "% of rel",
round( 100.0 * 8192 * count(*) FILTER (WHERE b.usagecount > 1) /
pg_table_size(c.oid) ) AS "% hot"
FROM pg_buffercache b
JOIN pg_class c ON pg_relation_filenode(c.oid) = b.relfilenode
WHERE b.reldatabase IN (
0, -- общие объекты кластера
(SELECT oid FROM pg_database
WHERE datname = current_database())
)
AND b.usagecount IS NOT NULL
GROUP BY c.relname, c.oid
ORDER BY 2 DESC
LIMIT 5; 
```


## Функция для просмотра информации о странице в таблице
```sql
CREATE FUNCTION heap_page(
relname text,
pageno integer
)
RETURNS TABLE(
ctid tid,
state text,
xmin text,
xmax text,
hhu text,
hot text,
t_ctid tid
) AS $$
SELECT (pageno,lp)::text::tid AS ctid,
CASE lp_flags
WHEN 0 THEN 'unused'
WHEN 1 THEN 'normal'
WHEN 2 THEN 'redirect to '||lp_off
WHEN 3 THEN 'dead'
END AS state,
t_xmin || CASE
WHEN (t_infomask & 256) > 0 THEN ' c'
WHEN (t_infomask & 512) > 0 THEN ' a'
ELSE ''
END AS xmin,
t_xmax || CASE
WHEN (t_infomask & 1024) > 0 THEN ' c'
WHEN (t_infomask & 2048) > 0 THEN ' a'
ELSE ''
END AS xmax,
CASE WHEN (t_infomask2 & 16384) > 0 THEN 't' END AS hhu,
CASE WHEN (t_infomask2 & 32768) > 0 THEN 't' END AS hot,
t_ctid
FROM heap_page_items(get_raw_page(relname,pageno))
ORDER BY lp;
$$ LANGUAGE sql;
```

## Функция просмотра информации об индексной странице

```sql
CREATE FUNCTION index_page(relname text, pageno integer)
RETURNS TABLE(itemoffset smallint, htid tid)
AS $$
SELECT itemoffset,
htid -- ctid до v.13
FROM bt_page_items(relname,pageno);
$$ LANGUAGE sql;
```

## Функция для определения текущего значения параметра либо из глобальной настройки, либо из настройки отдельной таблицы:

```sql
CREATE FUNCTION p(param text, c pg_class) RETURNS float
AS $$
SELECT coalesce(
-- если параметр хранения задан, то берем его
(SELECT option_value
FROM pg_options_to_table(c.reloptions)
WHERE option_name = CASE
-- для toast-таблиц имя параметра отличается
WHEN c.relkind = 't' THEN 'toast.' ELSE ''
END || param
),
-- иначе берем значение конфигурационного параметра
current_setting(param)
)::float;
$$ LANGUAGE sql;
```

## Представление для хранения информации о таблицах, которые требуют очистки:
```sql
CREATE VIEW need_vacuum AS
WITH c AS (
SELECT c.oid,
greatest(c.reltuples, 0) reltuples,
p('autovacuum_vacuum_threshold', c) threshold,
p('autovacuum_vacuum_scale_factor', c) scale_factor,
p('autovacuum_vacuum_insert_threshold', c) ins_threshold,
p('autovacuum_vacuum_insert_scale_factor', c) ins_scale_factor
FROM pg_class c
WHERE c.relkind IN ('r','m','t')
)
SELECT st.schemaname || '.' || st.relname AS tablename,
st.n_dead_tup AS dead_tup,
c.threshold + c.scale_factor * c.reltuples AS max_dead_tup,
st.n_ins_since_vacuum AS ins_tup,
c.ins_threshold + c.ins_scale_factor * c.reltuples AS max_ins_tup,
st.last_autovacuum
FROM pg_stat_all_tables st
JOIN c ON c.oid = st.relid;
```

## Представление для хранения информации о таблицах, которые требуют анализа:

```sql
CREATE VIEW need_analyze AS
WITH c AS (
SELECT c.oid,
greatest(c.reltuples, 0) reltuples,
p('autovacuum_analyze_threshold', c) threshold,
p('autovacuum_analyze_scale_factor', c) scale_factor
FROM pg_class c
WHERE c.relkind IN ('r','m')
)
SELECT st.schemaname || '.' || st.relname AS tablename,
st.n_mod_since_analyze AS mod_tup,
c.threshold + c.scale_factor * c.reltuples AS max_mod_tup,
st.last_autoanalyze
FROM pg_stat_all_tables st
JOIN c ON c.oid = st.relid;
```

## Функция для отображения диапазона страниц, расшифровки признака заморозки ("f") и показа
возраста транзакции xmin (используется системная функция age)

```sql
CREATE FUNCTION heap_page(
relname text, pageno_from integer, pageno_to integer
)
RETURNS TABLE(
ctid tid, state text,
xmin text, xmin_age integer, xmax text
) AS $$
SELECT (pageno,lp)::text::tid AS ctid,
CASE lp_flags
WHEN 0 THEN 'unused'
WHEN 1 THEN 'normal'
WHEN 2 THEN 'redirect to '||lp_off
WHEN 3 THEN 'dead'
END AS state,
t_xmin || CASE
WHEN (t_infomask & 256+512) = 256+512 THEN ' f'
WHEN (t_infomask & 256) > 0 THEN ' c'
WHEN (t_infomask & 512) > 0 THEN ' a'
ELSE ''
END AS xmin,
age(t_xmin) AS xmin_age,
t_xmax || CASE
WHEN (t_infomask & 1024) > 0 THEN ' c'
WHEN (t_infomask & 2048) > 0 THEN ' a'
ELSE ''
END AS xmax
FROM generate_series(pageno_from, pageno_to) p(pageno),
heap_page_items(get_raw_page(relname, pageno))
ORDER BY pageno, lp;
$$ LANGUAGE sql;
```


# Служебные таблицы

## Статистика по таблицам
Примерное число неактуальных версий строк (n_dead_tup) постоянно отслеживается в таблице: pg_stat_all_tables

Число вставленных строк с момента последней очистки (n_ins_since_vacuum).

Число измененных строк с момента последнего анализа (n_mod_since_analyze).

## Представление, показывающее прогресс ручной очистки
pg_stat_progress_vacuum

## Представление, показывающее прогресс ручного анализа
pg_stat_progress_analyze

## Представление с блокировками
pg_locks

# Возможные микрооптимизации и советы:

1. Еще одна возможная микрооптимизация—перенести в начало таблицы все
столбцы фиксированного размера, недопускающие неопределенных значений. Доступ к таким полям будет более эффективным благодаря возможности закешировать смещение поля от начала версии строки.


# Выполнение запросов
## Планирование и исполнение
Посмотреть именованные подготовленные операторы можно в представлении `pg_prepared_statements`.

```sql
SELECT name, statement, parameter_types
FROM pg_prepared_statements;
```

Также в этом представлении есть и статистика выбора планов:
```sql
SELECT name, generic_plans, custom_plans
FROM pg_prepared_statements;
```

Подготовленные операторы с параметрами первые пять раз всегда оптимизируются с учетом
фактических значений; при этом вычисляется средняя стоимость частных планов. Начиная с
шестого раза, если общий план оказывается в среднем выгоднее, чем частные (с учетом того,
что частные планы необходимо строить каждый раз заново), планировщик запоминает общий план
и дальше использует его, уже не повторяя оптимизацию. 

При неправильном автоматическом решении можнопринудительно выбрать общий либо частный 
план,установив соответствующее значение параметра `plan_cache_mode`.

```sql
SET plan_cache_mode = 'force_custom_plan';
```

## Статистика

Базовая статистика уровня отношения хранится в таблице `pg_class` системного каталога:
- reltuples - число строк
- relpages - размер отношения в страницах
- relallvisible - количество страниц, отмеченных в карте видимости

```sql
SELECT reltuples, relpages, relallvisible
FROM pg_class WHERE relname = 'TABLE_NAME';
```

Помимо самой простой, базовой статистики на уровне отношений, при анализе собирается
статистика для каждого столбца отношения. Она хранится в таблице системного каталога
`pg_statistics`. Но значительно проще пользоваться представлением `pg_stats`, которое 
показывает информацию в более удобном виде. 

`null_frac` - доля неопределенных значений, вычисленная при анализе

Примерная статистика по количеству строк, в которых для столбца COLUMN_NAME значение = NULL.
```sql
SELECT round(reltuples * s.null_frac) AS rows
FROM pg_class
JOIN pg_stats s ON s.tablename = relname
WHERE s.tablename = 'TABLE_NAME'
AND s.attname = 'COLUMN_NAME';
```

`n_distinct` - количество уникальных значений в столбце

`most_common_vals` - наиболее часто встречающиеся значения

`most_common_freqs` - частота появления наиболее часто встречающихся значений

```sql
SELECT most_common_vals AS mcv,
left(most_common_freqs::text,60) || '...' AS mcf
FROM pg_stats
WHERE tablename = 'flights' AND attname = 'aircraft_code' \gx

-[ RECORD 1 ]--------------------------------------------------------
mcv | {CN1,CR2,SU9,321,733,763,319,773}
mcf | {0.2778,0.2751,0.26073334,0.057066668,0.0385,0.037233334,0.0...
```

Для оценки селективности условия «столбец=начение» достаточно найти
значение в массиве most_common_vals и взять частоту из элемента массива
most_common_freqs с тем же номером:

```sql
SELECT round(reltuples * s.most_common_freqs[
array_position((s.most_common_vals::text::text[]),'733')
])
FROM pg_class
JOIN pg_stats s ON s.tablename = relname
WHERE s.tablename = 'flights'
AND s.attname = 'aircraft_code';
```

Создать расширенную статистику по выражению:
```sql
CREATE STATISTICS flights_expr_stat ON (extract(
month FROM scheduled_departure AT TIME ZONE 'Europe/Moscow'
))
FROM flights;
```

# Индексы
## Операторы и классы операторов

Класс операторов `text_pattern_ops` позволяет преодолеть ограничение на
поддержку оператора `~~` (который соответствует конструкции LIKE). В базе данных 
с правилом сортировки,отличным от C, этот оператор не может использовать обычный
индекс по текстовому полю. Другое дело — индекс с классом операторов `text_pattern_ops`:

```sql
=> SELECT datcollate FROM pg_database
   WHERE datname = current_database();
datcollate
−−−−−−−−−−−−−
en_US.UTF−8
(1 row)
=> CREATE INDEX ON tickets(passenger_name);
=> EXPLAIN (costs off)
SELECT * FROM tickets WHERE passenger_name LIKE 'ELENA%';

QUERY PLAN
−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−
Seq Scan on tickets
Filter: (passenger_name ~~ 'ELENA%'::text)
(2 rows)

=> CREATE INDEX tickets_passenger_name_pattern_idx
ON tickets(passenger_name text_pattern_ops);
=> EXPLAIN (costs off)
SELECT * FROM tickets WHERE passenger_name LIKE 'ELENA%';
QUERY PLAN
−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−
Bitmap Heap Scan on tickets
Filter: (passenger_name ~~ 'ELENA%'::text)
−> Bitmap Index Scan on tickets_passenger_name_pattern_idx
Index Cond: ((passenger_name ~>=~ 'ELENA'::text) AND
(passenger_name ~<~ 'ELENB'::text))
(5 rows)
```

## Индекс по выражению:
```sql
=> CREATE INDEX ON tickets( (initcap(passenger_name)) );
=> EXPLAIN (costs off)
SELECT * FROM tickets WHERE initcap(passenger_name) = 'Elena Belova';
QUERY PLAN
−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−−
Bitmap Heap Scan on tickets
Recheck Cond: (initcap(passenger_name) = 'Elena Belova'::text)
−> Bitmap Index Scan on tickets_initcap_idx
Index Cond: (initcap(passenger_name) = 'Elena Belova'::text)
(4 rows)
```

## Свойства индекса:
```sql
SELECT p.name, pg_index_has_property('seats_pkey', p.name)
FROM unnest(array[
'clusterable', 'index_scan', 'bitmap_scan', 'backward_scan'
]) p(name);
```

## Свойства столбцов:
```sql
SELECT p.name,
pg_index_column_has_property('seats_pkey', 1, p.name)
FROM unnest(array[
'asc', 'desc', 'nulls_first', 'nulls_last', 'orderable',
'distance_orderable', 'returnable', 'search_array', 'search_nulls'
]) p(name);
```
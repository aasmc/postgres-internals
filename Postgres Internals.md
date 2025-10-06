# Полезные функции и команды

## Работа с WAL

### Выбрать текущий LSN и LSN для следующей вставки
SELECT pg_current_wal_lsn(), pg_current_wal_insert_lsn();

### Заглянуть в заголовки журнальных записей можно либо утиллитой pg_waldump либо с помощью расширения pg_walinspect

CREATE EXTENSION pg_walinspect;

## Расшрение для просмотра информации о страницах
CREATE EXTENSION pageinspect;

## Расширение для просмотра информации о плотности распределения данных по страницам таблицы
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('TABLE_NAME');
SELECT * FROM pgstatindex('INDEX_NAME');

## Внутрь буферного кеша позволяет заглянуть расширение
CREATE EXTENSION pg_buffercache;

### Функция, позволяющая получить буферный кеши, относящиеся к таблице
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

## Запрос, чтобы посмотреть, какая часть таблицы закеширована в буфере и насколько активно используются эти данные:

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


## Функция для просмотра информации о странице в таблице

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

## Функция просмотра информации об индексной странице

CREATE FUNCTION index_page(relname text, pageno integer)
RETURNS TABLE(itemoffset smallint, htid tid)
AS $$
SELECT itemoffset,
htid -- ctid до v.13
FROM bt_page_items(relname,pageno);
$$ LANGUAGE sql;

## Функция для определения текущего значения параметра либо из глобальной настройки, либо из настройки отдельной таблицы:

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

## Представление для хранения информации о таблицах, которые требуют очистки:

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

## Представление для хранения информации о таблицах, которые требуют анализа:

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

## Функция для отображения диапазона страниц, расшифровки признака заморозки ("f") и показа
возраста транзакции xmin (используется системная функция age)

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


# Служебные таблицы

## Статистика по таблицам
Примерное число неактуальных версий строк (n_dead_tup) постоянно отслеживается в таблице: pg_stat_all_tables

Число вставленных строк с момента последней очистки (n_ins_since_vacuum).

Число измененных строк с момента последнего анализа (n_mod_since_analyze).

## Представление, показывающее прогресс ручной очистки
pg_stat_progress_vacuum

## Представление, показывающее прогресс ручного анализа
pg_stat_progress_analyze



# Возможные микрооптимизации и советы:

1. Еще одна возможная микрооптимизация—перенести в начало таблицы все
столбцы фиксированного размера, недопускающие неопределенных значений. Доступ к таким полям будет более эффективным благодаря возможности закешировать смещение поля от начала версии строки.


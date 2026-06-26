# OceanBase как БД для Zabbix + онлайн-миграция с PostgreSQL через Flink CDC. Экономия места в 15–18 раз

## Немного предыстории

Ещё в 2022 году внедрялся Zabbix у заказчика, нужен он был для мониторинга 10 MSSQL-серверов и был установлен на MySQL; с тех пор подключали к нему новые хосты, и начал он потихоньку умирать — всё-таки MySQL не для больших объёмов. Проблема миграции стала острой, а я как раз OceanBase последнее время занимаюсь очень активно.

Мне не удалось сразу уговорить заказчика на миграцию основного Zabbix на OceanBase, но была предложена миграция девелоперского Zabbix, который всё равно надо было переносить в целевой сегмент сети, и по результатам этой миграции уже будет принято решение о возможности миграции основного Zabbix, который на MySQL сейчас. Естественно, я согласился, хотя девелоперский Zabbix был на PG, но этот кейс даже более интересный. Мне будет очень интересно услышать отзывы или советы перед миграцией прома.

Итак, поехали.

Не расплываясь в этой статье про преимущества OceanBase, выделю только главное для Zabbix:

- Встроенное сжатие, дающее даже без колоночного формата 3–5-кратную экономию места по сравнению с PostgreSQL.
- Колоночное хранение с LSM-деревьями, благодаря которым не снижается скорость записи.
- Поддержка MySQL-синтаксиса.
- Настоящий enterprise HA без дополнительных компонентов.
- Multitenant-архитектура полезна на будущее, когда на одном кластере надо разграничить ресурсы на несколько проектов.
- И всё это уже в опенсорс CE-версии.

Вот главный плюс — размер топ-5 таблиц в PostgreSQL и OceanBase:

<!-- HABR: обернуть в <spoiler title="Размер топ-5 таблиц: PostgreSQL и OceanBase"> ... </spoiler> -->
<details>
<summary>SQL-запросы и вывод: размер топ-5 таблиц (PostgreSQL и OceanBase)</summary>

**PostgreSQL**

```sql
zabbix=# SELECT
    table_name,
    (xpath('/row/count/text()', xml_count))[1]::text::bigint AS row_count,
    pg_size_pretty(pg_total_relation_size(quote_ident(table_name))) AS total_size
FROM (
    SELECT
        table_name,
        query_to_xml(format('SELECT count(*) AS count FROM %I', table_name), false, true, '') AS xml_count
    FROM information_schema.tables
    WHERE table_schema = 'public'
) AS counts
ORDER BY row_count DESC LIMIT 5;
```

```text
  table_name   │ row_count │ total_size
--------------+-----------+------------
 history       │  14093149 │ 2102 MB
 history_uint  │  10940927 │ 1373 MB
 trends        │   2730276 │ 355 MB
 trends_uint   │   2614163 │ 331 MB
 event_tag     │     68147 │ 6872 kB
(5 rows)
```

**OceanBase**

```sql
[zabbix│zabbix] 07:44:48> SELECT
    t.table_name,
    COALESCE(s.row_count, 0) AS row_count,
    CONCAT(ROUND(s.data_size / 1024 / 1024, 2), ' MB') AS total_size
FROM information_schema.tables AS t
LEFT JOIN (
    SELECT
        table_name,
        SUM(data_length + index_length) AS data_size,
        SUM(table_rows) AS row_count
    FROM information_schema.tables
    WHERE table_schema = DATABASE()
    GROUP BY table_name
) AS s ON s.table_name = t.table_name
WHERE t.table_schema = DATABASE()
ORDER BY row_count DESC LIMIT 5;
```

```text
+--------------+-----------+------------+
│ table_name   │ row_count │ total_size │
+--------------+-----------+------------+
│ history      │  15049414 │ 100.00 MB  │
│ history_uint │  11826499 │ 74.00 MB   │
│ trends       │   2747343 │ 28.00 MB   │
│ trends_uint  │   2631177 │ 28.00 MB   │
│ event_tag    │     65595 │ 4.00 MB    │
+--------------+-----------+------------+
5 rows in set (0.66 sec)
```

</details>

## Как планировалась миграция

Мы решили мигрировать онлайн, не спеша, переключая клиентов по одному со старого на новый сервер. На время миграции было решено, что не будет изменений конфигурации, поэтому большинство таблиц переедут одним одноразовым джобом, а таблицы, в которые пишутся метрики и их история, будут мигрироваться онлайн через Flink CDC без потери данных. На время миграции хосты, оставшиеся на старом заббиксе, будут писать данные в старую базу на постгрес, а Flink CDC будет в реальном времени переливать данные из Postgres в OceanBase. Таким образом, клиентов можно будет переключать не спеша без потери метрик.

## Анализ схемы данных Zabbix в PostgreSQL перед миграцией

Схема Zabbix делится на два класса таблиц, и это определило стратегию миграции. Первый — конфигурация и операционные данные (хосты, шаблоны, items, триггеры, события — ~178 таблиц): они относительно статичны и переносятся разовым batch-снимком. Второй — метрики (history, history_uint, history_str, trends, trends_uint): это непрерывный высокочастотный поток, который нельзя «заморозить», поэтому именно эти пять таблиц мы переносили онлайн через CDC (initial snapshot + streaming).

Ключевое для миграции — как Zabbix генерирует идентификаторы. Вопреки ожиданиям, он почти не использует AUTO_INCREMENT СУБД: бизнес-ID раздаёт собственный аллокатор — таблица ids (тройки table_name, field_name, nextid, то есть просто числа, инкрементируемые в коде приложения). Единственный настоящий AUTO_INCREMENT — поле changelogid в служебной таблице changelog (механизм отслеживания изменений), которую мы оставили пустой. А пять таблиц метрик вообще не имеют суррогатных ID — у них натуральный первичный ключ (itemid, clock, ns для history; itemid, clock для trends).

Эта особенность оказалась удачной: отсутствие зависимости от автоинкремента СУБД снимает главную проблему мульти-мастер сценария — коллизии ID при параллельной записи на двух узлах. Натуральный ключ метрик вдобавок делает повторную вставку той же строки безопасной, а монотонный clock идеально ложится на партиционирование по времени в OceanBase.

## Миграция по шагам

1. Установили OceanBase CE 4.4.2 — кластер + OBProxy, создали тенант zb_tenant, БД zabbix (обязательно utf8mb4).
2. Установили Flink + Flink CDC — postgres-cdc и jdbc-коннекторы, настроили TaskManager-слоты.
3. Подготовили PostgreSQL-источник — wal_level=logical, пользователь репликации, публикация для CDC.
4. Мигрировали config-таблицы (178 шт.) — через Flink JDBC batch (не CDC, т.к. данные статичные), сверили построчно.
5. Перенесли триггеры (65 шт.) — из дампа.
6. Обошли проверку версии БД — Zabbix требует MySQL ≥8.0.30, а OBServer в handshake отдаёт 5.7.25; решили через OBProxy (mysql_version=8.0.30), подключив Zabbix через прокси.
7. Установили новый zabbix-server 7.0 на отдельном хосте, подключили к OceanBase, подняли web-интерфейс, проверили, что все хосты переехали, ошибок нет.
8. Спроектировали history/trends под OceanBase — columnstore + RANGE-партиции по clock, написали хранимую процедуру + EVENT для авто-нарезки будущих партиций.
9. Мигрировали history/trends (33M строк) — через Flink CDC (initial snapshot + streaming).
10. Обеспечили устойчивость CDC — checkpointing + restart-strategy + лимит удержания WAL.
11. Пережили реальный сетевой сбой — джобы восстановились с чекпоинтов без потери данных (стресс-тест «из жизни»).
12. Перенесли MSSQL ODBC (DSN + драйвер) — наш Zabbix напрямую опрашивает MSSQL.
13. Cutover — запустили новый сервер, переводим клиентов по одному, старый PostgreSQL-Zabbix дорабатывает параллельно.
14. За два дня все хосты перенесли.

Каждый пункт расписывать будет очень объёмно, да и от версии к версии Zabbix схемы данных меняются, и скрипты будут бесполезны. Я посоветую тем, кто пойдёт по этому пути, брать в помощь Opus 4.8 и выше — он вполне справится со всеми пунктами, особенно если дать эту статью как вводные данные.

Отметим только те пункты, в которых были проблемы, и как их решили.

### П. 6. Проверка версии

Сервер ставили на standalone-узел OceanBase, без кластера — рассчитывали подключить Zabbix напрямую к OBServer (2881), без OBProxy. Не вышло: Zabbix на старте требует MySQL ≥8.0.30, а OBServer в handshake жёстко отдаёт 5.7.25. AllowUnsupportedDBVersions=1 не помогает (слишком старая), строку версии в OBServer не поменять. Решение — OBProxy с mysql_version=8.0.30: подключили Zabbix через прокси (2883), проверка прошла. Итог: прокси пришлось поднять даже на standalone — ради единственной подмены версии в handshake.

### П. 8. Проектирование history/trends: columnstore + партиции

Пять таблиц метрик пересоздали под OceanBase: columnstore (колоночное хранение — сжатие + быстрая аналитика по году trends) и RANGE-партиции по clock (месячные, для быстрого DROP PARTITION вместо медленного housekeeper-DELETE).

Синтаксис для OB 4.4.2 делали эмпирически, два нюанса: граница партиции — только целый литерал (не UNIX_TIMESTAMP()), и WITH COLUMN GROUP идёт строго после PARTITION BY (иначе ERROR 1064). Пример history:

<!-- HABR: обернуть в <spoiler title="history: CREATE TABLE + процедура авто-нарезки партиций + EVENT"> ... </spoiler> -->
<details>
<summary>history: CREATE TABLE + процедура авто-нарезки партиций + EVENT</summary>

```sql
CREATE TABLE `history` (
  `itemid` bigint(20) unsigned NOT NULL,
  `clock`  int(11) NOT NULL DEFAULT '0',
  `value`  double  NOT NULL DEFAULT '0',
  `ns`     int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`itemid`,`clock`,`ns`)
) DEFAULT CHARSET = utf8mb4 COLLATE = utf8mb4_bin
  PARTITION BY RANGE (`clock`) (
    PARTITION `p2025_06` VALUES LESS THAN (1751328000)
  ) WITH COLUMN GROUP (each column);
```

Встроенный DYNAMIC_PARTITION_POLICY не подошёл — он только для DATE/TIMESTAMP, а clock это int (unix-секунды). Поэтому авто-нарезку будущих партиций сделали сами: хранимая процедура добирает месячные партиции вперёд до «сейчас + N месяцев», и EVENT вызывает её ежедневно.

```sql
CREATE PROCEDURE sp_manage_partitions(IN p_months_ahead INT)
BEGIN
  ...
  SET time_zone = '+00:00';  -- clock = unix UTC
  SET v_target = UNIX_TIMESTAMP(DATE_ADD(DATE_FORMAT(NOW(),'%Y-%m-01'),
                                         INTERVAL p_months_ahead MONTH));
  -- по каждой таблице: читаем max-границу, добавляем месяцы вперёд
  SET v_bound = UNIX_TIMESTAMP(DATE_ADD(FROM_UNIXTIME(v_bound), INTERVAL 1 MONTH));
  SET v_pname = CONCAT('p', DATE_FORMAT(FROM_UNIXTIME(v_bound - 1), '%Y_%m'));
  SET @sql = CONCAT('ALTER TABLE `', v_tbl, '` ADD PARTITION (PARTITION `',
                    v_pname, '` VALUES LESS THAN (', v_bound, '))');
  PREPARE st FROM @sql; EXECUTE st; DEALLOCATE PREPARE st;
  ...
END;

CREATE EVENT evt_manage_partitions
  ON SCHEDULE EVERY 1 DAY
  STARTS (CURRENT_TIMESTAMP + INTERVAL 1 HOUR)
  DO CALL sp_manage_partitions(3);
```

Процедуру нужно создавать/вызывать с collation_connection = utf8mb4_general_ci (иначе ERROR 1267 на сравнении с information_schema), event_scheduler = ON ставится только от root, а границы считаются строго в UTC (time_zone='+00:00'), т.к. clock — unix-время.

</details>

Сейчас старые метрики удаляет housekeeper Zabbix обычными DELETE — про партиции он не знает. Но партиционирование открывает путь к мгновенной очистке: отключив housekeeper и переложив ретеншн на свою процедуру, можно будет отбрасывать целый месяц одним DROP PARTITION вместо перебора миллионов строк. А на промышленном кластере из трёх узлов OceanBase распределит партиции по серверам, и запросы к истории пойдут параллельно по узлам — то есть быстрее. Авто-создание партиций мы уже сделали; ускоренное удаление через DROP — следующий шаг.

### П. 9. Динамическая миграция через Flink CDC

Заливку реализовали декларативно через Flink SQL. На каждую из пяти таблиц метрик — history, history_uint, history_str, trends, trends_uint — по три выражения: CREATE TABLE источника (коннектор postgres-cdc, режим initial — снапшот + последующий streaming), CREATE TABLE приёмника (коннектор jdbc в OceanBase через прокси) и INSERT INTO ... SELECT, который запускает джобу.

Возможно, спорный подход, но для этого случая подходящий.

Вот сами коды:

<!-- HABR: обернуть в <spoiler title="Flink SQL: пять пар src/snk + INSERT"> ... </spoiler> -->
<details>
<summary>Flink SQL: пять пар таблиц (источник/приёмник) + INSERT</summary>

```sql
-- ===== history (double) =====
CREATE TABLE src_history (
  itemid BIGINT, clock INT, `value` DOUBLE, ns INT,
  PRIMARY KEY (itemid, clock, ns) NOT ENFORCED
) WITH (
  'connector'='postgres-cdc','hostname'='192.168.88.41','port'='5432',
  'username'='cdc_zbx','password'='***',
  'database-name'='zabbix','schema-name'='public','table-name'='history',
  'slot.name'='flink_zbx_history','decoding.plugin.name'='pgoutput',
  'debezium.publication.name'='zbx_cdc_pub','debezium.publication.autocreate.mode'='disabled',
  'scan.incremental.snapshot.enabled'='true','scan.startup.mode'='initial'
);
CREATE TABLE snk_history (
  itemid BIGINT, clock INT, `value` DOUBLE, ns INT,
  PRIMARY KEY (itemid, clock, ns) NOT ENFORCED
) WITH (
  'connector'='jdbc',
  'url'='jdbc:mysql://10.10.5.205:2883/zabbix?useSSL=false&rewriteBatchedStatements=true&sessionVariables=foreign_key_checks=0',
  'driver'='com.mysql.cj.jdbc.Driver','username'='zabbix@zb_tenant','password'='***',
  'table-name'='history','sink.buffer-flush.max-rows'='200','sink.buffer-flush.interval'='3s'
);
INSERT INTO snk_history SELECT itemid,clock,`value`,ns FROM src_history;

-- ===== history_uint (bigint unsigned) =====
CREATE TABLE src_history_uint (
  itemid BIGINT, clock INT, `value` BIGINT, ns INT,
  PRIMARY KEY (itemid, clock, ns) NOT ENFORCED
) WITH (
  'connector'='postgres-cdc','hostname'='192.168.88.41','port'='5432',
  'username'='cdc_zbx','password'='***',
  'database-name'='zabbix','schema-name'='public','table-name'='history_uint',
  'slot.name'='flink_zbx_history_uint','decoding.plugin.name'='pgoutput',
  'debezium.publication.name'='zbx_cdc_pub','debezium.publication.autocreate.mode'='disabled',
  'scan.incremental.snapshot.enabled'='true','scan.startup.mode'='initial'
);
CREATE TABLE snk_history_uint (
  itemid BIGINT, clock INT, `value` BIGINT, ns INT,
  PRIMARY KEY (itemid, clock, ns) NOT ENFORCED
) WITH (
  'connector'='jdbc',
  'url'='jdbc:mysql://10.10.5.205:2883/zabbix?useSSL=false&rewriteBatchedStatements=true&sessionVariables=foreign_key_checks=0',
  'driver'='com.mysql.cj.jdbc.Driver','username'='zabbix@zb_tenant','password'='***',
  'table-name'='history_uint','sink.buffer-flush.max-rows'='200','sink.buffer-flush.interval'='3s'
);
INSERT INTO snk_history_uint SELECT itemid,clock,`value`,ns FROM src_history_uint;

-- ===== history_str (varchar 255) =====
CREATE TABLE src_history_str (
  itemid BIGINT, clock INT, `value` STRING, ns INT,
  PRIMARY KEY (itemid, clock, ns) NOT ENFORCED
) WITH (
  'connector'='postgres-cdc','hostname'='192.168.88.41','port'='5432',
  'username'='cdc_zbx','password'='***',
  'database-name'='zabbix','schema-name'='public','table-name'='history_str',
  'slot.name'='flink_zbx_history_str','decoding.plugin.name'='pgoutput',
  'debezium.publication.name'='zbx_cdc_pub','debezium.publication.autocreate.mode'='disabled',
  'scan.incremental.snapshot.enabled'='true','scan.startup.mode'='initial'
);
CREATE TABLE snk_history_str (
  itemid BIGINT, clock INT, `value` STRING, ns INT,
  PRIMARY KEY (itemid, clock, ns) NOT ENFORCED
) WITH (
  'connector'='jdbc',
  'url'='jdbc:mysql://10.10.5.205:2883/zabbix?useSSL=false&rewriteBatchedStatements=true&sessionVariables=foreign_key_checks=0',
  'driver'='com.mysql.cj.jdbc.Driver','username'='zabbix@zb_tenant','password'='***',
  'table-name'='history_str','sink.buffer-flush.max-rows'='200','sink.buffer-flush.interval'='3s'
);
INSERT INTO snk_history_str SELECT itemid,clock,`value`,ns FROM src_history_str;

-- ===== trends (double agg) =====
CREATE TABLE src_trends (
  itemid BIGINT, clock INT, num INT, value_min DOUBLE, value_avg DOUBLE, value_max DOUBLE,
  PRIMARY KEY (itemid, clock) NOT ENFORCED
) WITH (
  'connector'='postgres-cdc','hostname'='192.168.88.41','port'='5432',
  'username'='cdc_zbx','password'='***',
  'database-name'='zabbix','schema-name'='public','table-name'='trends',
  'slot.name'='flink_zbx_trends','decoding.plugin.name'='pgoutput',
  'debezium.publication.name'='zbx_cdc_pub','debezium.publication.autocreate.mode'='disabled',
  'scan.incremental.snapshot.enabled'='true','scan.startup.mode'='initial'
);
CREATE TABLE snk_trends (
  itemid BIGINT, clock INT, num INT, value_min DOUBLE, value_avg DOUBLE, value_max DOUBLE,
  PRIMARY KEY (itemid, clock) NOT ENFORCED
) WITH (
  'connector'='jdbc',
  'url'='jdbc:mysql://10.10.5.205:2883/zabbix?useSSL=false&rewriteBatchedStatements=true&sessionVariables=foreign_key_checks=0',
  'driver'='com.mysql.cj.jdbc.Driver','username'='zabbix@zb_tenant','password'='***',
  'table-name'='trends','sink.buffer-flush.max-rows'='200','sink.buffer-flush.interval'='3s'
);
INSERT INTO snk_trends SELECT itemid,clock,num,value_min,value_avg,value_max FROM src_trends;

-- ===== trends_uint (bigint unsigned agg) =====
CREATE TABLE src_trends_uint (
  itemid BIGINT, clock INT, num INT, value_min BIGINT, value_avg BIGINT, value_max BIGINT,
  PRIMARY KEY (itemid, clock) NOT ENFORCED
) WITH (
  'connector'='postgres-cdc','hostname'='192.168.88.41','port'='5432',
  'username'='cdc_zbx','password'='***',
  'database-name'='zabbix','schema-name'='public','table-name'='trends_uint',
  'slot.name'='flink_zbx_trends_uint','decoding.plugin.name'='pgoutput',
  'debezium.publication.name'='zbx_cdc_pub','debezium.publication.autocreate.mode'='disabled',
  'debezium.slot.drop.on.stop'='false',
  'scan.incremental.snapshot.enabled'='true','scan.startup.mode'='initial'
);
CREATE TABLE snk_trends_uint (
  itemid BIGINT, clock INT, num INT, value_min BIGINT, value_avg BIGINT, value_max BIGINT,
  PRIMARY KEY (itemid, clock) NOT ENFORCED
) WITH (
  'connector'='jdbc',
  'url'='jdbc:mysql://10.10.5.205:2883/zabbix?useSSL=false&rewriteBatchedStatements=true&sessionVariables=foreign_key_checks=0',
  'driver'='com.mysql.cj.jdbc.Driver','username'='zabbix@zb_tenant','password'='***',
  'table-name'='trends_uint','sink.buffer-flush.max-rows'='200','sink.buffer-flush.interval'='3s'
);
INSERT INTO snk_trends_uint SELECT itemid,clock,num,value_min,value_avg,value_max FROM src_trends_uint;
```

</details>

### П. 10 и 11. Обеспечили устойчивость CDC, пережили сбой

Ничего хитрого — несколько настроек, без которых CDC не переживёт сбой:

- Checkpointing (interval=30s, EXACTLY_ONCE) — Flink фиксирует позицию в WAL; после сбоя возобновляется с чекпоинта, а не с нуля.
- Restart-strategy (fixed-delay, бесконечно) — без неё джоба умирает при первой ошибке; с ней сама перезапускается.
- max_slot_wal_keep_size на PostgreSQL — страховка от переполнения диска источника отставшим слотом.
- slot.drop.on.stop=false — чтобы при рестарте слот не пересоздавался и не терялись события.

Связка checkpointing + restart-strategy и даёт самовосстановление: не записал → чекпоинт не зафиксирован → слот не сдвинулся → после рестарта перечитал с той же точки. Дублей нет — натуральный PK делает повторную вставку безопасной.

Во время переезда пару раз пропадала связь (одна из причин переноса сервера), и Flink CDC даже в односерверной конфигурации отлично справился с восстановлением репликации.

### Финальные пункты, проверка и переезд хостов

После настройки репликации и докачки данных — а это было оставлено на ночь, и я даже не могу сказать, сколько времени реально заняло: канал всё равно медленный, так что данные не релевантны для пром-миграции — все метрики появились и начали обновляться.

Хосты, которые опрашивались с сервера Zabbix, начали опрашиваться с обоих серверов, но на короткое время это не страшно: их на старом сервере задизейбили первыми. Напомню, что таблицы конфигурации мы перенесли один раз, и далее через CDC изменения не реплицировались. Хосты, на которые локальный агент сам отправлял данные на сервер, переводили вручную, не спеша, меняя сервер в локальном конфиге.

В проме планируется оставить тот же сервер, просто изменив настройки базы.

## Подводные камни / особенности

Коллации: объекты БД (триггеры, процедуры), обращающиеся к строкам, падали с ERROR 1267 из-за расхождения коллации сессии (utf8mb4_general_ci) и БД (utf8mb4_bin); лечится установкой collation_connection перед выполнением.

Беззнаковые таблицы (history_uint, trends_uint): PostgreSQL хранит их как numeric, OceanBase — как bigint unsigned, а во Flink беззнакового типа нет, и мы маппили в BIGINT — для значений до 2^63 это работает, но на огромных счётчиках (>2^63) возможно переполнение, и там надёжнее DECIMAL(20,0) сквозь весь конвейер.

На тестовой инсталляции суммарный объём данных на диске сократился с 4232 MB до 427 MB — примерно в 10 раз. Этот средний показатель занижен спецификой стенда: значительную долю занимают мелкие конфигурационные таблицы, которые практически не сжимаются. Сами таблицы истории — history (2102 → 114 MB) и history_uint (1373 → 78 MB) — дали 17–18-кратное сокращение. Поскольку в продакшене доля исторических данных в общем объёме существенно выше, реалистично ожидать 12–15-кратного сокращения требуемого дискового пространства на реальной нагрузке.

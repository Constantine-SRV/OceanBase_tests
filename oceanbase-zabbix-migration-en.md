# OceanBase as a Database for Zabbix + Online Migration from PostgreSQL via Flink CDC. 15–18× Storage Savings

## A bit of backstory

Back in 2022 we deployed Zabbix for a client to monitor 10 MSSQL servers, and it ran on MySQL. Over time we kept attaching new hosts to it, and it slowly started to die — MySQL just isn't built for large volumes. The migration problem became pressing, and OceanBase happens to be something I've been working with very intensively lately.

I couldn't immediately talk the client into migrating the main Zabbix onto OceanBase, but a migration of the development Zabbix was proposed — it had to be moved into the target network segment anyway, and based on the results of that migration a decision will be made about whether to migrate the main Zabbix (which currently runs on MySQL). Naturally I agreed, and even though the development Zabbix was on PG, this case is actually even more interesting. I'd be very glad to hear feedback or advice before the production migration.

So, let's go.

Without dwelling on the advantages of OceanBase in general, let me highlight only what matters for Zabbix:

- Built-in compression that yields a 3–5× space saving over PostgreSQL even without the columnar format.
- Columnar storage on top of LSM trees, thanks to which write speed doesn't drop.
- MySQL syntax support.
- Real enterprise HA without extra components.
- A multitenant architecture is handy down the road, when you need to isolate resources for several projects on a single cluster.
- And all of this is already in the open-source CE edition.

Here's the main win — the size of the top-5 tables in PostgreSQL and OceanBase:

<details>
<summary>SQL queries and output: size of the top-5 tables (PostgreSQL and OceanBase)</summary>

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

## How the migration was planned

We decided to migrate online, taking our time, switching clients over from the old server to the new one one at a time. For the duration of the migration we agreed there would be no configuration changes, so most tables would move in a single one-off job, while the tables that receive metrics and their history would be migrated online via Flink CDC with no data loss. During the migration, hosts that stayed on the old Zabbix would keep writing to the old Postgres database, while Flink CDC would replicate that data from Postgres into OceanBase in real time. This way clients can be switched over gradually without losing metrics.

## Analyzing the Zabbix PostgreSQL schema before migration

The Zabbix schema splits into two classes of tables, and that shaped the migration strategy. The first — configuration and operational data (hosts, templates, items, triggers, events — ~178 tables): these are relatively static and were moved with a one-time batch snapshot. The second — metrics (history, history_uint, history_str, trends, trends_uint): a continuous high-frequency stream that can't be "frozen," so it was precisely these five tables that we moved online via CDC (initial snapshot + streaming).

The key thing for the migration is how Zabbix generates identifiers. Contrary to expectations, it barely uses the DBMS's AUTO_INCREMENT: business IDs are handed out by its own allocator — the `ids` table (triples of table_name, field_name, nextid, i.e. just numbers incremented in the application code). The only genuine AUTO_INCREMENT is the `changelogid` column in the service table `changelog` (the change-tracking mechanism), which we left empty. And the five metrics tables have no surrogate IDs at all — they have a natural primary key (itemid, clock, ns for history; itemid, clock for trends).

This turned out to be a lucky property: not depending on the DBMS's auto-increment removes the main problem of a multi-master scenario — ID collisions on concurrent writes across two nodes. The natural key of the metrics additionally makes re-inserting the same row safe, and the monotonic `clock` maps perfectly onto time-based partitioning in OceanBase.

## Migration step by step

1. Installed OceanBase CE 4.4.2 — cluster + OBProxy, created the tenant zb_tenant and the database zabbix (utf8mb4 is mandatory).
2. Installed Flink + Flink CDC — postgres-cdc and jdbc connectors, configured TaskManager slots.
3. Prepared the PostgreSQL source — wal_level=logical, a replication user, a publication for CDC.
4. Migrated the config tables (178 of them) — via a Flink JDBC batch (not CDC, since the data is static), verified row by row.
5. Moved the triggers (65 of them) — from a dump.
6. Worked around the DB version check — Zabbix requires MySQL ≥8.0.30, while OBServer reports 5.7.25 in the handshake; solved it via OBProxy (mysql_version=8.0.30) by connecting Zabbix through the proxy.
7. Installed a fresh zabbix-server 7.0 on a separate host, pointed it at OceanBase, brought up the web UI, verified that all hosts had moved over and there were no errors.
8. Designed history/trends for OceanBase — columnstore + RANGE partitions on clock, wrote a stored procedure + EVENT to auto-cut future partitions.
9. Migrated history/trends (33M rows) — via Flink CDC (initial snapshot + streaming).
10. Made the CDC resilient — checkpointing + restart-strategy + a WAL-retention limit.
11. Survived a real network outage — the jobs recovered from checkpoints with no data loss (a "real-life" stress test).
12. Moved the MSSQL ODBC setup (DSN + driver) — our Zabbix polls MSSQL directly.
13. Cutover — started the new server, switching clients over one at a time while the old PostgreSQL Zabbix keeps running in parallel.
14. Within two days all hosts were moved.

Writing up every point would get very long, and besides, Zabbix's data schema changes from version to version, so the scripts would be useless. My advice to anyone going down this path: take Opus 4.8 or higher along as a helper — it will handle all the steps just fine, especially if you feed it this article as input.

Let me only call out the points where there were problems, and how we solved them.

### Point 6. The version check

We installed the server on a standalone OceanBase node, without a cluster — expecting to connect Zabbix directly to OBServer (2881), without OBProxy. It didn't work: at startup Zabbix requires MySQL ≥8.0.30, while OBServer rigidly reports 5.7.25 in the handshake. AllowUnsupportedDBVersions=1 doesn't help (too old), and the version string in OBServer can't be changed. The fix — OBProxy with mysql_version=8.0.30: we connected Zabbix through the proxy (2883) and the check passed. Bottom line: the proxy had to be brought up even on a standalone setup — just for the sake of that single version spoof in the handshake.

### Point 8. Designing history/trends: columnstore + partitions

The five metrics tables were recreated for OceanBase: columnstore (columnar storage — compression + fast year-long analytics over trends) and RANGE partitions on clock (monthly, for a fast DROP PARTITION instead of the slow housekeeper DELETE).

The syntax for OB 4.4.2 was figured out empirically, with two subtleties: a partition boundary must be a plain integer literal (not UNIX_TIMESTAMP()), and WITH COLUMN GROUP must come strictly after PARTITION BY (otherwise ERROR 1064). Example for history:

<details>
<summary>history: CREATE TABLE + the partition auto-cutting procedure + EVENT</summary>

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

The built-in DYNAMIC_PARTITION_POLICY wasn't a fit — it only works for DATE/TIMESTAMP, whereas clock is an int (unix seconds). So we built the auto-cutting of future partitions ourselves: a stored procedure tops up monthly partitions ahead, up to "now + N months," and an EVENT calls it daily.

```sql
CREATE PROCEDURE sp_manage_partitions(IN p_months_ahead INT)
BEGIN
  ...
  SET time_zone = '+00:00';  -- clock = unix UTC
  SET v_target = UNIX_TIMESTAMP(DATE_ADD(DATE_FORMAT(NOW(),'%Y-%m-01'),
                                         INTERVAL p_months_ahead MONTH));
  -- for each table: read the max boundary, add months forward
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

The procedure must be created/called with collation_connection = utf8mb4_general_ci (otherwise ERROR 1267 on the comparison against information_schema), event_scheduler = ON can only be set as root, and the boundaries are computed strictly in UTC (time_zone='+00:00'), since clock is unix time.

</details>

Right now old metrics are deleted by the Zabbix housekeeper with ordinary DELETEs — it knows nothing about partitions. But partitioning opens the door to instant cleanup: by disabling the housekeeper and moving retention onto our own procedure, we'll be able to drop a whole month with a single DROP PARTITION instead of scanning millions of rows. And on a production cluster of three nodes, OceanBase will spread the partitions across the servers, and queries against history will run in parallel across nodes — i.e. faster. Auto-creating partitions is already done; faster deletion via DROP is the next step.

### Point 9. The dynamic migration via Flink CDC

The load was implemented declaratively in Flink SQL. For each of the five metrics tables — history, history_uint, history_str, trends, trends_uint — three statements: a source CREATE TABLE (the postgres-cdc connector, mode initial — snapshot + subsequent streaming), a sink CREATE TABLE (the jdbc connector into OceanBase through the proxy), and an INSERT INTO ... SELECT that launches the job.

A debatable approach, perhaps, but a fitting one for this case.

Here is the code itself:

<details>
<summary>Flink SQL: five table pairs (source/sink) + INSERT</summary>

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

### Points 10 and 11. Making CDC resilient, surviving an outage

Nothing fancy — just a few settings, without which CDC won't survive a failure:

- Checkpointing (interval=30s, EXACTLY_ONCE) — Flink records its position in the WAL; after a failure it resumes from the checkpoint rather than from scratch.
- Restart-strategy (fixed-delay, infinite) — without it the job dies on the first error; with it, it restarts itself.
- max_slot_wal_keep_size on PostgreSQL — insurance against the source disk filling up due to a lagging slot.
- slot.drop.on.stop=false — so the slot isn't recreated on restart and events aren't lost.

The combination of checkpointing + restart-strategy is what gives self-recovery: didn't write → checkpoint not committed → slot didn't advance → after restart it re-reads from the same point. No duplicates — the natural PK makes re-inserting safe.

During the move the connection dropped a couple of times (one of the reasons for relocating the server), and Flink CDC, even in a single-server configuration, handled replication recovery beautifully.

### Final points: verification and moving the hosts

After setting up replication and the data catch-up — which was left to run overnight, and I honestly can't even say how long it actually took: the channel is slow anyway, so the numbers aren't relevant for a production migration — all the metrics appeared and started updating.

The hosts that were polled from the Zabbix server began being polled from both servers, but for a short while that's harmless: they were disabled on the old server first. As a reminder, the config tables were moved once, and after that changes were not replicated via CDC. Hosts whose local agent pushed data to the server itself were switched manually, taking our time, changing the server in the local config.

In production the plan is to keep the same server and simply change the database settings.

## Gotchas / quirks

Collations: database objects (triggers, procedures) that touch strings were failing with ERROR 1267 due to a mismatch between the session collation (utf8mb4_general_ci) and the database one (utf8mb4_bin); the fix is to set collation_connection before execution.

Unsigned tables (history_uint, trends_uint): PostgreSQL stores them as numeric, OceanBase as bigint unsigned, and Flink has no unsigned type, so we mapped them to BIGINT — for values up to 2^63 this works, but on huge counters (>2^63) overflow is possible, and there it's safer to run DECIMAL(20,0) through the whole pipeline.

On the test installation the total on-disk data size shrank from 4232 MB to 427 MB — roughly 10×. This average is understated by the nature of the bench: a significant share is taken by small configuration tables that barely compress. The history tables themselves — history (2102 → 114 MB) and history_uint (1373 → 78 MB) — gave a 17–18× reduction. Since in production the share of historical data in the total volume is substantially higher, it's realistic to expect a 12–15× reduction in required disk space under a real workload.

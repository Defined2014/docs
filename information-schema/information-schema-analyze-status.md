---
title: ANALYZE_STATUS
summary: Learn the `ANALYZE_STATUS` information_schema table.
---

# ANALYZE_STATUS {#analyze-status}

`ANALYZE_STATUS`テーブルは、統計を収集する実行中のタスクと限られた数の履歴タスクに関する情報を提供します。

TiDB v6.1.0 以降、 `ANALYZE_STATUS`テーブルはクラスターレベルのタスクの表示をサポートしています。 TiDB の再起動後でも、このテーブルを使用して再起動前のタスク レコードを表示できます。 TiDB v6.1.0 より前では、 `ANALYZE_STATUS`テーブルはインスタンス レベルのタスクのみを表示でき、タスク レコードは TiDB の再起動後にクリアされます。

TiDB v6.1.0 から、システム テーブルを介して過去 7 日間の履歴タスクを表示できます`mysql.analyze_jobs` 。

{{< copyable "" >}}

```sql
USE information_schema;
DESC analyze_status;
```

```sql
+----------------+---------------------+------+------+---------+-------+
| Field          | Type                | Null | Key  | Default | Extra |
+----------------+---------------------+------+------+---------+-------+
| TABLE_SCHEMA   | varchar(64)         | YES  |      | NULL    |       |
| TABLE_NAME     | varchar(64)         | YES  |      | NULL    |       |
| PARTITION_NAME | varchar(64)         | YES  |      | NULL    |       |
| JOB_INFO       | longtext            | YES  |      | NULL    |       |
| PROCESSED_ROWS | bigint(64) unsigned | YES  |      | NULL    |       |
| START_TIME     | datetime            | YES  |      | NULL    |       |
| END_TIME       | datetime            | YES  |      | NULL    |       |
| STATE          | varchar(64)         | YES  |      | NULL    |       |
| FAIL_REASON    | longtext            | YES  |      | NULL    |       |
| INSTANCE       | varchar(512)        | YES  |      | NULL    |       |
| PROCESS_ID     | bigint(64) unsigned | YES  |      | NULL    |       |
+----------------+---------------------+------+------+---------+-------+
11 rows in set (0.00 sec)
```

{{< copyable "" >}}

```sql
SELECT * FROM information_schema.analyze_status;
```

```sql
+--------------+------------+----------------+--------------------------------------------------------------------+----------------+---------------------+---------------------+----------+-------------+----------------+------------+
| TABLE_SCHEMA | TABLE_NAME | PARTITION_NAME | JOB_INFO                                                           | PROCESSED_ROWS | START_TIME          | END_TIME            | STATE    | FAIL_REASON | INSTANCE       | PROCESS_ID |
+--------------+------------+----------------+--------------------------------------------------------------------+----------------+---------------------+---------------------+----------+-------------+----------------+------------+
| test         | t          | p1             | analyze table all columns with 256 buckets, 500 topn, 1 samplerate |              0 | 2022-05-27 11:30:12 | 2022-05-27 11:30:12 | finished | NULL        | 127.0.0.1:4000 |       NULL |
| test         | t          | p0             | analyze table all columns with 256 buckets, 500 topn, 1 samplerate |              0 | 2022-05-27 11:30:12 | 2022-05-27 11:30:12 | finished | NULL        | 127.0.0.1:4000 |       NULL |
| test         | t          | p1             | analyze index idx                                                  |              0 | 2022-05-27 11:29:46 | 2022-05-27 11:29:46 | finished | NULL        | 127.0.0.1:4000 |       NULL |
| test         | t          | p0             | analyze index idx                                                  |              0 | 2022-05-27 11:29:46 | 2022-05-27 11:29:46 | finished | NULL        | 127.0.0.1:4000 |       NULL |
| test         | t          | p1             | analyze columns                                                    |              0 | 2022-05-27 11:29:46 | 2022-05-27 11:29:46 | finished | NULL        | 127.0.0.1:4000 |       NULL |
| test         | t          | p0             | analyze columns                                                    |              0 | 2022-05-27 11:29:46 | 2022-05-27 11:29:46 | finished | NULL        | 127.0.0.1:4000 |       NULL |
+--------------+------------+----------------+--------------------------------------------------------------------+----------------+---------------------+---------------------+----------+-------------+----------------+------------+
6 rows in set (0.00 sec)
```

`ANALYZE_STATUS`テーブルのフィールドは次のとおりです。

-   `TABLE_SCHEMA` : テーブルが属するデータベースの名前。
-   `TABLE_NAME` : テーブルの名前。
-   `PARTITION_NAME` :パーティションテーブルの名前。
-   `JOB_INFO` : `ANALYZE`タスクの情報。インデックスが分析される場合、この情報にはインデックス名が含まれます。 `tidb_analyze_version =2`の場合、この情報にはサンプルレートなどの構成項目が含まれます。
-   `PROCESSED_ROWS` : 処理された行数。
-   `START_TIME` : `ANALYZE`タスクの開始時刻。
-   `END_TIME` : `ANALYZE`タスクの終了時刻。
-   `STATE` : `ANALYZE`タスクの実行ステータス。その値は`pending` 、 `running` 、 `finished`または`failed`です。
-   `FAIL_REASON` : タスクが失敗した理由。実行が成功した場合、値は`NULL`です。
-   `INSTANCE` : タスクを実行する TiDB インスタンス。
-   `PROCESS_ID` : タスクを実行するプロセス ID。

---
title: TIFLASH_REPLICA
summary: Learn the `TIFLASH_REPLICA` information_schema table.
---

# TIFLASH_REPLICA {#tiflash-replica}

表`TIFLASH_REPLICA`は、利用可能なTiFlashレプリカに関する情報を提供します。

{{< copyable "" >}}

```sql
USE information_schema;
DESC tiflash_replica;
```

```
+-----------------+-------------+------+------+---------+-------+
| Field           | Type        | Null | Key  | Default | Extra |
+-----------------+-------------+------+------+---------+-------+
| TABLE_SCHEMA    | varchar(64) | YES  |      | NULL    |       |
| TABLE_NAME      | varchar(64) | YES  |      | NULL    |       |
| TABLE_ID        | bigint(21)  | YES  |      | NULL    |       |
| REPLICA_COUNT   | bigint(64)  | YES  |      | NULL    |       |
| LOCATION_LABELS | varchar(64) | YES  |      | NULL    |       |
| AVAILABLE       | tinyint(1)  | YES  |      | NULL    |       |
| PROGRESS        | double      | YES  |      | NULL    |       |
+-----------------+-------------+------+------+---------+-------+
7 rows in set (0.01 sec)
```

`TIFLASH_REPLICA`テーブルのフィールドは次のとおりです。

-   `TABLE_SCHEMA` : テーブルが属するデータベースの名前。
-   `TABLE_NAME` : テーブルの名前。
-   `TABLE_ID` : テーブルの内部 ID。TiDB クラスター内で一意です。
-   `REPLICA_COUNT` : TiFlashレプリカの数。
-   `LOCATION_LABELS` : TiFlashレプリカの作成時に設定される LocationLabelList。
-   `AVAILABLE` : テーブルのTiFlashレプリカが使用可能かどうかを示します。値が`1` (使用可能) の場合、TiDB オプティマイザーは、クエリ コストに基づいて、クエリを TiKV またはTiFlashにプッシュ ダウンすることをインテリジェントに選択できます。値が`0` (使用不可) の場合、TiDB はクエリをTiFlashにプッシュしません。このフィールドの値が`1` (使用可能) になると、変更されなくなります。
-   `PROGRESS` : TiFlashレプリカのレプリケーションの進行状況。精度は小数点以下 2 桁、分レベルです。このフィールドのスコープは`[0, 1]`です。 `AVAILABLE`フィールドが`1`で`PROGRESS`が 1 未満の場合、 TiFlashレプリカは TiKV よりもはるかに遅れており、 TiFlashにプッシュされたクエリは、データ レプリケーションの待機のタイムアウトにより失敗する可能性があります。

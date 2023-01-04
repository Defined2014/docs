---
title: DROP TABLE | TiDB SQL Statement Reference
summary: An overview of the usage of DROP TABLE for the TiDB database.
---

# ドロップテーブル {#drop-table}

このステートメントは、現在選択されているデータベースからテーブルを削除します。 `IF EXISTS`修飾子が使用されていない限り、テーブルが存在しない場合はエラーが返されます。

## あらすじ {#synopsis}

```ebnf+diagram
DropTableStmt ::=
    'DROP' OptTemporary TableOrTables IfExists TableNameList RestrictOrCascadeOpt

OptTemporary ::=
    ( 'TEMPORARY' | ('GLOBAL' 'TEMPORARY') )?

TableOrTables ::=
    'TABLE'
|   'TABLES'

TableNameList ::=
    TableName ( ',' TableName )*
```

## 一時テーブルを削除する {#drop-temporary-tables}

次の構文を使用して、通常のテーブルと一時テーブルを削除できます。

-   ローカル一時テーブルを削除するには、 `DROP TEMPORARY TABLE`を使用します。
-   グローバル一時テーブルを削除するには、 `DROP GLOBAL TEMPORARY TABLE`を使用します。
-   通常のテーブルまたは一時テーブルを削除するには、 `DROP TABLE`を使用します。

## 例 {#examples}

```sql
mysql> CREATE TABLE t1 (a INT);
Query OK, 0 rows affected (0.11 sec)

mysql> DROP TABLE t1;
Query OK, 0 rows affected (0.22 sec)

mysql> DROP TABLE table_not_exists;
ERROR 1051 (42S02): Unknown table 'test.table_not_exists'
mysql> DROP TABLE IF EXISTS table_not_exists;
Query OK, 0 rows affected (0.01 sec)

mysql> CREATE VIEW v1 AS SELECT 1;
Query OK, 0 rows affected (0.10 sec)

mysql> DROP TABLE v1;
Query OK, 0 rows affected (0.23 sec)
```

## MySQL の互換性 {#mysql-compatibility}

-   存在しないテーブルを削除しようとしても、 `IF EXISTS`を指定してテーブルを削除しても警告は返されません。 [問題＃7867](https://github.com/pingcap/tidb/issues/7867)
-   現在、 `RESTRICT`と`CASCADE`は構文的にのみサポートされています。

## こちらもご覧ください {#see-also}

-   [テーブルを作成](/sql-statements/sql-statement-create-table.md)
-   [テーブルの作成を表示](/sql-statements/sql-statement-show-create-table.md)
-   [テーブルを表示](/sql-statements/sql-statement-show-tables.md)

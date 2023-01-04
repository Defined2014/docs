---
title: SHOW CREATE DATABASE
summary: An overview of the use of SHOW CREATE DATABASE in the TiDB database.
---

# データベースの作成を表示 {#show-create-database}

`SHOW CREATE DATABASE`は、既存のデータベースを再作成するための正確な SQL ステートメントを示すために使用されます。 `SHOW CREATE SCHEMA`は同義語です。

## あらすじ {#synopsis}

**ShowCreateDatabaseStmt:**

```ebnf+diagram
ShowCreateDatabaseStmt ::=
    "SHOW" "CREATE" "DATABASE" | "SCHEMA" ("IF" "NOT" "EXISTS")? DBName
```

## 例 {#examples}

```sql
CREATE DATABASE test;
```

```sql
Query OK, 0 rows affected (0.12 sec)
```

```sql
SHOW CREATE DATABASE test;
```

```sql
+----------+------------------------------------------------------------------+
| Database | Create Database                                                  |
+----------+------------------------------------------------------------------+
| test     | CREATE DATABASE `test` /*!40100 DEFAULT CHARACTER SET utf8mb4 */ |
+----------+------------------------------------------------------------------+
1 row in set (0.00 sec)
```

```sql
SHOW CREATE SCHEMA IF NOT EXISTS test;
```

```sql
+----------+-------------------------------------------------------------------------------------------+
| Database | Create Database                                                                           |
+----------+-------------------------------------------------------------------------------------------+
| test     | CREATE DATABASE /*!32312 IF NOT EXISTS*/ `test` /*!40100 DEFAULT CHARACTER SET utf8mb4 */ |
+----------+-------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

## MySQL の互換性 {#mysql-compatibility}

`SHOW CREATE DATABASE`は、MySQL と完全に互換性があると予想されます。互換性の違いが見つかった場合は、GitHub [問題](https://github.com/pingcap/tidb/issues/new/choose)を送信してください。

## こちらもご覧ください {#see-also}

-   [テーブルを作成](/sql-statements/sql-statement-create-table.md)
-   [ドロップテーブル](/sql-statements/sql-statement-drop-table.md)
-   [テーブルを表示](/sql-statements/sql-statement-show-tables.md)
-   [次の列を表示](/sql-statements/sql-statement-show-columns-from.md)

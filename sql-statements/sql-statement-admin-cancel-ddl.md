---
title: ADMIN CANCEL DDL | TiDB SQL Statement Reference
summary: An overview of the usage of ADMIN CANCEL DDL for the TiDB database.
category: reference
---

# 管理者キャンセル DDL {#admin-cancel-ddl}

`ADMIN CANCEL DDL`ステートメントを使用すると、実行中の DDL ジョブをキャンセルできます。 `job_id`は[`ADMIN SHOW DDL JOBS`](/sql-statements/sql-statement-admin-show-ddl.md)を実行することで見つけることができます。

## あらすじ {#synopsis}

```ebnf+diagram
AdminStmt ::=
    'ADMIN' ( 'SHOW' ( 'DDL' ( 'JOBS' Int64Num? WhereClauseOptional | 'JOB' 'QUERIES' NumList )? | TableName 'NEXT_ROW_ID' | 'SLOW' AdminShowSlow ) | 'CHECK' ( 'TABLE' TableNameList | 'INDEX' TableName Identifier ( HandleRange ( ',' HandleRange )* )? ) | 'RECOVER' 'INDEX' TableName Identifier | 'CLEANUP' ( 'INDEX' TableName Identifier | 'TABLE' 'LOCK' TableNameList ) | 'CHECKSUM' 'TABLE' TableNameList | 'CANCEL' 'DDL' 'JOBS' NumList | 'RELOAD' ( 'EXPR_PUSHDOWN_BLACKLIST' | 'OPT_RULE_BLACKLIST' | 'BINDINGS' ) | 'PLUGINS' ( 'ENABLE' | 'DISABLE' ) PluginNameList | 'REPAIR' 'TABLE' TableName CreateTableStmt | ( 'FLUSH' | 'CAPTURE' | 'EVOLVE' ) 'BINDINGS' )

NumList ::=
    Int64Num ( ',' Int64Num )*
```

## 例 {#examples}

現在実行中の DDL ジョブをキャンセルし、対応するジョブが正常にキャンセルされたかどうかを返すには、 `ADMIN CANCEL DDL JOBS`を使用します。

{{< copyable "" >}}

```sql
ADMIN CANCEL DDL JOBS job_id [, job_id] ...;
```

操作でジョブをキャンセルできなかった場合は、具体的な理由が表示されます。

> **ノート：**
>
> -   この操作のみが DDL ジョブをキャンセルできます。他のすべての操作と環境の変更 (マシンの再起動やクラスターの再起動など) では、これらのジョブをキャンセルできません。
> -   この操作では、複数の DDL ジョブを同時にキャンセルできます。 [`ADMIN SHOW DDL JOBS`](/sql-statements/sql-statement-admin-show-ddl.md)ステートメントを使用して、DDL ジョブの ID を取得できます。
> -   キャンセルするジョブが終了している場合、キャンセル操作は失敗します。

## MySQL の互換性 {#mysql-compatibility}

このステートメントは、MySQL 構文に対する TiDB 拡張です。

## こちらもご覧ください {#see-also}

-   [`ADMIN SHOW DDL [JOBS|QUERIES]`](/sql-statements/sql-statement-admin-show-ddl.md)

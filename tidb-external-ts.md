---
title: Read Historical Data Using the `tidb_external_ts` Variable
summary: Learn how to read historical data using the `tidb_external_ts` variable.
---

# <code>tidb_external_ts</code>変数を使用して履歴データを読み取る {#read-historical-data-using-the-code-tidb-external-ts-code-variable}

履歴データの読み取りをサポートするために、TiDB v6.4.0 ではシステム変数[`tidb_external_ts`](/system-variables.md#tidb_external_ts-new-in-v640)が導入されています。このドキュメントでは、このシステム変数を使用して履歴データを読み取る方法について、詳細な使用例を含めて説明します。

> **警告：**
>
> 現在、 Stale ステイル読み取りをTiFlashと一緒に使用することはできません。クエリで[`tidb_enable_external_ts_read`](/system-variables.md#tidb_enable_external_ts_read-new-in-v640)を有効にし、TiDB がTiFlashレプリカからデータを読み取る可能性がある場合、エラー メッセージ`ERROR 1105 (HY000): stale requests require tikv backend`が発生する可能性があります。
>
> この問題を解決するには、次の操作のいずれかを実行して、Stale ステイル読み取りクエリのTiFlashレプリカを無効にします。
>
> -   `tidb_isolation_read_engines`変数を使用して、 TiFlashレプリカを無効にします: `SET SESSION tidb_isolation_read_engines='tidb,tikv'` 。
> -   ヒント[READ_FROM_STORAGE](/optimizer-hints.md#read_from_storagetiflasht1_name--tl_name--tikvt2_name--tl_name-)を使用して、TiDB が TiKV からデータを読み取るように強制します。

## シナリオ {#scenarios}

指定した時点からの履歴データの読み取りは、TiCDC などのデータ複製ツールに非常に役立ちます。データ複製ツールが特定の時点より前にデータの複製を完了した後、ダウンストリーム TiDB の`tidb_external_ts`システム変数を設定して、その時点より前のデータを読み取ることができます。これにより、データ複製によるデータの不整合が防止されます。

## 機能説明 {#feature-description}

システム変数[`tidb_external_ts`](/system-variables.md#tidb_external_ts-new-in-v640)は、 `tidb_enable_external_ts_read`が有効な場合に読み取られる履歴データのタイムスタンプを指定します。

システム変数[`tidb_enable_external_ts_read`](/system-variables.md#tidb_enable_external_ts_read-new-in-v640)は、現在のセッションまたはグローバルのどちらで履歴データを読み取るかを制御します。デフォルト値は`OFF`です。これは、履歴データの読み取り機能が無効になっていることを意味し、値`tidb_external_ts`は無視されます。 `tidb_enable_external_ts_read`がグローバルに`ON`に設定されている場合、すべてのクエリは`tidb_external_ts`で指定された時間より前の履歴データを読み取ります。特定のセッションに対してのみ`tidb_enable_external_ts_read`が`ON`に設定されている場合、そのセッションのクエリのみが履歴データを読み取ります。

`tidb_enable_external_ts_read`を有効にすると、TiDB は読み取り専用になります。すべての書き込みクエリは`ERROR 1836 (HY000): Running in read-only mode`のようなエラーで失敗します。

## 使用例 {#usage-examples}

このセクションでは、 `tidb_external_ts`変数を使用して履歴データを読み取る方法を例とともに説明します。

1.  テーブルを作成し、いくつかの行をテーブルに挿入します。

    ```sql
    CREATE TABLE t (c INT);
    ```

    ```
    Query OK, 0 rows affected (0.01 sec)
    ```

    ```sql
    INSERT INTO t VALUES (1), (2), (3);
    ```

    ```
    Query OK, 3 rows affected (0.00 sec)
    ```

2.  テーブル内のデータをビューします。

    ```sql
    SELECT * FROM t;
    ```

    ```
    +------+
    | c    |
    +------+
    |    1 |
    |    2 |
    |    3 |
    +------+
    3 rows in set (0.00 sec)
    ```

3.  `tidb_external_ts` ～ `@@tidb_current_ts`を設定:

    ```sql
    START TRANSACTION;
    SET GLOBAL tidb_external_ts=@@tidb_current_ts;
    COMMIT;
    ```

4.  新しい行を挿入し、挿入されたことを確認します。

    ```sql
    INSERT INTO t VALUES (4);
    ```

    ```
    Query OK, 1 row affected (0.001 sec)
    ```

    ```sql
    SELECT * FROM t;
    ```

    ```
    +------+
    | id   |
    +------+
    |    1 |
    |    2 |
    |    3 |
    |    4 |
    +------+
    4 rows in set (0.00 sec)
    ```

5.  `tidb_enable_external_ts_read`から`ON`を設定してから、テーブルのデータを表示します。

    ```sql
    SET tidb_enable_external_ts_read=ON;
    SELECT * FROM t;
    ```

    ```
    +------+
    | c    |
    +------+
    |    1 |
    |    2 |
    |    3 |
    +------+
    3 rows in set (0.00 sec)
    ```

    新しい行が挿入される前にタイムスタンプに`tidb_external_ts`が設定されるため、 `tidb_enable_external_ts_read`が有効になった後、新しく挿入された行は返されません。

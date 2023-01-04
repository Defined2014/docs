---
title: CLIENT_ERRORS_SUMMARY_BY_HOST
summary: Learn about the `CLIENT_ERRORS_SUMMARY_BY_HOST` information_schema table.
---

# CLIENT_ERRORS_SUMMARY_BY_HOST {#client-errors-summary-by-host}

表`CLIENT_ERRORS_SUMMARY_BY_HOST`は、TiDBサーバーに接続するクライアントに返された SQL エラーと警告の概要を示しています。これらには以下が含まれます：

-   不正な SQL ステートメント。
-   ゼロ エラーによる除算。
-   範囲外または重複するキー値を挿入しようとしています。
-   許可エラー。
-   存在しないテーブル。

これらのエラーは、MySQLサーバープロトコルを介してクライアントに返され、そこでアプリケーションは適切なアクションを実行することが期待されます。 `information_schema` 。表`CLIENT_ERRORS_SUMMARY_BY_HOST`は、アプリケーションが TiDBサーバーから返されたエラーを正しく処理 (またはログ記録) していないシナリオで、エラーを検査するための便利な方法を提供します。

`CLIENT_ERRORS_SUMMARY_BY_HOST`はリモート ホストごとにエラーを要約しているため、1 つのアプリケーションサーバーが他のサーバーよりも多くのエラーを生成しているシナリオを診断するのに役立ちます。考えられるシナリオは次のとおりです。

-   古い MySQL クライアント ライブラリ。
-   古いアプリケーション (おそらく、このサーバーは、新しい展開をロールアウトするときに見落とされたものです)。
-   ユーザー権限の「ホスト」部分の不適切な使用。
-   より多くのタイムアウトまたは切断された接続を生成する信頼性の低いネットワーク接続。

集計されたカウントは、ステートメント`FLUSH CLIENT_ERRORS_SUMMARY`を使用してリセットできます。要約は各 TiDBサーバーにローカルであり、メモリにのみ保持されます。 TiDBサーバーが再起動すると、サマリーは失われます。

{{< copyable "" >}}

```sql
USE information_schema;
DESC CLIENT_ERRORS_SUMMARY_BY_HOST;
```

```sql
+---------------+---------------+------+------+---------+-------+
| Field         | Type          | Null | Key  | Default | Extra |
+---------------+---------------+------+------+---------+-------+
| HOST          | varchar(255)  | NO   |      | NULL    |       |
| ERROR_NUMBER  | bigint(64)    | NO   |      | NULL    |       |
| ERROR_MESSAGE | varchar(1024) | NO   |      | NULL    |       |
| ERROR_COUNT   | bigint(64)    | NO   |      | NULL    |       |
| WARNING_COUNT | bigint(64)    | NO   |      | NULL    |       |
| FIRST_SEEN    | timestamp     | YES  |      | NULL    |       |
| LAST_SEEN     | timestamp     | YES  |      | NULL    |       |
+---------------+---------------+------+------+---------+-------+
7 rows in set (0.00 sec)
```

フィールドの説明:

-   `HOST` : クライアントのリモート ホスト。
-   `ERROR_NUMBER` : 返された MySQL 互換のエラー番号。
-   `ERROR_MESSAGE` : エラー番号に一致するエラー メッセージ (プリペアドステートメント形式)。
-   `ERROR_COUNT` : このエラーがクライアント ホストに返された回数。
-   `WARNING_COUNT` : この警告がクライアント ホストに返された回数。
-   `FIRST_SEEN` : このエラー (または警告) がクライアント ホストから初めて見られた時刻。
-   `LAST_SEEN` : このエラー (または警告) がクライアント ホストで最後に確認された時刻。

次の例は、クライアントがローカル TiDBサーバーに接続するときに生成される警告を示しています。サマリーは`FLUSH CLIENT_ERRORS_SUMMARY`を実行した後にリセットされます:

{{< copyable "" >}}

```sql
SELECT 0/0;
SELECT * FROM CLIENT_ERRORS_SUMMARY_BY_HOST;
FLUSH CLIENT_ERRORS_SUMMARY;
SELECT * FROM CLIENT_ERRORS_SUMMARY_BY_HOST;
```

```sql
+-----+
| 0/0 |
+-----+
| NULL |
+-----+
1 row in set, 1 warning (0.00 sec)

+-----------+--------------+---------------+-------------+---------------+---------------------+---------------------+
| HOST      | ERROR_NUMBER | ERROR_MESSAGE | ERROR_COUNT | WARNING_COUNT | FIRST_SEEN          | LAST_SEEN           |
+-----------+--------------+---------------+-------------+---------------+---------------------+---------------------+
| 127.0.0.1 |         1365 | Division by 0 |           0 |             1 | 2021-03-18 12:51:54 | 2021-03-18 12:51:54 |
+-----------+--------------+---------------+-------------+---------------+---------------------+---------------------+
1 row in set (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Empty set (0.00 sec)
```

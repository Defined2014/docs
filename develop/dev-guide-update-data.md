---
title: Update Data
summary: Learn about how to update data and batch update data.
---

# データの更新 {#update-data}

このドキュメントでは、次の SQL ステートメントを使用して、さまざまなプログラミング言語で TiDB のデータを更新する方法について説明します。

-   [アップデート](/sql-statements/sql-statement-update.md) : 指定されたテーブルのデータを変更するために使用されます。
-   [重複キーの更新時に挿入](/sql-statements/sql-statement-insert.md) : データを挿入し、主キーまたは一意のキーの競合がある場合にこのデータを更新するために使用されます。複数の一意のキー (主キーを含む) がある場合、このステートメントを使用することは**お勧め**しません。これは、一意のキー (主キーを含む) の競合が検出されると、このステートメントがデータを更新するためです。複数の行の競合がある場合、1 つの行のみを更新します。

## 始める前に {#before-you-start}

このドキュメントを読む前に、次の準備が必要です。

-   [TiDB Cloud(サーバーレス層) で TiDBクラスタを構築する](/develop/dev-guide-build-cluster-in-cloud.md) .
-   [スキーマ設計の概要](/develop/dev-guide-schema-design-overview.md) 、 [データベースを作成する](/develop/dev-guide-create-database.md) 、 [テーブルを作成する](/develop/dev-guide-create-table.md) 、および[セカンダリ インデックスの作成](/develop/dev-guide-create-secondary-indexes.md)を読んでください。
-   データを`UPDATE`つにしたい場合は、まず[データを挿入する](/develop/dev-guide-insert-data.md)にする必要があります。

## <code>UPDATE</code>を使用する {#use-code-update-code}

テーブル内の既存の行を更新するには、 [`UPDATE`ステートメント](/sql-statements/sql-statement-update.md)句と`WHERE`句を使用して、更新する列をフィルター処理する必要があります。

> **ノート：**
>
> 多数の行 (たとえば、1 万行以上) を更新する必要がある場合は、一度に完全な***更新***を行うのではなく、すべての行が更新されるまで、一度に一部ずつ繰り返し更新することをお勧めします。この操作をループするスクリプトまたはプログラムを作成できます。詳細は[一括更新](#bulk-update)を参照してください。

### <code>UPDATE</code> SQL 構文 {#code-update-code-sql-syntax}

SQL では、 `UPDATE`ステートメントは通常、次の形式になります。

{{< copyable "" >}}

```sql
UPDATE {table} SET {update_column} = {update_value} WHERE {filter_column} = {filter_value}
```

|       パラメータ名      |       説明       |
| :---------------: | :------------: |
|     `{table}`     |      テーブル名     |
| `{update_column}` |    更新するカラム名    |
|  `{update_value}` |    更新するカラムの値   |
| `{filter_column}` | フィルターに一致するカラム名 |
|  `{filter_value}` | フィルタに一致するカラムの値 |

詳細については、 [UPDATE 構文](/sql-statements/sql-statement-update.md)を参照してください。

### <code>UPDATE</code>のベスト プラクティス {#code-update-code-best-practices}

以下は、データを更新するためのいくつかのベスト プラクティスです。

-   `UPDATE`文には必ず`WHERE`節を指定してください。 `UPDATE`ステートメントに`WHERE`句がない場合、TiDB はテーブル内の***すべて***の行を更新します。

<CustomContent platform="tidb">

-   多数の行 (たとえば、1 万行以上) を更新する必要がある場合は、 [一括更新](#bulk-update)を使用します。 TiDB は 1 つのトランザクションのサイズを制限しているため (デフォルトでは[txn-合計サイズ制限](/tidb-configuration-file.md#txn-total-size-limit) MB)、一度に多くのデータを更新すると、ロックが長時間保持されたり ( [悲観的取引](/pessimistic-transaction.md) )、競合が発生したりします ( [楽観的取引](/optimistic-transaction.md) )。

</CustomContent>

<CustomContent platform="tidb-cloud">

-   多数の行 (たとえば、1 万行以上) を更新する必要がある場合は、 [一括更新](#bulk-update)を使用します。 TiDB はデフォルトで 1 つのトランザクションのサイズを 100 MB に制限しているため、一度に多くのデータ更新を行うと、ロックが長時間保持されたり ( [悲観的取引](/pessimistic-transaction.md) )、競合が発生したりします ( [楽観的取引](/optimistic-transaction.md) )。

</CustomContent>

### <code>UPDATE</code>例 {#code-update-code-example}

著者が名前を**Helen Haruki**に変更したとします。 [著者](/develop/dev-guide-bookshop-schema-design.md#authors-table)テーブルを変更する必要があります。彼女の一意の`id`が<strong>1</strong>で、フィルターが`id = 1`であるとします。

<SimpleTab groupId="language">
<div label="SQL" value="sql">

{{< copyable "" >}}

```sql
UPDATE `authors` SET `name` = "Helen Haruki" WHERE `id` = 1;
```

</div>

<div label="Java" value="java">

{{< copyable "" >}}

```java
// ds is an entity of com.mysql.cj.jdbc.MysqlDataSource
try (Connection connection = ds.getConnection()) {
    PreparedStatement pstmt = connection.prepareStatement("UPDATE `authors` SET `name` = ? WHERE `id` = ?");
    pstmt.setString(1, "Helen Haruki");
    pstmt.setInt(2, 1);
    pstmt.executeUpdate();
} catch (SQLException e) {
    e.printStackTrace();
}
```

</div>
</SimpleTab>

## <code>INSERT ON DUPLICATE KEY UPDATE</code>を使用する {#use-code-insert-on-duplicate-key-update-code}

新しいデータをテーブルに挿入する必要があるが、一意のキー (主キーも一意のキー) が競合している場合、最初に競合したレコードが更新されます。 `INSERT ... ON DUPLICATE KEY UPDATE ...`のステートメントを使用して挿入または更新できます。

### <code>INSERT ON DUPLICATE KEY UPDATE</code> SQL 構文 {#code-insert-on-duplicate-key-update-code-sql-syntax}

SQL では、 `INSERT ... ON DUPLICATE KEY UPDATE ...`ステートメントは通常、次の形式になります。

{{< copyable "" >}}

```sql
INSERT INTO {table} ({columns}) VALUES ({values})
    ON DUPLICATE KEY UPDATE {update_column} = {update_value};
```

|       パラメータ名      |     説明    |
| :---------------: | :-------: |
|     `{table}`     |   テーブル名   |
|    `{columns}`    |  挿入するカラム名 |
|     `{values}`    | 挿入するカラムの値 |
| `{update_column}` |  更新するカラム名 |
|  `{update_value}` | 更新するカラムの値 |

### <code>INSERT ON DUPLICATE KEY UPDATE</code>のベスト プラクティス {#code-insert-on-duplicate-key-update-code-best-practices}

-   一意のキーが 1 つのテーブルにのみ`INSERT ON DUPLICATE KEY UPDATE`を使用します。 ***UNIQUE KEY*** (主キーを含む) の競合が検出された場合、このステートメントはデータを更新します。複数行の競合がある場合は、1 行のみが更新されます。したがって、競合する行が 1 つだけであることを保証できない限り、複数の一意のキーを持つテーブルで`INSERT ON DUPLICATE KEY UPDATE`ステートメントを使用することはお勧めしません。
-   データを作成または更新するときに、このステートメントを使用します。

### <code>INSERT ON DUPLICATE KEY UPDATE</code>例 {#code-insert-on-duplicate-key-update-code-example}

たとえば、書籍に対するユーザーの評価を含めるように[評価](/develop/dev-guide-bookshop-schema-design.md#ratings-table)テーブルを更新する必要があります。ユーザーがまだ書籍を評価していない場合は、新しい評価が作成されます。ユーザーが既に評価している場合、以前の評価が更新されます。

次の例では、主キーは`book_id`と`user_id`の結合主キーです。ユーザ`user_id = 1`は本`book_id = 1000`に`5`の評価を与える。

<SimpleTab groupId="language">
<div label="SQL" value="sql">

{{< copyable "" >}}

```sql
INSERT INTO `ratings`
    (`book_id`, `user_id`, `score`, `rated_at`)
VALUES
    (1000, 1, 5, NOW())
ON DUPLICATE KEY UPDATE `score` = 5, `rated_at` = NOW();
```

</div>

<div label="Java" value="java">

{{< copyable "" >}}

```java
// ds is an entity of com.mysql.cj.jdbc.MysqlDataSource

try (Connection connection = ds.getConnection()) {
    PreparedStatement p = connection.prepareStatement("INSERT INTO `ratings` (`book_id`, `user_id`, `score`, `rated_at`)
VALUES (?, ?, ?, NOW()) ON DUPLICATE KEY UPDATE `score` = ?, `rated_at` = NOW()");
    p.setInt(1, 1000);
    p.setInt(2, 1);
    p.setInt(3, 5);
    p.setInt(4, 5);
    p.executeUpdate();
} catch (SQLException e) {
    e.printStackTrace();
}
```

</div>
</SimpleTab>

## 一括更新 {#bulk-update}

テーブル内の複数行のデータを更新する必要がある場合、 [`INSERT ON DUPLICATE KEY UPDATE`を使用する](#use-insert-on-duplicate-key-update)と`WHERE`節を使用して、更新が必要なデータをフィルター処理できます。

<CustomContent platform="tidb">

ただし、多数の行 (たとえば、1 万行以上) を更新する必要がある場合は、データを繰り返し更新することをお勧めします。つまり、更新が完了するまで、各繰り返しでデータの一部のみを更新します。 .これは、TiDB が[txn-合計サイズ制限](/tidb-configuration-file.md#txn-total-size-limit)つのトランザクションのサイズを制限しているためです (デフォルトでは 1、100 MB)。一度に多くのデータを更新すると、ロックが長時間保持される ( [悲観的取引](/pessimistic-transaction.md) 、または競合が発生する ( [楽観的取引](/optimistic-transaction.md) ) ) ことになります。プログラムまたはスクリプトでループを使用して、操作を完了することができます。

</CustomContent>

<CustomContent platform="tidb-cloud">

ただし、多数の行 (たとえば、1 万行以上) を更新する必要がある場合は、データを繰り返し更新することをお勧めします。つまり、更新が完了するまで、各繰り返しでデータの一部のみを更新します。 .これは、TiDB がデフォルトで 1 つのトランザクションのサイズを 100 MB に制限しているためです。一度に多くのデータを更新すると、ロックが長時間保持される ( [悲観的取引](/pessimistic-transaction.md) 、または競合が発生する ( [楽観的取引](/optimistic-transaction.md) ) ) ことになります。プログラムまたはスクリプトでループを使用して、操作を完了することができます。

</CustomContent>

このセクションでは、反復的な更新を処理するスクリプトの記述例を示します。この例は、一括更新を完了するために`SELECT`と`UPDATE`の組み合わせを実行する方法を示しています。

### 一括更新ループを書く {#write-bulk-update-loop}

まず、アプリケーションまたはスクリプトのループに`SELECT`クエリを記述する必要があります。このクエリの戻り値は、更新が必要な行の主キーとして使用できます。この`SELECT`クエリを定義するときは、 `WHERE`句を使用して、更新が必要な行をフィルタリングする必要があることに注意してください。

### 例 {#example}

過去 1 年間に`bookshop`の Web サイトで多くのユーザーから本の評価があったとしますが、元のデザインの 5 段階評価では、本の評価に違いがありませんでした。ほとんどの本の評価は`3`です。評価を差別化するために、5 点満点から 10 点満点に切り替えることにしました。

前の 5 ポイント スケールの`ratings`テーブルのデータに`2`を掛け、評価テーブルに新しい列を追加して、行が更新されたかどうかを示す必要があります。この列を使用すると、 `SELECT`で更新された行をフィルターで除外できます。これにより、スクリプトがクラッシュして複数回行を更新し、不合理なデータが生成されるのを防ぐことができます。

たとえば、10 ポイント スケールかどうかの識別子としてデータ型[ブール](/data-type-numeric.md#boolean-type)を使用して、 `ten_point`という名前の列を作成します。

{{< copyable "" >}}

```sql
ALTER TABLE `bookshop`.`ratings` ADD COLUMN `ten_point` BOOL NOT NULL DEFAULT FALSE;
```

> **ノート：**
>
> この一括更新アプリケーションは、 **DDL**ステートメントを使用して、データ テーブルにスキーマの変更を加えます。 TiDB のすべての DDL 変更操作はオンラインで実行されます。詳細については、 [列を追加](/sql-statements/sql-statement-add-column.md)を参照してください。

<SimpleTab groupId="language">
<div label="Golang" value="golang">

Golangでは、一括更新アプリケーションは次のようになります。

```go
package main

import (
    "database/sql"
    "fmt"
    _ "github.com/go-sql-driver/mysql"
    "strings"
    "time"
)

func main() {
    db, err := sql.Open("mysql", "root:@tcp(127.0.0.1:4000)/bookshop")
    if err != nil {
        panic(err)
    }
    defer db.Close()

    bookID, userID := updateBatch(db, true, 0, 0)
    fmt.Println("first time batch update success")
    for {
        time.Sleep(time.Second)
        bookID, userID = updateBatch(db, false, bookID, userID)
        fmt.Printf("batch update success, [bookID] %d, [userID] %d\n", bookID, userID)
    }
}

// updateBatch select at most 1000 lines data to update score
func updateBatch(db *sql.DB, firstTime bool, lastBookID, lastUserID int64) (bookID, userID int64) {
    // select at most 1000 primary keys in five-point scale data
    var err error
    var rows *sql.Rows

    if firstTime {
        rows, err = db.Query("SELECT `book_id`, `user_id` FROM `bookshop`.`ratings` " +
            "WHERE `ten_point` != true ORDER BY `book_id`, `user_id` LIMIT 1000")
    } else {
        rows, err = db.Query("SELECT `book_id`, `user_id` FROM `bookshop`.`ratings` "+
            "WHERE `ten_point` != true AND `book_id` > ? AND `user_id` > ? "+
            "ORDER BY `book_id`, `user_id` LIMIT 1000", lastBookID, lastUserID)
    }

    if err != nil || rows == nil {
        panic(fmt.Errorf("error occurred or rows nil: %+v", err))
    }

    // joint all id with a list
    var idList []interface{}
    for rows.Next() {
        var tempBookID, tempUserID int64
        if err := rows.Scan(&tempBookID, &tempUserID); err != nil {
            panic(err)
        }
        idList = append(idList, tempBookID, tempUserID)
        bookID, userID = tempBookID, tempUserID
    }

    bulkUpdateSql := fmt.Sprintf("UPDATE `bookshop`.`ratings` SET `ten_point` = true, "+
        "`score` = `score` * 2 WHERE (`book_id`, `user_id`) IN (%s)", placeHolder(len(idList)))
    db.Exec(bulkUpdateSql, idList...)

    return bookID, userID
}

// placeHolder format SQL place holder
func placeHolder(n int) string {
    holderList := make([]string, n/2, n/2)
    for i := range holderList {
        holderList[i] = "(?,?)"
    }
    return strings.Join(holderList, ",")
}
```

各反復で、主キーの順序で`SELECT`のクエリ。 10 ポイント スケール ( `ten_point`は`false` ) に更新されていない最大`1000`行の主キー値を選択します。各`SELECT`ステートメントは、重複を防ぐために、前の`SELECT`個の結果の最大のものよりも大きい主キーを選択します。次に、一括更新を使用し、 `score`列を`2`で乗算し、 `ten_point`を`true`に設定します。 `ten_point`を更新する目的は、クラッシュ後の再起動時に、更新アプリケーションが同じ行を繰り返し更新するのを防ぐことです。これにより、データが破損する可能性があります。各ループで`time.Sleep(time.Second)`を指定すると、更新アプリケーションが 1 秒間一時停止し、更新アプリケーションがハードウェア リソースを消費しすぎるのを防ぎます。

</div>

<div label="Java (JDBC)" value="jdbc">

Java (JDBC) では、一括更新アプリケーションは次のようになります。

**コード：**

{{< copyable "" >}}

```java
package com.pingcap.bulkUpdate;

import com.mysql.cj.jdbc.MysqlDataSource;

import java.sql.*;
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.TimeUnit;

public class BatchUpdateExample {
    static class UpdateID {
        private Long bookID;
        private Long userID;

        public UpdateID(Long bookID, Long userID) {
            this.bookID = bookID;
            this.userID = userID;
        }

        public Long getBookID() {
            return bookID;
        }

        public void setBookID(Long bookID) {
            this.bookID = bookID;
        }

        public Long getUserID() {
            return userID;
        }

        public void setUserID(Long userID) {
            this.userID = userID;
        }

        @Override
        public String toString() {
            return "[bookID] " + bookID + ", [userID] " + userID ;
        }
    }

    public static void main(String[] args) throws InterruptedException {
        // Configure the example database connection.

        // Create a mysql data source instance.
        MysqlDataSource mysqlDataSource = new MysqlDataSource();

        // Set server name, port, database name, username and password.
        mysqlDataSource.setServerName("localhost");
        mysqlDataSource.setPortNumber(4000);
        mysqlDataSource.setDatabaseName("bookshop");
        mysqlDataSource.setUser("root");
        mysqlDataSource.setPassword("");

        UpdateID lastID = batchUpdate(mysqlDataSource, null);

        System.out.println("first time batch update success");
        while (true) {
            TimeUnit.SECONDS.sleep(1);
            lastID = batchUpdate(mysqlDataSource, lastID);
            System.out.println("batch update success, [lastID] " + lastID);
        }
    }

    public static UpdateID batchUpdate (MysqlDataSource ds, UpdateID lastID) {
        try (Connection connection = ds.getConnection()) {
            UpdateID updateID = null;

            PreparedStatement selectPs;

            if (lastID == null) {
                selectPs = connection.prepareStatement(
                        "SELECT `book_id`, `user_id` FROM `bookshop`.`ratings` " +
                        "WHERE `ten_point` != true ORDER BY `book_id`, `user_id` LIMIT 1000");
            } else {
                selectPs = connection.prepareStatement(
                        "SELECT `book_id`, `user_id` FROM `bookshop`.`ratings` "+
                            "WHERE `ten_point` != true AND `book_id` > ? AND `user_id` > ? "+
                            "ORDER BY `book_id`, `user_id` LIMIT 1000");

                selectPs.setLong(1, lastID.getBookID());
                selectPs.setLong(2, lastID.getUserID());
            }

            List<Long> idList = new LinkedList<>();
            ResultSet res = selectPs.executeQuery();
            while (res.next()) {
                updateID = new UpdateID(
                        res.getLong("book_id"),
                        res.getLong("user_id")
                );
                idList.add(updateID.getBookID());
                idList.add(updateID.getUserID());
            }

            if (idList.isEmpty()) {
                System.out.println("no data should update");
                return null;
            }

            String updateSQL = "UPDATE `bookshop`.`ratings` SET `ten_point` = true, "+
                    "`score` = `score` * 2 WHERE (`book_id`, `user_id`) IN (" +
                    placeHolder(idList.size() / 2) + ")";
            PreparedStatement updatePs = connection.prepareStatement(updateSQL);
            for (int i = 0; i < idList.size(); i++) {
                updatePs.setLong(i + 1, idList.get(i));
            }
            int count = updatePs.executeUpdate();
            System.out.println("update " + count + " data");

            return updateID;
        } catch (SQLException e) {
            e.printStackTrace();
        }

        return null;
    }

    public static String placeHolder(int n) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < n ; i++) {
            sb.append(i == 0 ? "(?,?)" : ",(?,?)");
        }

        return sb.toString();
    }
}
```

-   `hibernate.cfg.xml`構成:

{{< copyable "" >}}

```xml
<?xml version='1.0' encoding='utf-8'?>
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>

        <!-- Database connection settings -->
        <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
        <property name="hibernate.dialect">org.hibernate.dialect.TiDBDialect</property>
        <property name="hibernate.connection.url">jdbc:mysql://localhost:4000/movie</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password"></property>
        <property name="hibernate.connection.autocommit">false</property>
        <property name="hibernate.jdbc.batch_size">20</property>

        <!-- Optional: Show SQL output for debugging -->
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>
    </session-factory>
</hibernate-configuration>
```

各反復で、主キーの順序で`SELECT`のクエリ。 10 ポイント スケール ( `ten_point`は`false` ) に更新されていない最大`1000`行の主キー値を選択します。各`SELECT`ステートメントは、重複を防ぐために、前の`SELECT`個の結果の最大のものよりも大きい主キーを選択します。次に、一括更新を使用し、 `score`列を`2`で乗算し、 `ten_point`を`true`に設定します。 `ten_point`を更新する目的は、クラッシュ後の再起動時に、更新アプリケーションが同じ行を繰り返し更新するのを防ぐことです。これにより、データが破損する可能性があります。各ループで`TimeUnit.SECONDS.sleep(1);`を指定すると、更新アプリケーションが 1 秒間一時停止し、更新アプリケーションがハードウェア リソースを消費しすぎるのを防ぎます。

</div>

</SimpleTab>

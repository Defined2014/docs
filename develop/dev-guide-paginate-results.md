---
title: Paginate Results
summary: Introduce paginate result feature in TiDB.
---

# 結果のページ付け {#paginate-results}

大きなクエリ結果をページングするには、「ページ分割」された方法で目的の部分を取得できます。

## クエリ結果のページ分割 {#paginate-query-results}

TiDB では、 `LIMIT`ステートメントを使用してクエリ結果をページ分割できます。例えば：

{{< copyable "" >}}

```sql
SELECT * FROM table_a t ORDER BY gmt_modified DESC LIMIT offset, row_count;
```

`offset`はレコードの開始数を示し、 `row_count`はページあたりのレコード数を示します。 TiDB は`LIMIT row_count OFFSET offset`の構文もサポートしています。

ページネーションを使用する場合、データをランダムに表示する必要がない限り、クエリ結果を`ORDER BY`ステートメントで並べ替えることが推奨されます。

<SimpleTab groupId="language">
<div label="SQL" value="sql">

たとえば、 [書店](/develop/dev-guide-bookshop-schema-design.md)アプリケーションのユーザーが最新の出版された書籍をページ分割された方法で表示できるようにするには、 `LIMIT 0, 10`ステートメントを使用できます。このステートメントは、結果リストの最初のページを返します。1 ページあたり最大 10 レコードです。 2 ページ目を取得するには、ステートメントを`LIMIT 10, 10`に変更します。

{{< copyable "" >}}

```sql
SELECT *
FROM books
ORDER BY published_at DESC
LIMIT 0, 10;
```

</div>
<div label="Java" value="java">

アプリケーション開発では、バックエンド プログラムは、 `offset`パラメーターの代わりに、 `page_number`パラメーター (要求されているページの数を意味する) と`page_size`パラメーター (ページあたりのレコード数を制御する) をフロントエンドから受け取ります。したがって、クエリを実行する前にいくつかの変換を行う必要がありました。

{{< copyable "" >}}

```java
public List<Book> getLatestBooksPage(Long pageNumber, Long pageSize) throws SQLException {
    pageNumber = pageNumber < 1L ? 1L : pageNumber;
    pageSize = pageSize < 10L ? 10L : pageSize;
    Long offset = (pageNumber - 1) * pageSize;
    Long limit = pageSize;
    List<Book> books = new ArrayList<>();
    try (Connection conn = ds.getConnection()) {
        PreparedStatement stmt = conn.prepareStatement("""
        SELECT id, title, published_at
        FROM books
        ORDER BY published_at DESC
        LIMIT ?, ?;
        """);
        stmt.setLong(1, offset);
        stmt.setLong(2, limit);
        ResultSet rs = stmt.executeQuery();
        while (rs.next()) {
            Book book = new Book();
            book.setId(rs.getLong("id"));
            book.setTitle(rs.getString("title"));
            book.setPublishedAt(rs.getDate("published_at"));
            books.add(book);
        }
    }
    return books;
}
```

</div>
</SimpleTab>

## 単一フィールド主キー テーブルのページング バッチ {#paging-batches-for-single-field-primary-key-tables}

通常、主キーまたは一意のインデックスを使用して結果を並べ替え、 `LIMIT`句の`offset`キーワードを使用して指定した行数でページを分割するページネーション SQL ステートメントを記述できます。その後、ページは独立したトランザクションにラップされ、柔軟なページング更新が実現されます。ただし、欠点も明らかです。主キーまたは一意のインデックスを並べ替える必要があるため、オフセットが大きいと、特に大量のデータの場合に、より多くのコンピューティング リソースが消費されます。

以下は、より効率的なページング バッチ処理方法を紹介します。

<SimpleTab groupId="language">
<div label="SQL" value="sql">

まず、データを主キーでソートし、ウィンドウ関数`row_number()`を呼び出して、各行の行番号を生成します。次に、集計関数を呼び出して、指定されたページ サイズで行番号をグループ化し、各ページの最小値と最大値を計算します。

{{< copyable "" >}}

```sql
SELECT
    floor((t.row_num - 1) / 1000) + 1 AS page_num,
    min(t.id) AS start_key,
    max(t.id) AS end_key,
    count(*) AS page_size
FROM (
    SELECT id, row_number() OVER (ORDER BY id) AS row_num
    FROM books
) t
GROUP BY page_num
ORDER BY page_num;
```

結果は次のとおりです。

```
+----------+------------+------------+-----------+
| page_num | start_key  | end_key    | page_size |
+----------+------------+------------+-----------+
|        1 |     268996 |  213168525 |      1000 |
|        2 |  213210359 |  430012226 |      1000 |
|        3 |  430137681 |  647846033 |      1000 |
|        4 |  647998334 |  848878952 |      1000 |
|        5 |  848899254 | 1040978080 |      1000 |
...
|       20 | 4077418867 | 4294004213 |      1000 |
+----------+------------+------------+-----------+
20 rows in set (0.01 sec)
```

次に、 `WHERE id BETWEEN start_key AND end_key`ステートメントを使用して、各スライスのデータをクエリします。データをより効率的に更新するために、データを変更するときに上記のスライス情報を使用できます。

ページ 1 のすべての書籍の基本情報を削除するには、上記の結果の`start_key`と`end_key`をページ 1 の値に置き換えます。

{{< copyable "" >}}

```sql
DELETE FROM books
WHERE
    id BETWEEN 268996 AND 213168525
ORDER BY id;
```

</div>
<div label="Java" value="java">

Javaで、ページのメタ情報を格納する`PageMeta`クラスを定義します。

{{< copyable "" >}}

```java
public class PageMeta<K> {
    private Long pageNum;
    private K startKey;
    private K endKey;
    private Long pageSize;

    // Skip the getters and setters.

}
```

ページのメタ情報一覧を取得するメソッドを`getPageMetaList()`つ定義し、ページのメタ情報に従ってデータを一括削除するメソッドを`deleteBooksByPageMeta()`定義します。

{{< copyable "" >}}

```java
public class BookDAO {
    public List<PageMeta<Long>> getPageMetaList() throws SQLException {
        List<PageMeta<Long>> pageMetaList = new ArrayList<>();
        try (Connection conn = ds.getConnection()) {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("""
            SELECT
                floor((t.row_num - 1) / 1000) + 1 AS page_num,
                min(t.id) AS start_key,
                max(t.id) AS end_key,
                count(*) AS page_size
            FROM (
                SELECT id, row_number() OVER (ORDER BY id) AS row_num
                FROM books
            ) t
            GROUP BY page_num
            ORDER BY page_num;
            """);
            while (rs.next()) {
                PageMeta<Long> pageMeta = new PageMeta<>();
                pageMeta.setPageNum(rs.getLong("page_num"));
                pageMeta.setStartKey(rs.getLong("start_key"));
                pageMeta.setEndKey(rs.getLong("end_key"));
                pageMeta.setPageSize(rs.getLong("page_size"));
                pageMetaList.add(pageMeta);
            }
        }
        return pageMetaList;
    }

    public void deleteBooksByPageMeta(PageMeta<Long> pageMeta) throws SQLException {
        try (Connection conn = ds.getConnection()) {
            PreparedStatement stmt = conn.prepareStatement("DELETE FROM books WHERE id >= ? AND id <= ?");
            stmt.setLong(1, pageMeta.getStartKey());
            stmt.setLong(2, pageMeta.getEndKey());
            stmt.executeUpdate();
        }
    }
}
```

次のステートメントは、ページ 1 のデータを削除します。

{{< copyable "" >}}

```java
List<PageMeta<Long>> pageMetaList = bookDAO.getPageMetaList();
if (pageMetaList.size() > 0) {
    bookDAO.deleteBooksByPageMeta(pageMetaList.get(0));
}
```

次のステートメントは、ページングによってすべてのブック データをバッチで削除します。

{{< copyable "" >}}

```java
List<PageMeta<Long>> pageMetaList = bookDAO.getPageMetaList();
pageMetaList.forEach((pageMeta) -> {
    try {
        bookDAO.deleteBooksByPageMeta(pageMeta);
    } catch (SQLException e) {
        e.printStackTrace();
    }
});
```

</div>
</SimpleTab>

この方法は、頻繁なデータの並べ替え操作によるコンピューティング リソースの浪費を回避することで、バッチ処理の効率を大幅に向上させます。

## 複合主キー テーブルのページング バッチ {#paging-batches-for-composite-primary-key-tables}

### 非クラスター化インデックス テーブル {#non-clustered-index-table}

非クラスター化インデックス テーブル (「非インデックス構成テーブル」とも呼ばれます) の場合、内部フィールド`_tidb_rowid`をページ付けキーとして使用できます。ページ付け方法は、単一フィールド主キー テーブルの場合と同じです。

> **ヒント：**
>
> `SHOW CREATE TABLE users;`ステートメントを使用して、テーブルの主キーが[クラスター化インデックス](/clustered-indexes.md)を使用しているかどうかを確認できます。

例えば：

{{< copyable "" >}}

```sql
SELECT
    floor((t.row_num - 1) / 1000) + 1 AS page_num,
    min(t._tidb_rowid) AS start_key,
    max(t._tidb_rowid) AS end_key,
    count(*) AS page_size
FROM (
    SELECT _tidb_rowid, row_number () OVER (ORDER BY _tidb_rowid) AS row_num
    FROM users
) t
GROUP BY page_num
ORDER BY page_num;
```

結果は次のとおりです。

```
+----------+-----------+---------+-----------+
| page_num | start_key | end_key | page_size |
+----------+-----------+---------+-----------+
|        1 |         1 |    1000 |      1000 |
|        2 |      1001 |    2000 |      1000 |
|        3 |      2001 |    3000 |      1000 |
|        4 |      3001 |    4000 |      1000 |
|        5 |      4001 |    5000 |      1000 |
|        6 |      5001 |    6000 |      1000 |
|        7 |      6001 |    7000 |      1000 |
|        8 |      7001 |    8000 |      1000 |
|        9 |      8001 |    9000 |      1000 |
|       10 |      9001 |    9990 |       990 |
+----------+-----------+---------+-----------+
10 rows in set (0.00 sec)
```

### クラスタ化インデックス テーブル {#clustered-index-table}

クラスター化されたインデックス テーブル (&quot;インデックス構成テーブル&quot; とも呼ばれます) の場合、 `concat`関数を使用して複数の列の値をキーとして連結し、ウィンドウ関数を使用してページング情報をクエリできます。

現時点では、キーは文字列であり、 `min`と`max`の集計関数によってスライス内の正しい`start_key`と`end_key`を取得するには、文字列の長さが常に同じであることを確認する必要があることに注意してください。文字列連結のフィールドの長さが固定されていない場合は、 `LPAD`関数を使用して埋め込むことができます。

たとえば、次のように`ratings`テーブルのデータのページング バッチを実装できます。

次のステートメントを使用して、メタ情報テーブルを作成します。 `bigint`種類の`book_id`と`user_id`で連結されたキーは同じ長さに変換できないため、 `LPAD`関数を使用して、 `bigint`の最大ビット 19 に従って長さを`0`でパディングします。

{{< copyable "" >}}

```sql
SELECT
    floor((t1.row_num - 1) / 10000) + 1 AS page_num,
    min(mvalue) AS start_key,
    max(mvalue) AS end_key,
    count(*) AS page_size
FROM (
    SELECT
        concat('(', LPAD(book_id, 19, 0), ',', LPAD(user_id, 19, 0), ')') AS mvalue,
        row_number() OVER (ORDER BY book_id, user_id) AS row_num
    FROM ratings
) t1
GROUP BY page_num
ORDER BY page_num;
```

> **ノート：**
>
> 前の SQL ステートメントは`TableFullScan`として実行されます。データ量が多いとクエリが遅くなるため、高速化することができ[TiFlashを使う](/tiflash/tiflash-overview.md#use-tiflash) 。

結果は次のとおりです。

```
+----------+-------------------------------------------+-------------------------------------------+-----------+
| page_num | start_key                                 | end_key                                   | page_size |
+----------+-------------------------------------------+-------------------------------------------+-----------+
|        1 | (0000000000000268996,0000000000092104804) | (0000000000140982742,0000000000374645100) |     10000 |
|        2 | (0000000000140982742,0000000000456757551) | (0000000000287195082,0000000004053200550) |     10000 |
|        3 | (0000000000287196791,0000000000191962769) | (0000000000434010216,0000000000237646714) |     10000 |
|        4 | (0000000000434010216,0000000000375066168) | (0000000000578893327,0000000002167504460) |     10000 |
|        5 | (0000000000578893327,0000000002457322286) | (0000000000718287668,0000000001502744628) |     10000 |
...
|       29 | (0000000004002523918,0000000000902930986) | (0000000004147203315,0000000004090920746) |     10000 |
|       30 | (0000000004147421329,0000000000319181561) | (0000000004294004213,0000000003586311166) |      9972 |
+----------+-------------------------------------------+-------------------------------------------+-----------+
30 rows in set (0.28 sec)
```

ページ 1 のすべての評価レコードを削除するには、上記の結果の`start_key`と`end_key`をページ 1 の値に置き換えます。

{{< copyable "" >}}

```sql
SELECT * FROM ratings
WHERE
    (book_id > 268996 AND book_id < 140982742)
    OR (
        book_id = 268996 AND user_id >= 92104804
    )
    OR (
        book_id = 140982742 AND user_id <= 374645100
    )
ORDER BY book_id, user_id;
```

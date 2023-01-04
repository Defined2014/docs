---
title: Predicates Push Down
summary: Introduce one of the TiDB's logic optimization rules—Predicate Push Down (PPD).
---

# Predicate Push Down(PPD) {#predicates-push-down-ppd}

このドキュメントでは、TiDB のロジック最適化ルールの 1 つである Predicate Push Down (PPD) を紹介します。述語のプッシュ ダウンを理解し、その適用可能なシナリオと適用できないシナリオを理解できるようにすることを目的としています。

PPD は、データ フィルタリングをできるだけ早く完了するために、選択演算子をデータ ソースのできるだけ近くにプッシュ ダウンします。これにより、データ送信または計算のコストが大幅に削減されます。

## 例 {#examples}

次のケースでは、PPD の最適化について説明します。ケース 1、2、および 3 は PPD が適用されるシナリオであり、ケース 4、5、および 6 は PPD が適用されないシナリオです。

### ケース 1: 述語をストレージレイヤーにプッシュする {#case-1-push-predicates-to-storage-layer}

```sql
create table t(id int primary key, a int);
explain select * from t where a < 1;
+-------------------------+----------+-----------+---------------+--------------------------------+
| id                      | estRows  | task      | access object | operator info                  |
+-------------------------+----------+-----------+---------------+--------------------------------+
| TableReader_7           | 3323.33  | root      |               | data:Selection_6               |
| └─Selection_6           | 3323.33  | cop[tikv] |               | lt(test.t.a, 1)                |
|   └─TableFullScan_5     | 10000.00 | cop[tikv] | table:t       | keep order:false, stats:pseudo |
+-------------------------+----------+-----------+---------------+--------------------------------+
3 rows in set (0.00 sec)
```

このクエリでは、述語`a < 1`を TiKVレイヤーにプッシュ ダウンしてデータをフィルター処理することで、ネットワーク送信のオーバーヘッドを削減できます。

### ケース 2: 述語をストレージレイヤーにプッシュする {#case-2-push-predicates-to-storage-layer}

```sql
create table t(id int primary key, a int not null);
explain select * from t where a < substring('123', 1, 1);
+-------------------------+----------+-----------+---------------+--------------------------------+
| id                      | estRows  | task      | access object | operator info                  |
+-------------------------+----------+-----------+---------------+--------------------------------+
| TableReader_7           | 3323.33  | root      |               | data:Selection_6               |
| └─Selection_6           | 3323.33  | cop[tikv] |               | lt(test.t.a, 1)                |
|   └─TableFullScan_5     | 10000.00 | cop[tikv] | table:t       | keep order:false, stats:pseudo |
+-------------------------+----------+-----------+---------------+--------------------------------+
```

このクエリは、述語`a < substring('123', 1, 1)`の`substring`の入力パラメーターが定数であるため、ケース 1 のクエリと同じ実行プランを持ち、事前に計算することができます。次に、述語は同等の述語`a < 1`に簡略化されます。その後、TiDB は`a < 1`を TiKV にプッシュできます。

### ケース 3: 結合演算子の下に述語をプッシュする {#case-3-push-predicates-below-join-operator}

```sql
create table t(id int primary key, a int not null);
create table s(id int primary key, a int not null);
explain select * from t join s on t.a = s.a where t.a < 1;
+------------------------------+----------+-----------+---------------+--------------------------------------------+
| id                           | estRows  | task      | access object | operator info                              |
+------------------------------+----------+-----------+---------------+--------------------------------------------+
| HashJoin_8                   | 4154.17  | root      |               | inner join, equal:[eq(test.t.a, test.s.a)] |
| ├─TableReader_15(Build)      | 3323.33  | root      |               | data:Selection_14                          |
| │ └─Selection_14             | 3323.33  | cop[tikv] |               | lt(test.s.a, 1)                            |
| │   └─TableFullScan_13       | 10000.00 | cop[tikv] | table:s       | keep order:false, stats:pseudo             |
| └─TableReader_12(Probe)      | 3323.33  | root      |               | data:Selection_11                          |
|   └─Selection_11             | 3323.33  | cop[tikv] |               | lt(test.t.a, 1)                            |
|     └─TableFullScan_10       | 10000.00 | cop[tikv] | table:t       | keep order:false, stats:pseudo             |
+------------------------------+----------+-----------+---------------+--------------------------------------------+
7 rows in set (0.00 sec)
```

このクエリでは、述語`t.a < 1`を join の下にプッシュして事前にフィルター処理することで、join の計算オーバーヘッドを削減できます。

また、このSQL文は内部結合が実行されており、 `ON`条件は`t.a = s.a`です。述語`s.a <1`は`t.a < 1`から派生し、結合演算子の下の`s`テーブルにプッシュダウンできます。 `s`テーブルをフィルタリングすると、結合の計算オーバーヘッドをさらに削減できます。

### ケース 4: ストレージ レイヤーでサポートされていない述語をプッシュ ダウンできない {#case-4-predicates-that-are-not-supported-by-storage-layers-cannot-be-pushed-down}

```sql
create table t(id int primary key, a int not null);
desc select * from t where substring('123', a, 1) = '1';
+-------------------------+---------+-----------+---------------+----------------------------------------+
| id                      | estRows | task      | access object | operator info                          |
+-------------------------+---------+-----------+---------------+----------------------------------------+
| Selection_7             | 2.00    | root      |               | eq(substring("123", test.t.a, 1), "1") |
| └─TableReader_6         | 2.00    | root      |               | data:TableFullScan_5                   |
|   └─TableFullScan_5     | 2.00    | cop[tikv] | table:t       | keep order:false, stats:pseudo         |
+-------------------------+---------+-----------+---------------+----------------------------------------+
```

このクエリには、述語`substring('123', a, 1) = '1'`があります。

`explain`の結果から、述語が計算のために TiKV にプッシュ ダウンされていないことがわかります。これは、TiKV コプロセッサが組み込み関数`substring`をサポートしていないためです。

### ケース 5: 外部結合の内部テーブルの述語をプッシュ ダウンできない {#case-5-predicates-of-inner-tables-on-the-outer-join-can-t-be-pushed-down}

```sql
create table t(id int primary key, a int not null);
create table s(id int primary key, a int not null);
explain select * from t left join s on t.a = s.a where s.a is null;
+-------------------------------+----------+-----------+---------------+-------------------------------------------------+
| id                            | estRows  | task      | access object | operator info                                   |
+-------------------------------+----------+-----------+---------------+-------------------------------------------------+
| Selection_7                   | 10000.00 | root      |               | isnull(test.s.a)                                |
| └─HashJoin_8                  | 12500.00 | root      |               | left outer join, equal:[eq(test.t.a, test.s.a)] |
|   ├─TableReader_13(Build)     | 10000.00 | root      |               | data:TableFullScan_12                           |
|   │ └─TableFullScan_12        | 10000.00 | cop[tikv] | table:s       | keep order:false, stats:pseudo                  |
|   └─TableReader_11(Probe)     | 10000.00 | root      |               | data:TableFullScan_10                           |
|     └─TableFullScan_10        | 10000.00 | cop[tikv] | table:t       | keep order:false, stats:pseudo                  |
+-------------------------------+----------+-----------+---------------+-------------------------------------------------+
6 rows in set (0.00 sec)
```

この問合せでは、内部表`s`に述語`s.a is null`があります。

`explain`の結果から、述語が結合演算子の下にプッシュされていないことがわかります。これは、 `on`の条件が満たされない場合、外部結合によって内部テーブルに`NULL`の値が入力され、結合後の結果をフィルター処理するために述語`s.a is null`が使用されるためです。結合の下の内部テーブルにプッシュ ダウンされると、実行計画は元の実行計画と同等ではなくなります。

### ケース 6: ユーザー変数を含む述語をプッシュダウンできない {#case-6-the-predicates-which-contain-user-variables-cannot-be-pushed-down}

```sql
create table t(id int primary key, a char);
set @a = 1;
explain select * from t where a < @a;
+-------------------------+----------+-----------+---------------+--------------------------------+
| id                      | estRows  | task      | access object | operator info                  |
+-------------------------+----------+-----------+---------------+--------------------------------+
| Selection_5             | 8000.00  | root      |               | lt(test.t.a, getvar("a"))      |
| └─TableReader_7         | 10000.00 | root      |               | data:TableFullScan_6           |
|   └─TableFullScan_6     | 10000.00 | cop[tikv] | table:t       | keep order:false, stats:pseudo |
+-------------------------+----------+-----------+---------------+--------------------------------+
3 rows in set (0.00 sec)
```

このクエリでは、テーブル`t`に述語`a < @a`があります。述語の`@a`はユーザー変数です。

`explain`の結果からわかるように、述語はケース 2 のようではなく、 `a < 1`に単純化され、TiKV にプッシュダウンされます。これは、ユーザー変数`@a`の値が計算中に変更される可能性があり、TiKV が変更を認識しないためです。そのため、TiDB は`@a`を`1`に置き換えたり、TiKV にプッシュしたりしません。

理解に役立つ例は次のとおりです。

```sql
create table t(id int primary key, a int);
insert into t values(1, 1), (2,2);
set @a = 1;
select id, a, @a:=@a+1 from t where a = @a;
+----+------+----------+
| id | a    | @a:=@a+1 |
+----+------+----------+
|  1 |    1 | 2        |
|  2 |    2 | 3        |
+----+------+----------+
2 rows in set (0.00 sec)
```

このクエリからわかるように、 `@a`の値はクエリ中に変化します。したがって、 `a = @a`を`a = 1`に置き換えて TiKV にプッシュすると、同等の実行計画にはなりません。

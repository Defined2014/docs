---
title: Latency Breakdown
summary: Introduce more details about TiDB latency and how to analyze latency in real use cases.
---

# レイテンシの内訳 {#latency-breakdown}

このドキュメントでは、レイテンシーをメトリクスに分解し、ユーザーの観点から次の側面から分析します。

-   [一般的な SQLレイヤー](#general-sql-layer)
-   [クエリを読む](#read-queries)
-   [クエリを書く](#write-queries)
-   [バッチ クライアント](#batch-client)
-   [TiKV スナップショット](#tikv-snapshot)
-   [非同期書き込み](#async-write)

これらの分析により、 TiDB SQLクエリ中の時間コストに関する深い洞察が得られます。 TiDBのクリティカルパス診断のご案内です。また、第[診断のユースケース](#diagnosis-use-cases)セクションでは、実際のユース ケースでレイテンシーを分析する方法を紹介します。

このドキュメントの前に[パフォーマンス分析とチューニング](/performance-tuning-methods.md)を読んだほうがよいでしょう。レイテンシーをメトリクスに分割する場合、特定の低速クエリではなく、期間またはレイテンシーの平均値が計算されることに注意してください。多くのメトリックは、期間またはレイテンシーの分布であるヒストグラムとして表示されます。平均レイテンシーを計算するには、次の合計およびカウント カウンターを使用する必要があります。

```
avg = ${metric_name}_sum / ${metric_name}_count
```

このドキュメントで説明されているメトリクスは、TiDB の Prometheus ダッシュボードから直接読み取ることができます。

## 一般的な SQLレイヤー {#general-sql-layer}

この一般的な SQLレイヤーのレイテンシーは TiDB の最上位に存在し、すべての SQL クエリで共有されます。以下は、一般的な SQLレイヤー操作の時間コスト図です。

```railroad+diagram
Diagram(
    NonTerminal("Token wait duration"),
    Choice(
        0,
        Comment("Prepared statement"),
        NonTerminal("Parse duration"),
    ),
    OneOrMore(
        Sequence(
        Choice(
            0,
            NonTerminal("Optimize prepared plan duration"),
            Sequence(
            Comment("Plan cache miss"),
            NonTerminal("Compile duration"),
            ),
        ),
        NonTerminal("TSO wait duration"),
        NonTerminal("Execution duration"),
        ),
        Comment("Retry"),
    ),
)
```

一般的な SQLレイヤーのレイテンシーは`e2e duration`のメトリックとして観察でき、次のように計算されます。

```text
e2e duration =
    tidb_server_get_token_duration_seconds +
    tidb_session_parse_duration_seconds +
    tidb_session_compile_duration_seconds +
    tidb_session_execute_duration_seconds{type="general"}
```

-   `tidb_server_get_token_duration_seconds`は、トークン待機の期間を記録します。これは通常 1 ミリ秒未満であり、無視できるほど小さいです。
-   `tidb_session_parse_duration_seconds`は、SQL クエリを抽象構文木 (AST) に解析する期間を記録します。これは[`PREPARE/EXECUTE`ステートメント](/develop/dev-guide-optimize-sql-best-practices.md#use-prepare)でスキップできます。
-   `tidb_session_compile_duration_seconds`は、AST を実行計画にコンパイルする期間を記録します。これは[SQL準備実行計画キャッシュ](/sql-prepared-plan-cache.md)でスキップできます。
-   `tidb_session_execute_duration_seconds{type="general"}`は実行期間を記録し、すべてのタイプのユーザー クエリが混在します。パフォーマンスの問題やボトルネックを分析するには、これを細かい期間に分割する必要があります。

一般に、OLTP (オンライン トランザクション処理) のワークロードは、いくつかの重要なコードを共有する読み取りクエリと書き込みクエリに分けることができます。次のセクションでは、実行方法が異なる[クエリを読む](#read-queries)と[クエリを書く](#write-queries)のレイテンシーについて説明します。

## クエリを読む {#read-queries}

読み取りクエリには、単一のプロセス フォームしかありません。

### ポイントゲット {#point-get}

以下は、 [ポイントゲット](/glossary.md#point-get)操作の時間コスト図です。

```railroad+diagram
Diagram(
    Choice(
        0,
        NonTerminal("Resolve TSO"),
        Comment("Read by clustered PK in auto-commit-txn mode or snapshot read"),
    ),
    Choice(
        0,
        NonTerminal("Read handle by index key"),
        Comment("Read by clustered PK, encode handle by key"),
    ),
    NonTerminal("Read value by handle"),
)
```

ポイント取得中、 `tidb_session_execute_duration_seconds{type="general"}`期間は次のように計算されます。

```text
tidb_session_execute_duration_seconds{type="general"} =
    pd_client_cmd_handle_cmds_duration_seconds{type="wait"} +
    read handle duration +
    read value duration
```

`pd_client_cmd_handle_cmds_duration_seconds{type="wait"}`は、PD から[TSO (タイムスタンプ Oracle)](/glossary.md#tso)をフェッチする期間を記録します。クラスター化されたプライマリ インデックスを使用して、またはスナップショットから自動コミット トランザクション モードで読み取る場合、値はゼロになります。

`read handle duration`と`read value duration`は次のように計算されます。

```text
read handle duration = read value duration =
    tidb_tikvclient_txn_cmd_duration_seconds{type="get"} =
    send request duration =
    tidb_tikvclient_request_seconds{type="Get"} =
    tidb_tikvclient_batch_wait_duration +
    tidb_tikvclient_batch_send_latency +
    tikv_grpc_msg_duration_seconds{type="kv_get"} +
    tidb_tikvclient_rpc_net_latency_seconds{store="?"}
```

`tidb_tikvclient_request_seconds{type="Get"}`は、バッチ化された gRPC ラッパーを介して TiKV に直接送信される取得リクエストの期間を記録します。 `tidb_tikvclient_batch_wait_duration` 、 `tidb_tikvclient_batch_send_latency` 、および`tidb_tikvclient_rpc_net_latency_seconds{store="?"}`などの前述のバッチ クライアント期間の詳細については、 [バッチ クライアント](#batch-client)セクションを参照してください。

`tikv_grpc_msg_duration_seconds{type="kv_get"}`期間は次のように計算されます。

```text
tikv_grpc_msg_duration_seconds{type="kv_get"} =
    tikv_storage_engine_async_request_duration_seconds{type="snapshot"} +
    tikv_engine_seek_micro_seconds{type="seek_average"} +
    read value duration +
    read value duration(non-short value)
```

現時点では、リクエストは TiKV にあります。 TiKV プロセスは、1 回のシークと 1 回または 2 回の読み取りアクションでリクエストを取得します (短い値は書き込みカラムファミリーにエンコードされ、1 回の読み取りで十分です)。 TiKV は、読み取り要求を処理する前にスナップショットを取得します。 TiKV スナップショットの期間の詳細については、 [TiKV スナップショット](#tikv-snapshot)セクションを参照してください。

`read value duration(from disk)`は次のように計算されます。

```text
read value duration(from disk) =
    sum(rate(tikv_storage_rocksdb_perf{metric="block_read_time",req="get/batch_get_command"})) / sum(rate(tikv_storage_rocksdb_perf{metric="block_read_count",req="get/batch_get_command"}))
```

TiKV は RocksDB をストレージ エンジンとして使用します。必要な値がブロックキャッシュにない場合、TiKV はディスクから値を読み込む必要があります。 `tikv_storage_rocksdb_perf`の場合、get リクエストは`get`または`batch_get_command`のいずれかになります。

### バッチポイント取得 {#batch-point-get}

以下は、バッチ ポイント取得操作の時間コスト ダイアグラムです。

```railroad+diagram
Diagram(
  NonTerminal("Resolve TSO"),
  Choice(
    0,
    NonTerminal("Read all handles by index keys"),
    Comment("Read by clustered PK, encode handle by keys"),
  ),
  NonTerminal("Read values by handles"),
)
```

バッチ ポイント取得中、 `tidb_session_execute_duration_seconds{type="general"}`は次のように計算されます。

```text
tidb_session_execute_duration_seconds{type="general"} =
    pd_client_cmd_handle_cmds_duration_seconds{type="wait"} +
    read handles duration +
    read values duration
```

バッチ ポイント get のプロセスは、バッチ ポイント get が同時に複数の値を読み取ることを除いて、 [ポイントゲット](#point-get)とほとんど同じです。

`read handles duration`と`read values duration`は次のように計算されます。

```text
read handles duration = read values duration =
    tidb_tikvclient_txn_cmd_duration_seconds{type="batch_get"} =
    send request duration =
    tidb_tikvclient_request_seconds{type="BatchGet"} =
    tidb_tikvclient_batch_wait_duration(transaction) +
    tidb_tikvclient_batch_send_latency(transaction) +
    tikv_grpc_msg_duration_seconds{type="kv_batch_get"} +
    tidb_tikvclient_rpc_net_latency_seconds{store="?"}(transaction)
```

`tidb_tikvclient_batch_wait_duration(transaction)` 、 `tidb_tikvclient_batch_send_latency(transaction)` 、および`tidb_tikvclient_rpc_net_latency_seconds{store="?"}(transaction)`などの前述のバッチ クライアント期間の詳細については、 [バッチ クライアント](#batch-client)セクションを参照してください。

`tikv_grpc_msg_duration_seconds{type="kv_batch_get"}`期間は次のように計算されます。

```text
tikv_grpc_msg_duration_seconds{type="kv_batch_get"} =
    tikv_storage_engine_async_request_duration_seconds{type="snapshot"} +
    n * (
        tikv_engine_seek_micro_seconds{type="seek_max"} +
        read value duration +
        read value duration(non-short value)
    )

read value duration(from disk) =
    sum(rate(tikv_storage_rocksdb_perf{metric="block_read_time",req="batch_get"})) / sum(rate(tikv_storage_rocksdb_perf{metric="block_read_count",req="batch_get"}))
```

スナップショットを取得した後、TiKV は同じスナップショットから複数の値を読み取ります。読み取り期間は[ポイントゲット](#point-get)と同じです。 TiKV がディスクからデータをロードする場合、平均期間は`tikv_storage_rocksdb_perf`と`req="batch_get"`で計算できます。

### テーブルスキャンとインデックススキャン {#table-scan-x26-index-scan}

以下は、テーブル スキャン操作とインデックス スキャン操作の時間コストの図です。

```railroad+diagram
Diagram(
    Stack(
        NonTerminal("Resolve TSO"),
        NonTerminal("Load region cache for related table/index ranges"),
        OneOrMore(
            NonTerminal("Wait for result"),
            Comment("Next loop: drain the result"),
        ),
    ),
)
```

テーブル スキャンとインデックス スキャンの間、 `tidb_session_execute_duration_seconds{type="general"}`時間は次のように計算されます。

```text
tidb_session_execute_duration_seconds{type="general"} =
    pd_client_cmd_handle_cmds_duration_seconds{type="wait"} +
    req_per_copr * (
        tidb_distsql_handle_query_duration_seconds{sql_type="general"}
    )
    tidb_distsql_handle_query_duration_seconds{sql_type="general"} <= send request duration
```

テーブル スキャンとインデックス スキャンは同じ方法で処理されます。 `req_per_copr`は分散タスク数です。コプロセッサーの実行とクライアントに応答するデータは別のスレッドにあるため、 `tidb_distsql_handle_query_duration_seconds{sql_type="general"}`は待機時間であり、 `send request duration`よりも短くなります。

`send request duration`と`req_per_copr`は次のように計算されます。

```text
send request duration =
    tidb_tikvclient_batch_wait_duration +
    tidb_tikvclient_batch_send_latency +
    tikv_grpc_msg_duration_seconds{type="coprocessor"} +
    tidb_tikvclient_rpc_net_latency_seconds{store="?"}

tikv_grpc_msg_duration_seconds{type="coprocessor"} =
    tikv_coprocessor_request_wait_seconds{type="snapshot"} +
    tikv_coprocessor_request_wait_seconds{type="schedule"} +
    tikv_coprocessor_request_handler_build_seconds{type="index/select"} +
    tikv_coprocessor_request_handle_seconds{type="index/select"}

req_per_copr = rate(tidb_distsql_handle_query_duration_seconds_count) / rate(tidb_distsql_scan_keys_partial_num_count)
```

TiKV では、テーブル スキャン タイプは`select`で、インデックス スキャン タイプは`index`です。 `select`型と`index`型の持続時間の詳細は同じです。

### インデックスルックアップ {#index-look-up}

以下は、インデックス ルックアップ操作の時間コスト ダイアグラムです。

```railroad+diagram
Diagram(
    Stack(
        NonTerminal("Resolve TSO"),
        NonTerminal("Load region cache for related index ranges"),
        OneOrMore(
            Sequence(
                NonTerminal("Wait for index scan result"),
                NonTerminal("Wait for table scan result"),
            ),
        Comment("Next loop: drain the result"),
        ),
    ),
)
```

インデックス ルックアップ中、 `tidb_session_execute_duration_seconds{type="general"}`期間は次のように計算されます。

```text
tidb_session_execute_duration_seconds{type="general"} =
    pd_client_cmd_handle_cmds_duration_seconds{type="wait"} +
    req_per_copr * (
        tidb_distsql_handle_query_duration_seconds{sql_type="general"}
    ) +
    req_per_copr * (
        tidb_distsql_handle_query_duration_seconds{sql_type="general"}
    )

req_per_copr = rate(tidb_distsql_handle_query_duration_seconds_count) / rate(tidb_distsql_scan_keys_partial_num_count)
```

インデックス ルックアップは、パイプラインで処理されるインデックス スキャンとテーブル スキャンを組み合わせたものです。

## クエリを書く {#write-queries}

書き込みクエリは、読み取りクエリよりもはるかに複雑です。書き込みクエリにはいくつかのバリエーションがあります。以下は、書き込みクエリ操作の時間コストの図です。

```railroad+diagram
Diagram(
    NonTerminal("Execute write query"),
    Choice(
        0,
        NonTerminal("Pessimistic lock keys"),
        Comment("bypass in optimistic transaction"),
    ),
    Choice(
        0,
        NonTerminal("Auto Commit Transaction"),
        Comment("bypass in non-auto-commit or explicit transaction"),
    ),
)
```

|         | 悲観的な取引          | 楽観的な取引    |
| ------- | --------------- | --------- |
| 自動コミット  | 実行 + ロック + コミット | 実行 + コミット |
| 非自動コミット | 実行 + ロック        | 実行する      |

書き込みクエリは、次の 3 つのフェーズに分かれています。

-   実行フェーズ: ミューテーションを実行し、TiDB のメモリに書き込みます。
-   ロック フェーズ: 実行結果に対して悲観的ロックを取得します。
-   コミット フェーズ: 2 フェーズ コミット プロトコル (2PC) を介してトランザクションをコミットします。

実行フェーズでは、TiDB がメモリのデータを操作し、主なレイテンシーは必要なデータの読み取りから発生します。更新クエリと削除クエリの場合、TiDB はまず TiKV からデータを読み取り、次にメモリの行を更新または削除します。

例外は、単一のリモート プロシージャ コール (RPC) で読み取りとロックを実行するポイント get とバッチ ポイント get を使用したロック時の読み取り操作 ( `SELECT FOR UPDATE` ) です。

### ロック時刻取得 {#lock-time-point-get}

以下は、lock-time point get 操作の時間コスト ダイアグラムです。

```railroad+diagram
Diagram(
    Choice(
        0,
        Sequence(
            NonTerminal("Read handle key by index key"),
            NonTerminal("Lock index key"),
        ),
        Comment("Clustered index"),
    ),
    NonTerminal("Lock handle key"),
    NonTerminal("Read value from pessimistic lock cache"),
)
```

lock-time point get の間、 `execution(clustered PK)`および`execution(non-clustered PK or UK)`の期間は次のように計算されます。

```text
execution(clustered PK) =
    tidb_tikvclient_txn_cmd_duration_seconds{type="lock_keys"}
execution(non-clustered PK or UK) =
    2 * tidb_tikvclient_txn_cmd_duration_seconds{type="lock_keys"}
```

Lock-time point get はキーをロックし、その値を返します。実行後のロックフェーズと比較して、これは 1 ラウンド トリップを節約します。 lock-time point get の期間は[ロック期間](#lock)と同じように扱うことができます。

### ロックタイム バッチ ポイントの取得 {#lock-time-batch-point-get}

以下は、ロック時のバッチ ポイント取得操作の時間コスト ダイアグラムです。

```railroad+diagram
Diagram(
    Choice(
        0,
        NonTerminal("Read handle keys by index keys"),
        Comment("Clustered index"),
    ),
    NonTerminal("Lock index and handle keys"),
    NonTerminal("Read values from pessimistic lock cache"),
)
```

ロック時のバッチ ポイントの取得中に、 `execution(clustered PK)`と`execution(non-clustered PK or UK)`の期間は次のように計算されます。

```text
execution(clustered PK) =
    tidb_tikvclient_txn_cmd_duration_seconds{type="lock_keys"}
execution(non-clustered PK or UK) =
    tidb_tikvclient_txn_cmd_duration_seconds{type="batch_get"} +
    tidb_tikvclient_txn_cmd_duration_seconds{type="lock_keys"}
```

lock-time batch point get の実行は、lock-time batch point get が単一の RPC で複数の値を読み取ることを除いて、 [ロック時刻取得](#lock-time-point-get)と似ています。 `tidb_tikvclient_txn_cmd_duration_seconds{type="batch_get"}`期間の詳細については、 [バッチポイント取得](#batch-point-get)セクションを参照してください。

### ロック {#lock}

このセクションでは、ロック期間について説明します。

```text
round = ceil(
    sum(rate(tidb_tikvclient_txn_regions_num_sum{type="2pc_pessimistic_lock"})) /
    sum(rate(tidb_tikvclient_txn_regions_num_count{type="2pc_pessimistic_lock"})) /
    committer-concurrency
)

lock = tidb_tikvclient_txn_cmd_duration_seconds{type="lock_keys"} =
    round * tidb_tikvclient_request_seconds{type="PessimisticLock"}
```

ロックは、フロー制御メカニズムを持つ 2PC 構造を通じて取得されます。フロー制御は、同時オンザフライ リクエストを`committer-concurrency`に制限します (デフォルト値は`128`です)。簡単にするために、フロー制御はリクエストレイテンシーの増幅として扱うことができます ( `round` )。

`tidb_tikvclient_request_seconds{type="PessimisticLock"}`は次のように計算されます。

```text
tidb_tikvclient_request_seconds{type="PessimisticLock"} =
    tidb_tikvclient_batch_wait_duration +
    tidb_tikvclient_batch_send_latency +
    tikv_grpc_msg_duration_seconds{type="kv_pessimistic_lock"} +
    tidb_tikvclient_rpc_net_latency_seconds{store="?"}
```

`tidb_tikvclient_batch_wait_duration` 、 `tidb_tikvclient_batch_send_latency` 、および`tidb_tikvclient_rpc_net_latency_seconds{store="?"}`などの前述のバッチ クライアント期間の詳細については、 [バッチ クライアント](#batch-client)セクションを参照してください。

`tikv_grpc_msg_duration_seconds{type="kv_pessimistic_lock"}`期間は次のように計算されます。

```text
tikv_grpc_msg_duration_seconds{type="kv_pessimistic_lock"} =
    tikv_scheduler_latch_wait_duration_seconds{type="acquire_pessimistic_lock"} +
    tikv_storage_engine_async_request_duration_seconds{type="snapshot"} +
    (lock in-mem key count + lock on-disk key count) * lock read duration +
    lock on-disk key count / (lock in-mem key count + lock on-disk key count) *
    lock write duration
```

-   TiDB v6.0 以降、TiKV はデフォルトで[インメモリ悲観的ロック](/pessimistic-transaction.md#in-memory-pessimistic-lock)を使用します。インメモリ悲観的ロックは、非同期書き込みプロセスをバイパスします。

-   `tikv_storage_engine_async_request_duration_seconds{type="snapshot"}`はスナップショット タイプの期間です。詳細については、 [TiKV スナップショット](#tikv-snapshot)セクションを参照してください。

-   `lock in-mem key count`と`lock on-disk key count`は次のように計算されます。

    ```text
    lock in-mem key count =
        sum(rate(tikv_in_memory_pessimistic_locking{result="success"})) /
        sum(rate(tikv_grpc_msg_duration_seconds_count{type="kv_pessimistic_lock"}}))

    lock on-disk key count =
        sum(rate(tikv_in_memory_pessimistic_locking{result="full"})) /
        sum(rate(tikv_grpc_msg_duration_seconds_count{type="kv_pessimistic_lock"}}))
    ```

    メモリ内およびディスク上のロックされたキーの数は、メモリ内ロック カウンターによって計算できます。 TiKV はロックを取得する前にキーの値を読み取り、読み取り期間は RocksDB パフォーマンス コンテキストによって計算できます。

    ```text
    lock read duration(from disk) =
        sum(rate(tikv_storage_rocksdb_perf{metric="block_read_time",req="acquire_pessimistic_lock"})) / sum(rate(tikv_storage_rocksdb_perf{metric="block_read_count",req="acquire_pessimistic_lock"}))
    ```

-   `lock write duration`は、オンディスク ロックの書き込み期間です。詳細については、 [非同期書き込み](#async-write)セクションを参照してください。

### 専念 {#commit}

このセクションでは、コミット期間について説明します。以下は、コミット操作の時間コスト図です。

```railroad+diagram
Diagram(
    Stack(
        Sequence(
            Choice(
                0,
                Comment("use 2pc or causal consistency"),
                NonTerminal("Get min-commit-ts"),
            ),
            Optional("Async prewrite binlog"),
            NonTerminal("Prewrite mutations"),
            Optional("Wait prewrite binlog result"),
        ),
        Sequence(
            Choice(
                1,
                Comment("1pc"),
                Sequence(
                    Comment("2pc"),
                    NonTerminal("Get commit-ts"),
                    NonTerminal("Check schema, try to amend if needed"),
                    NonTerminal("Commit PK mutation"),
                ),
                Sequence(
                    Comment("async-commit"),
                    NonTerminal("Commit mutations asynchronously"),
                ),
            ),
            Choice(
                0,
                Comment("committed"),
                NonTerminal("Async cleanup"),
            ),
            Optional("Commit binlog"),
        ),
    ),
)
```

コミット フェーズの期間は、次のように計算されます。

```text
commit =
    Get_latest_ts_time +
    Prewrite_time +
    Get_commit_ts_time +
    Commit_time

Get_latest_ts_time = Get_commit_ts_time =
    pd_client_cmd_handle_cmds_duration_seconds{type="wait"}

prewrite_round = ceil(
    sum(rate(tidb_tikvclient_txn_regions_num_sum{type="2pc_prewrite"})) /
    sum(rate(tidb_tikvclient_txn_regions_num_count{type="2pc_prewrite"})) /
    committer-concurrency
)

commit_round = ceil(
    sum(rate(tidb_tikvclient_txn_regions_num_sum{type="2pc_commit"})) /
    sum(rate(tidb_tikvclient_txn_regions_num_count{type="2pc_commit"})) /
    committer-concurrency
)

Prewrite_time =
    prewrite_round * tidb_tikvclient_request_seconds{type="Prewrite"}

Commit_time =
    commit_round * tidb_tikvclient_request_seconds{type="Commit"}
```

コミット期間は、次の 4 つのメトリックに分類できます。

-   `Get_latest_ts_time`は、非同期コミットまたは単一フェーズ コミット (1PC) トランザクションで最新の TSO を取得する期間を記録します。
-   `Prewrite_time`は、事前書き込みフェーズの期間を記録します。
-   `Get_commit_ts_time`は、一般的な 2PC トランザクションの期間を記録します。
-   `Commit_time`は、コミット フェーズの期間を記録します。 async-commit または 1PC トランザクションにはこのフェーズがないことに注意してください。

悲観的ロックと同様に、フロー制御はレイテンシーの増幅として機能します (前の式の`prewrite_round`と`commit_round` )。

`tidb_tikvclient_request_seconds{type="Prewrite"}`と`tidb_tikvclient_request_seconds{type="Commit"}`の期間は次のように計算されます。

```text
tidb_tikvclient_request_seconds{type="Prewrite"} =
    tidb_tikvclient_batch_wait_duration +
    tidb_tikvclient_batch_send_latency +
    tikv_grpc_msg_duration_seconds{type="kv_prewrite"} +
    tidb_tikvclient_rpc_net_latency_seconds{store="?"}

tidb_tikvclient_request_seconds{type="Commit"} =
    tidb_tikvclient_batch_wait_duration +
    tidb_tikvclient_batch_send_latency +
    tikv_grpc_msg_duration_seconds{type="kv_commit"} +
    tidb_tikvclient_rpc_net_latency_seconds{store="?"}
```

`tidb_tikvclient_batch_wait_duration` 、 `tidb_tikvclient_batch_send_latency` 、および`tidb_tikvclient_rpc_net_latency_seconds{store="?"}`などの前述のバッチ クライアント期間の詳細については、 [バッチ クライアント](#batch-client)セクションを参照してください。

`tikv_grpc_msg_duration_seconds{type="kv_prewrite"}`は次のように計算されます。

```text
tikv_grpc_msg_duration_seconds{type="kv_prewrite"} =
    prewrite key count * prewrite read duration +
    prewrite write duration

prewrite key count =
    sum(rate(tikv_scheduler_kv_command_key_write_sum{type="prewrite"})) /
    sum(rate(tikv_scheduler_kv_command_key_write_count{type="prewrite"}))

prewrite read duration(from disk) =
    sum(rate(tikv_storage_rocksdb_perf{metric="block_read_time",req="prewrite"})) / sum(rate(tikv_storage_rocksdb_perf{metric="block_read_count",req="prewrite"}))
```

TiKV のロックと同様に、事前書き込みは読み取りフェーズと書き込みフェーズで処理されます。読み取り期間は、RocksDB パフォーマンス コンテキストから計算できます。書き込み期間の詳細については、 [非同期書き込み](#async-write)セクションを参照してください。

`tikv_grpc_msg_duration_seconds{type="kv_commit"}`は次のように計算されます。

```text
tikv_grpc_msg_duration_seconds{type="kv_commit"} =
    commit key count * commit read duration +
    commit write duration

commit key count =
    sum(rate(tikv_scheduler_kv_command_key_write_sum{type="commit"})) /
    sum(rate(tikv_scheduler_kv_command_key_write_count{type="commit"}))

commit read duration(from disk) =
    sum(rate(tikv_storage_rocksdb_perf{metric="block_read_time",req="commit"})) / sum(rate(tikv_storage_rocksdb_perf{metric="block_read_count",req="commit"})) (storage)
```

`kv_commit`の持続時間は`kv_prewrite`とほぼ同じです。書き込み期間の詳細については、 [非同期書き込み](#async-write)セクションを参照してください。

## バッチ クライアント {#batch-client}

以下は、バッチ クライアントの時間コスト ダイアグラムです。

```railroad+diagram
Diagram(
    NonTerminal("Get conn pool to the target store"),
    Choice(
        0,
        Sequence(
            Comment("Batch enabled"),
                NonTerminal("Push request to channel"),
                NonTerminal("Wait response"),
            ),
            Sequence(
            NonTerminal("Get conn from pool"),
            NonTerminal("Call RPC"),
            Choice(
                0,
                Comment("Unary call"),
                NonTerminal("Recv first"),
            ),
        ),
    ),
)
```

-   リクエストを送信する全体の期間は`tidb_tikvclient_request_seconds`として観測されます。
-   RPC クライアントは、各ストアへの接続プール (ConnArray という名前) を維持し、各プールには、バッチ要求 (送信) チャネルを持つ BatchConn があります。
-   バッチは、ストアが TiKV で、バッチ サイズが正の場合に有効になります。これはほとんどの場合に当てはまります。
-   バッチ リクエスト チャネルのサイズは[`tikv-client.max-batch-size`](/tidb-configuration-file.md#max-batch-size) (デフォルトは`128` ) で、エンキューの期間は`tidb_tikvclient_batch_wait_duration`として観察されます。
-   ストリーム要求には`CmdBatchCop` 、 `CmdCopStream` 、および`CmdMPPConn`の 3 種類があり、ストリームから最初の応答を取得するために追加の`recv()`呼び出しが必要です。

見逃されたレイテンシーがまだいくらかありますが、 `tidb_tikvclient_request_seconds`はおおよそ次のように計算できます。

```text
tidb_tikvclient_request_seconds{type="?"} =
    tidb_tikvclient_batch_wait_duration +
    tidb_tikvclient_batch_send_latency +
    tikv_grpc_msg_duration_seconds{type="kv_?"} +
    tidb_tikvclient_rpc_net_latency_seconds{store="?"}
```

-   `tidb_tikvclient_batch_wait_duration`は、バッチ システムでの待機時間を記録します。
-   `tidb_tikvclient_batch_send_latency`は、バッチ システムでのエンコード期間を記録します。
-   `tikv_grpc_msg_duration_seconds{type="kv_?"}`は TiKV 処理期間です。
-   `tidb_tikvclient_rpc_net_latency_seconds`はネットワークレイテンシーを記録します。

## TiKV スナップショット {#tikv-snapshot}

以下は、TiKV スナップショット操作の時間コスト図です。

```railroad+diagram
Diagram(
    Choice(
        0,
        Comment("Local Read"),
        Sequence(
            NonTerminal("Propose Wait"),
            NonTerminal("Read index Read Wait"),
        ),
    ),
    NonTerminal("Fetch A Snapshot From KV Engine"),
)
```

TiKV スナップショットの全体の期間は`tikv_storage_engine_async_request_duration_seconds{type="snapshot"}`として観測され、次のように計算されます。

```text
tikv_storage_engine_async_request_duration_seconds{type="snapshot"} =
    tikv_coprocessor_request_wait_seconds{type="snapshot"} =
    tikv_raftstore_request_wait_time_duration_secs +
    tikv_raftstore_commit_log_duration_seconds +
    get snapshot from rocksdb duration
```

リーダーのリースが期限切れになると、TiKV は RocksDB からスナップショットを取得する前にインデックスの読み取りコマンドを提案します。 `tikv_raftstore_request_wait_time_duration_secs`と`tikv_raftstore_commit_log_duration_seconds`は、読み取りインデックス コマンドをコミットする期間です。

通常、RocksDB からスナップショットを取得する操作は高速であるため、 `get snapshot from rocksdb duration`は無視されます。

## 非同期書き込み {#async-write}

非同期書き込みは、TiKV がデータを Raft ベースの複製されたステート マシンにコールバックで非同期に書き込むプロセスです。

-   以下は、非同期 IO が無効になっている場合の非同期書き込み操作の時間コストの図です。

    ```railroad+diagram
    Diagram(
        NonTerminal("Propose Wait"),
        NonTerminal("Process Command"),
        Choice(
            0,
            Sequence(
                NonTerminal("Wait Current Batch"),
                NonTerminal("Write to Log Engine"),
            ),
            Sequence(
                NonTerminal("RaftMsg Send Wait"),
                NonTerminal("Commit Log Wait"),
            ),
        ),
        NonTerminal("Apply Wait"),
        NonTerminal("Apply Log"),
    )
    ```

-   以下は、非同期 IO が有効な場合の非同期書き込み操作の時間コスト図です。

    ```railroad+diagram
    Diagram(
        NonTerminal("Propose Wait"),
        NonTerminal("Process Command"),
        Choice(
            0,
            NonTerminal("Wait Until Persisted by Write Worker"),
            Sequence(
                NonTerminal("RaftMsg Send Wait"),
                NonTerminal("Commit Log Wait"),
            ),
        ),
        NonTerminal("Apply Wait"),
        NonTerminal("Apply Log"),
    )
    ```

非同期書き込み期間は次のように計算されます。

```text
async write duration(async io disabled) =
    propose +
    async io disabled commit +
    tikv_raftstore_apply_wait_time_duration_secs +
    tikv_raftstore_apply_log_duration_seconds

async write duration(async io enabled) =
    propose +
    async io enabled commit +
    tikv_raftstore_apply_wait_time_duration_secs +
    tikv_raftstore_apply_log_duration_seconds
```

非同期書き込みは、次の 3 つのフェーズに分けることができます。

-   提案する
-   専念
-   適用: 前の式の`tikv_raftstore_apply_wait_time_duration_secs + tikv_raftstore_apply_log_duration_seconds`

提案フェーズの期間は、次のように計算されます。

```text
propose =
    propose wait duration +
    propose duration

propose wait duration =
    tikv_raftstore_store_wf_batch_wait_duration_seconds

propose duration =
    tikv_raftstore_store_wf_send_to_queue_duration_seconds -
    tikv_raftstore_store_wf_batch_wait_duration_seconds
```

Raftプロセスはウォーターフォール形式で記録されます。したがって、提案期間は、2 つのメトリックの差から計算されます。

コミット フェーズの期間は、次のように計算されます。

```text
async io disabled commit = max(
    persist log locally duration,
    replicate log duration
)

async io enabled commit = max(
    wait by write worker duration,
    replicate log duration
)
```

v5.3.0 以降、TiKV は Async IO Raft (StoreWriter スレッド プールによるRaftログの書き込み) をサポートします。 Async IO Raftは、 [`store-io-pool-size`](/tikv-configuration-file.md#store-io-pool-size-new-in-v530)が正の値に設定されている場合にのみ有効になり、コミットのプロセスが変更されます。 `persist log locally duration`と`wait by write worker duration`は次のように計算されます。

```text
persist log locally duration =
    batch wait duration +
    write to raft db duration

batch wait duration =
    tikv_raftstore_store_wf_before_write_duration_seconds -
    tikv_raftstore_store_wf_send_to_queue_duration_seconds

write to raft db duration =
    tikv_raftstore_store_wf_write_end_duration_seconds -
    tikv_raftstore_store_wf_before_write_duration_seconds

wait by write worker duration =
    tikv_raftstore_store_wf_persist_duration_seconds -
    tikv_raftstore_store_wf_send_to_queue_duration_seconds
```

Async IO がある場合とない場合の違いは、ログがローカルに保持される期間です。 Async IO を使用すると、ログをローカルに保持する期間をウォーターフォール メトリックから直接計算できます (バッチ待機期間をスキップします)。

レプリケート ログ期間は、クォーラム ピアに保持されるログの期間を記録します。これには、RPC 期間と、大部分のログが保持される期間が含まれます。 `replicate log duration`は次のように計算されます。

```text
replicate log duration =
    raftmsg send wait duration +
    commit log wait duration

raftmsg send wait duration =
    tikv_raftstore_store_wf_send_proposal_duration_seconds -
    tikv_raftstore_store_wf_send_to_queue_duration_seconds

commit log wait duration =
    tikv_raftstore_store_wf_commit_log_duration -
    tikv_raftstore_store_wf_send_proposal_duration_seconds
```

### RaftDB {#raft-db}

以下は、 Raft DB 操作の時間コスト図です。

```railroad+diagram
Diagram(
    NonTerminal("Wait for Writer Leader"),
    NonTerminal("Write and Sync Log"),
    NonTerminal("Apply Log to Memtable"),
)
```

```text
write to raft db duration = raft db write duration
commit log wait duration >= raft db write duration

raft db write duration(raft engine enabled) =
    raft_engine_write_preprocess_duration_seconds +
    raft_engine_write_leader_duration_seconds +
    raft_engine_write_apply_duration_seconds

raft db write duration(raft engine disabled) =
    tikv_raftstore_store_perf_context_time_duration_secs{type="write_thread_wait"} +
    tikv_raftstore_store_perf_context_time_duration_secs{type="write_scheduling_flushes_compactions_time"} +
    tikv_raftstore_store_perf_context_time_duration_secs{type="write_wal_time"} +
    tikv_raftstore_store_perf_context_time_duration_secs{type="write_memtable_time"}
```

`commit log wait duration`はクォーラム ピアの最長期間であるため、 `raft db write duration`より大きい場合があります。

v6.1.0 以降、TiKV はデフォルトのログ ストレージ エンジンとして[Raft Engine](/glossary.md#raft-engine)を使用し、ログの書き込みプロセスを変更します。

### KV DB {#kv-db}

以下は、KV DB 操作の時間コスト図です。

```railroad+diagram
Diagram(
    NonTerminal("Wait for Writer Leader"),
    NonTerminal("Preprocess"),
    Choice(
        0,
        Comment("No Need to Switch"),
        NonTerminal("Switch WAL or Memtable"),
    ),
    NonTerminal("Write and Sync WAL"),
    NonTerminal("Apply to Memtable"),
)
```

```text
tikv_raftstore_apply_log_duration_seconds =
    tikv_raftstore_apply_perf_context_time_duration_secs{type="write_thread_wait"} +
    tikv_raftstore_apply_perf_context_time_duration_secs{type="write_scheduling_flushes_compactions_time"} +
    tikv_raftstore_apply_perf_context_time_duration_secs{type="write_wal_time"} +
    tikv_raftstore_apply_perf_context_time_duration_secs{type="write_memtable_time"}
```

非同期書き込みプロセスでは、コミットされたログを KV DB に適用する必要があります。適用期間は、RocksDB パフォーマンス コンテキストから計算できます。

## 診断のユースケース {#diagnosis-use-cases}

前のセクションでは、クエリ実行中の時間コスト メトリックについて詳しく説明しました。このセクションでは、低速の読み取りまたは書き込みクエリが発生した場合のメトリック分析の一般的な手順を紹介します。すべてのメトリックは、 [パフォーマンス概要ダッシュボード](/grafana-performance-overview-dashboard.md)の [データベース時間] パネルで確認できます。

### 遅い読み取りクエリ {#slow-read-queries}

`SELECT`のステートメントがデータベース時間のかなりの部分を占めている場合、TiDB の読み取りクエリが遅いと見なすことができます。

低速クエリの実行計画は、TiDB ダッシュボードの[Top SQLステートメント](/dashboard/dashboard-overview.md#top-sql-statements)パネルで確認できます。低速読み取りクエリの時間コストを調査するには、前述の説明に従って[ポイントゲット](#point-get) 、 [バッチポイント取得](#batch-point-get) 、およびいくつかの[単純なコプロセッサー照会](#table-scan--index-scan)を分析できます。

### 遅い書き込みクエリ {#slow-write-queries}

遅い書き込みを調査する前に、 `tikv_scheduler_latch_wait_duration_seconds_sum{type="acquire_pessimistic_lock"} by (instance)`を確認して競合の原因をトラブルシューティングする必要があります。

-   一部の特定の TiKV インスタンスでこのメトリクスが高い場合、ホット リージョンで競合が発生している可能性があります。
-   このメトリックがすべてのインスタンスで高い場合、アプリケーションで競合が発生している可能性があります。

アプリケーションから競合の原因を確認した後、 [ロック](#lock)と[専念](#commit)の期間を分析することで、書き込みが遅いクエリを調査できます。

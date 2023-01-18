---
title: TiDB 5.0 RC Release Notes
---

# TiDB 5.0 RC リリースノート {#tidb-5-0-rc-release-notes}

発売日：2021年1月12日

TiDB バージョン: 5.0.0-rc

TiDB v5.0.0-rc は、TiDB v5.0 の先行バージョンです。 v5.0 では、PingCAP は企業が TiDB に基づいて迅速にアプリケーションを構築できるよう支援し、データベースのパフォーマンス、パフォーマンスのジッター、セキュリティ、高可用性、災害復旧、SQL パフォーマンスのトラブルシューティングなどに関する心配から解放します。

v5.0 の主な新機能または改善点は次のとおりです。

-   クラスタ化されたインデックス。この機能を有効にすると、データベースのパフォーマンスが向上します。たとえば、TPC-C tpmC テストでは、クラスター化インデックスを有効にした TiDB のパフォーマンスは 39% 向上します。
-   非同期コミット。この機能を有効にすると、書き込みレイテンシーが短縮されます。たとえば、Sysbench olpt-insert テストでは、非同期コミットを有効にした TiDB の書き込みレイテンシーは 37.3% 短縮されました。
-   ジッターの低減。これは、オプティマイザの安定性を向上させ、システム タスクによる I/O、ネットワーク、CPU、およびメモリ リソースの使用を制限することによって実現されます。たとえば、72 時間のパフォーマンス テストでは、Sysbench TPS ジッターの標準偏差が 11.09% から 3.36% に減少しました。
-   Raftジョイント コンセンサス アルゴリズム。これにより、リージョンメンバーシップの変更中にシステムの可用性が保証されます。
-   データベース管理者 (DBA) が SQL ステートメントをより効率的にデバッグするのに役立つ、最適化された`EXPLAIN`機能と非表示のインデックス。
-   企業データの信頼性を保証します。 TiDB から AWS S3 ストレージおよび Google Cloud GCS にデータをバックアップしたり、これらのクラウド ストレージ プラットフォームからデータを復元したりできます。
-   AWS S3 ストレージまたは TiDB/MySQL からのデータ インポートまたはデータ エクスポートのパフォーマンスが向上し、企業がクラウド上でアプリケーションを迅速に構築するのに役立ちます。たとえば、TPC-C テストでは、1 TiB のデータをインポートするパフォーマンスが 254 GiB/h から 366 GiB/h に 40% 向上します。

## SQL {#sql}

### クラスタ化インデックスをサポート (実験的) {#support-clustered-index-experimental}

クラスター化インデックス機能を有効にすると、次の場合に TiDB のパフォーマンスが大幅に向上します (たとえば、TPC-C tpmC テストでは、クラスター化インデックスを有効にした TiDB のパフォーマンスは 39% 向上します)。

-   データが挿入されると、クラスター化インデックスは、ネットワークからのインデックス データの 1 回の書き込みを減らします。
-   同等の条件を持つクエリに主キーのみが含まれる場合、クラスター化インデックスは、ネットワークからのインデックス データの 1 回の読み取りを減らします。
-   範囲条件を含むクエリに主キーのみが含まれる場合、クラスター化インデックスにより、ネットワークからのインデックス データの複数回の読み取りが削減されます。
-   同等または範囲条件を含むクエリに主キー プレフィックスが含まれる場合、クラスター化インデックスは、ネットワークからのインデックス データの複数回の読み取りを減らします。

クラスター化インデックスは、テーブル内のデータの物理的な格納順序を定義します。テーブル内のデータは、クラスター化インデックスの定義に従ってのみ並べ替えられます。各テーブルには、クラスター化インデックスが 1 つだけあります。

ユーザーは、変数`tidb_enable_clustered_index`を変更することで、クラスター化インデックス機能を有効にすることができます。有効にすると、この機能は新しく作成されたテーブルでのみ有効になり、複数の列を持つ主キーまたは単一の列に整数型ではない主キーに適用されます。主キーが単一列の整数型である場合、またはテーブルに主キーがない場合、データは以前と同じ方法で並べ替えられ、クラスター化インデックスの影響を受けません。

たとえば、テーブル ( `tbl_name` ) にクラスター化インデックスがあるかどうかを確認するには、 `select tidb_pk_type from information_schema.tables where table_name = '{tbl_name}'`を実行します。

-   [ユーザー文書](/system-variables.md#tidb_enable_clustered_index-new-in-v50)
-   関連する問題: [#4841](https://github.com/pingcap/tidb/issues/4841)

### 非表示のインデックスをサポート {#support-invisible-indexes}

ユーザーがパフォーマンスを調整するとき、または最適なインデックスを選択するときに、SQL ステートメントを使用してインデックスを`Visible`または`Invisible`に設定できます。この設定により、リソースを消費する操作 ( `DROP INDEX`や`ADD INDEX`など) の実行を回避できます。

インデックスの可視性を変更するには、 `ALTER INDEX`ステートメントを使用します。変更後、オプティマイザーは、インデックスの可視性に基づいて、このインデックスをインデックス リストに追加するかどうかを決定します。

-   [ユーザー文書](/sql-statements/sql-statement-alter-index.md)
-   関連する問題: [#9246](https://github.com/pingcap/tidb/issues/9246)

### <code>EXCEPT</code>および<code>INTERSECT</code>演算子をサポート {#support-code-except-code-and-code-intersect-code-operators}

`INTERSECT`演算子は集合演算子であり、2 つ以上のクエリの結果セットの共通部分を返します。ある程度、これは`InnerJoin`演算子の代わりになります。

`EXCEPT`演算子はセット演算子で、2 つのクエリの結果セットを結合し、最初のクエリ結果には含まれるが 2 番目のクエリ結果には含まれない要素を返します。

-   [ユーザー文書](/functions-and-operators/set-operators.md)
-   関連する問題: [#18031](https://github.com/pingcap/tidb/issues/18031)

## 取引 {#transaction}

### 悲観的トランザクションの実行の成功率を高める {#increase-the-success-rate-of-executing-pessimistic-transactions}

悲観的トランザクション モードでは、トランザクションに関連するテーブルに同時 DDL 操作または`SCHEMA VERSION`の変更が含まれている場合、システムはトランザクションの`SCHEMA VERSION`を最新のものに自動的に更新して、DDL 操作によってトランザクションが中断されるのを回避し、トランザクションのコミットが成功するようにします。トランザクションが中断されると、クライアントは`Information schema is changed`エラー メッセージを受け取ります。

-   [ユーザー文書](/system-variables.md#tidb_enable_amend_pessimistic_txn-new-in-v407)
-   関連する問題: [#18005](https://github.com/pingcap/tidb/issues/18005)

## 文字セットと照合順序 {#character-set-and-collation}

文字セットの大文字と小文字を区別しない比較ソートをサポートします。

-   [ユーザー文書](/character-set-and-collation.md#new-framework-for-collations)
-   関連する問題: [#17596](https://github.com/pingcap/tidb/issues/17596)

## 安全 {#security}

### エラーメッセージとログファイルの感度低下をサポート {#support-desensitizing-error-messages-and-log-files}

TiDB は、ID 情報やクレジット カード番号などの機密情報の漏洩を防ぐために、エラー メッセージとログ ファイルの感度を下げることをサポートするようになりました。

ユーザーは、さまざまなコンポーネントの感度低下機能を有効にすることができます。

-   TiDB 側では、tidb-server で SQL ステートメントを使用して`tidb_redact_log=1`変数を設定します。
-   TiKV 側は、tikv-server に`security.redact-info-log = true`の構成を設定します。
-   PD 側は、pd-server に`security.redact-info-log = true`の構成を設定します。 [#2852](https://github.com/tikv/pd/issues/2852) [#3011](https://github.com/tikv/pd/pull/3011)
-   TiFlash側は、tiflash-server に`security.redact_info_log = true`を、tiflash-learner に`security.redact-info-log = true`を設定します。

[ユーザー文書](/log-redaction.md)

関連する問題: [#18566](https://github.com/pingcap/tidb/issues/18566)

## パフォーマンスの向上 {#performance-improvements}

### 非同期コミットをサポート (実験的) {#support-async-commit-experimental}

非同期コミット機能を有効にすると、トランザクションのレイテンシーを大幅に短縮できます。たとえば、この機能が有効になっている場合、Sysbench oltp-insert テストでのトランザクションのレイテンシーは、この機能が無効になっている場合よりも 37.3% 低くなります。

以前は非同期コミット機能がなく、書き込まれたステートメントは、2 フェーズ トランザクション コミットが完了した後にのみクライアントに返されました。現在、非同期コミット機能は、2 フェーズ コミットの最初のフェーズが終了した後にクライアントに結果を返すことをサポートしています。次に、2 番目のフェーズがバックグラウンドで非同期に実行されるため、トランザクション コミットのレイテンシーが短縮されます。

ただし、非同期コミットが有効な場合、トランザクションの外部整合性は`tidb_guarantee_external_consistency = ON`が設定されている場合に**のみ**保証されます。非同期コミットを有効にすると、パフォーマンスが低下する可能性があります。

ユーザーは、グローバル変数`tidb_enable_async_commit = ON`を設定することで、この機能を有効にすることができます。

-   [ユーザー文書](/system-variables.md#tidb_enable_async_commit-new-in-v50)
-   関連する問題: [#8316](https://github.com/tikv/tikv/issues/8316)

### インデックス選択におけるオプティマイザーの安定性を向上させます (実験的)。 {#improve-the-optimizer-s-stability-in-index-selection-experimental}

比較的適切なインデックスを常に選択するオプティマイザの機能は、クエリのレイテンシーが安定しているかどうかを大きく左右します。統計モジュールを改善およびリファクタリングして、同じ SQL ステートメントに対して、統計が欠落しているか不正確であるためにオプティマイザーが毎回複数の候補インデックスから異なるインデックスを選択しないようにしました。オプティマイザーが比較的適切なインデックスを選択するのに役立つ主な改善点は次のとおりです。

-   複数列の NDV、複数列の順序の依存関係、複数列の関数の依存関係など、より多くの情報を統計モジュールに追加します。
-   統計モジュールをリファクタリングします。
    -   `CMSKetch`から`TopN`の値を削除します。
    -   `TopN`の検索ロジックをリファクタリングします。
    -   ヒストグラムから`TopN`の情報を削除し、ヒストグラムのインデックスを作成して、バケット NDV のメンテナンスを容易にします。

関連する問題: [#18065](https://github.com/pingcap/tidb/issues/18065)

### 不完全なスケジューリングまたは不完全な I/O フロー制御によって引き起こされるパフォーマンス ジッタを最適化します。 {#optimize-performance-jitter-caused-by-imperfect-scheduling-or-imperfect-i-o-flow-control}

TiDB スケジューリング プロセスは、I/O、ネットワーク、CPU、メモリなどのリソースを占有します。 TiDB がスケジュールされたタスクを制御しない場合、QPS と遅延により、リソースのプリエンプションが原因でパフォーマンスのジッターが発生する可能性があります。次の最適化の後、72 時間のテストで、Sysbench TPS ジッターの標準偏差が 11.09% から 3.36% に減少しました。

-   ノード容量の変動 (常にウォーターライン付近) および PD の`store-limit`構成値の設定が大きすぎることが原因で発生する、冗長なスケジューリングの問題を軽減します。これは、 `region-score-formula-version = v2`の構成アイテムを介して有効になる新しいスケジューリング計算式のセットを導入することによって実現されます。 [#3269](https://github.com/tikv/pd/pull/3269)
-   `enable-cross-table-merge = true`を変更して空のリージョンの数を減らすことで、リージョン間のマージ機能を有効にします。 [#3129](https://github.com/tikv/pd/pull/3129)
-   TiKV バックグラウンドでのデータ圧縮は、多くの I/O リソースを占有します。システムは圧縮率を自動的に調整して、バックグラウンド タスクとフォアグラウンドの読み取りおよび書き込みの間の I/O リソースの競合のバランスを取ります。 `rate-limiter-auto-tuned`の設定項目でこの機能を有効にすると、遅延ジッターが大幅に減少します。 [#18011](https://github.com/pingcap/tidb/issues/18011)
-   TiKV がガベージコレクション(GC) とデータ圧縮を実行するとき、パーティションは CPU と I/O リソースを占有します。これら 2 つのタスクの実行中に重複データが存在します。 I/O 使用量を削減するために、GC 圧縮フィルター機能はこれら 2 つのタスクを 1 つに結合し、同じタスクで実行します。この機能はまだ実験的段階であり、 `gc.enable-compaction-filter = true`で有効にすることができます。 [#18009](https://github.com/pingcap/tidb/issues/18009)
-   TiFlashがデータを圧縮またはソートすると、大量の I/O リソースが占有されます。システムは、圧縮とデータの並べ替えによる I/O リソースの使用を制限することで、リソースの競合を軽減します。この機能はまだ実験的段階であり、 `bg_task_io_rate_limit`で有効にすることができます。

関連する問題: [#18005](https://github.com/pingcap/tidb/issues/18005)

### リアルタイム BI / データ ウェアハウジングのシナリオでTiFlashの安定性を向上 {#improve-the-stability-of-tiflash-in-real-time-bi-data-warehousing-scenarios}

-   DeltaIndex のメモリ使用量を制限して、大量のデータ ボリュームのシナリオで過剰なメモリ使用量が原因で発生するシステム メモリ不足 (OOM) を回避します。
-   バックグラウンド データ並べ替えタスクで使用される I/O 書き込みトラフィックを制限して、フォアグラウンド タスクへの影響を軽減します。
-   コプロセッサー・タスクをキューに入れるための新しいスレッド・プールを追加します。これにより、コプロセッサーを高い並行性で処理する際の過度のメモリー使用によって引き起こされるシステム OOM が回避されます。

### その他のパフォーマンスの最適化 {#other-performance-optimizations}

-   `delete from table where id <?`ステートメントの実行パフォーマンスを向上させます。その P99 パフォーマンスは 4 倍向上します。 [#18028](https://github.com/pingcap/tidb/issues/18028)
-   TiFlashは、パフォーマンスを向上させるために、複数のローカル ディスクで同時にデータを読み書きすることをサポートしています。

## 高可用性と災害復旧 {#high-availability-and-disaster-recovery}

### リージョンメンバーシップの変更中のシステムの可用性を向上させる (実験的) {#improve-system-availability-during-region-membership-change-experimental}

リージョンメンバーシップの変更プロセスでは、「メンバーの追加」と「メンバーの削除」の 2 つの操作が 2 つのステップで実行されます。メンバーシップの変更が完了するときに障害が発生した場合、リージョンは使用できなくなり、フォアグラウンド アプリケーションのエラーが返されます。導入されたRaft Joint Consensus アルゴリズムは、リージョンメンバーシップの変更中のシステムの可用性を向上させることができます。会員変更時の「会員追加」「会員削除」の操作をまとめて全会員に送信します。変更プロセス中、リージョンは中間状態にあります。変更されたメンバーに障害が発生した場合でも、システムは引き続き使用できます。ユーザーは、 `pd-ctl config set enable-joint-consensus true`を実行してメンバーシップ変数を変更することで、この機能を有効にすることができます。 [#7587](https://github.com/tikv/tikv/issues/7587) [#2860](https://github.com/tikv/pd/issues/2860)

-   [ユーザー文書](/pd-configuration-file.md#enable-joint-consensus-new-in-v50)
-   関連する問題: [#18079](https://github.com/pingcap/tidb/issues/18079)

### メモリ管理モジュールを最適化して、システム OOM のリスクを軽減します {#optimize-the-memory-management-module-to-reduce-system-oom-risks}

-   キャッシュ統計のメモリ消費を減らします。
-   Dumplingツールを使用して、データのエクスポートのメモリ消費を削減します。
-   データの暗号化された中間結果をディスクに保存することで、メモリ消費を削減しました。

## バックアップと復元 {#backup-and-restore}

-   バックアップと復元ツール (BR) は、AWS S3 と Google Cloud GCS へのデータのバックアップをサポートしています。 ( [ユーザー文書](/br/backup-and-restore-storages.md) )
-   Backup &amp; Restore ツール (BR) は、AWS S3 および Google Cloud GCS から TiDB へのデータの復元をサポートしています。 ( [ユーザー文書](/br/backup-and-restore-storages.md) )
-   関連する問題: [#89](https://github.com/pingcap/br/issues/89)

## データのインポートとエクスポート {#data-import-and-export}

-   TiDB Lightningは、AWS S3 ストレージから TiDB へのAuroraスナップショット データのインポートをサポートしています。 (関連号: [#266](https://github.com/pingcap/tidb-lightning/issues/266) )
-   1 TiB のデータを DBaaS T1.standard にインポートする TPC-C テストでは、パフォーマンスが 254 GiB/h から 366 GiB/h に 40% 向上します。
-   Dumplingは、TiDB/MySQL から AWS S3 ストレージへのデータのエクスポートをサポートします (実験的) (関連する問題: [#8](https://github.com/pingcap/dumpling/issues/8) 、 [ユーザー文書](/dumpling-overview.md#export-data-to-amazon-s3-cloud-storage) )

## 診断 {#diagnostics}

### より多くの情報を収集して最適化された<code>EXPLAIN</code>機能は、ユーザーがパフォーマンスの問題をトラブルシューティングするのに役立ちます {#optimized-code-explain-code-features-with-more-collected-information-help-users-troubleshoot-performance-issues}

ユーザーが SQL パフォーマンスの問題をトラブルシューティングする場合、パフォーマンスの問題の原因を特定するための詳細な診断情報が必要です。以前のバージョンの TiDB では、 `EXPLAIN`ステートメントによって収集された情報は十分に詳細ではありませんでした。 DBA は、ログ情報、監視情報、または推測に基づいてのみトラブルシューティングを行っていたため、非効率的である可能性があります。 TiDB v5.0 では、ユーザーがパフォーマンスの問題をより効率的にトラブルシューティングできるように、次の改善が行われています。

-   `EXPLAIN ANALYZE`は、すべての DML ステートメントの分析をサポートし、実際のパフォーマンス プランと各オペレーターの実行情報を示します。 [#18056](https://github.com/pingcap/tidb/issues/18056)
-   ユーザーは`EXPLAIN FOR CONNECTION`を使用して、実行中の SQL ステートメントの状況情報を分析できます。この情報には、各演算子の実行時間と処理された行数が含まれます。 [#18233](https://github.com/pingcap/tidb/issues/18233)
-   `EXPLAIN ANALYZE`の出力には、オペレーターが送信した RPC リクエストの数、ロックの競合を解決する期間、ネットワークレイテンシー、RocksDB 内の削除されたデータのスキャン ボリューム、RocksDB キャッシュのヒット率など、より多くの情報が含まれています。 [#18663](https://github.com/pingcap/tidb/issues/18663)
-   SQL ステートメントの詳細な実行情報はスロー ログに記録されます。これは、 `EXPLAIN ANALYZE`の出力情報と一致しています。この情報には、各オペレーターが消費した時間、処理された行数、および送信された RPC 要求の数が含まれます。 [#15009](https://github.com/pingcap/tidb/issues/15009)

[ユーザー文書](/sql-statements/sql-statement-explain.md)

## 展開とメンテナンス {#deployment-and-maintenance}

-   以前は、TiDB Ansible の構成情報をTiUPにインポートすると、 TiUPはユーザー構成を`ansible-imported-configs`ディレクトリに配置していました。ユーザーが後で`tiup cluster edit-config`を使用して構成を編集する必要が生じたときに、インポートされた構成がエディター インターフェイスに表示されず、ユーザーが混乱する可能性がありました。 TiDB v5.0 では、TiDB Ansible 構成がインポートされると、 TiUPは構成情報を`ansible-imported-configs`とエディター インターフェイスの両方に配置します。この改善により、ユーザーはクラスター構成を編集しているときに、インポートされた構成を確認できます。
-   複数のミラーを 1 つにマージし、ローカル ミラーにコンポーネントを公開し、ローカル ミラーにコンポーネント所有者を追加することをサポートする拡張`mirror`コマンド。 [#814](https://github.com/pingcap/tiup/issues/814)
    -   大企業、特に金融業界の場合、本番環境の変更は慎重に検討されます。バージョンごとに、ユーザーがインストールに CD を使用する必要がある場合、面倒な場合があります。 TiDB v5.0 では、 TiUPの`merge`コマンドで複数のインストール パッケージを 1 つにマージできるようになり、インストールが容易になりました。
    -   v4.0 では、ユーザーは自分で構築したミラーを公開するために tiup-server を起動する必要があり、これは十分に便利ではありませんでした。 v5.0 では、ユーザーは`tiup mirror set`を使用して現在のミラーをローカル ミラーに設定するだけで、セルフビルド ミラーを公開できます。

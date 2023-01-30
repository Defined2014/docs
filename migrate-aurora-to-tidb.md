---
title: Migrate Data from Amazon Aurora to TiDB
summary: Learn how to migrate data from Amazon Aurora to TiDB using DB snapshot.
---

# Amazon Auroraから TiDB にデータを移行する {#migrate-data-from-amazon-aurora-to-tidb}

このドキュメントでは、Amazon Auroraから TiDB にデータを移行する方法について説明します。移行プロセスでは[DB スナップショット](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Managing.Backups.html)を使用するため、スペースと時間を大幅に節約できます。

移行全体には 2 つのプロセスがあります。

-   TiDB Lightningを使用して TiDB に完全なデータをインポートする
-   DM を使用して増分データを TiDB に複製する (オプション)

## 前提条件 {#prerequisites}

-   [DumplingとTiDB Lightningをインストールする](/migration-tools.md)
-   [TiDB Lightningに必要なターゲット データベース権限を取得する](/tidb-lightning/tidb-lightning-faq.md#what-are-the-privilege-requirements-for-the-target-database) .

## 完全なデータを TiDB にインポートする {#import-full-data-to-tidb}

### ステップAuroraスナップショットを Amazon S3 にエクスポートする {#step-1-export-an-aurora-snapshot-to-amazon-s3}

1.  Auroraで、次のコマンドを実行して現在の binlog の位置を照会します。

    ```sql
    mysql> SHOW MASTER STATUS;
    ```

    出力は次のようになります。後で使用するためにバイナリログの名前と位置を記録します。

    ```
    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000002 |    52806 |              |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+
    1 row in set (0.012 sec)
    ```

2.  Auroraスナップショットをエクスポートします。詳細な手順については、 [DB スナップショット データを Amazon S3 にエクスポートする](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_ExportSnapshot.html)を参照してください。

binlog の位置を取得したら、5 分以内にスナップショットをエクスポートします。そうしないと、記録されたバイナリログの位置が古くなり、増分レプリケーション中にデータの競合が発生する可能性があります。

上記の 2 つの手順を実行したら、次の情報が準備できていることを確認してください。

-   スナップショット作成時のAuroraバイナリログの名前と位置。
-   スナップショットが保存されている S3 パス、および S3 パスにアクセスできる SecretKey と AccessKey。

### ステップ 2. スキーマのエクスポート {#step-2-export-schema}

Auroraのスナップショット ファイルには DDL ステートメントが含まれていないため、Dumple を使用してスキーマをエクスポートし、 Dumpling TiDB Lightningを使用してターゲット データベースにスキーマを作成する必要があります。スキーマを手動で作成する場合は、この手順を省略できます。

次のコマンドを実行して、 Dumplingを使用してスキーマをエクスポートします。このコマンドには、目的のテーブル スキーマのみをエクスポートするための`--filter`つのパラメーターが含まれています。

{{< copyable "" >}}

```shell
tiup dumpling --host ${host} --port 3306 --user root --password ${password} --filter 'my_db1.table[12]' --no-data --output 's3://my-bucket/schema-backup?region=us-west-2' --filter "mydb.*"
```

上記のコマンドで使用されるパラメーターは次のとおりです。その他のパラメーターについては、 [Dumplingの概要](/dumpling-overview.md)を参照してください。

| パラメータ                  | 説明                                                                                                |
| ---------------------- | ------------------------------------------------------------------------------------------------- |
| `-u`または`--user`        | Aurora MySQL ユーザー                                                                                 |
| `-p`または`--password`    | MySQL ユーザーのパスワード                                                                                  |
| `-P`または`--port`        | MySQL ポート                                                                                         |
| `-h`または`--host`        | MySQL IP アドレス                                                                                     |
| `-t`または`--thread`      | エクスポートに使用されるスレッドの数                                                                                |
| `-o`または`--output`      | エクスポートされたファイルを格納するディレクトリ。ローカル パスまたは[外部ストレージ URL](/br/backup-and-restore-storages.md)をサポート         |
| `-r`または`--row`         | 1 つのファイルの最大行数                                                                                     |
| `-F`                   | 1 つのファイルの最大サイズ (MiB 単位)。推奨値: 256 MiB。                                                             |
| `-B`または`--database`    | エクスポートするデータベースを指定します                                                                              |
| `-T`または`--tables-list` | 指定されたテーブルをエクスポートします                                                                               |
| `-d`または`--no-data`     | データをエクスポートしません。スキーマのみをエクスポートします。                                                                  |
| `-f`または`--filter`      | パターンに一致するテーブルをエクスポートします。 `-f`と`-T`を同時に使用しないでください。構文については、 [テーブルフィルター](/table-filter.md)を参照してください。 |

### ステップ 3. TiDB Lightning構成ファイルを作成する {#step-3-create-the-tidb-lightning-configuration-file}

次のように`tidb-lightning.toml`の構成ファイルを作成します。

{{< copyable "" >}}

```shell
vim tidb-lightning.toml
```

{{< copyable "" >}}

```toml
[tidb]

# The target TiDB cluster information.
host = ${host}                # e.g.: 172.16.32.1
port = ${port}                # e.g.: 4000
user = "${user_name}          # e.g.: "root"
password = "${password}"      # e.g.: "rootroot"
status-port = ${status-port}  # Obtains the table schema information from TiDB status port, e.g.: 10080
pd-addr = "${ip}:${port}"     # The cluster PD address, e.g.: 172.16.31.3:2379. TiDB Lightning obtains some information from PD. When backend = "local", you must specify status-port and pd-addr correctly. Otherwise, the import will be abnormal.

[tikv-importer]
# "local": Default backend. The local backend is recommended to import large volumes of data (1 TiB or more). During the import, the target TiDB cluster cannot provide any service.
# "tidb": The "tidb" backend is recommended to import data less than 1 TiB. During the import, the target TiDB cluster can provide service normally.
backend = "local"

# Set the temporary storage directory for the sorted Key-Value files. The directory must be empty, and the storage space must be greater than the size of the dataset to be imported. For better import performance, it is recommended to use a directory different from `data-source-dir` and use flash storage, which can use I/O exclusively.
sorted-kv-dir = "/mnt/ssd/sorted-kv-dir"

[mydumper]
# The path that stores the snapshot file.
data-source-dir = "${s3_path}"  # e.g.: s3://my-bucket/sql-backup?region=us-west-2

[[mydumper.files]]
# The expression that parses the parquet file.
pattern = '(?i)^(?:[^/]*/)*([a-z0-9_]+)\.([a-z0-9_]+)/(?:[^/]*/)*(?:[a-z0-9\-_.]+\.(parquet))$'
schema = '$1'
table = '$2'
type = '$3'
```

TiDB クラスターで TLS を有効にする必要がある場合は、 [TiDB LightningConfiguration / コンフィグレーション](/tidb-lightning/tidb-lightning-configuration.md)を参照してください。

### ステップ 4. 完全なデータを TiDB にインポートする {#step-4-import-full-data-to-tidb}

1.  TiDB Lightningを使用してターゲット データベースにテーブルを作成します。

    {{< copyable "" >}}

    ```shell
    tiup tidb-lightning -config tidb-lightning.toml -d ./schema -no-schema=false
    ```

2.  `tidb-lightning`を実行してインポートを開始します。コマンド ラインでプログラムを直接起動すると、プロセスが SIGHUP シグナルの受信後に予期せず終了することがあります。この場合、 `nohup`または`screen`ツールを使用してプログラムを実行することをお勧めします。例えば：

    S3 ストレージ パスにアクセスできる SecretKey と AccessKey を環境変数としてDumplingノードに渡します。 `~/.aws/credentials`から資格情報を読み取ることもできます。

    {{< copyable "" >}}

    ```shell
    export AWS_ACCESS_KEY_ID=${access_key}
    export AWS_SECRET_ACCESS_KEY=${secret_key}
    nohup tiup tidb-lightning -config tidb-lightning.toml -no-schema=true > nohup.out 2>&1 &
    ```

3.  インポートの開始後、次のいずれかの方法でインポートの進行状況を確認できます。

    -   `grep`ログのキーワード`progress` 。デフォルトでは、進行状況は 5 分ごとに更新されます。
    -   [監視ダッシュボード](/tidb-lightning/monitor-tidb-lightning.md)で進行状況を確認します。
    -   [TiDB Lightning Web インターフェイス](/tidb-lightning/tidb-lightning-web-interface.md)で進行状況を確認します。

4.  TiDB Lightningがインポートを完了すると、自動的に終了します。 log print `the whole procedure completed`の最後の 5 行が見つかった場合、インポートは成功です。

> **ノート：**
>
> インポートが成功したかどうかに関係なく、ログの最後の行に`tidb lightning exit`が表示されます。これは、 TiDB Lightningが正常に終了したことを意味しますが、必ずしもインポートが成功したことを意味するものではありません。

インポート中に問題が発生した場合は、トラブルシューティングについて[TiDB LightningFAQ](/tidb-lightning/tidb-lightning-faq.md)を参照してください。

## 増分データを TiDB に複製する (オプション) {#replicate-incremental-data-to-tidb-optional}

### 前提条件 {#prerequisites}

-   [DMをインストール](/dm/deploy-a-dm-cluster-using-tiup.md) .
-   [DM に必要なソース データベースとターゲット データベースの権限を取得する](/dm/dm-worker-intro.md) .

### ステップ 1: データ ソースを作成する {#step-1-create-the-data-source}

1.  次のように`source1.yaml`ファイルを作成します。

    {{< copyable "" >}}

    ```yaml
    # Must be unique.
    source-id: "mysql-01"
    # Configures whether DM-worker uses the global transaction identifier (GTID) to pull binlogs. To enable this mode, the upstream MySQL must also enable GTID. If the upstream MySQL service is configured to switch master between different nodes automatically, GTID mode is required.
    enable-gtid: false

    from:
      host: "${host}"         # e.g.: 172.16.10.81
      user: "root"
      password: "${password}" # Supported but not recommended to use plaintext password. It is recommended to use `dmctl encrypt` to encrypt the plaintext password before using it.
      port: 3306
    ```

2.  次のコマンドを実行して、 `tiup dmctl`を使用してデータ ソース構成を DM クラスターに読み込みます。

    {{< copyable "" >}}

    ```shell
    tiup dmctl --master-addr ${advertise-addr} operate-source create source1.yaml
    ```

    上記のコマンドで使用されるパラメーターは、次のとおりです。

    | パラメータ                   | 説明                                                                    |
    | ----------------------- | --------------------------------------------------------------------- |
    | `--master-addr`         | `dmctl`が接続されるクラスタ内の任意の DM マスターの`{advertise-addr}`例: 172.16.10.71:8261 |
    | `operate-source create` | データ ソースを DM クラスターに読み込みます。                                             |

### ステップ 2: 移行タスクを作成する {#step-2-create-the-migration-task}

次のように`task1.yaml`ファイルを作成します。

{{< copyable "" >}}

```yaml
# Task name. Multiple tasks that are running at the same time must each have a unique name.
name: "test"
# Task mode. Options are:
# - full: only performs full data migration.
# - incremental: only performs binlog real-time replication.
# - all: full data migration + binlog real-time replication.
task-mode: "incremental"
# The configuration of the target TiDB database.
target-database:
  host: "${host}"                   # e.g.: 172.16.10.83
  port: 4000
  user: "root"
  password: "${password}"           # Supported but not recommended to use a plaintext password. It is recommended to use `dmctl encrypt` to encrypt the plaintext password before using it.

# Global configuration for block and allow lists. Each instance can reference the configuration by name.
block-allow-list:                     # If the DM version is earlier than v2.0.0-beta.2, use black-white-list.
  listA:                              # Name.
    do-tables:                        # Allow list for the upstream tables to be migrated.
    - db-name: "test_db"              # Name of databases to be migrated.
      tbl-name: "test_table"          # Name of tables to be migrated.

# Configures the data source.
mysql-instances:
  - source-id: "mysql-01"               # Data source ID，i.e., source-id in source1.yaml
    block-allow-list: "listA"           # References the block-allow-list configuration above.
#       syncer-config-name: "global"    # Name of the syncer configuration.
    meta:                               # When task-mode is "incremental" and the downstream database does not have a checkpoint, DM uses the binlog position as the starting point. If the downstream database has a checkpoint, DM uses the checkpoint as the starting point.
      binlog-name: "mysql-bin.000004"   # The binlog position recorded in "Step 1. Export an Aurora snapshot to Amazon S3". When the upstream database has source-replica switching, GTID mode is required.
      binlog-pos: 109227
      # binlog-gtid: "09bec856-ba95-11ea-850a-58f2b4af5188:1-9"

# (Optional) If you need to incrementally replicate data that has already been migrated in the full data migration, you need to enable the safe mode to avoid the incremental data replication error.
   # This scenario is common in the following case: the full migration data does not belong to the data source's consistency snapshot, and after that, DM starts to replicate incremental data from a position earlier than the full migration.
   # syncers:            # The running configurations of the sync processing unit.
   #   global:            # Configuration name.
   #     safe-mode: true  # If this field is set to true, DM changes INSERT of the data source to REPLACE for the target database, and changes UPDATE of the data source to DELETE and REPLACE for the target database. This is to ensure that when the table schema contains a primary key or unique index, DML statements can be imported repeatedly. In the first minute of starting or resuming an incremental replication task, DM automatically enables the safe mode.
```

上記の YAML ファイルは、移行タスクに必要な最小限の構成です。その他の設定項目については、 [DM 拡張タスクConfiguration / コンフィグレーションファイル](/dm/task-configuration-file-full.md)を参照してください。

### ステップ 3.移行タスクを実行する {#step-3-run-the-migration-task}

移行タスクを開始する前に、エラーの可能性を減らすために、次の`check-task`コマンドを実行して、構成が DM の要件を満たしていることを確認することをお勧めします。

{{< copyable "" >}}

```shell
tiup dmctl --master-addr ${advertise-addr} check-task task.yaml
```

その後、 `tiup dmctl`を実行して移行タスクを開始します。

{{< copyable "" >}}

```shell
tiup dmctl --master-addr ${advertise-addr} start-task task.yaml
```

上記のコマンドで使用されるパラメーターは、次のとおりです。

| パラメータ           | 説明                                                                    |
| --------------- | --------------------------------------------------------------------- |
| `--master-addr` | `dmctl`が接続されるクラスタ内の任意の DM マスターの`{advertise-addr}`例: 172.16.10.71:8261 |
| `start-task`    | 移行タスクを開始します。                                                          |

タスクの開始に失敗した場合は、プロンプト メッセージを確認し、構成を修正します。その後、上記のコマンドを再実行してタスクを開始できます。

問題が発生した場合は、 [DM エラー処理](/dm/dm-error-handling.md)および[DMFAQ](/dm/dm-faq.md)を参照してください。

### ステップ 4. 移行タスクのステータスを確認する {#step-4-check-the-migration-task-status}

DM クラスターに進行中の移行タスクとタスクのステータスがあるかどうかを確認するには、 `tiup dmctl`を使用して`query-status`コマンドを実行します。

{{< copyable "" >}}

```shell
tiup dmctl --master-addr ${advertise-addr} query-status ${task-name}
```

結果の詳細な解釈については、 [クエリのステータス](/dm/dm-query-status.md)を参照してください。

### ステップ 5. タスクを監視してログを表示する {#step-5-monitor-the-task-and-view-logs}

移行タスクの履歴ステータスとその他の内部メトリックを表示するには、次の手順を実行します。

TiUP を使用して DM をデプロイしたときに Prometheus、Alertmanager、および Grafana をデプロイした場合は、デプロイ中に指定された IP アドレスとポートを使用してTiUPにアクセスできます。次に、DM ダッシュボードを選択して、DM 関連のモニタリング メトリックを表示できます。

DM が実行されている場合、DM-worker、DM-master、および dmctl は関連情報をログに出力します。これらのコンポーネントのログ ディレクトリは次のとおりです。

-   DM-master: DM-master プロセス パラメータ`--log-file`によって指定されます。 TiUPを使用して DM を展開する場合、ログ ディレクトリはデフォルトで`/dm-deploy/dm-master-8261/log/`です。
-   DM-worker: DM-worker プロセス パラメータ`--log-file`によって指定されます。 TiUPを使用して DM を展開する場合、ログ ディレクトリはデフォルトで`/dm-deploy/dm-worker-8262/log/`です。

## 次は何ですか {#what-s-next}

-   [移行タスクを一時停止します](/dm/dm-pause-task.md) .
-   [移行タスクを再開します](/dm/dm-resume-task.md) .
-   [移行タスクを停止する](/dm/dm-stop-task.md) .
-   [クラスター データ ソースとタスク構成のエクスポートとインポート](/dm/dm-export-import-config.md) .
-   [失敗した DDL ステートメントを処理する](/dm/handle-failed-ddl-statements.md) .

---
title: TiDB Lightning Configuration
summary: Learn about the CLI usage and sample configuration in TiDB Lightning.
---

# TiDB LightningConfiguration / コンフィグレーション {#tidb-lightning-configuration}

このドキュメントでは、グローバル構成とタスク構成のサンプルを提供し、コマンドライン パラメータの使用法について説明します。

## Configuration / コンフィグレーションファイル {#configuration-files}

TiDB Lightningには「global」と「task」の 2 つの構成クラスがあり、それらは互換性のある構造を持っています。それらの区別は、 [サーバーモード](/tidb-lightning/tidb-lightning-web-interface.md)が有効になっている場合にのみ発生します。サーバーモードが無効になっている場合 (デフォルト)、 TiDB Lightningは 1 つのタスクのみを実行し、同じ構成ファイルがグローバル構成とタスク構成の両方に使用されます。

### TiDB Lightning(グローバル) {#tidb-lightning-global}

```toml
### tidb-lightning global configuration

[lightning]
# The HTTP port for displaying the web interface, pulling Prometheus metrics, exposing debug data, and submitting import tasks (in server mode). Setting it to 0 disables the port.
status-addr = ':8289'

# Server mode. Defaults to false, which means an import task starts immediately after you execute the command.
# If this value is set to true, after you execute the command, TiDB Lightning waits until you submit an import task in the web interface.
# See the "TiDB Lightning Web Interface" section for details.
server-mode = false

# Logging
level = "info"
file = "tidb-lightning.log"
max-size = 128 # MB
max-days = 28
max-backups = 14
```

### TiDB Lightning(タスク) {#tidb-lightning-task}

```toml
### tidb-lightning task configuration

[lightning]
# Checks whether the cluster satisfies the minimum requirement before starting.
#check-requirements = true

# The maximum number of engines to be opened concurrently.
# Each table is split into one "index engine" to store indices, and multiple
# "data engines" to store row data. These settings control the maximum
# concurrent number for each type of engines.
# These values affect the memory and disk usage of tikv-importer.
# The sum of these two values must not exceed the max-open-engines setting
# for tikv-importer.
index-concurrency = 2
table-concurrency = 6

# The concurrency number of data. It is set to the number of logical CPU
# cores by default. When deploying together with other components, you can
# set it to 75% of the size of logical CPU cores to limit the CPU usage.
#region-concurrency =

# The maximum I/O concurrency. Excessive I/O concurrency causes an increase in
# I/O latency because the disk's internal buffer is frequently refreshed,
# which causes the cache miss and slows down the read speed. Depending on the storage
# medium, this value might need to be adjusted for optimal performance.
io-concurrency = 5

# The maximum number of non-fatal errors to tolerate before stopping TiDB Lightning.
# Non-fatal errors are localized to a few rows, and ignoring those rows allows the import process to continue.
# Setting this to N means that TiDB Lightning will stop as soon as possible when the (N+1)-th error is encountered.
# The skipped rows will be inserted into tables inside the "task info" schema on the target TiDB, which can be configured below.
max-error = 0
# task-info-schema-name is the name of the schema or database that stores TiDB Lightning execution results.
# To disable error recording, set this to an empty string.
# task-info-schema-name = 'lightning_task_info'

# In parallel import mode, the schema name that stores the meta information for each TiDB Lightning instance in the target cluster. By default, the value is "lightning_metadata".
# Configure this parameter only if parallel import is enabled.
# **Note:**
# - The value set for this parameter must be the same for each TiDB Lightning instance that participates in the same parallel import; otherwise, the correctness of the imported data cannot be ensured.
# - If parallel import mode is enabled, make sure that the user used for import (for the tidb.user configuration) has permissions to create and access the databases corresponding to this configuration.
# - TiDB Lightning removes this schema after the import is completed. So do not use any existing schema name to configure this parameter.
meta-schema-name = "lightning_metadata"

[security]
# Specifies certificates and keys for TLS connections within the cluster.
# Public certificate of the CA. Leave empty to disable TLS.
# ca-path = "/path/to/ca.pem"
# Public certificate of this service.
# cert-path = "/path/to/lightning.pem"
# Private key of this service.
# key-path = "/path/to/lightning.key"

[checkpoint]
# Whether to enable checkpoints.
# While importing data, TiDB Lightning records which tables have been imported, so
# even if TiDB Lightning or another component crashes, you can start from a known
# good state instead of restarting from scratch.
enable = true
# The schema name (database name) to store the checkpoints.
schema = "tidb_lightning_checkpoint"
# Where to store the checkpoints.
#  - file:  store as a local file.
#  - mysql: store into a remote MySQL-compatible database
driver = "file"
# The data source name (DSN) indicating the location of the checkpoint storage.
# For the "file" driver, the DSN is a path. If the path is not specified, TiDB Lightning would
# default to "/tmp/CHECKPOINT_SCHEMA.pb".
# For the "mysql" driver, the DSN is a URL in the form of "USER:PASS@tcp(HOST:PORT)/".
# If the URL is not specified, the TiDB server from the [tidb] section is used to
# store the checkpoints. You should specify a different MySQL-compatible
# database server to reduce the load of the target TiDB cluster.
# dsn = "/tmp/tidb_lightning_checkpoint.pb"
# Whether to keep the checkpoints after all data are imported. If false, the
# checkpoints will be deleted. Keeping the checkpoints can aid debugging but
# will leak metadata about the data source.
# keep-after-success = false

[tikv-importer]
# "local": Physical import mode, used by default. It applies to large dataset import, for example, greater than 1 TiB. However, during the import, downstream TiDB is not available to provide services.
# "tidb": Logical import mode. You can use this mode for small dataset import, for example, smaller than 1 TiB. During the import, downstream TiDB is available to provide services.
# backend = "local"
# Whether to allow importing data to tables with data. The default value is `false`.
# When you use parallel import mode, you must set it to `true`, because multiple TiDB Lightning instances are importing the same table at the same time.
# incremental-import = false

# The listening address of tikv-importer when backend is "importer". Change it to the actual address.
addr = "172.16.31.10:8287"
# Action to do when trying to insert a duplicated entry in the logical import mode.
#  - replace: use new entry to replace the existing entry
#  - ignore: keep the existing entry, and ignore the new entry
#  - error: report error and quit the program
# on-duplicate = "replace"

# Whether to detect and resolve duplicate records (unique key conflict) in the physical import mode.
# The following resolution algorithms are supported:
#  - record: only records duplicate records to the `lightning_task_info.conflict_error_v1` table on the target TiDB. Note that the
#    required version of the target TiKV is no earlier than v5.2.0; otherwise it falls back to 'none'.
#  - none: does not detect duplicate records, which has the best performance of the three algorithms, but might lead to
#    inconsistent data in the target TiDB.
#  - remove: records all duplicate records to the lightning_task_info database, like the 'record' algorithm. But it removes all duplicate records from the target table to ensure a consistent
#    state in the target TiDB.
# duplicate-resolution = 'none'
# The number of KV pairs sent in one request in the physical import mode.
# send-kv-pairs = 32768
# The directory of local KV sorting in the physical import mode. If the disk
# performance is low (such as in HDD), it is recommended to set the directory
# on a different disk from `data-source-dir` to improve import speed.
# sorted-kv-dir = ""
# The concurrency that TiKV writes KV data in the physical import mode.
# When the network transmission speed between TiDB Lightning and TiKV
# exceeds 10 Gigabit, you can increase this value accordingly.
# range-concurrency = 16
# Limits the bandwidth in which TiDB Lightning writes data into each TiKV
# node in the physical import mode. 0 by default, which means no limit.
# store-write-bwlimit = "128MiB"

# Specifies the disk quota for local temporary files when physical import mode is used.
# When the disk quota is insufficient, TiDB Lightning stops reading source data and writing temporary files,
# but prioritizes writing the already sorted key-value pairs to TiKV.
# After TiDB Lightning deletes the local temporary files, the import process continues.
# This option takes effect only when you set the `backend` option to `local`.
# The default value is `MaxInt64` bytes, that is, 9223372036854775807 bytes.
# disk-quota = "10GB"

[mydumper]
# Block size for file reading. Keep it longer than the longest string of the data source.
read-block-size = "64KiB" # default value

# The engine file needs to be imported sequentially. Due to parallel processing,
# multiple data engines will be imported at nearly the same time, and this
# creates a queue and wastes resources. Therefore, TiDB Lightning slightly
# increases the size of the first few batches to properly distribute
# resources. The scale up factor is controlled by this parameter, which
# expresses the ratio of duration between the "import" and "write" steps
# with full concurrency. This can be calculated by using the ratio
# (import duration/write duration) of a single table of size around 1 GiB.
# The exact timing can be found in the log. If "import" is faster, the batch
# size variance is smaller, and a ratio of zero means a uniform batch size.
# This value should be in the range (0 <= batch-import-ratio < 1).
batch-import-ratio = 0.75

# Local source data directory or the URL of the external storage.
data-source-dir = "/data/my_database"

# The character set of the schema files, containing CREATE TABLE statements;
# only supports one of:
#  - utf8mb4: the schema files must be encoded as UTF-8; otherwise, an error is reported.
#  - gb18030: the schema files must be encoded as GB-18030; otherwise,
#             an error is reported
#  - auto:    (default) automatically detects whether the schema is UTF-8 or
#             GB-18030. An error is reported if the encoding is neither.
#  - binary:  do not try to decode the schema files
character-set = "auto"

# Specifies the character set of the source data file. Lightning converts the source file from the specified character set to UTF-8 encoding when importing.
# Currently, this configuration only specifies the character set of the CSV files with the following options supported:
# - utf8mb4: Indicates that the source data file uses UTF-8 encoding.
# - GB18030: Indicates that the source data file uses the GB-18030 encoding.
# - GBK: The source data file uses GBK encoding (GBK encoding is an extension of the GB-2312 character set, also known as Code Page 936).
# - binary: Indicates that Lightning does not convert the encoding (by default).
# If left blank, the default value "binary" is used, that is to say, Lightning does not convert the encoding.
# Note that Lightning does not predict about the character set of the source data file and only converts the source file and import the data based on this configuration.
# If the value of this configuration is not the same as the actual encoding of the source data file, a failed import, data loss or data disorder might appear.
data-character-set = "binary"
# Specifies the replacement character in case of incompatible characters during the character set conversion of the source data file.
# This configuration must not be duplicated with field separators, quote definers, and line breaks.
# The default value is "\uFFFD", which is the "error" Rune or Unicode replacement character in UTF-8 encoding.
# Changing the default value might result in potential degradation of parsing performance for the source data file.
data-invalid-char-replace = "\uFFFD"

# the input data in a "strict" format speeds up processing.
# "strict-format = true" requires that:
# in CSV, every value cannot contain literal new lines (U+000A and U+000D, or \r and \n) even
# when quoted, which means new lines are strictly used to separate rows.
# "Strict" format allows TiDB Lightning to quickly locate split positions of a large file for parallel processing.
# However, if the input data is not "strict", it may split a valid data in half and
# corrupt the result.
# The default value is false for safety instead of speed.
strict-format = false

# If strict-format is true, TiDB Lightning splits large CSV files into multiple chunks to process in
# parallel. max-region-size is the maximum size of each chunk after splitting.
# max-region-size = "256MiB" # default value

# Only import tables if these wildcard rules are matched. See the corresponding section for details.
filter = ['*.*', '!mysql.*', '!sys.*', '!INFORMATION_SCHEMA.*', '!PERFORMANCE_SCHEMA.*', '!METRICS_SCHEMA.*', '!INSPECTION_SCHEMA.*']

# Configures how CSV files are parsed.
[mydumper.csv]
# Separator between fields. Must not be empty.
separator = ','
# Quoting delimiter. Empty value means no quoting.
delimiter = '"'
# Line terminator. Empty value means both "\n" (LF) and "\r\n" (CRLF) are line terminators.
terminator = ''
# Whether the CSV files contain a header.
# If `header` is true, the first line will be skipped.
header = true
# Whether the CSV contains any NULL value.
# If `not-null` is true, all columns from CSV cannot be NULL.
not-null = false
# When `not-null` is false (that is, CSV can contain NULL),
# fields equal to this value will be treated as NULL.
null = '\N'
# Whether to interpret backslash escapes inside fields.
backslash-escape = true
# If a line ends with a separator, remove it.
trim-last-separator = false

# [[mydumper.files]]
# Expression used for parsing AWS Aurora parquet files
# pattern = '(?i)^(?:[^/]*/)*([a-z0-9_]+)\.([a-z0-9_]+)/(?:[^/]*/)*(?:[a-z0-9\-_.]+\.(parquet))$'
# schema = '$1'
# table = '$2'
# type = '$3'

[tidb]
# Configuration of any TiDB server from the cluster.
host = "172.16.31.1"
port = 4000
user = "root"
# Configure the password to connect to TiDB. The password can either be plaintext or Base64 encoded.
password = ""
# Table schema information is fetched from TiDB via this status-port.
status-port = 10080
# Address of any PD server from the cluster.
pd-addr = "172.16.31.4:2379"
# tidb-lightning imports TiDB as a library and generates some logs itself.
# This setting controls the log level of the TiDB library.
log-level = "error"

# Sets the TiDB session variable to speed up the Checksum and Analyze operations.
# See https://pingcap.com/docs/dev/reference/performance/statistics/#control-analyze-concurrency
# for the meaning of each setting
build-stats-concurrency = 20
distsql-scan-concurrency = 100
index-serial-scan-concurrency = 20
checksum-table-concurrency = 16

# The default SQL mode used to parse and execute the SQL statements.
sql-mode = "ONLY_FULL_GROUP_BY,NO_ENGINE_SUBSTITUTION"
# Sets maximum packet size allowed for SQL connections.
# Set this to 0 to automatically fetch the `max_allowed_packet` variable from server on every connection.
max-allowed-packet = 67_108_864

# Whether to use TLS for SQL connections. Valid values are:
#  * ""            - force TLS (same as "cluster") if [tidb.security] section is populated, otherwise same as "false"
#  * "false"       - disable TLS
#  * "cluster"     - force TLS and verify the server's certificate with the CA specified in the [tidb.security] section
#  * "skip-verify" - force TLS but do not verify the server's certificate (insecure!)
#  * "preferred"   - same as "skip-verify", but if the server does not support TLS, fallback to unencrypted connection
# tls = ""

# Specifies certificates and keys for TLS-enabled MySQL connections.
# Defaults to a copy of the [security] section.
# [tidb.security]
# Public certificate of the CA. Set to empty string to disable TLS for SQL.
# ca-path = "/path/to/ca.pem"
# Public certificate of this service. Default to copy of `security.cert-path`
# cert-path = "/path/to/lightning.pem"
# Private key of this service. Default to copy of `security.key-path`
# key-path = "/path/to/lightning.key"

# In the physical import mode, when data importing is complete, tidb-lightning can
# automatically perform the Checksum and Analyze operations. It is recommended
# to leave these as true in the production environment.
# The execution order: Checksum -> Analyze.
# In the logical import mode, Checksum and Analyze is not needed, and they are always
# skipped in the actual operation.
[post-restore]
# Specifies whether to perform `ADMIN CHECKSUM TABLE <table>` for each table to verify data integrity after importing.
# The following options are available:
# - "required" (default value): Perform admin checksum. If checksum fails, TiDB Lightning will exit with failure.
# - "optional": Perform admin checksum. If checksum fails, TiDB Lightning will report a WARN log but ignore any error.
# - "off": Do not perform checksum.
# Note that since v4.0.8, the default value has changed from "true" to "required".
# For backward compatibility, bool values "true" and "false" are also allowed for this field.
# "true" is equivalent to "required" and "false" is equivalent to "off".
checksum = "required"
# Specifies whether to perform `ANALYZE TABLE <table>` for each table after checksum is done.
# Options available for this field are the same as `checksum`. However, the default value for this field is "optional".
analyze = "optional"

# If the value is set to `true`, a level-1 compaction is performed
# every time a table is imported.
# The default value is `false`.
level-1-compact = false

# If the value is set to `true`, a full compaction on the whole
# TiKV cluster is performed at the end of the import.
# The default value is `false`.
compact = false

# Configures the background periodic actions.
# Supported units: h (hour), m (minute), s (second).
[cron]
# Duration between which TiDB Lightning automatically refreshes the import mode
# status. Should be shorter than the corresponding TiKV setting.
switch-mode = "5m"
# Duration between which an import progress is printed to the log.
log-progress = "5m"
# The time interval for checking the local disk quota when you use the physical import mode.
# The default value is 60 seconds.
# check-disk-quota = "60s"
```

## コマンド ライン パラメータ {#command-line-parameters}

### <code>tidb-lightning</code>の使い方 {#usage-of-code-tidb-lightning-code}

| パラメータ                                                                      | 説明                                                                                              | 対応設定                           |
| :------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------- | :----------------------------- |
| --config*ファイル*                                                             | *ファイル*からグローバル構成を読み取ります。指定しない場合、デフォルトの構成が使用されます。                                                 |                                |
| -V                                                                         | プログラムのバージョンを出力します                                                                               |                                |
| -d*ディレクトリ*                                                                 | 読み取るデータ ダンプのディレクトリまたは[外部ストレージ URL](/br/backup-and-restore-storages.md)つ                         | `mydumper.data-source-dir`     |
| -L*レベル*                                                                    | ログ レベル: debug、info、warn、error、fatal (デフォルト = info)                                              | `lightning.log-level`          |
| -f*ルール*                                                                    | [テーブル フィルター ルール](/table-filter.md) (複数指定可)                                                      | `mydumper.filter`              |
| --backend*<a href="/tidb-lightning/tidb-lightning-overview.md">バックエンド</a>* | インポート モードを選択します。 `local`は物理的なインポート モードを指します。 `tidb`は、論理インポート モードを指します。                          | `local`                        |
| --log-file*ファイル*                                                           | ログ ファイルのパス。デフォルトでは`/tmp/lightning.log.{timestamp}`です。 「-」に設定すると、ログ ファイルが stdout に出力されることを意味します。 | `lightning.log-file`           |
| --status-addr *ip:ポート*                                                     | TiDB Lightningサーバーのリッスン アドレス                                                                    | `lightning.status-port`        |
| --インポーター*ホスト:ポート*                                                          | TiKV インポーターの住所                                                                                  | `tikv-importer.addr`           |
| --pd-urls*ホスト:ポート*                                                         | PD エンドポイント アドレス                                                                                 | `tidb.pd-addr`                 |
| --tidb-host*ホスト*                                                           | TiDBサーバーホスト                                                                                     | `tidb.host`                    |
| --tidb-port*ポート*                                                           | TiDBサーバーポート (デフォルト = 4000)                                                                      | `tidb.port`                    |
| --tidb-status*ポート*                                                         | TiDB ステータス ポート (デフォルト = 10080)                                                                  | `tidb.status-port`             |
| --tidb-user*ユーザー*                                                          | TiDB に接続するためのユーザー名                                                                              | `tidb.user`                    |
| --tidb-password*パスワード*                                                     | TiDB に接続するためのパスワード。パスワードは、プレーンテキストまたは Base64 エンコードのいずれかです。                                      | `tidb.password`                |
| --enable-checkpoint*ブール*                                                   | チェックポイントを有効にするかどうか (デフォルト = true)                                                               | `checkpoint.enable`            |
| --分析*レベル*                                                                  | インポート後にテーブルを分析します。使用可能な値は、「required」、「optional」(デフォルト値)、および「off」です。                             | `post-restore.analyze`         |
| -- チェックサム*レベル*                                                             | インポート後にチェックサムを比較します。使用可能な値は、「required」(デフォルト値)、「optional」、および「off」です。                           | `post-restore.checksum`        |
| --check-requirements*ブール*                                                  | 開始前にクラスターのバージョンの互換性を確認する (デフォルト = true)                                                         | `lightning.check-requirements` |
| --ca*ファイル*                                                                 | TLS 接続の CA 証明書パス                                                                                | `security.ca-path`             |
| --証明書*ファイル*                                                                | TLS 接続の証明書パス                                                                                    | `security.cert-path`           |
| --キー*ファイル*                                                                 | TLS 接続の秘密鍵パス                                                                                    | `security.key-path`            |
| -- サーバーモード                                                                 | サーバーモードでTiDB Lightningを起動する                                                                     | `lightning.server-mode`        |

コマンド ライン パラメーターと構成ファイル内の対応する設定の両方が指定されている場合は、コマンド ライン パラメーターが使用されます。たとえば、 `./tidb-lightning -L debug --config cfg.toml`を実行すると、 `cfg.toml`の内容に関係なく、常にログ レベルが「debug」に設定されます。

## <code>tidb-lightning-ctl</code>使い方 {#usage-of-code-tidb-lightning-ctl-code}

このツールは、次のパラメーターのいずれかを指定して、さまざまなアクションを実行できます。

| パラメータ                               | 説明                                              |
| :---------------------------------- | :---------------------------------------------- |
| - コンパクト                             | 完全な圧縮を実行します                                     |
| --switch-mode*モード*                  | すべての TiKV ストアを特定のモードに切り替えます: 通常、インポート           |
| --fetch-mode                        | すべての TiKV ストアの現在のモードを出力します                      |
| -- インポートエンジン*uuid*                  | 閉じたエンジン ファイルを TiKV インポーターから TiKV クラスターにインポートします |
| --cleanup-engine *uuid*             | エンジン ファイルを TiKV Importer から削除します                |
| --checkpoint-dump*フォルダー*            | 現在のチェックポイントを CSV としてフォルダーにダンプします                |
| *--checkpoint* -error-destroy テーブル名 | チェックポイントを削除し、エラーが発生した場合はテーブルを削除します              |
| *--checkpoint* -error-ignore テーブル名  | 指定されたテーブルに関連するチェックポイントに記録されたエラーを無視します           |
| *--checkpoint* -remove テーブル名        | テーブルのチェックポイントを無条件に削除します                         |

*tablename*は、形式`` `db`.`tbl` `` (逆引用符を含む) の修飾テーブル名か、キーワード「all」のいずれかでなければなりません。

さらに、上記のセクションで説明した`tidb-lightning`のすべてのパラメーターは`tidb-lightning-ctl`で有効です。

---
title: Upgrade TiDB Using TiUP
summary: Learn how to upgrade TiDB using TiUP.
---

# TiUP を使用して TiDB をアップグレードする {#upgrade-tidb-using-tiup}

このドキュメントは、次のアップグレード パスを対象としています。

-   TiDB 4.0 バージョンから TiDB 5.4 バージョンにアップグレードします。
-   TiDB 5.0 バージョンから TiDB 5.4 バージョンにアップグレードします。
-   TiDB 5.1 バージョンから TiDB 5.4 バージョンにアップグレードします。
-   TiDB 5.2 バージョンから TiDB 5.4 バージョンにアップグレードします。
-   TiDB 5.3 バージョンから TiDB 5.4 バージョンにアップグレードします。

> **警告：**
>
> -   TiFlash を 5.3 より前のバージョンから 5.3 以降にオンラインでアップグレードすることはできません。代わりに、最初に初期バージョンのすべての TiFlash インスタンスを停止してから、クラスターをオフラインでアップグレードする必要があります。他のコンポーネント (TiDB や TiKV など) がオンライン アップグレードをサポートしていない場合は、 [オンラインアップグレード](#online-upgrade)の警告の指示に従ってください。
> -   DDL ステートメントがクラスターで実行されているときは、 **TiDB**クラスターをアップグレードしないでください (通常、 `ADD INDEX`のような時間のかかる DDL ステートメントや列の型の変更のため)。
> -   アップグレードの前に、 [`ADMIN SHOW DDL`](/sql-statements/sql-statement-admin-show-ddl.md)コマンドを使用して、TiDB クラスターに進行中の DDL ジョブがあるかどうかを確認することをお勧めします。クラスターに DDL ジョブがある場合、クラスターをアップグレードするには、DDL の実行が完了するまで待つか、 [`ADMIN CANCEL DDL`](/sql-statements/sql-statement-admin-cancel-ddl.md)コマンドを使用して DDL ジョブをキャンセルしてからクラスターをアップグレードします。
> -   また、クラスターのアップグレード中は、DDL ステートメントを実行し**ない**でください。そうしないと、未定義の動作の問題が発生する可能性があります。

> **ノート：**
>
> アップグレードするクラスターが v3.1 またはそれ以前のバージョン (v3.0 または v2.1) である場合、v5.4 またはそのパッチ バージョンへの直接アップグレードはサポートされていません。最初にクラスターを v4.0 にアップグレードし、次に v5.4 にアップグレードする必要があります。

## アップグレードの注意事項 {#upgrade-caveat}

-   TiDB は現在、バージョンのダウングレードまたはアップグレード後の以前のバージョンへのロールバックをサポートしていません。
-   TiDB Ansible を使用して管理される v4.0 クラスターの場合、クラスターを TiUP ( `tiup cluster` ) にインポートして、 [TiUP (v4.0) を使用して TiDB をアップグレードする](https://docs.pingcap.com/tidb/v4.0/upgrade-tidb-using-tiup#import-tidb-ansible-and-the-inventoryini-configuration-to-tiup)に従って新しい管理を行う必要があります。その後、このドキュメントに従ってクラスターを v5.4 またはそのパッチ バージョンにアップグレードできます。
-   3.0 より前のバージョンを 5.4 に更新するには:
    1.  [TiDB アンシブル](https://docs.pingcap.com/tidb/v3.0/upgrade-tidb-using-ansible)を使用して、このバージョンを 3.0 に更新します。
    2.  TiUP ( `tiup cluster` ) を使用して、TiDB Ansible 構成をインポートします。
    3.  [TiUP (v4.0) を使用して TiDB をアップグレードする](https://docs.pingcap.com/tidb/v4.0/upgrade-tidb-using-tiup#import-tidb-ansible-and-the-inventoryini-configuration-to-tiup)に従って、バージョン 3.0 を 4.0 に更新します。
    4.  このドキュメントに従って、クラスターを v5.4 にアップグレードします。
-   TiDB Binlog、TiCDC、TiFlash、およびその他のコンポーネントのバージョンのアップグレードをサポートします。
-   異なるバージョンの詳細な互換性の変更については、各バージョンの[リリースノート](/releases/release-notes.md)を参照してください。対応するリリース ノートの「互換性の変更」セクションに従って、クラスター構成を変更します。
-   v5.3 より前のバージョンから v5.3 以降のバージョンにアップグレードするクラスターの場合、デフォルトでデプロイされた Prometheus は v2.8.1 から v2.27.1 にアップグレードされます。 Prometheus v2.27.1 は、より多くの機能を提供し、セキュリティの問題を修正します。 v2.8.1 と比較して、v2.27.1 のアラート時間の表現が変更されました。詳細については、 [プロメテウスコミット](https://github.com/prometheus/prometheus/commit/7646cbca328278585be15fa615e22f2a50b47d06)を参照してください。

## 準備 {#preparations}

このセクションでは、TiUP および TiUPクラスタコンポーネントのアップグレードを含め、TiDB クラスターをアップグレードする前に必要な準備作業を紹介します。

### 手順 1: TiUP または TiUP オフライン ミラーをアップグレードする {#step-1-upgrade-tiup-or-tiup-offline-mirror}

TiDB クラスターをアップグレードする前に、まず TiUP または TiUP ミラーをアップグレードする必要があります。

#### TiUP および TiUPクラスタのアップグレード {#upgrade-tiup-and-tiup-cluster}

> **ノート：**
>
> アップグレードするクラスタの制御マシンが`https://tiup-mirrors.pingcap.com`にアクセスできない場合は、このセクションをスキップして[TiUP オフライン ミラーのアップグレード](#upgrade-tiup-offline-mirror)を参照してください。

1.  TiUPのバージョンアップ。 TiUPのバージョンは`1.9.0`以降を推奨します。

    {{< copyable "" >}}

    ```shell
    tiup update --self
    tiup --version
    ```

2.  TiUPクラスタのバージョンをアップグレードします。 TiUP クラスタのバージョンは`1.9.0`以降を推奨します。

    {{< copyable "" >}}

    ```shell
    tiup update cluster
    tiup cluster --version
    ```

#### TiUP オフライン ミラーのアップグレード {#upgrade-tiup-offline-mirror}

> **ノート：**
>
> アップグレードするクラスターがオフラインの方法を使用せずにデプロイされた場合は、この手順をスキップしてください。

[TiUP を使用して TiDBクラスタをデプロイする - TiUP をオフラインでデプロイ](/production-deployment-using-tiup.md#deploy-tiup-offline)を参照して、新しいバージョンの TiUP ミラーをダウンロードし、制御マシンにアップロードします。 `local_install.sh`を実行すると、TiUP は上書きアップグレードを完了します。

{{< copyable "" >}}

```shell
tar xzvf tidb-community-server-${version}-linux-amd64.tar.gz
sh tidb-community-server-${version}-linux-amd64/local_install.sh
source /home/tidb/.bash_profile
```

上書きアップグレード後、以下のコマンドを実行して TiUP クラスタコンポーネントをアップグレードします。

{{< copyable "" >}}

```shell
tiup update cluster
```

これで、オフライン ミラーが正常にアップグレードされました。上書き後のTiUP操作でエラーが発生した場合、 `manifest`が更新されていない可能性があります。 TiUP を再度実行する前に`rm -rf ~/.tiup/manifests/*`を試すことができます。

### 手順 2: TiUP トポロジ構成ファイルを編集する {#step-2-edit-tiup-topology-configuration-file}

> **ノート：**
>
> 次のいずれかの状況に該当する場合は、この手順をスキップしてください。
>
> -   元のクラスターの構成パラメーターを変更していません。または、 `tiup cluster`を使用して構成パラメーターを変更しましたが、それ以上の変更は必要ありません。
> -   アップグレード後、変更されていない構成アイテムに対して v5.4 のデフォルトのパラメーター値を使用したいと考えています。

1.  `vi`編集モードに入り、トポロジ ファイルを編集します。

    {{< copyable "" >}}

    ```shell
    tiup cluster edit-config <cluster-name>
    ```

2.  [トポロジー](https://github.com/pingcap/tiup/blob/master/embed/examples/cluster/topology.example.yaml)構成テンプレートのフォーマットを参照し、変更するパラメーターをトポロジー ファイルの`server_configs`セクションに入力します。

3.  変更後、 <kbd>:</kbd> + <kbd>w</kbd> + <kbd>q</kbd>を入力して変更を保存し、編集モードを終了します。 <kbd>Y</kbd>を入力して変更を確認します。

> **ノート：**
>
> クラスターを v5.4 にアップグレードする前に、v4.0 で変更したパラメーターが v5.4 で互換性があることを確認してください。詳細については、 [TiKVConfiguration / コンフィグレーションファイル](/tikv-configuration-file.md)を参照してください。

### ステップ 3: 現在のクラスターのヘルス ステータスを確認する {#step-3-check-the-health-status-of-the-current-cluster}

アップグレード中の未定義の動作やその他の問題を回避するには、アップグレード前に現在のクラスターのリージョンのヘルス ステータスを確認することをお勧めします。これを行うには、 `check`サブコマンドを使用できます。

{{< copyable "" >}}

```shell
tiup cluster check <cluster-name> --cluster
```

コマンド実行後、「リージョンの状態」チェック結果が出力されます。

-   結果が「すべてのリージョンが正常」である場合、現在のクラスター内のすべてのリージョンは正常であり、アップグレードを続行できます。
-   結果が「リージョンが完全に正常ではありません: m miss-peer、n pending-peer」の場合、「他の操作の前に異常なリージョンを修正してください」。現在のクラスターの一部のリージョンが異常です。チェック結果が「すべてのリージョンが正常」になるまで、異常をトラブルシューティングする必要があります。その後、アップグレードを続行できます。

## TiDB クラスターをアップグレードする {#upgrade-the-tidb-cluster}

このセクションでは、TiDB クラスターをアップグレードし、アップグレード後にバージョンを確認する方法について説明します。

### TiDB クラスターを指定されたバージョンにアップグレードする {#upgrade-the-tidb-cluster-to-a-specified-version}

クラスタは、オンライン アップグレードとオフライン アップグレードの 2 つの方法のいずれかでアップグレードできます。

デフォルトでは、TiUPクラスタはオンライン方式を使用して TiDB クラスターをアップグレードします。これは、TiDB クラスターがアップグレード プロセス中にサービスを提供できることを意味します。オンライン方式では、リーダーはアップグレードと再起動の前に各ノードで 1 つずつ移行されます。したがって、大規模なクラスターの場合、アップグレード操作全体を完了するには長い時間がかかります。

メンテナンスのためにデータベースを停止するメンテナンス ウィンドウがアプリケーションにある場合は、オフライン アップグレード方法を使用して、アップグレード操作をすばやく実行できます。

#### オンラインアップグレード {#online-upgrade}

{{< copyable "" >}}

```shell
tiup cluster upgrade <cluster-name> <version>
```

たとえば、クラスターを v5.4.3 にアップグレードする場合:

{{< copyable "" >}}

```shell
tiup cluster upgrade <cluster-name> v5.4.3
```

> **ノート：**
>
> -   オンライン アップグレードでは、すべてのコンポーネントが 1 つずつアップグレードされます。 TiKV のアップグレード中、TiKV インスタンス内のすべてのリーダーは、インスタンスを停止する前に削除されます。デフォルトのタイムアウト時間は 5 分 (300 秒) です。このタイムアウト時間が経過すると、インスタンスは直接停止されます。
>
> -   `--force`パラメーターを使用して、リーダーを削除せずにすぐにクラスターをアップグレードできます。ただし、アップグレード中に発生するエラーは無視されます。つまり、アップグレードの失敗は通知されません。したがって、 `--force`パラメータは注意して使用してください。
>
> -   安定したパフォーマンスを維持するには、インスタンスを停止する前に、TiKV インスタンス内のすべてのリーダーが削除されていることを確認してください。 `--transfer-timeout`を`--transfer-timeout 3600` (単位: 秒) など、より大きな値に設定できます。
>
> <!---->
>
> -   TiFlash を 5.3 より前のバージョンから 5.3 以降にアップグレードするには、TiFlash を停止してからアップグレードする必要があります。次の手順は、他のコンポーネントを中断することなく TiFlash をアップグレードするのに役立ちます。
>     1.  TiFlash インスタンスを停止します: `tiup cluster stop <cluster-name> -R tiflash`
>     2.  再起動せずに TiDB クラスターをアップグレードします (ファイルの更新のみ): `tiup cluster upgrade <cluster-name> <version> --offline`
>     3.  TiDB クラスターをリロードします。 `tiup cluster reload <cluster-name>` .リロード後、TiFlash インスタンスが開始されるため、手動で開始する必要はありません。

#### オフライン アップグレード {#offline-upgrade}

1.  オフライン アップグレードの前に、まずクラスター全体を停止する必要があります。

    {{< copyable "" >}}

    ```shell
    tiup cluster stop <cluster-name>
    ```

2.  `upgrade`コマンドと`--offline`オプションを使用して、オフライン アップグレードを実行します。

    {{< copyable "" >}}

    ```shell
    tiup cluster upgrade <cluster-name> <version> --offline
    ```

3.  アップグレード後、クラスターは自動的に再起動されません。 `start`コマンドを使用して再起動する必要があります。

    {{< copyable "" >}}

    ```shell
    tiup cluster start <cluster-name>
    ```

### クラスターのバージョンを確認する {#verify-the-cluster-version}

`display`コマンドを実行して、最新のクラスター バージョン`TiDB Version`を表示します。

{{< copyable "" >}}

```shell
tiup cluster display <cluster-name>
```

```
Cluster type:       tidb
Cluster name:       <cluster-name>
Cluster version:    v5.4.3
```

> **ノート：**
>
> デフォルトでは、TiUP と TiDB は使用状況の詳細を PingCAP と共有して、製品の改善方法を理解できるようにします。共有される内容と共有を無効にする方法の詳細については、 [テレメトリー](/telemetry.md)を参照してください。

## FAQ {#faq}

このセクションでは、TiUP を使用して TiDB クラスターを更新するときに発生する一般的な問題について説明します。

### エラーが発生してアップグレードが中断された場合、このエラーを修正した後にアップグレードを再開するにはどうすればよいですか? {#if-an-error-occurs-and-the-upgrade-is-interrupted-how-to-resume-the-upgrade-after-fixing-this-error}

`tiup cluster upgrade`コマンドを再実行して、アップグレードを再開します。アップグレード操作により、以前にアップグレードされたノードが再起動されます。アップグレードしたノードを再起動したくない場合は、 `replay`サブコマンドを使用して操作を再試行します。

1.  `tiup cluster audit`を実行して操作記録を表示します。

    {{< copyable "" >}}

    ```shell
    tiup cluster audit
    ```

    失敗したアップグレード操作レコードを見つけて、この操作レコードの ID を保持します。 ID は、次のステップの`<audit-id>`の値です。

2.  `tiup cluster replay <audit-id>`を実行して、対応する操作を再試行します。

    {{< copyable "" >}}

    ```shell
    tiup cluster replay <audit-id>
    ```

### エビクト リーダーは、アップグレード中に長時間待機しました。迅速なアップグレードのためにこの手順をスキップするにはどうすればよいですか? {#the-evict-leader-has-waited-too-long-during-the-upgrade-how-to-skip-this-step-for-a-quick-upgrade}

`--force`を指定できます。その後、PD リーダーの転送と TiKV リーダーの削除のプロセスは、アップグレード中にスキップされます。クラスターを直接再起動してバージョンを更新するため、オンラインで実行されるクラスターに大きな影響を与えます。コマンドは次のとおりです。

{{< copyable "" >}}

```shell
tiup cluster upgrade <cluster-name> <version> --force
```

### TiDB クラスターをアップグレードした後、pd-ctl などのツールのバージョンを更新する方法を教えてください。 {#how-to-update-the-version-of-tools-such-as-pd-ctl-after-upgrading-the-tidb-cluster}

TiUP を使用して、対応するバージョンの`ctl`つのコンポーネントをインストールすることで、ツールのバージョンをアップグレードできます。

{{< copyable "" >}}

```shell
tiup install ctl:v5.4.3
```

## TiDB 5.4 の互換性の変更 {#tidb-5-4-compatibility-changes}

-   互換性の変更については、TiDB 5.4 リリース ノートを参照してください。
-   TiDB Binlogを使用してクラスターにローリング更新を適用するときは、新しいクラスター化インデックス テーブルを作成しないようにしてください。

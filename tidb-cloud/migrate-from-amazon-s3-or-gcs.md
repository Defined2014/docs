---
title: Import or Migrate from Amazon S3 or GCS to TiDB Cloud
summary: Learn how to import or migrate data from Amazon Simple Storage Service (Amazon S3) or Google Cloud Storage (GCS) to TiDB Cloud.
---

# Amazon S3 または GCS からTiDB Cloudへのインポートまたは移行 {#import-or-migrate-from-amazon-s3-or-gcs-to-tidb-cloud}

このドキュメントでは、データをTiDB Cloudにインポートまたは移行するためのステージング領域として Amazon Simple Storage Service (Amazon S3) または Google Cloud Storage (GCS) を使用する方法について説明します。

> **ノート：**
>
> アップストリーム データベースが Amazon Aurora MySQL の場合は、このドキュメントを参照する代わりに、 [Amazon Aurora MySQL からTiDB Cloudに一括移行する](/tidb-cloud/migrate-from-aurora-bulk-import.md)の手順に従ってください。

## Amazon S3 からTiDB Cloudへのインポートまたは移行 {#import-or-migrate-from-amazon-s3-to-tidb-cloud}

組織が AWS のサービスとしてTiDB Cloudを使用している場合、 TiDB Cloudにデータをインポートまたは移行するためのステージング領域として Amazon S3 を使用できます。

### 前提条件 {#prerequisites}

Amazon S3 からTiDB Cloudにデータを移行する前に、企業所有の AWS アカウントへの管理者アクセス権があることを確認してください。

### ステップ 1.Amazon S3 バケットを作成し、ソースデータファイルを準備する {#step-1-create-an-amazon-s3-bucket-and-prepare-source-data-files}

1.  企業所有の AWS アカウントで Amazon S3 バケットを作成します。

    詳細については、AWS ユーザーガイドの[バケットの作成](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html)を参照してください。

    > **ノート：**
    >
    > エグレス料金とレイテンシーを最小限に抑えるには、Amazon S3 バケットとTiDB Cloudデータベース クラスターを同じリージョンに作成します。

2.  アップストリーム データベースからデータを移行する場合は、最初にソース データをエクスポートする必要があります。

    詳細については、 [MySQL 互換データベースからデータを移行する](/tidb-cloud/migrate-data-into-tidb.md)を参照してください。

3.  ソース データがローカル ファイルにある場合は、Amazon S3 コンソールまたは AWS CLI を使用してファイルを Amazon S3 バケットにアップロードできます。

    -   Amazon S3 コンソールを使用してファイルをアップロードするには、AWS ユーザーガイドの[オブジェクトのアップロード](https://docs.aws.amazon.com/AmazonS3/latest/userguide/upload-objects.html)を参照してください。
    -   AWS CLI を使用してファイルをアップロードするには、次のコマンドを使用します。

        ```shell
        aws s3 sync <Local path> <Amazon S3 bucket URI>
        ```

        例えば：

        ```shell
        aws s3 sync ./tidbcloud-samples-us-west-2/ s3://tidb-cloud-source-data
        ```

> **ノート：**
>
> -   ソース データをTiDB Cloudでサポートされているファイル形式にコピーできることを確認してください。サポートされている形式には、CSV、Dumpling、およびAuroraバックアップ スナップショットが含まれます。ソース ファイルが CSV 形式の場合は、 [TiDB がサポートする命名規則](https://docs.pingcap.com/tidb/stable/migrate-from-csv-using-tidb-lightning#file-name)に従う必要があります。
> -   可能な場合は、大きなソース ファイルを最大サイズ 256 MB の小さなファイルに分割することをお勧めします。これにより、 TiDB Cloudはスレッド間でファイルを並行して読み取ることができるため、インポートのパフォーマンスが向上する可能性があります。

### ステップ 2.Amazon S3 アクセスを構成する {#step-2-configure-amazon-s3-access}

TiDB Cloudが Amazon S3 バケットのソース データにアクセスできるようにするには、 TiDB Cloudのバケット アクセスを設定し、Role-ARN を取得する必要があります。プロジェクト内の 1 つの TiDB クラスターの設定が完了すると、そのプロジェクト内のすべての TiDB クラスターが同じ Role-ARN を使用して Amazon S3 バケットにアクセスできるようになります。

詳細な手順については、 [Amazon S3 アクセスの構成](/tidb-cloud/config-s3-and-gcs-access.md#configure-amazon-s3-access)を参照してください。

### ステップ 3. データをTiDB Cloudにインポートする {#step-3-import-data-into-tidb-cloud}

1.  [TiDB Cloudコンソール](https://tidbcloud.com/)にログインし、[ **Clusters** ] ページに移動します。

2.  ターゲット クラスターを見つけて、クラスター領域の右上隅にある [ **...** ] をクリックし、 [<strong>データのインポート</strong>] を選択します。 [<strong>データのインポート]</strong>ページが表示されます。

3.  [**データのインポート**] ページで、次の情報を入力します。

    -   **データ形式**: データの形式を選択します。
    -   **場所**: `AWS`
    -   **バケット URI** : ソース データのバケット URI を入力します。
    -   **Role ARN** : [ステップ2](#step-2-configure-amazon-s3-access)で取得した Role-ARN を入力します。
    -   **ターゲットクラスタ**: クラスター名とリージョン名が表示されます。

    バケットのリージョンがクラスターと異なる場合は、クロス リージョンのコンプライアンスを確認します。 [**次へ**] をクリックします。

    TiDB Cloudは、指定されたバケット URI でデータにアクセスできるかどうかの検証を開始します。検証後、 TiDB Cloudはデフォルトのファイル命名パターンを使用してデータ ソース内のすべてのファイルをスキャンしようとし、次のページの左側にスキャンの概要結果を返します。 `AccessDenied`エラーが発生した場合は、 [S3 からのデータ インポート中のアクセス拒否エラーのトラブルシューティング](/tidb-cloud/troubleshoot-import-access-denied-error.md)を参照してください。

4.  ファイル パターンを変更し、必要に応じてテーブル フィルター ルールを追加します。

5.  [**次へ**] をクリックします。

6.  [**プレビュー**] ページでインポートするデータを確認し、[<strong>インポートの開始</strong>] をクリックします。

データがインポートされた後、 TiDB Cloudの Amazon S3 アクセスを削除する場合は、 [ステップ 2.Amazon S3 アクセスを構成する](#step-2-configure-amazon-s3-access)で追加したポリシーを削除するだけです。

## GCS からTiDB Cloudへのインポートまたは移行 {#import-or-migrate-from-gcs-to-tidb-cloud}

組織が Google Cloud Platform (GCP) 上のサービスとしてTiDB Cloudを使用している場合、 TiDB Cloudにデータをインポートまたは移行するためのステージング領域として Google Cloud Storage (GCS) を使用できます。

### 前提条件 {#prerequisites}

GCS からTiDB Cloudにデータを移行する前に、次のことを確認してください。

-   企業所有の GCP アカウントへの管理者アクセス権があります。
-   TiDB Cloud管理ポータルへの管理者アクセス権があります。

### ステップ 1. GCS バケットを作成し、ソースデータ ファイルを準備する {#step-1-create-a-gcs-bucket-and-prepare-source-data-files}

1.  企業所有の GCP アカウントに GCS バケットを作成します。

    詳細については、Google Cloud Storage ドキュメントの[ストレージ バケットの作成](https://cloud.google.com/storage/docs/creating-buckets)を参照してください。

2.  アップストリーム データベースからデータを移行する場合は、最初にソース データをエクスポートする必要があります。

    詳細については、 [TiUPをインストールする](/tidb-cloud/migrate-data-into-tidb.md#step-1-install-tiup)および[MySQL 互換データベースからデータをエクスポートする](/tidb-cloud/migrate-data-into-tidb.md#step-2-export-data-from-mysql-compatible-databases)を参照してください。

> **ノート：**
>
> -   ソース データをTiDB Cloudでサポートされているファイル形式にコピーできることを確認してください。サポートされている形式には、CSV、Dumpling、およびAuroraバックアップ スナップショットが含まれます。ソース ファイルが CSV 形式の場合は、 [TiDB がサポートする命名規則](https://docs.pingcap.com/tidb/stable/migrate-from-csv-using-tidb-lightning#file-name)に従う必要があります。
> -   可能な場合は、大きなソース ファイルを最大サイズ 256 MB の小さなファイルに分割することをお勧めします。これにより、 TiDB Cloudがスレッド間でファイルを並行して読み取ることができるようになり、インポートのパフォーマンスが向上します。

### ステップ 2.GCS アクセスを構成する {#step-2-configure-gcs-access}

TiDB クラウドが GCS バケット内のソース データにアクセスできるようにするには、GCP プロジェクトと GCS バケット ペアのサービスとして各TiDB Cloudの GCS アクセスを構成する必要があります。プロジェクト内の 1 つのクラスタの構成が完了すると、そのプロジェクト内のすべてのデータベース クラスタが GCS バケットにアクセスできるようになります。

詳細な手順については、 [GCS アクセスの構成](/tidb-cloud/config-s3-and-gcs-access.md#configure-gcs-access)を参照してください。

### ステップ 3. ソース データ ファイルを GCS にコピーし、データをTiDB Cloudにインポートする {#step-3-copy-source-data-files-to-gcs-and-import-data-into-tidb-cloud}

1.  ソース データ ファイルを GCS バケットにコピーするには、Google Cloud Console または gsutil を使用してデータを GCS バケットにアップロードします。

    -   Google Cloud Console を使用してデータをアップロードするには、Google Cloud Storage ドキュメントの[ストレージ バケットの作成](https://cloud.google.com/storage/docs/creating-buckets)を参照してください。
    -   gsutil を使用してデータをアップロードするには、次のコマンドを使用します。

        ```shell
        gsutil rsync -r <Local path> <GCS URI>
        ```

        例えば：

        ```shell
        gsutil rsync -r ./tidbcloud-samples-us-west-2/ gs://target-url-in-gcs
        ```

2.  TiDB Cloudコンソールから [**クラスター**] ページに移動し、ターゲット クラスターの名前をクリックして、独自の概要ページに移動します。 [<strong>インポート</strong>] 領域で [<strong>データのインポート</strong>] をクリックし、[<strong>データのインポート</strong>] ページでインポート関連の情報を入力します。

> **ノート：**
>
> 送信料金とレイテンシーを最小限に抑えるには、GCS バケットとTiDB Cloudデータベース クラスタを同じリージョンに配置します。

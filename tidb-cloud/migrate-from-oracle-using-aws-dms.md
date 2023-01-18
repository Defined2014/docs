---
title:  Migrate from Amazon RDS for Oracle to TiDB Cloud Using AWS DMS
summary: Learn how to migrate data from Amazon RDS for Oracle into TiDB Cloud Serverless Tier using AWS Database Migration Service (AWS DMS).
---

# AWS DMS を使用して Amazon RDS for Oracle からTiDB Cloudに移行する {#migrate-from-amazon-rds-for-oracle-to-tidb-cloud-using-aws-dms}

このドキュメントでは、AWS Database Migration Service (AWS DMS) を使用して Amazon RDS for Oracle から[TiDB CloudServerless Tier](https://tidbcloud.com/console/clusters/create-cluster)にデータを移行する方法の例を段階的に説明します。

TiDB Cloudと AWS DMS について詳しく知りたい場合は、以下を参照してください。

-   [TiDB Cloud](https://docs.pingcap.com/tidbcloud/)
-   [TiDB 開発者ガイド](https://docs.pingcap.com/tidbcloud/dev-guide-overview)
-   [AWS DMS ドキュメント](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_GettingStarted.html)

## AWS DMS を使用する理由 {#why-use-aws-dms}

AWS DMS は、リレーショナル データベース、データ ウェアハウス、NoSQL データベース、およびその他のタイプのデータ ストアの移行を可能にするクラウド サービスです。

PostgreSQL、Oracle、SQL Server などの異種データベースからTiDB Cloudにデータを移行する場合は、AWS DMS を使用することをお勧めします。

## 導入アーキテクチャ {#deployment-architecture}

大まかに言うと、次の手順に従います。

1.  ソース Amazon RDS for Oracle をセットアップします。
2.  ターゲットを設定します[TiDB CloudServerless Tier](https://tidbcloud.com/console/clusters/create-cluster) 。
3.  AWS DMS を使用してデータ移行 (全負荷) をセットアップします。

次の図は、アーキテクチャの概要を示しています。

![Architecture](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-0.png)

## 前提条件 {#prerequisites}

開始する前に、次の前提条件をお読みください。

-   [AWS DMS の前提条件](/tidb-cloud/migrate-from-mysql-using-aws-dms.md#prerequisites)
-   [AWS クラウド アカウント](https://aws.amazon.com)
-   [TiDB Cloudアカウント](https://tidbcloud.com)
-   [DBeaver](https://dbeaver.io/)

次に、AWS DMS を使用して Amazon RDS for Oracle からTiDB Cloudにデータを移行する方法を学習します。

## ステップ 1.VPC を作成する {#step-1-create-a-vpc}

[AWS コンソール](https://console.aws.amazon.com/vpc/home#vpcs:)にログインし、AWS VPC を作成します。後でこの VPC に Oracle RDS および DMS インスタンスを作成する必要があります。

VPC の作成方法については、 [VPC の作成](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-vpcs.html#Create-VPC)を参照してください。

![Create VPC](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-1.png)

## ステップ 2.Oracle DB インスタンスを作成する {#step-2-create-an-oracle-db-instance}

作成したばかりの VPC に Oracle DB インスタンスを作成し、パスワードを覚えておいてパブリック アクセスを許可します。 AWS Schema Conversion Tool を使用するには、パブリック アクセスを有効にする必要があります。本番環境でパブリック アクセスを許可することはお勧めできません。

Oracle DB インスタンスの作成方法については、 [Oracle DB インスタンスの作成と Oracle DB インスタンス上のデータベースへの接続](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.Oracle.html)を参照してください。

![Create Oracle RDS](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-2.png)

## ステップ 3. Oracle でテーブル データを準備する {#step-3-prepare-the-table-data-in-oracle}

次のスクリプトを使用して、github_events テーブルに 10000 行のデータを作成して入力します。 github イベント データセットを使用して、 [GHアーカイブ](https://gharchive.org/)からダウンロードできます。 10000 行のデータが含まれています。次の SQL スクリプトを使用して、Oracle で実行します。

-   [table_schema_oracle.sql](https://github.com/pingcap-inc/tidb-integration-script/blob/main/aws-dms/oracle_table_schema.sql)
-   [oracle_data.sql](https://github.com/pingcap-inc/tidb-integration-script/blob/main/aws-dms/oracle_data.sql)

SQL スクリプトの実行が終了したら、Oracle でデータを確認します。次の例では、 [DBeaver](https://dbeaver.io/)を使用してデータをクエリします。

![Oracle RDS Data](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-3.png)

## ステップ 4. TiDB Cloud Serverless Tierクラスターを作成する {#step-4-create-a-tidb-cloud-serverless-tier-cluster}

1.  [TiDB Cloudコンソール](https://tidbcloud.com/console/clusters)にログインします。

2.  [無料のServerless Tierクラスターを作成する](/tidb-cloud/tidb-cloud-quickstart.md) .

3.  [**クラスター**](https://tidbcloud.com/console/clusters)ページで、ターゲット クラスタ名をクリックしてその概要ページに移動します。

4.  右上隅にある [**接続**] をクリックします。

5.  [パスワードの**作成] をクリックしてパスワード**を生成し、生成されたパスワードをコピーします。

6.  希望する接続方法とオペレーティング システムを選択し、表示された接続文字列を使用してクラスターに接続します。

## ステップ 5.AWS DMS レプリケーションインスタンスを作成する {#step-5-create-an-aws-dms-replication-instance}

1.  AWS DMS コンソールの[レプリケーション インスタンス](https://console.aws.amazon.com/dms/v2/home#replicationInstances)ページに移動し、対応するリージョンに切り替えます。

2.  VPC に`dms.t3.large`で AWS DMS レプリケーション インスタンスを作成します。

    ![Create AWS DMS Instance](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-8.png)

## ステップ 6. DMS エンドポイントを作成する {#step-6-create-dms-endpoints}

1.  [AWS DMS コンソール](https://console.aws.amazon.com/dms/v2/home)で、左ペインの`Endpoints`メニュー項目をクリックします。

2.  Oracle ソース エンドポイントと TiDB ターゲット エンドポイントを作成します。

    次のスクリーンショットは、ソース エンドポイントの構成を示しています。

    ![Create AWS DMS Source endpoint](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-9.png)

    次のスクリーンショットは、ターゲット エンドポイントの構成を示しています。

    ![Create AWS DMS Target endpoint](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-10.png)

## ステップ 7. スキーマを移行する {#step-7-migrate-the-schema}

この例では、スキーマ定義が単純であるため、AWS DMS がスキーマを自動的に処理します。

AWS Schema Conversion Tool を使用してスキーマを移行する場合は、 [AWS SCT のインストール](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Installing.html#CHAP_Installing.Procedure)を参照してください。

詳細については、 [AWS SCT を使用してソース スキーマをターゲット データベースに移行する](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_GettingStarted.SCT.html)を参照してください。

## ステップ 8. データベース移行タスクを作成する {#step-8-create-a-database-migration-task}

1.  AWS DMS コンソールで、 [データ移行タスク](https://console.aws.amazon.com/dms/v2/home#tasks)ページに移動します。お住まいの地域に切り替えます。次に、ウィンドウの右上隅にある [**タスクの作成**] をクリックします。

    ![Create task](/media/tidb-cloud/aws-dms-to-tidb-cloud-create-task.png)

2.  データベース移行タスクを作成し、**選択ルール**を指定します。

    ![Create AWS DMS migration task](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-11.png)

    ![AWS DMS migration task selection rules](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-12.png)

3.  タスクを作成して開始し、タスクが完了するまで待ちます。

4.  **テーブル統計**をクリックして、テーブルを確認します。スキーマ名は`ADMIN`です。

    ![Check AWS DMS migration task](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-13.png)

## ステップ 9. ダウンストリーム TiDB クラスターのデータを確認する {#step-9-check-data-in-the-downstream-tidb-cluster}

[Serverless Tierクラスター](https://tidbcloud.com/console/clusters/create-cluster)に接続し、 `admin.github_event`のテーブルデータを確認します。次のスクリーンショットに示すように、DMS はテーブル`github_events`と 10000 行のデータを正常に移行しました。

![Check Data In TiDB](/media/tidb-cloud/aws-dms-from-oracle-to-tidb-14.png)

## 概要 {#summary}

AWS DMS を使用すると、このドキュメントの例に従って、アップストリームの AWS RDS データベースからデータを正常に移行できます。

移行中に問題や障害が発生した場合は、 [クラウドウォッチ](https://console.aws.amazon.com/cloudwatch/home)のログ情報を確認して問題をトラブルシューティングできます。

![Troubleshooting](/media/tidb-cloud/aws-dms-to-tidb-cloud-troubleshooting.png)

## こちらもご覧ください {#see-also}

-   [AWS DMS を使用して MySQL 互換データベースから移行する](/tidb-cloud/migrate-from-mysql-using-aws-dms.md)

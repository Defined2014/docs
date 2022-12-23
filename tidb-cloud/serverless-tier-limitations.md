---
title: Serverless Tier Limitations
summary: Learn about the limitations of TiDB Cloud Serverless Tier.
---

# サーバーレス層の制限 {#serverless-tier-limitations}

<!-- markdownlint-disable MD026 -->

このドキュメントでは、サーバーレス層の制限について説明します。

サーバーレス層と専用層の間の機能のギャップを常に埋めています。ギャップでこれらの機能が必要な場合は、機能リクエストに[専用ティア](/tidb-cloud/select-cluster-tier.md#dedicated-tier)または[お問い合わせ](https://www.pingcap.com/contact-us/?from=en)を使用します。

## 一般的な制限 {#general-limitations}

-   TiDB Cloudアカウントごとに、ベータ段階で最大 5 つの無料の Serverless Tier クラスターを作成できます。
-   各サーバーレス層クラスターには次の制限があります。
    -   ストレージ サイズは、OLTP ストレージの 5 GiB (論理サイズ) と OLAP ストレージの 5 GiB (論理サイズ) に制限されています。
    -   コンピューティング リソースは、1 つの vCPU と 1 GiB RAM に制限されています。
    -   **注**: 今後数か月以内に、無料のスターター プランの提供を維持しながら、追加のリソースとパフォーマンスの向上のために使用量ベースの課金プランを提供する予定です。今後のリリースでは、無料のサーバーレス層の制限が変更される可能性があります。

## 取引 {#transaction}

-   ベータ フェーズ中のサーバーレス ティアでは、1 つのトランザクションの合計サイズが 10 MB を超えないように設定されています。

## 繋がり {#connection}

-   使用できるのは[標準接続](/tidb-cloud/connect-via-standard-connection.md)だけです。 [プライベート エンドポイント](/tidb-cloud/set-up-private-endpoint-connections.md)または[VPC ピアリング](/tidb-cloud/set-up-vpc-peering-connections.md)を使用して Serverless Tier クラスターに接続することはできません。
-   「IP アクセス リスト」はサポートされていません。

## バックアップと復元 {#backup-and-restore}

-   [バックアップと復元](/tidb-cloud/backup-and-restore.md)は現在、サーバーレス ティアではサポートされていません。

## モニタリング {#monitoring}

-   [ビルトインモニタリング](/tidb-cloud/built-in-monitoring.md)は現在、サーバーレス ティアでは使用できません。
-   [サードパーティのモニタリング統合](/tidb-cloud/third-party-monitoring-integrations.md)は現在、サーバーレス ティアでは使用できません。

## その他 {#others}

-   [TiDB CloudAPI](/tidb-cloud/api-overview.md)と[Terraform 統合](/tidb-cloud/terraform-tidbcloud-provider-overview.md)は現在、サーバーレス ティアでは使用できません。

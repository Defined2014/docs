---
title: Export Data from TiDB
summary: This page has instructions for exporting data from your TiDB cluster in TiDB Cloud.
---

# TiDB からのデータのエクスポート {#export-data-from-tidb}

このページでは、 TiDB Cloudのクラスターからデータをエクスポートする方法について説明します。

TiDB はデータをロックしません。 TiDB から他のデータ プラットフォームにデータを移行できるようにしたい場合があります。 TiDB は MySQL との互換性が高いため、MySQL に適したエクスポート ツールはすべて TiDB にも使用できます。

ツール[Dumpling](https://github.com/pingcap/dumpling)を使用してデータをエクスポートできます。

1.  TiUPをダウンロードしてインストールします。

    {{< copyable "" >}}

    ```shell
    curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
    ```

2.  グローバル環境変数を宣言します。

    > **ノート：**
    >
    > インストール後、 TiUPは対応する`profile`ファイルの絶対パスを表示します。次のコマンドの`.bash_profile`を`profile`ファイルのパスに変更する必要があります。

    {{< copyable "" >}}

    ```shell
    source .bash_profile
    ```

3.  Dumplingをインストールします。

    {{< copyable "" >}}

    ```shell
    tiup install dumpling
    ```

4.  TiDB からDumplingを使用してデータをエクスポートします。

    {{< copyable "" >}}

    ```shell
    tiup dumpling -h ${tidb-endpoint} -P 3306 -u ${user} -F 67108864 -t 4 -o /path/to/export/dir
    ```

    指定したデータベースのみをエクスポートする場合は、 `-B`を使用してデータベース名のコンマ区切りリストを指定します。

    最低限必要な権限は次のとおりです。

    -   `SELECT`
    -   `RELOAD`
    -   `LOCK TABLES`
    -   `REPLICATION CLIENT`

    現在、 Dumplingは Mydumper 形式の出力のみをサポートしており、これは[TiDB Lightning](https://github.com/pingcap/tidb-lightning)を使用して MySQL 互換データベースに簡単に復元できます。

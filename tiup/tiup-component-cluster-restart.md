---
title: tiup cluster restart
---

# tiup cluster restart {#tiup-cluster-restart}

コマンド`tiup cluster restart`は、指定されたクラスターのすべてまたは一部のサービスを再起動するために使用されます。

> **ノート：**
>
> 再起動プロセス中は、関連するサービスが一定期間利用できなくなります。

## 構文 {#syntax}

```shell
tiup cluster restart <cluster-name> [flags]
```

`<cluster-name>` : 操作するクラスターの名前。クラスター名を忘れた場合は、 [クラスタ リスト](/tiup/tiup-component-cluster-list.md)コマンドで確認できます。

## オプション {#options}

### -N, --ノード {#n-node}

-   再起動するノードを指定します。このオプションの値は、ノード ID のコンマ区切りリストです。 `tiup cluster display`コマンドで返される[クラスタ ステータス テーブル](/tiup/tiup-component-cluster-display.md)の最初の列からノード ID を取得できます。
-   データ型: `STRING`
-   このオプションが指定されていない場合、 TiUPはデフォルトですべてのノードを再起動します。

> **ノート：**
>
> オプション`-R, --role`を同時に指定すると、 TiUPは`-N, --node`と`-R, --role`の両方の要件に一致するサービス ノードを再起動します。

### -R, --role {#r-role}

-   再起動するノードの役割を指定します。このオプションの値は、ノードの役割のコンマ区切りリストです。 `tiup cluster display`コマンドによって返される[クラスタ ステータス テーブル](/tiup/tiup-component-cluster-display.md)の 2 列目から、ノードの役割を取得できます。
-   データ型: `STRING`
-   このオプションが指定されていない場合、 TiUPはデフォルトですべてのロールのノードを再起動します。

> **ノート：**
>
> オプション`-N, --node`を同時に指定すると、 TiUPは`-N, --node`と`-R, --role`の両方の要件に一致するサービス ノードを再起動します。

### -h, --help {#h-help}

-   ヘルプ情報を出力します。
-   データ型: `BOOLEAN`
-   このオプションはデフォルトで無効になっており、値は`false`です。このオプションを有効にするには、このオプションをコマンドに追加し、値`true`を渡すか、値を何も渡さないでください。

## 出力 {#outputs}

サービスの再起動プロセスのログ。

[&lt;&lt; 前のページに戻る - TiUP クラスタコマンド一覧](/tiup/tiup-component-cluster.md#command-list)

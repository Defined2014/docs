---
title: TiDB Binlog Troubleshooting
summary: Learn the troubleshooting process of TiDB Binlog.
---

# Binlogバイナリログのトラブルシューティング {#tidb-binlog-troubleshooting}

このドキュメントでは、TiDB Binlogをトラブルシューティングして問題を見つける方法について説明します。

TiDB Binlogの実行中にエラーが発生した場合は、次の手順に従ってトラブルシューティングを行います。

1.  各監視メトリックが正常かどうかを確認します。詳細は[Binlogバイナリログ監視](/tidb-binlog/monitor-tidb-binlog-cluster.md)を参照してください。

2.  [binlogctl ツール](/tidb-binlog/binlog-control.md)を使用して、各PumpまたはDrainerノードの状態が正常かどうかを確認します。

3.  PumpログまたはDrainerログに`ERROR`または`WARN`が存在するかどうかを確認してください。

上記の手順で問題を特定した後、解決方法については[FAQ](/tidb-binlog/tidb-binlog-faq.md)と[TiDB Binlogエラー処理](/tidb-binlog/handle-tidb-binlog-errors.md)を参照してください。解決策が見つからない場合、または提供された解決策が役に立たない場合は、 [問題](https://github.com/pingcap/tidb-binlog/issues)を送信してヘルプを求めてください。

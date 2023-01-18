---
title: TiDB 3.0.0-rc.3 Release Notes
---

# TiDB 3.0.0-rc.3 リリースノート {#tidb-3-0-0-rc-3-release-notes}

発売日：2019年6月21日

TiDB バージョン: 3.0.0-rc.3

TiDB アンシブル バージョン: 3.0.0-rc.3

## 概要 {#overview}

2019 年 6 月 21 日に、TiDB 3.0.0-rc.3 がリリースされました。対応する TiDB Ansible のバージョンは 3.0.0-rc.3 です。 TiDB 3.0.0-rc.2 と比較して、このリリースでは、安定性、使いやすさ、機能、SQL オプティマイザー、統計、および実行エンジンが大幅に改善されています。

## TiDB {#tidb}

-   SQL オプティマイザー
    -   仮想生成列の統計を収集する機能を削除します[#10629](https://github.com/pingcap/tidb/pull/10629)
    -   ポイントクエリで主キー定数がオーバーフローする問題を修正[#10699](https://github.com/pingcap/tidb/pull/10699)
    -   `fast analyze`で初期化されていない情報を使用するとpanic[#10691](https://github.com/pingcap/tidb/pull/10691)が発生する問題を修正します
    -   `prepare`を使用して`create view`ステートメントを実行すると、間違った列情報が原因でpanicが発生する問題を修正します[#10713](https://github.com/pingcap/tidb/pull/10713)
    -   ウィンドウ関数の処理時に列情報が複製されない問題を修正[#10720](https://github.com/pingcap/tidb/pull/10720)
    -   インデックス結合[#10854](https://github.com/pingcap/tidb/pull/10854)の内部テーブル選択の選択率の間違った見積もりを修正します。
    -   `stats-lease`変数値が 0 [#10811](https://github.com/pingcap/tidb/pull/10811)の場合の自動読み込み統計をサポート

-   実行エンジン
    -   `StreamAggExec` [#10636](https://github.com/pingcap/tidb/pull/10636)で`Close`関数を呼び出したときにリソースが正しく解放されない問題を修正
    -   分割テーブル[#10689](https://github.com/pingcap/tidb/pull/10689)に対して`show create table`ステートメントを実行した結果、 `table_option`と`partition_options`の順序が正しくない問題を修正
    -   逆順でのデータのスキャンをサポートすることにより、 `admin show ddl jobs`のパフォーマンスを向上させます[#10687](https://github.com/pingcap/tidb/pull/10687)
    -   このステートメントに`current_user`フィールド[#10684](https://github.com/pingcap/tidb/pull/10684)がある場合、RBAC の`show grants`ステートメントの結果が MySQL の結果と互換性がない問題を修正します。
    -   UUID が複数のノードで重複した値を生成する可能性がある問題を修正します[#10712](https://github.com/pingcap/tidb/pull/10712)
    -   `explain` [#10635](https://github.com/pingcap/tidb/pull/10635)で`show view`の権限が考慮されない問題を修正
    -   `split table region`のステートメントを追加して、テーブルリージョンを手動で分割し、ホットスポットの問題を軽減します[#10765](https://github.com/pingcap/tidb/pull/10765)
    -   `split index region`ステートメントを追加して、インデックスリージョンを手動で分割し、ホットスポットの問題を軽減します[#10764](https://github.com/pingcap/tidb/pull/10764)
    -   `create user` 、 `grant` 、または`revoke`などの複数のステートメントを連続して実行すると、正しく実行されない問題を修正します[#10737](https://github.com/pingcap/tidb/pull/10737)
    -   コプロセッサー[#10791](https://github.com/pingcap/tidb/pull/10791)への式のプッシュ ダウンを禁止するブロックリストを追加します。
    -   クエリがメモリ構成の制限を超えたときに`expensive query`ログを出力する機能を追加します[#10849](https://github.com/pingcap/tidb/pull/10849)
    -   `bind-info-lease`の構成項目を追加して、変更されたバインディング実行プランの更新時間を制御します[#10727](https://github.com/pingcap/tidb/pull/10727)
    -   `execdetails.ExecDetails`ポインター[#10832](https://github.com/pingcap/tidb/pull/10832)が原因で、コプロセッサーリソースを迅速に解放できなかったことが原因で発生した同時実行の多いシナリオでの OOM の問題を修正します。
    -   場合によっては`kill`ステートメントによって引き起こされるpanicの問題を修正します[#10876](https://github.com/pingcap/tidb/pull/10876)

-   サーバ
    -   GC [#10683](https://github.com/pingcap/tidb/pull/10683)の修復時にゴルーチンがリークする可能性がある問題を修正
    -   スロークエリで`host`情報の表示をサポート[#10693](https://github.com/pingcap/tidb/pull/10693)
    -   TiKV [#10632](https://github.com/pingcap/tidb/pull/10632)とやり取りするアイドル リンクの再利用をサポート
    -   RBAC [#10738](https://github.com/pingcap/tidb/pull/10738)で`skip-grant-table`オプションを有効にするためのサポートを修正
    -   `pessimistic-txn`構成が無効になる問題を修正[#10825](https://github.com/pingcap/tidb/pull/10825)
    -   アクティブにキャンセルされた ticlient リクエストがまだ再試行される問題を修正します[#10850](https://github.com/pingcap/tidb/pull/10850)
    -   悲観的トランザクションと楽観的なトランザクションが競合する場合のパフォーマンスを改善する[#10881](https://github.com/pingcap/tidb/pull/10881)

-   DDL
    -   `alter table`を使用して charset を変更すると`blob`型が変更される問題を修正します[#10698](https://github.com/pingcap/tidb/pull/10698)
    -   ホットスポットの問題を軽減するために、列に`AUTO_INCREMENT`属性が含まれている場合に`SHARD_ROW_ID_BITS`を使用して行 ID を分散させる機能を追加します[#10794](https://github.com/pingcap/tidb/pull/10794)
    -   `alter table`ステートメント[#10808](https://github.com/pingcap/tidb/pull/10808)を使用して、格納された生成列の追加を禁止する
    -   クラスターのアップグレード後に DDL 操作が遅くなる期間を短縮するために、DDL メタデータの無効な存続時間を最適化します[#10795](https://github.com/pingcap/tidb/pull/10795)

## PD {#pd}

-   `enable-two-way-merge`つの構成アイテムを追加して、一方向のマージのみを許可する[#1583](https://github.com/pingcap/pd/pull/1583)
-   `AddLightLearner`と`AddLightPeer`のスケジューリング操作を追加して、 リージョン Scatter スケジューリングが制限メカニズムによって制限されないようにする[#1563](https://github.com/pingcap/pd/pull/1563)
-   システムの起動時にデータのレプリカ複製が 1 つしかない可能性があるため、信頼性が不十分になる問題を修正します[#1581](https://github.com/pingcap/pd/pull/1581)
-   構成チェック ロジックを最適化して構成アイテム エラーを回避する[#1585](https://github.com/pingcap/pd/pull/1585)
-   `store-balance-rate`構成の定義を、1 分間に生成されるバランス オペレーターの数の上限に調整します[#1591](https://github.com/pingcap/pd/pull/1591)
-   ストアがスケジュールされた操作を生成できなかった可能性がある問題を修正します[#1590](https://github.com/pingcap/pd/pull/1590)

## TiKV {#tikv}

-   エンジン
    -   イテレーターがステータスをチェックしないために、システムで不完全なスナップショットが生成される問題を修正します[#4936](https://github.com/tikv/tikv/pull/4936)
    -   異常な状態での停電後にスナップショットを受信すると、ディスクへのデータのフラッシュの遅延によって引き起こされるデータ損失の問題を修正します[#4850](https://github.com/tikv/tikv/pull/4850)

-   サーバ
    -   `block-size`構成の有効性をチェックする機能を追加します[#4928](https://github.com/tikv/tikv/pull/4928)
    -   追加`READ_INDEX`関連の監視指標[#4830](https://github.com/tikv/tikv/pull/4830)
    -   GC ワーカー関連のモニタリング指標を追加する[#4922](https://github.com/tikv/tikv/pull/4922)

-   ラフトストア
    -   ローカル リーダーのキャッシュが正しくクリアされない問題を修正します[#4778](https://github.com/tikv/tikv/pull/4778)
    -   リーダーの転送と変更時にリクエストの遅延が増加する可能性がある問題を修正`conf` [#4734](https://github.com/tikv/tikv/pull/4734)
    -   古いコマンドが誤って報告される問題を修正[#4682](https://github.com/tikv/tikv/pull/4682)
    -   コマンドが長時間保留になる可能性がある問題を修正します[#4810](https://github.com/tikv/tikv/pull/4810)
    -   スナップショットファイルのディスクへの同期の遅延が原因で、停電後にファイルが破損する問題を修正します[#4807](https://github.com/tikv/tikv/pull/4807) 、 [#4850](https://github.com/tikv/tikv/pull/4850)

-   コプロセッサー
    -   ベクトル計算でトップ N をサポート[#4827](https://github.com/tikv/tikv/pull/4827)
    -   ベクトル計算で`Stream`集計をサポート[#4786](https://github.com/tikv/tikv/pull/4786)
    -   ベクトル計算で`AVG`集計関数をサポート[#4777](https://github.com/tikv/tikv/pull/4777)
    -   ベクトル計算で`First`集計関数をサポート[#4771](https://github.com/tikv/tikv/pull/4771)
    -   ベクトル計算で`SUM`集計関数をサポート[#4797](https://github.com/tikv/tikv/pull/4797)
    -   ベクトル計算で`MAX` / `MIN`集計関数をサポート[#4837](https://github.com/tikv/tikv/pull/4837)
    -   ベクトル計算[#4747](https://github.com/tikv/tikv/pull/4747)で`Like`式をサポート
    -   ベクトル計算[#4849](https://github.com/tikv/tikv/pull/4849)で`MultiplyDecimal`式をサポート
    -   ベクトル計算で`BitAnd` / `BitOr` / `BitXor`式をサポート[#4724](https://github.com/tikv/tikv/pull/4724)
    -   ベクトル計算[#4808](https://github.com/tikv/tikv/pull/4808)で`UnaryNot`式をサポート

-   取引
    -   悲観的なトランザクションで非悲観的的なロック競合が原因でエラーが発生する問題を修正[#4801](https://github.com/tikv/tikv/pull/4801) 、 [#4883](https://github.com/tikv/tikv/pull/4883)
    -   悲観的なトランザクションを有効にした後、楽観的トランザクションの不要な計算を減らしてパフォーマンスを向上させます[#4813](https://github.com/tikv/tikv/pull/4813)
    -   単一ステートメントのロールバックの機能を追加して、デッドロック状況でトランザクション全体がロールバック操作を必要としないようにします[#4848](https://github.com/tikv/tikv/pull/4848)
    -   悲観的トランザクション関連の監視項目を追加[#4852](https://github.com/tikv/tikv/pull/4852)
    -   `ResolveLockLite`コマンドを使用して軽量ロックを解決し、重大な競合が存在する場合のパフォーマンスを向上させるサポート[#4882](https://github.com/tikv/tikv/pull/4882)

-   tikv-ctl
    -   `bad-regions`コマンドを追加して、より多くの異常状態のチェックをサポートします[#4862](https://github.com/tikv/tikv/pull/4862)
    -   `tombstone`コマンドを強制実行する機能を追加[#4862](https://github.com/tikv/tikv/pull/4862)

-   その他
    -   `dist_release`コンパイル コマンド[#4841](https://github.com/tikv/tikv/pull/4841)を追加します。

## ツール {#tools}

-   Binlog
    -   データの書き込みに失敗したときにPumpが戻り値をチェックしないことによって引き起こされる間違ったオフセットの問題を修正します[#640](https://github.com/pingcap/tidb-binlog/pull/640)
    -   Drainerに`advertise-addr`の構成を追加して、コンテナー環境でブリッジ モードをサポートする[#634](https://github.com/pingcap/tidb-binlog/pull/634)
    -   Pumpに`GetMvccByEncodeKey`関数を追加して、トランザクション ステータスのクエリを高速化します[#632](https://github.com/pingcap/tidb-binlog/pull/632)

## TiDB アンシブル {#tidb-ansible}

-   クラスタの最大 QPS 値を予測するモニタリング項目を追加 (デフォルトでは「非表示」) [#f5cfa4d](https://github.com/pingcap/tidb-ansible/commit/f5cfa4d903bbcd77e01eddc8d31eabb6e6157f73)

---
title: TiDB Data Migration Support for Online DDL Tools
summary: Learn about the support for common online DDL tools, usage, and precautions in DM.
---

# オンライン DDL ツールの TiDB データ移行サポート {#tidb-data-migration-support-for-online-ddl-tools}

MySQL エコシステムでは、gh-ost や pt-osc などのツールが広く使用されています。 TiDB データ移行 (DM) は、不要な中間データの移行を避けるために、これらのツールをサポートします。

このドキュメントでは、一般的なオンライン DDL ツールのサポート、使用方法、および DM での注意事項について紹介します。

オンライン DDL ツールの DM の動作原理と実装方法については、 [オンライン ddl](/dm/feature-online-ddl.md)を参照してください。

## 制限 {#restrictions}

-   DM は gh-ost と pt-osc のみをサポートします。
-   `online-ddl`が有効な場合、増分レプリケーションに対応するチェックポイントは、オンライン DDL 実行のプロセスにあってはなりません。たとえば、アップストリームのオンライン DDL 操作がバイナリログの`position-A`で開始し、 `position-B`で終了する場合、増分レプリケーションの開始点は`position-A`より前または`position-B`より後である必要があります。そうしないと、エラーが発生します。詳細は[FAQ](/dm/dm-faq.md#how-to-handle-the-error-returned-by-the-ddl-operation-related-to-the-gh-ost-table-after-online-ddl-scheme-gh-ost-is-set)を参照してください。

## パラメータの構成 {#configure-parameters}

<SimpleTab>
<div label="v2.0.5 and later">

v2.0.5 以降のバージョンでは、 `task`構成ファイルの`online-ddl`構成アイテムを使用する必要があります。

-   上流の MySQL/MariaDB が (同時に) gh-ost または pt-osc ツールを使用する場合は、タスク構成ファイルで`online-ddl`から`true`を設定します。

```yml
online-ddl: true
```

> **ノート：**
>
> v2.0.5 以降、 `online-ddl-scheme`は廃止されたため、 `online-ddl-scheme`の代わりに`online-ddl`を使用する必要があります。つまり、設定`online-ddl: true`は`online-ddl-scheme`を上書きし、設定`online-ddl-scheme: "pt"`または`online-ddl-scheme: "gh-ost"`は`online-ddl: true`に変換されます。

</div>

<div label="earlier than v2.0.5">

v2.0.5 より前 (v2.0.5 を除く) では、 `task`構成ファイルの`online-ddl-scheme`構成アイテムを使用する必要があります。

-   アップストリームの MySQL/MariaDB が gh-ost ツールを使用している場合は、タスク構成ファイルで設定します。

```yml
online-ddl-scheme: "gh-ost"
```

-   アップストリームの MySQL/MariaDB が pt ツールを使用している場合は、タスク構成ファイルで設定します。

```yml
online-ddl-scheme: "pt"
```

</div>
</SimpleTab>

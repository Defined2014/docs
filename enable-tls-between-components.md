---
title: Enable TLS Between TiDB Components
summary: Learn how to enable TLS authentication between TiDB components.
---

# TiDB コンポーネント間の TLS を有効にする {#enable-tls-between-tidb-components}

このドキュメントでは、TiDB クラスター内のコンポーネント間で暗号化されたデータ転送を有効にする方法について説明します。有効にすると、次のコンポーネント間で暗号化された転送が使用されます。

-   TiDB および TiKV; TiDB と PD
-   TiKVとPD
-   TiDB コントロールと TiDB。 TiKV Controlと TiKV; PD Controlと PD
-   各 TiKV、PD、TiDB クラスター内の内部通信

現在、一部の特定のコンポーネントの暗号化された送信のみを有効にすることはサポートされていません。

## 暗号化されたデータ転送を構成して有効にする {#configure-and-enable-encrypted-data-transmission}

1.  証明書を準備します。

    TiDB、TiKV、PD のサーバー証明書は別途用意することをお勧めします。これらのコンポーネントが相互に認証できることを確認してください。 TiDB、TiKV、および PD の制御ツールは、1 つのクライアント証明書を共有することを選択できます。

    `openssl` 、 `easy-rsa` 、 `cfssl`などのツールを使用して、自己署名証明書を生成できます。

    <CustomContent platform="tidb">

    `openssl`を選択した場合は[自己署名証明書の生成](/generate-self-signed-certificates.md)を参照できます。

    </CustomContent>

    <CustomContent platform="tidb-cloud">

    `openssl`を選択した場合は[自己署名証明書の生成](https://docs.pingcap.com/tidb/stable/generate-self-signed-certificates)を参照できます。

    </CustomContent>

2.  証明書を構成します。

    TiDB コンポーネント間の相互認証を有効にするには、TiDB、TiKV、および PD の証明書を次のように構成します。

    -   TiDB

        構成ファイルまたはコマンドライン引数で構成します。

        ```toml
        [security]
        # Path of the file that contains list of trusted SSL CAs for connection with cluster components.
        cluster-ssl-ca = "/path/to/ca.pem"
        # Path of the file that contains X509 certificate in PEM format for connection with cluster components.
        cluster-ssl-cert = "/path/to/tidb-server.pem"
        # Path of the file that contains X509 key in PEM format for connection with cluster components.
        cluster-ssl-key = "/path/to/tidb-server-key.pem"
        ```

    -   TiKV

        構成ファイルまたはコマンドライン引数で構成し、対応する URL を`https`に設定します。

        ```toml
        [security]
        ## The path for certificates. An empty string means that secure connections are disabled.
        # Path of the file that contains a list of trusted SSL CAs. If it is set, the following settings `cert_path` and `key_path` are also needed.
        ca-path = "/path/to/ca.pem"
        # Path of the file that contains X509 certificate in PEM format.
        cert-path = "/path/to/tikv-server.pem"
        # Path of the file that contains X509 key in PEM format.
        key-path = "/path/to/tikv-server-key.pem"
        ```

    -   PD

        構成ファイルまたはコマンドライン引数で構成し、対応する URL を`https`に設定します。

        ```toml
        [security]
        ## The path for certificates. An empty string means that secure connections are disabled.
        # Path of the file that contains a list of trusted SSL CAs. If it is set, the following settings `cert_path` and `key_path` are also needed.
        cacert-path = "/path/to/ca.pem"
        # Path of the file that contains X509 certificate in PEM format.
        cert-path = "/path/to/pd-server.pem"
        # Path of the file that contains X509 key in PEM format.
        key-path = "/path/to/pd-server-key.pem"
        ```

    -   TiFlash (v4.0.5 の新機能)

        `tiflash.toml`ファイルで構成し、 `http_port`項目を`https_port`に変更します。

        ```toml
        [security]
        ## The path for certificates. An empty string means that secure connections are disabled.
        # Path of the file that contains a list of trusted SSL CAs. If it is set, the following settings `cert_path` and `key_path` are also needed.
        ca_path = "/path/to/ca.pem"
        # Path of the file that contains X509 certificate in PEM format.
        cert_path = "/path/to/tiflash-server.pem"
        # Path of the file that contains X509 key in PEM format.
        key_path = "/path/to/tiflash-server-key.pem"
        ```

        `tiflash-learner.toml`のファイルで構成します。

        ```toml
        [security]
        # Path of the file that contains a list of trusted SSL CAs. If it is set, the following settings `cert_path` and `key_path` are also needed.
        ca-path = "/path/to/ca.pem"
        # Path of the file that contains X509 certificate in PEM format.
        cert-path = "/path/to/tiflash-server.pem"
        # Path of the file that contains X509 key in PEM format.
        key-path = "/path/to/tiflash-server-key.pem"
        ```

    -   TiCDC

        構成ファイルで構成します：

        ```toml
        [security]
        ca-path = "/path/to/ca.pem"
        cert-path = "/path/to/cdc-server.pem"
        key-path = "/path/to/cdc-server-key.pem"
        ```

        または、コマンドライン引数で構成し、対応する URL を`https`に設定します。

        {{< copyable "" >}}

        ```bash
        cdc server --pd=https://127.0.0.1:2379 --log-file=ticdc.log --addr=0.0.0.0:8301 --advertise-addr=127.0.0.1:8301 --ca=/path/to/ca.pem --cert=/path/to/ticdc-cert.pem --key=/path/to/ticdc-key.pem
        ```

        現在、TiDB コンポーネント間の暗号化された送信が有効になっています。

    > **ノート：**
    >
    > TiDB クラスターで暗号化送信を有効にした後、tidb-ctl、tikv-ctl、または pd-ctl を使用してクラスターに接続する必要がある場合は、クライアント証明書を指定します。例えば：

    {{< copyable "" >}}

    ```bash
    ./tidb-ctl -u https://127.0.0.1:10080 --ca /path/to/ca.pem --ssl-cert /path/to/client.pem --ssl-key /path/to/client-key.pem
    ```

    {{< copyable "" >}}

    ```bash
    tiup ctl:<cluster-version> pd -u https://127.0.0.1:2379 --cacert /path/to/ca.pem --cert /path/to/client.pem --key /path/to/client-key.pem
    ```

    {{< copyable "" >}}

    ```bash
    ./tikv-ctl --host="127.0.0.1:20160" --ca-path="/path/to/ca.pem" --cert-path="/path/to/client.pem" --key-path="/path/to/clinet-key.pem"
    ```

### コンポーネントの呼び出し元の身元を確認する {#verify-component-caller-s-identity}

Common Name は発信者の確認に使用されます。一般に、呼び出し先は、呼び出し元から提供されたキー、証明書、および CA の確認に加えて、呼び出し元の身元を確認する必要があります。たとえば、TiKV は TiDB からのみアクセスでき、他の訪問者は正当な証明書を持っていてもブロックされます。

コンポーネントの呼び出し元の ID を確認するには、証明書の生成時に`Common Name`を使用して証明書のユーザー ID をマークし、呼び出し先の`Common Name`リストを構成して呼び出し元の ID を確認する必要があります。

-   TiDB

    構成ファイルまたはコマンドライン引数で構成します。

    ```toml
    [security]
    cluster-verify-cn = [
        "TiDB-Server",
        "TiKV-Control",
    ]
    ```

-   TiKV

    構成ファイルまたはコマンドライン引数で構成します。

    ```toml
    [security]
    cert-allowed-cn = [
        "TiDB-Server", "PD-Server", "TiKV-Control", "RawKvClient1",
    ]
    ```

-   PD

    構成ファイルまたはコマンドライン引数で構成します。

    ```toml
    [security]
    cert-allowed-cn = ["TiKV-Server", "TiDB-Server", "PD-Control"]
    ```

-   TiFlash (v4.0.5 の新機能)

    `tiflash.toml`のファイルまたはコマンドライン引数で構成します。

    ```toml
    [security]
    cert_allowed_cn = ["TiKV-Server", "TiDB-Server"]
    ```

    `tiflash-learner.toml`のファイルで構成します。

    ```toml
    [security]
    cert-allowed-cn = ["PD-Server", "TiKV-Server", "TiFlash-Server"]
    ```

### 証明書のリロード {#reload-certificates}

証明書とキーをリロードするために、TiDB、PD、TiKV、およびすべての種類のクライアントは、新しい接続が作成されるたびに現在の証明書とキー ファイルを再読み込みします。現在、CA 証明書を再ロードすることはできません。

## こちらもご覧ください {#see-also}

-   [TiDB クライアントとサーバー間で TLS を有効にする](/enable-tls-between-clients-and-servers.md)

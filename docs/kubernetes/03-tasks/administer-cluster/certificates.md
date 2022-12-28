# 手動生成證書

原文: [Generate Certificates Manually](https://kubernetes.io/docs/tasks/administer-cluster/certificates/)

在使用客戶端證書認證的場景下，你可以通過 easyrsa、 openssl 或 cfssl 等工具以手工方式生成證書。

## openssl

openssl 支持以手動的方式為你的集群生成證書。

1. 生成一個 2048 位元的 `ca.key` 文件

    ```bash
    # Generate CA private key
    openssl genrsa -out ca.key 2048
    ```

2. 在 `ca.key` 文件的基礎上，生成 `ca.crt` 文件（用參數 `-days` 設置證書有效期）

    ```bash
    # Generate Self Signed certificate
    openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt
    ```

3. 生成一個 2048 位元的 `server.key` 文件：

    ```bash
    openssl genrsa -out server.key 2048
    ```

4. 創建一個用於生成證書籤名請求（CSR）的配置文件。保存文件（例如：csr.conf）前，記得用真實值替換掉尖括號中的值（例如：<MASTER_IP>）。注意：MASTER_CLUSTER_IP就像前一小節所述，它的值是 API 服務器的服務集群 IP。下面的例子假定你的默認 DNS 域名為 cluster.local。

    ``` title="csr.conf"
    [ req ]
    default_bits = 2048
    prompt = no
    default_md = sha256
    req_extensions = req_ext
    distinguished_name = dn

    [ dn ]
    C = <country>
    ST = <state>
    L = <city>
    O = <organization>
    OU = <organization unit>
    CN = <MASTER_IP>

    [ req_ext ]
    subjectAltName = @alt_names

    [ alt_names ]
    DNS.1 = kubernetes
    DNS.2 = kubernetes.default
    DNS.3 = kubernetes.default.svc
    DNS.4 = kubernetes.default.svc.cluster
    DNS.5 = kubernetes.default.svc.cluster.local
    IP.1 = <MASTER_IP>
    IP.2 = <MASTER_CLUSTER_IP>

    [ v3_ext ]
    authorityKeyIdentifier=keyid,issuer:always
    basicConstraints=CA:FALSE
    keyUsage=keyEncipherment,dataEncipherment
    extendedKeyUsage=serverAuth,clientAuth
    subjectAltName=@alt_names
    ```

5. 基於上面的配置文件生成證書簽名請求：

    ```bash
    openssl req -new -key server.key -out server.csr -config csr.conf
    ```

6. 基於 `ca.key`、`ca.crt` 和 `server.csr` 等三個文件生成服務端證書：

    ```bash
    openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
        -CAcreateserial -out server.crt -days 10000 \
        -extensions v3_ext -extfile csr.conf -sha256
    ```

7. 查看證書籤名請求：

    ```bash
    openssl req  -noout -text -in ./server.csr
    ```

8. 查看證書：

    ```bash
    openssl x509  -noout -text -in ./server.crt
    ```

最後，為API 服務器添加相同的啟動參數。


## 分發自簽名的 CA 證書

客戶端節點可能不認可自簽名 CA 證書的有效性。對於非生產環境，或者運行在公司防火牆後的環境，你可以分發自簽名的 CA 證書到所有客戶節點，並刷新本地列表以使證書生效。

在每一個客戶節點，執行以下操作：

```bash
sudo cp ca.crt /usr/local/share/ca-certificates/kubernetes.crt
sudo update-ca-certificates
```

結果:

```
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d....
done.
```

## 證書API

你可以通過 `certificates.k8s.io` API 提供 x509 證書，用來做身份驗證， 如管理集群中的TLS 認證文檔所述。





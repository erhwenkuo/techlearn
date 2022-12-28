# OpenSSL 命令大全，CA 證書生成，客戶端證書生成實例

![](./assets/Sertifikat-SSL.png)

原文:

- [OpenSSL命令大全，CA證書生成，客戶端證書生成實例](https://www.twblogs.net/a/5bb048112b7177781a0feb9b)
- [Self Signed SSL/TLS Certificate with IP Address](https://nodeployfriday.com/posts/self-signed-cert/)

## X509 證書鏈

在創建 x509 證書時一般會用到三類的文件:

- `key` 是私用密鑰，openssl 格式，通常是 rsa 算法。
- `csr` 是證書請求文件，用於申請證書。在製作 csr 文件的時候，必須使用自己的私鑰來簽署申請，還可以設定一個密鑰。
- `crt` 是 CA 認證後的證書文件（windows下面的 csr，其實是 crt），簽署人用自己的 key 給你簽署的憑證。

## Openssl 相關文件說明

- **.key 格式**：私有的密鑰
- **.csr 格式**：證書籤名請求（證書請求文件），含有公鑰信息，`Certificate Signing Request` 的縮寫
- **.crt 格式**：證書文件，certificate 的縮寫
- **.crl 格式**：證書吊銷列表，`Certificate Revocation List` 的縮寫
- **.pem 格式**：用於導出，導入證書時候用的證書格式，有證書開頭，結尾的格式

接著讓我們使用 openssl 來構建根憑證(root CA)。

## Root CA 證書的生成步驟

1. 生成 CA (.key) 私鑰
2. 生成 CA (.csr) 證書請求
3. 自簽 CA (.crt) 根證書: CA 給自已頒發的證書

在實際的軟件開發工作中，往往服務器就採用這種自簽名的方式，因爲畢竟找第三方簽名機構是要給錢的，也是需要花時間的。

### 1. 生成 CA (.key) 私鑰
```bash
# Generate CA private key
openssl genrsa -out ca.key 2048
```

結果:

```
Generating RSA private key, 2048 bit long modulus (2 primes)
...............................+++++
................................................................................................................+++++
e is 65537 (0x010001)
```

使用文字編輯器來打開 `ca.key`:

``` title="caKey.pem"
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAmHXAyNC6XafYiPKKnOF7LREOYcVhjfaqp1WYHvRvTa3GsPDw
6b3wOtRqR9k3uBzcj5uxa7tZSCa1/VoH8GbI44oQ1x2WvTCrlPLqqioAKQnvaciI
tlG5MzgLTMMueY9KbewwKV1mVjD9+FGqqmetjGty3aLl+agp+/yZQstWun/e8sFC
l1Ye0YW+U9mIonPKbXxelaaBpk0f5D2RJmSTUmESO8y4pInUz6Tj9m3N9VVNsbrB
xX5ZvvVp2dnHXb02NM20TOsPKhyUA2Qst96QRUV4RLkEo34Auj62fXE+ihWe01FC
3Ose7/X45PmaLUHsJlKXs5kJu9bK6LBRVM+oFQIDAQABAoIBAE8xSzSEh2nCl816
/tlCnnLoWyoaFvRg3oARC/V4TqBw1bZvURR5HuHQGGy9vh2akE7gNqaZKGU8BmhV
ba5IFa1ruBelPPmE4Ht8OrytGGw4xu2RxsG9bY/XWrdC408tSSIT/2hdJZ070ZA9
C4N8Wz+HSKErzn2CBlzn9swlRyWUHQMF2blFAXRTauPJyisk/dOXJkuJYX+moiTu
G6Y2x4FbFVmRxzTKnNZmlakzUjTFZddliVSkT0tB8RiFf/3ERHgnsA+6RExGiu53
EH727PifOqzxe7dlrpwsHMLj9c0gLQkcbePXUNvvmRXSDqZUH7VSX75MFsGJ4WJV
5fOzQoECgYEAxhhxEUhryntjIyzJM4H/vSZ7s5yR7gkkGYv3g5WDBnzjzKZFymwe
Hy6pCVxkP9X7LcMj7ckyClhZGueNhpLRViwLGdzUM6SiVupQw30lIflCjRQ1O0Tf
6GT4Ao6LHE608/NVQ1BhVqE3YDP38n/P71tzwU4m3YfYv28hFaFvwZECgYEAxQZj
xPcHm3tabSKqvGqIaLe4znKfft2bUTA4NiwT/0oIIhG0tQxS+JhuJO6ld4BRya9s
wUoHXbzIKb+blibM4BsyxZPh8huWfX4hmhCZ2MkGP3VtkTPkiyLqf8f4KWm7WAnj
+LBHoVQzYFMxHlO9Sy+29TGSzVUAO9e3gkHqvEUCgYAi1s2b4obCl6y51PiVzHkz
xP7gedrYaFcm/wzK4ZKno3NU3LFNhnJNvaEQ+mTPLUz9oWJCQa5BT4RsTOkBD/Ut
GJXjOIlBg+ThMFh/6RSMww/HTBSIlfZjIs7TdEpW8ii09si6C/ryh2v7yWgECCKD
0CuthZckQu4FzP+elEPZYQKBgEnfB3FGHbgSG+lnYhSa7OI2YDgnid2LQTzDk4/c
HvNM0pfTg6mOIV3L6SA3yhpwJjo0Z9Zg5zoiGfptOOynR5GPIQ4rUD/yUA5lSDv3
lXdOh+UrJhWXG01/neSjGUyNtAxiSPNpRLIcW9b12ijAxOx3y2VLXPtXr2rAirFv
/Y7xAoGADq4vUdpPcjDXrFsj5WwidEfT4XTf94GhJOAyh22+AGXPcVkmqwAapLbN
XIcSwXgvLeYU1Arm2NiHeToetEQ3yBB/lNL1Ttf5XuT7GsdaJJpYNqXk0IIrZ5GT
z7CHvAcEATdQ+YYSDTcqfSPsmJRZ+IWcIKkQwF4IzznVtygeZZo=
-----END RSA PRIVATE KEY-----
```

### 2. 生成 CA (.csr) 證書請求

內含公鑰

```bash
# Generate CSR
openssl req -new -key ca.key -out ca.csr
```

自定義區域:

|欄位名|說明|範例|
|-----|---|----|
|Country Name (2 letter code) [AU]|證書持有者所在國家(2碼)|TW|
|State or Province Name (full name) [Some-State]|證書持有者所在州或省份（可省略不填）|Taiwan|
|Locality Name (eg, city) []|證書持有者所在城市（可省略不填）|Tapei|
|Organization Name (eg, company) [Internet Widgits Pty Ltd]|證書持有者所屬組織或公司|Wistron|
|Organizational Unit Name (eg, section) []|證書持有者所屬部門（可省略不填）|dxlab|
|Common Name (e.g. server FQDN or YOUR name) []|域名|dxlab.wistron.com|
|Email Address []|電子郵件信箱（可省略不填）||
|A challenge password []|自定義密碼||
|An optional company name []|公司名稱（可省略不填）||

使用文字編輯器來打開 `ca.csr`:

``` title="ca.csr"
-----BEGIN CERTIFICATE REQUEST-----
MIIC3TCCAcUCAQAwfTELMAkGA1UEBhMCVFcxCzAJBgNVBAgMAlRXMQ8wDQYDVQQH
DAZUYWlwZWkxEDAOBgNVBAoMB1dpc3Ryb24xDjAMBgNVBAsMBWR4bGFiMQ4wDAYD
VQQDDAVkeGxhYjEeMBwGCSqGSIb3DQEJARYPYWRtaW5AZHhsYWIuY29tMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAmHXAyNC6XafYiPKKnOF7LREOYcVh
jfaqp1WYHvRvTa3GsPDw6b3wOtRqR9k3uBzcj5uxa7tZSCa1/VoH8GbI44oQ1x2W
vTCrlPLqqioAKQnvaciItlG5MzgLTMMueY9KbewwKV1mVjD9+FGqqmetjGty3aLl
+agp+/yZQstWun/e8sFCl1Ye0YW+U9mIonPKbXxelaaBpk0f5D2RJmSTUmESO8y4
pInUz6Tj9m3N9VVNsbrBxX5ZvvVp2dnHXb02NM20TOsPKhyUA2Qst96QRUV4RLkE
o34Auj62fXE+ihWe01FC3Ose7/X45PmaLUHsJlKXs5kJu9bK6LBRVM+oFQIDAQAB
oBswGQYJKoZIhvcNAQkHMQwMCndpc3Ryb244ODgwDQYJKoZIhvcNAQELBQADggEB
AAci3tAoVwmzRxc0oT8pqqZ+d9tOSwDzuzdAkQAxdHsa3QlWXWVzCWvf4outqdnE
RjSqmAwCt7x6qGGxxW+kDh/zwttT5fl3URO6MOhHw7AoBtXNS4gScSmeC+lb/Tk9
VW5AqpMw/Lijy3NJraDW4YmW89CpFSvXH25+NyLsIzzVzawNIxXe04CURUL+NVB3
abjEjXVKnQ8ymqE3AIIi3c9Rd6nNOOmxLYnX/6XDCNvo2hdMFTcPA99Lq2guuqgL
KOR5ZBlYYeApEMvVDZc40a49FmM5IF24xs5VIBWo4VtYjskUdrEXp/J4m6jIPhZn
0J6rt5KrzDrgcwFrxacX0qc=
-----END CERTIFICATE REQUEST-----
```

#### 使用 -subj 引數

創建 CSR 時的另一種選擇是使用 `-subj` 在命令本身中提供所有必要的信息。

使用以下命令在生成 CSR 時取消問題提示：

```bash
# Generate CSR
openssl req -new -key ca.key -out ca.csr \
-subj "/C=TW/ST=Taiwan/L=Taipei/O=Wistron, Inc./OU=IT/CN=dxlab.wistron.com"
```

#### 驗證 CSR 信息

使用私鑰創建 CSR 後，我們建議驗證 CSR 中包含的信息是否正確，以及文件是否未被修改或損壞。

使用以下命令查看您的 CSR 中的信息:

```bash
openssl req -text -in ca.csr -noout -verify
```

- `-noout` 忽略 CSR 編碼版本的輸出。 
- `-verify` 檢查文件的簽名以確保它沒有被修改。

結果:

```
Certificate request self-signature verify OK
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C = TW, ST = Taiwan, L = Taipei, O = "Wistron, Inc.", OU = IT, CN = dxlab.wistron.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:9a:fe:97:54:72:b7:f5:c9:cd:da:3d:40:91:60:
                    f9:30:90:ae:0e:06:96:e0:f9:18:53:ae:ff:fc:3a:
                    e8:0c:40:92:45:30:61:19:30:e7:27:70:27:4a:04:
                    53:f1:2e:6a:4b:d7:e1:81:32:0b:13:44:2c:cb:0c:
                    3a:39:42:54:c9:02:ce:6a:e9:db:c9:f4:64:eb:64:
                    dc:3b:27:83:84:2a:0d:d8:00:2a:28:00:25:80:4d:
                    ae:8b:4c:60:0c:06:6e:c6:8d:f7:e9:c4:52:1c:4f:
                    9a:0d:d8:84:6b:4f:c0:fe:0a:c3:25:88:23:1a:28:
                    fe:81:21:81:86:0a:3c:91:f9:87:44:27:25:34:aa:
                    51:92:bf:42:25:3a:29:d7:69:7b:dd:27:47:5b:b0:
                    e2:d1:0f:72:25:8a:93:b4:e7:5d:46:29:49:f1:94:
                    01:c6:e1:12:5c:b3:3f:fd:21:84:d1:ea:74:19:21:
                    1e:85:14:06:e9:e3:51:06:a8:ec:af:b0:6b:64:b6:
                    0b:80:ae:0f:34:49:a2:06:8f:da:25:ba:f3:22:b2:
                    5c:83:76:84:25:ef:07:66:21:20:08:38:be:52:2e:
                    cf:ee:72:a4:97:84:d0:06:e1:3b:9f:a6:f4:38:94:
                    e8:90:55:2f:7d:d0:10:7c:bd:55:b6:a5:83:70:92:
                    00:71
                Exponent: 65537 (0x10001)
        Attributes:
            (none)
            Requested Extensions:
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        21:30:47:b7:c6:78:2c:23:a6:fb:6d:0e:d2:cd:b1:af:1e:c8:
        ab:2c:97:73:aa:20:00:8e:a6:4d:4d:4f:73:3c:bb:98:89:e1:
        e2:b9:a5:04:56:e1:b5:ab:17:3d:58:fe:4e:01:56:32:53:85:
        35:a2:39:4c:63:35:03:09:9d:39:cf:66:31:dc:fc:c6:60:7a:
        5d:31:23:81:73:e0:01:2b:09:5c:8e:d2:f4:cd:d8:85:ec:4c:
        a1:31:a6:eb:b4:fc:06:e2:fa:b7:0a:d1:c6:f6:fb:57:93:b1:
        f2:f8:8a:58:1c:38:59:c9:ce:55:7b:8d:39:1b:45:6b:3d:3a:
        35:18:d4:29:1b:9c:af:6d:c1:81:71:a5:9a:67:5f:ea:3b:f4:
        0e:da:b8:b0:8d:4c:bf:37:4d:f7:a3:a9:11:47:94:c9:b2:f6:
        a9:bf:8d:ab:65:68:87:5e:0a:3e:eb:47:47:bc:a1:d0:29:a0:
        70:f2:a7:72:f9:65:71:a0:c3:19:ca:cd:ae:27:c0:3f:2e:17:
        1d:25:b4:a0:c7:e5:4b:97:53:09:6d:8d:cf:81:02:2e:4a:5d:
        9d:e9:96:d8:98:3a:6d:ca:22:28:97:e5:31:f0:b6:c7:26:c2:
        01:aa:8d:3e:81:97:f6:3a:a4:05:8c:6a:c4:58:2c:8c:cc:a3:
        fa:15:59:58
```

### 3. 自簽 CA (.crt) 根證書

```bash
# Generate Self Signed certificate
openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt
```

- `genrsa` - generate an RSA private key
- `req` - PKCS#10 certificate request and certificate generating utility
- `x509` - Certificate display and signing utility
- `-days` 證書有效期
- `-x509` 在 req 中直接生成 crt 代替 csr

``` title="ca.crt"
-----BEGIN CERTIFICATE-----
MIIDgTCCAmkCFB/qgGKxU5oUQSJMeoOr06XnzOIxMA0GCSqGSIb3DQEBCwUAMH0x
CzAJBgNVBAYTAlRXMQswCQYDVQQIDAJUVzEPMA0GA1UEBwwGVGFpcGVpMRAwDgYD
VQQKDAdXaXN0cm9uMQ4wDAYDVQQLDAVkeGxhYjEOMAwGA1UEAwwFZHhsYWIxHjAc
BgkqhkiG9w0BCQEWD2FkbWluQGR4bGFiLmNvbTAeFw0yMjEyMjYwOTMwMzVaFw0y
MzEyMjYwOTMwMzVaMH0xCzAJBgNVBAYTAlRXMQswCQYDVQQIDAJUVzEPMA0GA1UE
BwwGVGFpcGVpMRAwDgYDVQQKDAdXaXN0cm9uMQ4wDAYDVQQLDAVkeGxhYjEOMAwG
A1UEAwwFZHhsYWIxHjAcBgkqhkiG9w0BCQEWD2FkbWluQGR4bGFiLmNvbTCCASIw
DQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJh1wMjQul2n2Ijyipzhey0RDmHF
YY32qqdVmB70b02txrDw8Om98DrUakfZN7gc3I+bsWu7WUgmtf1aB/BmyOOKENcd
lr0wq5Ty6qoqACkJ72nIiLZRuTM4C0zDLnmPSm3sMCldZlYw/fhRqqpnrYxrct2i
5fmoKfv8mULLVrp/3vLBQpdWHtGFvlPZiKJzym18XpWmgaZNH+Q9kSZkk1JhEjvM
uKSJ1M+k4/ZtzfVVTbG6wcV+Wb71adnZx129NjTNtEzrDyoclANkLLfekEVFeES5
BKN+ALo+tn1xPooVntNRQtzrHu/1+OT5mi1B7CZSl7OZCbvWyuiwUVTPqBUCAwEA
ATANBgkqhkiG9w0BAQsFAAOCAQEAEcSjnzK9F3QqOuWVvIEEjWjHQ4FrhcggOCo2
oclG36BEGidu3+zuli/gyUrflsKZVDQccpIvg94VRIyDiZCh31NQkEAO3nTpMt8Q
ODEL47Zt4TrzTOC0MOwa/sgwZpONS3lerrC3JGL3uVNM7NNimt5VroeJxO8VsEOp
C/7LAfVGYmsXEbPbRTXXZrS+GIaTIBjt8ZOysHIOwfh8YESV6N828Ajgdnbq0Yy5
yM90ppSgEbIKaNXMtFC+zfvoXYYb91jt0VNMUgOHHqcL0gv0qowxiSpRRtAmxsZs
ox4C63BrKslebF2CtJaqWuS+QnoMn5zbNJrcuDD9xK4ChJ084g==
-----END CERTIFICATE-----
```

### 4. 將證書導出成瀏覽器支持的p12格式

```bash
openssl pkcs12 -export -clcerts -in ca.crt -inkey ca.key -out ca.p12
```

## 用戶證書的生成步驟

一般說的生成用戶證書分兩種，客戶端證書和服務端證書，除了名稱不一樣步驟命令完全一樣。

1. 生成私鑰（.key）
2. 生成證書請求（.csr）
3. 用 CA 根證書簽署證書（.crt）

### 1. 生成私鑰(.key) 

```bash
# Generate CA private key
openssl genrsa -out server.key 2048
```

### 2. 生成證書請求(.csr)

接著我們將要使用 IP 地址而不是主機名或域名來創建自簽名證書。

為了讓自簽名證書僅與 IP（而不是域名）一起使用，我們將為 IP 指定主題備用名稱 (SAN)。

#### 創建請求配置文件

``` title="ssl.conf" hl_lines="14 25"
[req]
prompt = no
default_bits = 2048
default_md = sha256
distinguished_name = req_distinguished_name
x509_extensions = v3_req

[req_distinguished_name]
C = TW
ST = Taiwan
L = Taipei
O = Wistron Inc.
OU = IT
CN = 192.168.50.191

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.localhost
DNS.2 = localhost
IP.1 = 127.0.0.1
IP.2 = 192.168.50.191
```

!!! tip
    您需要關注的兩個關鍵設定是 **CN 欄位** 和底部的 **alt_names** 部分。Chrome會擋掉沒有寫在[alt_names]的來源，所以訪問來源要確實寫好

`CN 欄位` 需要是服務器的 IP 地址。 `alt_names` 部分必須有一個帶有 IP 地址的條目。

```bash
# Generate CSR
openssl req -new -key server.key -out server.csr
```

### 3. 籤發證書進行簽發，生成 x509證書

使用 CA 證書及CA密鑰 對請求籤發證書進行簽發，生成 x509證書

```bash
# Generate Self Signed certificate
openssl x509 -req -days 365 -in server.csr -signkey server.key \
 -CA ca.crt -CAkey ca.key CAcreateserial -out server.crt
```
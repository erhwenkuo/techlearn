# NVIDIA GPU 的 MIG 切割編輯器

原文: [MIG Partiton Editor for NVIDIA GPUs](https://github.com/NVIDIA/mig-parted)

MIG（Multi-Instance GPU 的縮寫）是最新一代 NVIDIA Ampere GPU 中的一種操作模式。它允許將一個 GPU 劃分為一組“MIG 設備”，在使用它們的軟件看來，每個設備都像一個迷你 GPU，具有固定的內存分區和固定的計算資源分區。有關 MIG 及其提供的功能的詳細說明，請參閱 [MIG 用戶指南](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html)。



## 安裝

MIG Partiton Editor (`nvidia-mig-parted`) 是一款專為系統管理員設計的工具，可讓他們更輕鬆地處理 MIG 分割。

它允許管理員以聲明方式定義一組他們希望應用於節點上所有 GPU 的可能的 MIG 配置。在運行時，將 `nvidia-mig-parted` 指向這些預先準備好的 MIG 配置，`nvidia-mig-parted` 負責應用這些配置聲明來切割 Nvidia GPU。

通過這種方式，相同的配置文件可以分佈在集群中的所有節點上，並且可以使用運行時標誌（或環境變量）來決定在任何給定時間將這些配置中的哪些實際應用於節點。

請在[此處](https://github.com/NVIDIA/mig-parted/releases)的發布頁面下載 `nvidia-mig-parted` 並安裝它們。


下載最新的 `nvidia-mig-parted`(v0.5.0):

```bash
wget https://github.com/NVIDIA/mig-parted/releases/download/v0.5.0/nvidia-mig-manager-0.5.0-1.x86_64.tar.gz

tar -xzvf nvidia-mig-manager-0.5.0-1.x86_64.tar.gz

cd nvidia-mig-manager-0.5.0-1

sudo cp nvidia-mig-parted /usr/local/bin
```

解壓縮的目錄裡:

``` hl_lines="9"
.
├── LICENSE
├── config-default.yaml
├── hooks-default.yaml
├── hooks-minimal.yaml
├── hooks.sh
├── install.sh
├── nvidia-mig-manager.service
├── nvidia-mig-parted
├── nvidia-mig-parted.sh
├── override.conf
├── service.sh
├── uninstall.sh
└── utils.sh
```

## 快速上手

在深入了解 `nvidia-mig-parted` 的可能選項的細節之前，先看幾個最常見用法的示例是很有用的。下面的所有命令都會使用一個 `config.yaml` 的範例配置文件。

```yaml title="config.yaml"
version: v1
mig-configs:
  all-disabled:
    - devices: all
      mig-enabled: false

  all-enabled:
    - devices: all
      mig-enabled: true
      mig-devices: {}

  all-1g.10gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "1g.10gb": 7

  all-2g.20gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3

  all-3g.40gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "3g.40gb": 2

  all-balanced:
    - devices: all
      mig-enabled: true
      mig-devices:
        "1g.10gb": 2
        "2g.20gb": 1
        "3g.40gb": 1

  custom-config:
    - devices: [0,1,2,3]
      mig-enabled: false
    - devices: [4]
      mig-enabled: true
      mig-devices:
        "1g.10gb": 7
    - devices: [5]
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3
    - devices: [6]
      mig-enabled: true
      mig-devices:
        "3g.40gb": 2
    - devices: [7]
      mig-enabled: true
      mig-devices:
        "1g.10gb": 2
        "2g.20gb": 1
        "3g.40gb": 1
```

從配置文件應用特定的 MIG 配置:

```bash
nvidia-mig-parted apply -f config.yaml -c all-1g.10gb
```

應用配置以僅更改配置的 MIG 模式設置:

```bash
nvidia-mig-parted apply --mode-only -f config.yaml -c all-1g.10gb
```

應用帶有調試輸出的 MIG 配置:

```bash
nvidia-mig-parted -d apply -f config.yaml -c all-1g.10gb
```

在沒有配置文件的情況下應用一次性 MIG 配置:

```bash
cat <<EOF | nvidia-mig-parted apply -f -
version: v1
mig-configs:
  all-1g.10gb:
  - devices: all
    mig-enabled: true
    mig-devices:
      1g.10gb: 7
EOF
```

應用一次性 MIG 配置以僅更改 MIG 模式:

```bash
cat <<EOF | nvidia-mig-parted apply --mode-only -f -
version: v1
mig-configs:
  whatever:
  - devices: all
    mig-enabled: true
    mig-devices: {}
EOF
```

導出當前的 MIG 配置:

```bash
nvidia-mig-parted export
```

檢查當前應用了特定的 MIG 配置:

```bash
nvidia-mig-parted assert -f config.yaml -c all-1g.10gb
```

檢查並執行當前應用了 MIG 配置的 MIG 模式設置:

```bash
nvidia-mig-parted assert --mode-only -f config.yaml -c all-1g.10gb
```

在沒有配置文件的情況下檢查並執行一次性 MIG 配置:

```bash
cat <<EOF | nvidia-mig-parted assert -f -
version: v1
mig-configs:
  all-1g.5gb:
  - devices: all
    mig-enabled: true
    mig-devices: 
      1g.5gb: 7
EOF
```

檢查一次性 MIG 配置的 MIG 模式設置:

```bash
cat <<EOF | nvidia-mig-parted assert --mode-only -f -
version: v1
mig-configs:
  whatever:
  - devices: all
    mig-enabled: true
    mig-devices: {}
EOF
```

### 練習

創建一個範例 MIG 配置檔 `config.yaml` 來對一個擁有 8　張 NVIDIA A100(80gb) 的 GPU 節點進行 MIG 切割的設定:

```yaml title="config.yaml"
version: v1
mig-configs:
  all-disabled:
    - devices: all
      mig-enabled: false

  all-enabled:
    - devices: all
      mig-enabled: true
      mig-devices: {}

  all-1g.10gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "1g.10gb": 7

  all-2g.20gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3

  all-3g.40gb:
    - devices: all
      mig-enabled: true
      mig-devices:
        "3g.40gb": 2

  all-balanced:
    - devices: all
      mig-enabled: true
      mig-devices:
        "1g.10gb": 2
        "2g.20gb": 1
        "3g.40gb": 1

  custom-config:
    - devices: [0,1,2,3]
      mig-enabled: false
    - devices: [4]
      mig-enabled: true
      mig-devices:
        "1g.10gb": 7
    - devices: [5]
      mig-enabled: true
      mig-devices:
        "2g.20gb": 3
    - devices: [6]
      mig-enabled: true
      mig-devices:
        "3g.40gb": 2
    - devices: [7]
      mig-enabled: true
      mig-devices:
        "1g.10gb": 2
        "2g.20gb": 1
        "3g.40gb": 1
```

`mig-configs` 下的每個部分都是由用戶定義的 MIG 分割配置，當要啟動時可利用這些自定義標籤來引用它們。

例如，`all-disabled` 標籤是指為節點上的所有 GPU 禁用 MIG 的 MIG 配置。同樣，`all-1g.10gb` 標籤指的是將節點上的所有 GPU 切片為 `1g.10gb` 設備的 MIG 配置。

最後，`custom-config` 標籤定義了一個完全自定義的配置，它在節點的前 4 個 GPU 上禁用 MIG，並在其餘部分應用混合的 MIG 設備。

使用此工具，可以運行以下命令以依次應用這些配置中的每一個:

```bash
sudo nvidia-mig-parted apply -f config.yaml -c all-disabled
```

結果:

```
MIG configuration applied successfully
```

用 `nvidia-smi` 查看:

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.86.01    Driver Version: 515.86.01    CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100 80G...  On   | 00000001:00:00.0 Off |                   On |
| N/A   35C    P0    45W / 300W |      0MiB / 81920MiB |     N/A      Default |
|                               |                      |            Disabled* |
+-------------------------------+----------------------+----------------------+
```

由 MIG M. 是 `Disabled*` 可了解必需要重新開機才會生效。

```bash
sudo shutdown now -r
```

重新用`nvidia-smi` 查看:

```bash
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.86.01    Driver Version: 515.86.01    CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100 80G...  On   | 00000001:00:00.0 Off |                    0 |
| N/A   35C    P0    45W / 300W |     99MiB / 81920MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
```

接下來要來配置 MIG 的相關配置, 由於之後我們把 MIG mode 給停用了先啟用起來:

```bash
sudo nvidia-smi -mig 1
```

重新開機才會生效。

```bash
sudo shutdown now -r
```

啟用 `all-1g.10gb` 的配置:

```bash
sudo nvidia-mig-parted apply -f config.yaml -c all-1g.10gb
```

用`nvidia-smi` 查看:

```bash
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.86.01    Driver Version: 515.86.01    CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100 80G...  On   | 00000001:00:00.0 Off |                   On |
| N/A   34C    P0    44W / 300W |     45MiB / 81920MiB |     N/A      Default |
|                               |                      |              Enabled |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| MIG devices:                                                                |
+------------------+----------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |         Memory-Usage |        Vol|         Shared        |
|      ID  ID  Dev |           BAR1-Usage | SM     Unc| CE  ENC  DEC  OFA  JPG|
|                  |                      |        ECC|                       |
|==================+======================+===========+=======================|
|  0    7   0   0  |      6MiB /  9728MiB | 14      0 |  1   0    0    0    0 |
|                  |      0MiB / 16383MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0    8   0   1  |      6MiB /  9728MiB | 14      0 |  1   0    0    0    0 |
|                  |      0MiB / 16383MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0    9   0   2  |      6MiB /  9728MiB | 14      0 |  1   0    0    0    0 |
|                  |      0MiB / 16383MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0   10   0   3  |      6MiB /  9728MiB | 14      0 |  1   0    0    0    0 |
|                  |      0MiB / 16383MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0   11   0   4  |      6MiB /  9728MiB | 14      0 |  1   0    0    0    0 |
|                  |      0MiB / 16383MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0   12   0   5  |      6MiB /  9728MiB | 14      0 |  1   0    0    0    0 |
|                  |      0MiB / 16383MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0   13   0   6  |      6MiB /  9728MiB | 14      0 |  1   0    0    0    0 |
|                  |      0MiB / 16383MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
```

啟用 `all-2g.20gb` 的配置:

```bash
sudo nvidia-mig-parted apply -f config.yaml -c all-2g.20gb
```

用 `nvidia-smi` 查看:

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.86.01    Driver Version: 515.86.01    CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100 80G...  On   | 00000001:00:00.0 Off |                   On |
| N/A   35C    P0    45W / 300W |     39MiB / 81920MiB |     N/A      Default |
|                               |                      |              Enabled |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| MIG devices:                                                                |
+------------------+----------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |         Memory-Usage |        Vol|         Shared        |
|      ID  ID  Dev |           BAR1-Usage | SM     Unc| CE  ENC  DEC  OFA  JPG|
|                  |                      |        ECC|                       |
|==================+======================+===========+=======================|
|  0    3   0   0  |     13MiB / 19968MiB | 28      0 |  2   0    1    0    0 |
|                  |      0MiB / 32767MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0    4   0   1  |     13MiB / 19968MiB | 28      0 |  2   0    1    0    0 |
|                  |      0MiB / 32767MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0    5   0   2  |     13MiB / 19968MiB | 28      0 |  2   0    1    0    0 |
|                  |      0MiB / 32767MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
```

啟用 `all-3g.40gb` 的配置:

```bash
sudo nvidia-mig-parted apply -f config.yaml -c all-3g.40gb
```

用 `nvidia-smi` 查看:

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.86.01    Driver Version: 515.86.01    CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100 80G...  On   | 00000001:00:00.0 Off |                   On |
| N/A   35C    P0    45W / 300W |     38MiB / 81920MiB |     N/A      Default |
|                               |                      |              Enabled |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| MIG devices:                                                                |
+------------------+----------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |         Memory-Usage |        Vol|         Shared        |
|      ID  ID  Dev |           BAR1-Usage | SM     Unc| CE  ENC  DEC  OFA  JPG|
|                  |                      |        ECC|                       |
|==================+======================+===========+=======================|
|  0    1   0   0  |     19MiB / 40192MiB | 42      0 |  3   0    2    0    0 |
|                  |      0MiB / 65535MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0    2   0   1  |     19MiB / 40192MiB | 42      0 |  3   0    2    0    0 |
|                  |      0MiB / 65535MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
```

啟用 `all-balanced` 的配置:

```bash
sudo nvidia-mig-parted apply -f config.yaml -c all-balanced
```

用 `nvidia-smi` 查看:

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.86.01    Driver Version: 515.86.01    CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A100 80G...  On   | 00000001:00:00.0 Off |                   On |
| N/A   35C    P0    45W / 300W |     45MiB / 81920MiB |     N/A      Default |
|                               |                      |              Enabled |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| MIG devices:                                                                |
+------------------+----------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |         Memory-Usage |        Vol|         Shared        |
|      ID  ID  Dev |           BAR1-Usage | SM     Unc| CE  ENC  DEC  OFA  JPG|
|                  |                      |        ECC|                       |
|==================+======================+===========+=======================|
|  0    2   0   0  |     19MiB / 40192MiB | 42      0 |  3   0    2    0    0 |
|                  |      0MiB / 65535MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0    3   0   1  |     13MiB / 19968MiB | 28      0 |  2   0    1    0    0 |
|                  |      0MiB / 32767MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0    9   0   2  |      6MiB /  9728MiB | 14      0 |  1   0    0    0    0 |
|                  |      0MiB / 16383MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
|  0   10   0   3  |      6MiB /  9728MiB | 14      0 |  1   0    0    0    0 |
|                  |      0MiB / 16383MiB |           |                       |
+------------------+----------------------+-----------+-----------------------+
```

然後可以通過以下方式查找當前應用的配置:

```bash
sudo nvidia-mig-parted export
```

結果:

```
version: v1
mig-configs:
  current:
  - devices: all
    mig-enabled: true
    mig-devices:
      1g.10gb: 2
      2g.20gb: 1
      3g.40gb: 1
```

檢查設定檔的格式與內容:

```bash
sudo nvidia-mig-parted assert -f config.yaml -c all-balanced -m
```

參數：

- `-f config.yaml` 配置文件的路徑
- `-c all-balanced` 設定要應用於節點的配置文件 mig-config 裡的設定標籤
- `-m` 僅驗證來自所選配置的 MIG 模式設置的正確性，而不會配置 MIG 設備

!!! info
    注意：單獨的 `nvidia-mig-parted` 工具無法確保您的節點處於 MIG 模式更改和 MIG 設備配置將乾淨地應用的狀態。此外，它不能確保 MIG 設備配置在節點重新啟動後仍然存在。

    為了解決這個問題，開發了一個 `systemd` 服務和一組支持腳本來包裝 `nvidia-mig-parted` 並提供這些非常需要的功能。請參閱 [deployments/systemd](https://github.com/NVIDIA/mig-parted/tree/main/deployments/systemd) 下的 README.md 以獲取更多詳細信息。

    如果是使用 Nvidia GPU Operator 在 Kubernetes 裡來管理 mig 配置的話, MIG 配置的保存手法會利用 Kubernetes 的機制來進行。詳情見:[GPU Operator 與 MIG 配置](./gpu-operator-mig.md)

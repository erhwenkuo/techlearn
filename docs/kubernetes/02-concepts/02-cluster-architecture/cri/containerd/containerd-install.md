# 如何在 Ubuntu 22.04 上安裝 Containerd

![](./assets/footer-logo.png)

原文: [如何在 Ubuntu 22.04 上安装 Containerd 容器运行时](https://0xzx.com/2022091003502621716.html)

Containerd 是一個 Container Runtime，支持 OCI Image Spec 和 OCI Runtime Spec（Open Container Initiative）。 Containerd 的創建強調容器部署和生命週期的簡單性、健壯性和可移植性。它的設計目的是易於嵌入到 Docker 和 Kubernetes 等後期系統。在底層，新的 Docker 版本使用 Containerd 來管理容器生命週期。對於 Kubernetes，你可以通過 CRI 使用 Containerd 作為容器運行時來管理 Kubernetes 集群上的容器生命週期。

本教程將介紹如何在 Ubuntu 22.04 服務器上安裝 Containerd Container Runtime。本教程將介紹兩種不同的安裝 Containerd 的方法，手動下載二進制包或通過 APT 存儲庫安裝 Containerd。在本教程結束時，你將擁有用於開發的容器環境，你也可以將其用作 Kubernetes 集群安裝的一部分。

![](./assets/containerd-1.1.png)

## 先決條件

要學習本教程，你需要滿足以下要求：

- 一台 Ubuntu 22.04 服務器
- 具有 root/sudo 管理權限的非 root 用戶

## 手動安裝 Containerd

### 安裝 Containerd Container Runtime

第一步，你將學習通過二進制包手動安裝 Containerd Container Runtime。這意味著你還需要為 Containerd 本身安裝一些組件，包括 `runc` 和 `CNI（容器網絡接口）插件`。

Containerd 及其所有組件都可以在其 GitHub 存儲庫中找到並準備安裝。

要安裝 Containerd，請查看 Containerd GitHub 發布頁面以獲取最新版本。然後，複製鏈接並使用 `wget` 下載Containerd的二進制包。之後，將文件解壓縮到 `/usr/local` 目錄。

在撰寫本文時，Containerd 的最新版本是 v1.6.15。使用 `wget` 命令將其下載到你的 Ubuntu 服務器，並通過 `tar` 命令將其解壓縮到 `/usr/local` 目錄。

```bash title="執行下列命令  >_"
sudo su

wget https://github.com/containerd/containerd/releases/download/v1.6.15/containerd-1.6.15-linux-amd64.tar.gz

tar Cxzvf /usr/local containerd-1.6.15-linux-amd64.tar.gz
```

結果:

``` title="/usr/local"
bin/
bin/containerd-stress
bin/containerd-shim
bin/containerd-shim-runc-v1
bin/containerd-shim-runc-v2
bin/containerd
bin/ctr
```

### 安裝 Runc

**Runc** 是一個命令行工具，用於根據 OCI（Open Container Initiative）規範在 Linux 系統上生成容器。要安裝 `runc`，你需要訪問官方 `runc` GitHub 發布頁面並獲取最新版本的 `runc` 並使用 `wget` 命令下載它。

在撰寫本文時，最新版本的 `runc` 是 v1.1.4。使用 `wget` 命令下載並安裝如下。

```bash title="執行下列命令  >_"
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64

sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

現在使用以下命令檢查 `runc` 二進製文件。你應該看到 `runc` 二進製文件在 `/usr/local/sbin` 目錄中可用。

```bash title="執行下列命令  >_"
$ which runc

/usr/local/sbin/runc
```

### 安裝 CNI 插件

現在你需要為 Containerd Container Runtime 安裝 CNI（容器網絡接口）插件。 CNI 插件為 Linux 系統上的容器提供網絡功能。

在下載 CNI 插件之前，請務必訪問官方 GitHub 發布頁面以獲取最新版本的 CNI 插件。

在撰寫本文時，CNI 插件的最新版本是 `v1.1.1`。使用 `wget` 命令下載它，如下所示。

```bash title="執行下列命令  >_"
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
```

接下來，新建一個目錄 `/opt/cni/bin`，作為 CNI 插件的目標安裝目錄。然後，通過 `tar` 命令將 CNI 插件解壓縮到其中。

```bash title="執行下列命令  >_"
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

### 配置 Containerd

安裝完所有三個 Containerd Container Runtime 組件後，你將開始配置 Containerd。

運行以下命令創建一個新目錄 `/etc/containerd`。然後，使用 “containerd” 命令生成默認的 Containerd 配置，如下所示。

```bash title="執行下列命令  >_"
sudo mkdir -p /etc/containerd/

containerd config default | sudo tee /etc/containerd/config.toml
```

現在運行以下命令為 Containerd 容器運行時啟用 `SystemdCgroup`。此命令會將 Containerd 配置文件 `/etc/containerd/config.toml` 上的選項 `SystemdCgroup = false` 替換為 `SystemdCgroup = true`。

```bash title="執行下列命令  >_"
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

現在運行以下命令，將 Containerd 的 systemd 服務文件下載到 `/etc/systemd/system` 目錄。

```bash title="執行下列命令  >_"
sudo curl -L https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -o /etc/systemd/system/containerd.service
```

然後，重新加載 systemd 管理器以應用新的服務文件。

```bash title="執行下列命令  >_"
sudo systemctl daemon-reload
```

最後，使用以下 systemctl 命令啟動並啟用 `containerd` 服務。如果你的安裝成功，你將看到在 `containerd` 服務啟動過程中沒有發現錯誤消息。

```bash title="執行下列命令  >_"
sudo systemctl start containerd
sudo systemctl enable containerd
```

使用以下命令檢查並驗證 `containerd` 服務。你應該看到 `containerd` 服務當前處於 “active(running)” 狀態並且已啟用，這意味著將在系統啟動時自動運行。

```bash title="執行下列命令  >_"
sudo systemctl status containerd
```

## 通過 APT Docker 存儲庫安裝 Containerd

### 安裝 Containerd Container Runtime

現在你將學習如何通過 APT Docker 存儲庫安裝 Containerd Container Runtime。

首先，運行以下命令來安裝一些包依賴項。當提示確認安裝時，輸入 Y 並按 ENTER。

```bash title="執行下列命令  >_"
sudo apt install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

接下來，運行以下命令來創建一個新目錄並下載 Docker 存儲庫的 GPG 密鑰。

```bash title="執行下列命令  >_"
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

使用以下命令將 Docker 存儲庫添加到你的系統。

```bash title="執行下列命令  >_"
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

現在運行以下 apt 命令來更新和刷新 Ubuntu 系統的包索引。然後，安裝包 `containerd.io` 作為 Containerd Container Runtime。安裝將自動開始。

```bash title="執行下列命令  >_"
sudo apt update

sudo apt install containerd.io
```

安裝完成後，使用以下 `systemctl` 命令啟動並啟用 `containerd` 服務。

```bash title="執行下列命令  >_"
sudo systemctl start containerd
sudo systemctl enable containerd
```

然後，使用以下命令檢查並驗證 “containerd” 服務。你應該會看到 “containerd” 服務處於活動狀態並正在運行。

```bash title="執行下列命令  >_"
sudo systemctl status containerd
```

結果:

```
vagrant@ubuntu-jammy:~$ sudo systemctl status containerd
● containerd.service - containerd container runtime
     Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2023-01-06 07:25:23 UTC; 1min 14s ago
       Docs: https://containerd.io
   Main PID: 2657 (containerd)
      Tasks: 8
     Memory: 13.4M
        CPU: 429ms
     CGroup: /system.slice/containerd.service
             └─2657 /usr/bin/containerd

Jan 06 07:25:23 ubuntu-jammy containerd[2657]: time="2023-01-06T07:25:23.885534908Z" level=info msg="loading plugin \"io.containerd.grpc.v1.tasks\"..." >
Jan 06 07:25:23 ubuntu-jammy containerd[2657]: time="2023-01-06T07:25:23.885684866Z" level=info msg="loading plugin \"io.containerd.grpc.v1.version\"...>
Jan 06 07:25:23 ubuntu-jammy containerd[2657]: time="2023-01-06T07:25:23.885826498Z" level=info msg="loading plugin \"io.containerd.tracing.processor.v1>
Jan 06 07:25:23 ubuntu-jammy containerd[2657]: time="2023-01-06T07:25:23.885996801Z" level=info msg="skip loading plugin \"io.containerd.tracing.process>
Jan 06 07:25:23 ubuntu-jammy containerd[2657]: time="2023-01-06T07:25:23.886163134Z" level=info msg="loading plugin \"io.containerd.internal.v1.tracing\>
Jan 06 07:25:23 ubuntu-jammy containerd[2657]: time="2023-01-06T07:25:23.886312776Z" level=error msg="failed to initialize a tracing processor \"otlp\"">
Jan 06 07:25:23 ubuntu-jammy containerd[2657]: time="2023-01-06T07:25:23.886679588Z" level=info msg=serving... address=/run/containerd/containerd.sock.t>
Jan 06 07:25:23 ubuntu-jammy containerd[2657]: time="2023-01-06T07:25:23.886861616Z" level=info msg=serving... address=/run/containerd/containerd.sock
Jan 06 07:25:23 ubuntu-jammy systemd[1]: Started containerd container runtime.
Jan 06 07:25:23 ubuntu-jammy containerd[2657]: time="2023-01-06T07:25:23.890647614Z" level=info msg="containerd successfully booted in 0.750610s"
```

### 安裝 CNI 插件

接下來，你還需要通過 APT Docker 存儲庫為 Containerd 安裝安裝 CNI（容器網絡接口）插件。

運行下面的 wget 命令下載 CNI 插件。

```bash title="執行下列命令  >_"
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
```

現在使用以下命令創建一個新目錄 `/opt/cni/bin`。然後，通過下面的 tar 命令解壓 CNI 插件。

```bash title="執行下列命令  >_"
sudo mkdir -p /opt/cni/bin

sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

### 配置 Containerd

接下來，你將為 Containerd Container Runtime 生成一個新的配置文件。

運行以下命令以後端 Docker 存儲庫提供的默認配置。然後，使用“containerd”命令生成一個新的配置文件，如下所示。

```bash title="執行下列命令  >_"
sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.orig

containerd config default | sudo tee /etc/containerd/config.toml
```

現在運行以下命令為 Containerd 啟用“SystemdCgroup”。

```bash title="執行下列命令  >_"
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

最後，運行以下命令重新啟動 Containerd 服務並應用新更改。

```bash title="執行下列命令  >_"
sudo systemctl restart containerd
```

此時，Containerd 容器運行時現在與 CNI 插件和 SystemdCgroup 一起運行。

使用以下命令檢查並驗證 `containerd` 服務。你應該看到 `containerd` 服務當前處於 “active(running)” 狀態並且已啟用，這意味著將在系統啟動時自動運行。

```bash title="執行下列命令  >_"
sudo systemctl status containerd
```

## 使用 nerdctl 與 Containerd 通信

`nrdctl` 是一個命令行工具，用於管理 Container Runtime 的容器。它與 Docker CLI for Docker 兼容，並且具有與 `docker` 命令相同的 UI/UX。

`nrdctl` 命令行通過 `nerdctl compose` 命令支持 Docker Compose，還支持容器 rootless 模式，通過 OverlayBD 和 Stargz 進行延遲拉取鏡像，支持加密貨幣鏡像，以及通過 IPFS 進行 P2P 鏡像分發。

運行以下命令下載 `nerctl` 二進製文件。然後，通過 `tar` 命令將其解壓到 `/usr/local/bin` 目錄。

```bash title="執行下列命令  >_"
wget https://github.com/containerd/nerdctl/releases/download/v1.1.0/nerdctl-1.1.0-linux-amd64.tar.gz

sudo tar Cxzvf /usr/local/bin nerdctl-1.1.0-linux-amd64.tar.gz
```

現在使用以下命令檢查 nerdctl 二進製文件。

```bash title="執行下列命令  >_"
which nerdctl
```

接下來，你將學習如何通過 `nerdctl` 命令使用 Containerd 容器運行時運行容器。 `nerdctl` 命令行工具是 Docker Compatible，所以管理容器的命令與 Docker 命令類似。

使用下面的 `nerdctl` 命令運行名為 `nginx` 的新容器。 `nginx` 容器將在容器和主機的端口 80 上運行，它基於映像 `nginx:alpine`。

```bash title="執行下列命令  >_"
sudo nerdctl run -d --name nginx -p 80:80 nginx:alpine
```

現在使用下面的 `nerdctl` 命令檢查系統上當前運行的容器。你應該會看到 nginx 容器正在運行。

```bash title="執行下列命令  >_"
sudo nerdctl ps
```

接下來，使用以下命令檢查並驗證系統上可用容器映像的列表。你應該會看到下載的 `nginx:alpine` 鏡像。

```bash title="執行下列命令  >_"
sudo nerdctl images
```

最後，運行下面的 `curl` 命令來檢查 nginx 容器。你應該會看到 nginx 容器的默認 `index.html` 頁面。

```bash title="執行下列命令  >_"
curl localhost
```

此外，你還可以使用以下 `nerdctl` 命令檢查和驗證 nginx 容器的日誌。

```bash title="執行下列命令  >_"
sudo nerdctl logs nginx
```

## 結論

在本教程中，你學習了兩種安裝 Containerd 容器運行時的方法。你已經學習瞭如何通過下載 Container 二進制包並通過 APT Docker 存儲庫安裝 Containerd 來手動安裝 Containerd。

最後，你現在已經了解瞭如何安裝 nerdctl 命令行工具以及 nerdctl 用於管理在 Containerd 容器運行時環境下運行的容器的基本用法。你現在還可以使用 Containerd 容器運行時設置 Kubernetes 集群。

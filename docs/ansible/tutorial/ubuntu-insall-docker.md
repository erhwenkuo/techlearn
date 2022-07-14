# 如何在 Ubuntu 20.04 上安裝 Docker

原文: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04

Docker 是一個應用程序，它簡化了在容器中管理應用程序進程的過程。容器讓你可以在資源隔離的進程中運行你的應用程序。它們類似於虛擬機，但容器更便攜、更資源友好，並且更依賴於主機操作系統。

在本教程中，你將在 Ubuntu 20.04 上安裝和使用 Docker 社區版 (CE)。你將安裝 Docker 本身，使用容器和映像檔，並將映像檔推送到 Docker 存儲庫。

## 先決條件

要學習本教程，你將需要準備以下環境：

- 按照 Ubuntu 20.04 初始服務器設置指南設置一台 Ubuntu 20.04 服務器，包括 sudo 非 root 用戶和防火牆。(在本教程中我們使用Vagrant與VirtualBox來啟動一台VM)
- Docker Hub 上的帳戶，如果你希望創建自己的圖像並將它們推送到 Docker Hub，如步驟 7 和 8 所示。

讓我們創建一個新目錄并且切換到新目錄裡。

```
$ mkdir ubuntu_install_docker
$ cd ubuntu_install_docker
$ vagrant init ubuntu/focal64
$ vagrant up
$ vagrant ssh
```

使用 Vagrant 與 VirtualBox 來在本機啟動一台 Ubuntu 20.04 的虛擬機。

```
$ vagrant init ubuntu/focal64
$ vagrant up
$ vagrant ssh
```

結果:

```
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-113-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Jun  7 21:29:14 UTC 2022

  System load:  0.06              Processes:               122
  Usage of /:   3.5% of 38.71GB   Users logged in:         0
  Memory usage: 21%               IPv4 address for enp0s3: 10.0.2.15
  Swap usage:   0%


1 update can be applied immediately.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
```

## 安裝 Docker

Ubuntu 官方存儲庫中提供的 Docker 安裝包可能不是最新版本。為確保我們獲得最新版本，我們將從官方 Docker 存儲庫安裝 Docker。為此，我們將添加一個新的套件源，從 Docker 添加 GPG 密鑰以確保下載有效，然後安裝套件。

首先，更新現有的軟件套件列表：

```bash
$ sudo apt update
```

接下來，安裝一些必備套件，讓 `apt` 可通過 HTTPS 來下載套件：

```bash
$ sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

然後將官方 Docker 存儲庫的 GPG 密鑰添加到你的系統：

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

將 Docker 存儲庫添加到 APT 套件源列表：

```
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```

這也將使用新添加的 repo 中的 Docker 套件更新我們的套件數據庫。

確保你要從 Docker 存儲庫而不是默認的 Ubuntu 存儲庫進行安裝：

```
$ apt-cache policy docker-ce
```

你會看到這樣的輸出，儘管 Docker 的版本號可能不同：

```
docker-ce:
  Installed: (none)
  Candidate: 5:20.10.17~3-0~ubuntu-focal
  Version table:
     5:20.10.17~3-0~ubuntu-focal 500
        500 https://download.docker.com/linux/ubuntu focal/stable amd64 Packages
    ...
    ...
```

請注意，此時我們還未安裝 docker-ce，但安裝的候選者來自 Ubuntu 20.04 的 Docker 存儲庫。

最後，安裝 Docker：

```
$ sudo apt install docker-ce
```

Docker 現在應該已安裝，守護程序已啟動，並且該進程已啟用以在虛擬機啟動時啟動。檢查它是否正在運行：

```bash
$ sudo systemctl status docker
```

輸出應類似於以下內容，表明該服務處於活動狀態且正在運行：

```
Processing triggers for systemd (245.4-4ubuntu3.17) ...
vagrant@ubuntu-focal:~$ sudo systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-06-07 21:53:53 UTC; 1min 39s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 5211 (dockerd)
      Tasks: 8
     Memory: 30.6M
     CGroup: /system.slice/docker.service
             └─5211 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

```
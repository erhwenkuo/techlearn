# 如何在 Ubuntu 22.04 上安裝和配置 Ansible

原文: https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-22-04

配置管理系統旨在為管理員和運營團隊簡化控制大量服務器的過程。它們允許您從一個中央位置以自動化方式控制許多不同的系統。

雖然有許多流行的配置管理工具可用於 Linux 系統，例如 Chef 和 Puppet，但這些工具通常比許多人想要或需要的複雜。 Ansible 是這些選項的絕佳替代品，因為它提供的架構不需要在節點上安裝特殊軟件，使用 SSH 來執行自動化任務並使用 YAML 文件來定義配置細節。

在本教程中，我們將討論如何在 `Ubuntu 22.04` 服務器上安裝 Ansible，並介紹如何使用該軟件的一些基礎知識。有關 Ansible 作為配置管理工具的更高級概述，請參閱 [An Introduction to Configuration Management with Ansible](https://www.digitalocean.com/community/conceptual_articles/an-introduction-to-configuration-management-with-ansible)。

## 先決條件

要遵循本教程，您將需要：

1. 一個 Ansible 控制節點：Ansible 控制節點是我們將用來通過 SSH 連接和控制 Ansible 主機的機器。您的 Ansible 控制節點可以是您的本地機器或專用於運行 Ansible 的服務器，但本指南假定您的控制節點是 Ubuntu 22.04 系統。確保控制節點具有：
     - **具有 `sudo` 權限的非 root 用戶**。要進行此設置，您可以按照我們的 [Ubuntu 22.04 初始服務器設置指南](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04)的第 2 步和第 3 步進行操作。但是，請注意，如果您使用遠程服務器作為 Ansible 控制節點，則應遵循本指南的每個步驟。這樣做將使用 `ufw` 在服務器上配置防火牆並啟用對非 root 用戶配置文件的外部訪問，這兩者都將有助於保持遠程服務器的安全。
     - **與此用戶關聯的 SSH 密鑰對**。要進行設置，您可以按照我們關於[如何在 Ubuntu 22.04 上設置 SSH 密鑰的指南](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-22-04)的第 1 步進行操作。

### Step 1 — Installing Ansible

要開始使用 Ansible 作為管理服務器基礎架構的一種方式，您需要在將用作 Ansible 控制節點的機器上安裝 Ansible 軟件。我們將為此使用默認的 Ubuntu 存儲庫。

首先，使用以下命令刷新系統的包索引：

```bash
$ sudo apt update
```

在此更新之後，您可以安裝 Ansible 軟件：

```bash
$ sudo apt install ansible
```

提示確認安裝時按 `Y`。如果系統提示您重新啟動任何服務，請按 `ENTER` 接受默認設置並繼續。

您的 Ansible 控制節點現在擁有管理主機所需​​的所有軟件。接下來，我們將介紹如何設置清單文件，以便 Ansible 可以與您的託管節點通信。

### Step 2 — Setting Up the Inventory File

Inventory 清單文件包含有關您將使用 Ansible 管理的主機的信息。您可以在 inventory 清單文件中包含從一台到數百台服務器的任意位置，並且可以將主機組織成群組和子群組。Inventory 清單文件還經常用於設置僅對特定主機或組有效的變量，以便在劇本和模板中使用。一些變量也會影響 playbook 的運行方式，例如我們稍後會看到的 `ansible_python_interpreter` 變量。

要編輯默認 Ansible 的 inventory 清單的內容，首先使用 `mkdir` 命令創建 `/etc/ansible` 目錄。然後在 Ansible 控制節點上使用您選擇的文本編輯器打開 `/etc/ansible/hosts` 文件：

```bash
$ sudo mkdir /etc/ansible
$ sudo nano /etc/ansible/hosts
```

!!! tip
    儘管 Ansible 通常會在 `etc/ansible/hosts` 中創建默認清單文件，但您可以在任何更適合您需求的位置自由創建清單文件。在這種情況下，您需要在運行 Ansible 命令和 playbook 時使用 `-i` 參數提供自定義清單文件的路徑。使用每個項目的清單文件是一種很好的做法，可以最大程度地降低在錯誤的服務器組上運行劇本的風險。

Ansible 安裝提供的默認清單文件包含許多示例，您可以將它們用作設置清單的參考。以下示例定義了一個名為 `[servers]` 的組，其中包含三個不同的服務器，每個服務器由一個自定義別名標識：`server1`、`server2` 和 `server3`。請務必將突出下列顯示的 IP 替換為你要控制的 Ansible 主機的 IP 地址。

```init title="/etc/ansible/hosts"
[servers]
server1 ansible_host=203.0.113.111
server2 ansible_host=203.0.113.112
server3 ansible_host=203.0.113.113

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

`all:vars` 子群組設置 `ansible_python_interpreter` 主機參數，該參數將對該清單中包含的所有主機有效。此參數確保遠程服務器使用 `/usr/bin/python3` Python 3 可執行文件，而不是 `/usr/bin/python `(Python 2.7)，這在最近的 Ubuntu 版本中不存在。

完成後，按 CTRL+X 然後按 Y 和 ENTER 確認更改，保存並關閉文件。

每當您想檢查庫存時，都可以運行：

```bash
$ ansible-inventory --list -y
```

您將看到與此類似的輸出，但包含您自己的服務器基礎設施，如清單文件中所定義：

```yaml
all:
  children:
    servers:
      hosts:
        server1:
          ansible_host: 203.0.113.111
          ansible_python_interpreter: /usr/bin/python3
        server2:
          ansible_host: 203.0.113.112
          ansible_python_interpreter: /usr/bin/python3
        server3:
          ansible_host: 203.0.113.113
          ansible_python_interpreter: /usr/bin/python3
    ungrouped: {}
```

現在您已經配置了清單文件，您擁有測試與 Ansible 主機的連接所需的一切。


### Step 3 — Testing Connection

在設置清單文件以包含您的服務器之後，是時候檢查 Ansible 是否能夠連接到這些服務器並通過 `SSH` 運行命令。

對於本指南，我們將使用 Ubuntu root 帳戶，因為這通常是新創建的服務器上默認可用的唯一帳戶。如果您的 Ansible 主機已經創建了一個普通的 sudo 用戶，我們鼓勵您改用該帳戶。

您可以使用 `-u` 參數來指定遠程系統用戶。如果未提供，Ansible 將嘗試以您當前系統用戶的身份連接到控制節點。

從您的本地機器或 Ansible 控制節點，運行：

```bash
$ ansible all -m ping -u root
```

此命令將使用 Ansible 的內置 `ping` 模塊在默認清單中的所有節點上運行連接測試，以 `root` 身份連接。 `ping` 模塊將測試：

- 主機是否可以訪問；
- 如果您擁有有效的 SSH 憑據；
- 如果主機能夠使用 Python 運行 Ansible 模塊。

你應該得到類似這樣的輸出：

```
server1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
server2 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
server3 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

如果這是您第一次通過 `SSH` 連接到這些服務器，系統會要求您確認通過 Ansible 連接的主機的真實性。出現提示時，鍵入 `yes`，然後按 `ENTER` 確認。

一旦您從主機收到 “`pong`” 回覆，這意味著您已準備好在該服務器上運行 Ansible 命令和 playbook。

!!! info
    如果您無法從服務器獲得成功的響應，請查看我們的 [Ansible 備忘單指南](../ansible-cheatsheet.md)，了解有關如何使用不同連接選項運行 Ansible 命令的更多信息。

### Step 4 — Running Ad-Hoc Commands (Optional)

在確認您的 Ansible 控制節點能夠與您的主機通信後，您可以開始在您的服務器上運行臨時命令和 playbook。

您通常通過 SSH 在遠程服務器上執行的任何命令都可以在清單文件中指定的服務器上使用 Ansible 運行。例如，您可以使用以下命令檢查所有服務器上的磁盤使用情況：

```bash
$ ansible all -a "df -h" -u root
```

結果:

```
server1 | CHANGED | rc=0 >>
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           798M  624K  798M   1% /run
/dev/vda1       155G  2.3G  153G   2% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/vda15      105M  3.6M  101M   4% /boot/efi
tmpfs           798M     0  798M   0% /run/user/0

server2 | CHANGED | rc=0 >>
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
tmpfs           395M  608K  394M   1% /run
/dev/vda1        78G  2.2G   76G   3% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/vda15      105M  3.6M  101M   4% /boot/efi
tmpfs           395M     0  395M   0% /run/user/0

...
```

突出顯示的命令 df -h 可以替換為您想要的任何命令。

您還可以通過 `ad-hoc` 命令執行 Ansible 其它模塊，類似於我們之前使用 `ping` 模塊測試連接所做的操作。例如，以下是我們如何使用 `apt` 模塊在庫存中的所有服務器上安裝最新版本的 `vim`：

```bash
$ ansible all -m apt -a "name=vim state=latest" -u root
```

在運行 Ansible 命令時，您還可以針對單個主機以及組群和子群組。例如，這是您檢查服務器組中每個主機的正常運行時間的方式：

```bash
$ ansible servers -a "uptime" -u root
```

我們可以通過用冒號分隔來指定多個主機：

```bash
$ ansible server1:server2 -m ping -u root
```

有關如何使用 Ansible 的更多信息，包括如何執行 playbook 以自動化服務器設置，您可以查看我們的 [Ansible 參考指南](../ansible-cheatsheet.md)。

## 結論

在本指南中，您已經安裝了 Ansible 並設置了一個清單文件以從 Ansible 控制節點執行臨時命令。

一旦您確認您能夠從中央 Ansible 控制器機器連接和控制您的基礎架構，您就可以在這些主機上執行您想要的任何命令或劇本。
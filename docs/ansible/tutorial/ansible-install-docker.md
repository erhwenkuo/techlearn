# 如何使用 Ansible 在 Ubuntu 20.04 上安裝和設置 Docker

參考: https://www.digitalocean.com/community/tutorials/how-to-use-ansible-to-install-and-set-up-docker-on-ubuntu-20-04

由於現代應用程序環境的一次性性質，服務器自動化現在在系統管理中發揮著重要作用。 Ansible 等配置管理工具通常用於通過為新服務器建立標準程序來簡化自動化服務器設置的過程，同時減少與手動設置相關的人為錯誤。

Ansible 提供了一個簡單的架構，不需要在節點上安裝特殊軟件。它還提供了一組強大的功能和內置模塊，有助於編寫自動化腳本。

本教程說明如何使用 Ansible 自動執行我們關於[如何在 Ubuntu 20.04 上安裝和使用 Docker](ubuntu-insall-docker.md) 的指南中包含的步驟。 Docker 是一個應用程序，它簡化了管理容器的過程，這些資源隔離的進程的行為方式與虛擬機相似，但更便攜、對資源更友好，並且對主機操作系統的依賴程度更高。

## 先決條件

為了執行本指南中的劇本提供的自動設置，你需要：

1. **一個 Ansible 控制節點**：一台安裝了 Ansible 並配置為使用 SSH 密鑰連接到 Ansible 主機的 Ubuntu 20.04 機器。確保控制節點有一個具有 sudo 權限的普通用戶並啟用了防火牆，如我們的[初始服務器設置指南](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04)中所述。要設置 Ansible，請按照我們關於[如何在 Ubuntu 20.04 上安裝和配置 Ansible 的指南](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-20-04)進行操作。
2. **一台或多台 Ansible 主機**：一台或多台遠程 Ubuntu 20.04 服務器之前已按照[如何使用 Ansible 在 Ubuntu 20.04 上自動化初始服務器設置的指南](https://www.digitalocean.com/community/tutorials/how-to-use-ansible-to-automate-initial-server-setup-on-ubuntu-20-04)進行設置。

## 這個 Playbook 有什麼作用？

這個 Ansible playbook 提供了一種替代方法來手動運行我們的指南中概述的過程，該指南是[如何在 Ubuntu 20.04 上安裝和使用 Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04)。設置你的劇本一次，然後在每次安裝後使用它。

運行此 playbook 將在你的 Ansible 主機上執行以下操作：

1. 安裝 `aptitude`，Ansible 首選它作為 `apt` 包管理器的替代品。
2. 安裝所需的系統包。
3. 安裝 Docker GPG APT 密鑰。
4. 將官方 Docker 存儲庫添加到 `apt` 源點。
5. 安裝 Docker。
6. 通過 pip 安裝 Python Docker 模塊。
7. 從 Docker Hub 拉取 `default_container_image` 指定的默認鏡像。
8. 創建由 `container_count` 變量定義的容器數量，每個容器使用 default_container_image 定義的鏡像，並在每個新容器中執行 default_container_command 中定義的命令。

一旦 playbook 完成運行，你將根據你在配置變量中定義的選項創建許多容器。

首先，在 Ansible 控制節點服務器上登錄啟用 `sudo` 的用戶。

### 1. 準備你的 Playbook

`playbook.yml` 文件是定義所有 `task` 的地方。`Task` 是你可以使用 Ansible playbook 自動執行的最小操作單元。但首先，使用你喜歡的文本編輯器創建你的 playbook 文件：

```yaml
$ nano playbook.yml
```

這將打開一個空的 YAML 文件。在開始向你的 playbook 添加任務之前，請先添加以下內容：

```yaml title="playbook.yml"
---
- hosts: all
  become: true
  vars:
    container_count: 4
    default_container_name: docker
    default_container_image: ubuntu
    default_container_command: sleep 1
```

幾乎你遇到的每一個 playbook 都會以類似的聲明開頭。 `hosts` 聲明 Ansible 控制節點將使用此 playbook 定位哪些服務器。 `become` 狀態是否所有命令都將使用升級的 `root` 權限完成。

`vars` 允許你將數據存儲在變量中。如果你決定將來更改這些，你只需在文件中編輯這些單行。以下是每個變量的簡要說明：

- `container_count`：要創建的容器數。
- `default_container_name`：默認容器名稱。
- `default_container_image`：創建容器時使用的默認 Docker 映像。
- `default_container_command`：在新容器上運行的默認命令。


### 2. 將安裝套件的任務添加到你的 Playbook

默認情況下，{++Ansible 在 playbook 中按照從上到下的順序同步執行任務++}。這意味著任務順序很重要，你可以放心地假設一個任務將在下一個任務開始之前完成執行。

此 playbook 中的所有任務都可以獨立存在，並可在你的其他劇本中重複使用。

添加安裝 `aptitude`（與 Linux 包管理器交互的工具）和安裝所需系統包的第一個任務。 Ansible 將確保這些軟件包始終安裝在你的服務器上：

```yaml title="playbook.yml"
tasks:
    - name: Install aptitude
      apt:
        name: aptitude
        state: latest
        update_cache: true

    - name: Install required system packages
      apt:
        pkg:
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
          - python3-pip
          - virtualenv
          - python3-setuptools
        state: latest
        update_cache: true

```

在這裡，你使用 `apt` Ansible 內置模塊來指導 Ansible 安裝你的套件。 Ansible 中的模塊是執行操作的快捷方式，否則你必須將其作為原始 bash 命令運行。如果 aptitude 不可用，Ansible 會安全地使用 apt 來安裝軟件包，但 Ansible 歷來偏愛 aptitude。

你可以根據自己的喜好添加或刪除軟件套件。這將確保所有軟件套件不僅存在，而且在最新版本上，並且在調用 apt 更新後完成。

### 3. 將 Docker 安裝任務添加到你的 Playbook

你的任務將從官方存儲庫安裝最新版本的 Docker。添加 Docker GPG 密鑰以驗證下載，添加官方存儲庫作為新的套件源，並將安裝 Docker。此外，還將安裝 Python 的 Docker 模塊：

```yaml title="playbook.yml"
    - name: Add Docker GPG apt Key
          apt_key:
            url: https://download.docker.com/linux/ubuntu/gpg
            state: present

    - name: Add Docker Repository
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu focal stable
        state: present

    - name: Update apt and install docker-ce
      apt:
        name: docker-ce
        state: latest
        update_cache: true

    - name: Install Docker Module for Python
      pip:
        name: docker
```

你會看到 `apt_key` 和 `apt_repository` 內置 Ansible 模塊首先指向正確的 URL，然後負責確保它們存在。這允許安裝最新版本的 Docker，以及使用 pip 安裝 Python 模塊。

### 4. 將 Docker 映像和容器任務添加到你的 Playbook

Docker 容器的實際創建從拉取所需的 Docker 映像開始。默認情況下，這些鏡像來自官方的 Docker Hub。使用此映像，將根據劇本頂部聲明的變量所規定的規範創建容器：

```yaml title="playbook.yml"
    - name: Pull default Docker image
          community.docker.docker_image:
            name: "{{ default_container_image }}"
            source: pull

    - name: Create default containers
      community.docker.docker_container:
        name: "{{ default_container_name }}{{ item }}"
        image: "{{ default_container_image }}"
        command: "{{ default_container_command }}"
        state: present
      with_sequence: count={{ container_count }}
```

`docker_image` 用於提取要用作容器基礎的 Docker 映像。 `docker_container` 允許你指定你創建的容器的細節，以及你想要傳遞它們的命令。

`with_sequence` 是 Ansible 創建循環的方式，在這種情況下，它將根據你指定的計數循環創建容器。這是一個基本的計數循環，所以這裡的 item 變量提供了一個代表當前循環迭代的數字。這個數字在這裡用來命名你的容器。

### 5. 回顧你的完整 Playbook

你的 playbook 應大致如下所示，根據你的自定義設置略有不同：

```yaml title="playbook.yml"
---
- hosts: all
  become: true
  vars:
    container_count: 4
    default_container_name: docker
    default_container_image: ubuntu
    default_container_command: sleep 1d

  tasks:
    - name: Install aptitude
      apt:
        name: aptitude
        state: latest
        update_cache: true

    - name: Install required system packages
      apt:
        pkg:
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
          - python3-pip
          - virtualenv
          - python3-setuptools
        state: latest
        update_cache: true

    - name: Add Docker GPG apt Key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker Repository
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu focal stable
        state: present

    - name: Update apt and install docker-ce
      apt:
        name: docker-ce
        state: latest
        update_cache: true

    - name: Install Docker Module for Python
      pip:
        name: docker

    - name: Pull default Docker image
      community.docker.docker_image:
        name: "{{ default_container_image }}"
        source: pull

    - name: Create default containers
      community.docker.docker_container:
        name: "{{ default_container_name }}{{ item }}"
        image: "{{ default_container_image }}"
        command: "{{ default_container_command }}"
        state: present
      with_sequence: count={{ container_count }}
```

隨意修改此手冊以最適合你自己工作流程中的個人需求。例如，你可以使用 `docker_image` 模塊將映像推送到 `Docker Hub` 或使用 `docker_container` 模塊來設置容器網絡。

一旦你對你的劇本感到滿意，你可以退出你的文本編輯器並保存。

### 6. 運行你的 Playbook

你現在已準備好在一台或多台服務器上運行此 playbook。大多數劇本默認配置為在你庫存中的每台服務器上執行，但這次你將指定你的服務器。

要僅在 server1 上執行 playbook，以 sammy 身份連接，你可以使用以下命令：

```bash
$ ansible-playbook playbook.yml -l server1 -u sammy
```

-l 標誌指定你的服務器，-u 標誌指定要在遠程服務器上登錄的用戶。你會得到類似這樣的輸出：

```
Output
. . .
changed: [server1]

TASK [Create default containers] *****************************************************************************************************************
changed: [server1] => (item=1)
changed: [server1] => (item=2)
changed: [server1] => (item=3)
changed: [server1] => (item=4)

PLAY RECAP ***************************************************************************************************************************************
server1                  : ok=9    changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

這表明你的服務器設置已完成！你的輸出不必完全相同，但零故障很重要。

當 playbook 運行完成後，通過 SSH 登錄到 Ansible 提供的服務器，檢查容器是否創建成功。

使用以下命令登錄遠程服務器：

```bash
ssh sammy@your_remote_server_ip

vagrant ssh
```

並列出遠程服務器上的 Docker 容器：

```bash
$ sudo docker ps -a
```

你應該會看到與此類似的輸出：

```bash
CONTAINER ID   IMAGE     COMMAND      CREATED         STATUS    PORTS     NAMES
9885b07bb59b   ubuntu    "sleep 1d"   3 minutes ago   Created             docker4
06d42806b828   ubuntu    "sleep 1d"   3 minutes ago   Created             docker3
83a27e8fe4db   ubuntu    "sleep 1d"   3 minutes ago   Created             docker2
c6e52400d0c9   ubuntu    "sleep 1d"   3 minutes ago   Created             docker1
```

這意味著 playbook 中定義的容器已成功創建。由於這是 playbook 中的最後一個任務，因此它還確認 playbook 已在此服務器上完全執行。

## 結論

自動化你的基礎設施設置不僅可以節省你的時間，而且還有助於確保你的服務器遵循可根據你的需求進行定制的標準配置。由於現代應用程序的分佈式特性以及不同暫存環境之間對一致性的需求，這樣的自動化已成為許多團隊開發過程中的核心組件。

在本指南中，你演示瞭如何使用 Ansible 自動化在遠程服務器上安裝和設置 Docker 的過程。因為每個人在使用容器時通常有不同的需求，我們鼓勵你查看官方 Ansible 文檔以獲取更多信息和 docker_container Ansible 模塊的用例。


# 如何使用 Ansible：參考指南

Ansible 是一種現代化的配置管理工具，可幫助您完成設置和維護遠程服務器的任務。

這個 cheatsheet 風格的指南提供了使用 Ansible 時常用的命令和實踐的快速參考。有關 Ansible 的概述以及如何安裝和配置它，請查看我們關於[如何在 Ubuntu 20.04 上安裝和配置 Ansible 的指南]()。

**如何使用本指南：**

- 本指南採用備忘單格式，帶有獨立的命令行片段。
- 跳轉到與您嘗試完成的任務相關的任何部分。
- 當您在本指南的命令中看到{==突出顯示的文本==}時，請記住，此{==文本==}是對應到您自己的清單中的主機、用戶名和 IP 地址。

**Ansible 詞彙表**

本指南中主要使用以下 Ansible 特定術語：

- `Control Machine / Node`: 安裝和配置 Ansible 以在節點上連接和執行命令的系統。
- `Node`: 由 Ansible 控制的服務器。
- `Inventory File`: 包含有關 Ansible 控制的服務器信息的文件，通常位於 `/etc/ansible/hosts`。
- `Playbook`: 包含要在遠程服務器上執行的一系列任務的文件。
- `Role`: 與目標相關的劇本和其他文件的集合，例如安裝 Web 服務器。
- `Play`: 完整的 Ansible 運行。一個劇本可以有多個劇本和角色，包含在一個作為入口點的劇本中。

讓我們使用Ansible與Vagrant來構建一個VM來練習本教程的相關指令。

首先在 Ansible 控制節點上創建一個新目錄，你將在其中設置 Ansible 文件和一個演示靜態 HTML 網站以部署到遠程服務器。這可以在你的主文件夾中你選擇的任何位置。在本例中，我們將使用 `~/ansible-cheatsheet`。讓我們創建一個新目錄并且切換到新目錄裡。

```bash
$ mkdir ansible-cheatsheet
$ cd ansible-cheatsheet
```

使用 Vagrant 與 VirtualBox 來在本機啟動一台 Ubuntu 20.04 的虛擬機。

```bash
$ vagrant init ubuntu/focal64
$ vagrant up
$ vagrant ssh-config
```

取得虛擬機的IP與ssh port:

```hl_lines="4"
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2203
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/dxlab/opt/ws_ansible/ansible-nginx-demo/.vagrant/machines/default/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
```

創建 `inventory` 檔案:

```yaml title="inventory"
demo-vm ansible_host=127.0.0.1 ansible_port=2203
```

- `demo-vm` 我們給予這個虛擬機的一個主機名
- `ansible_host` Ansible 要連線的控制節點 IP
- `ansible_port` Ansible 要使用SSH連線的port number

創建 `ansible.cfg` 檔案:

```ini title="ansible.cfg"
[defaults]
inventory = inventory
remote_user = vagrant
private_key_file = .vagrant/machines/default/virtualbox/private_key
host_key_checking = False
```

- `inventory` Ansible 控制節點主機資訊列表檔案位置與檔名
- `remote_user` Ansible 要連線到節點主機時要登入的使用者ID
- `private_key_file` Ansible 要連線到節點主機時的連線憑證檔案位置與檔名

驗證 Ansible 可連接至 Ubuntu 20.04 的虛擬機:

```bash
$ ansible all -m ping

demo-vm | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

## 測試與節點的連接

要測試 Ansible 是否能夠在您的節點上連接和運行命令和 playbook，您可以使用以下命令：

```bash
ansible all -m ping
```

結果:

```
demo-vm | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

除了測試 Ansible 是否能夠在遠程服務器上運行 Python 腳本之外，`ping` 模塊還將測試您是否具有連接到清單文件中定義的節點的有效憑據。 pong 回复意味著 Ansible 已準備好在該節點上運行命令和 playbook。

## 以不同用戶身份連接

默認情況下，Ansible 會嘗試使用其對應的 SSH 密鑰對以您當前的系統用戶身份連接到節點。要以其他用戶身份連接，請在命令後面附加 `-u` 標誌和預期用戶的名稱：

```bash
$ ansible all -m ping -u vagrant
```

同樣的flag也適用於 ansible-playbook：

```bash
$ ansible-playbook myplaybook.yml -u vagrant
```

## 使用自定義 SSH 密鑰

如果您使用自定義 SSH 密鑰連接到遠程服務器，您可以在執行時使用 `--private-key` 選項提供它：

```bash
$ ansible all -m ping --private-key=~/.ssh/custom_id
```

此選項對 ansible-playbook 也有效：

```bash
ansible-playbook myplaybook.yml --private-key=~/.ssh/custom_id
```

## 使用基於密碼的身份驗證

如果您需要使用基於密碼的身份驗證來連接到節點，則需要將選項 `--ask-pass` 附加到您的 Ansible 命令。

這將使 Ansible 提示您輸入您嘗試連接的遠程服務器上的用戶密碼：

```bash
$ ansible all -m ping --ask-pass
```

此選項對 ansible-playbook 也有效：

```bash
$ ansible-playbook myplaybook.yml --ask-pass
```

## 提供 `sudo` 密碼

如果遠程用戶需要提供密碼才能運行 `sudo` 命令，您可以在 Ansible 命令中包含選項 `--ask-become-pass `。這將提示您提供遠程用戶 sudo 密碼：

```bash
$ ansible all -m ping --ask-become-pass
```

此選項對 ansible-playbook 也有效：

```bash
$ ansible-playbook myplaybook.yml --ask-become-pass
```

## 使用自定義主機 inventory 文件

默認清單文件通常位於 `/etc/ansible/hosts`，但您也可以在運行 Ansible 命令和 playbook 時使用 `-i` 選項指向自定義清單文件。 Ansible 還支持用於構建動態清單文件的清單腳本，以便在您的清單波動時，經常創建和銷毀服務器。自定義清單文件對於設置可以包含在版本控制系統（例如 Git）中的每個項目的清單很有用：

```bash
$ ansible all -m ping -i my_custom_inventory
```

此選項對 ansible-playbook 也有效：

```bash
ansible-playbook myplaybook.yml -i my_custom_inventory
```

## 運行臨時命令 (ad-hoc)

要在節點上執行命令，請使用 `-a` 選項，後跟要運行的命令，用引號括起來。

這將在清單中的所有節點上執行 `uname -a`：

```bash
$ ansible all -a "uname -a"
```

也可以使用選項 `-m` 運行 Ansible 模塊。以下命令將從您的清單中將軟件包 `vim` 安裝到 server1 上：

```bash
$ ansible server1 -m apt -a "name=vim"
```

在對節點進行更改之前，您可以進行試運行以預測服務器將如何受到您的命令的影響。這可以通過包含 `--check` 選項來完成：

```
$ ansible server1 -m apt -a "name=vim" --check
```

## 運行劇本 Playbook

要運行劇本並執行其中定義的所有任務，請使用 ansible-playbook 命令：

```bash
$ ansible-playbook myplaybook.yml
```

要覆蓋 playbook 中的默認主機選項並將執行限制為特定組或主機，請在命令中包含選項 `-l`：

```bash
$ ansible-playbook -l server1 myplaybook.yml
```

## 獲取有關 `Play` 的信息

```bash
$ ansible-playbook myplaybook.yml --list-tasks
```

同樣，可以列出所有受 `Play` 影響的主機，而無需在遠程服務器上運行任何任務：

```bash
$ ansible-playbook myplaybook.yml --list-hosts
```

您可以使用標籤來限製`Play`的執行。要列出播放中可用的所有標籤，請使用選項 `--list-tags`：

```bash
$ ansible-playbook myplaybook.yml --list-tags
```

## 控制劇本 Playbook 執行

您可以使用選項 `--start-at-task` 為您的劇本定義一個新的入口點。 Ansible 然後將跳過指定任務之前的任何內容，從該點開始執行剩餘的播放。此選項需要一個有效的任務名稱作為參數：

```bash
$ ansible-playbook myplaybook.yml --start-at-task="Set Up Nginx"
```

要僅執行與特定標籤關聯的任務，您可以使用選項 --tags。例如，如果您只想執行標記為 nginx 或 mysql 的任務，您可以使用：

```bash
$ ansible-playbook myplaybook.yml --tags=mysql,nginx
```

如果要跳過特定標籤下的所有任務，請使用 --skip-tags。以下命令將執行 myplaybook.yml，跳過所有標記為 mysql 的任務：

```bash
$ ansible-playbook myplaybook.yml --skip-tags=mysql
```

## 使用 Ansible Vault 存儲敏感數據

如果您的 Ansible playbook 處理密碼、API 密鑰和憑據等敏感數據，則使用加密機制確保這些數據的安全非常重要。 Ansible 提供 ansible-vault 來加密文件和變量。

儘管可以加密任何 Ansible 數據文件以及二進製文件，但更常見的是使用 ansible-vault 來加密包含敏感數據的變量文件。使用此工具加密文件後，您只能通過提供首次加密文件時定義的相關密碼來執行、編輯或查看其內容。

### 創建新的加密文件

您可以使用以下命令創建一個新的加密 Ansible 文件：

```bash
$ ansible-vault create credentials.yml
```

該命令將執行以下操作：

- 首先，它會提示您輸入新密碼。每當您訪問文件內容時，您都需要提供此密碼，無論是用於編輯、查看還是僅使用這些值運行 playbook 或命令。
- 接下來，它將打開您的默認命令行編輯器，以便您可以使用所需內容填充文件。
- 最後，當你完成編輯後，ansible-vault 會將文件保存為加密數據。

### 加密現有的 Ansible File

要加密現有的 Ansible 文件，可以使用以下語法：

```bash
$ ansible-vault encrypt credentials.yml
```

這將提示您輸入密碼，每當您訪問文件 `credentials.yml` 時都需要輸入該密碼。

### 查看加密文件的內容

如果您想查看之前使用 ansible-vault 加密的文件的內容並且不需要更改其內容，可以使用：

```bash
$ ansible-vault view credentials.yml
```

這將提示您提供首次使用 ansible-vault 加密文件時選擇的密碼。

### 編輯加密文件

要編輯之前使用 Ansible Vault 加密的文件的內容，請運行：

```bash
$ ansible-vault edit credentials.yml
```

這將提示您提供首次使用 `ansible-vault` 加密文件 `credentials.yml` 時選擇的密碼。密碼驗證後，您的默認命令行編輯器將打開文件的未加密內容，允許您進行更改。完成後，您可以照常保存和關閉文件，更新的內容將保存為加密數據。

### 解密加密文件

如果您希望將以前使用 `ansible-vault` 加密的文件永久還原為其未加密版本，您可以使用以下語法執行此操作：

```bash
$ ansible-vault decrypt credentials.yml
```

這將提示您提供首次使用 `ansible-vault` 加密文件 `credentials.yml` 時使用的相同密碼。密碼驗證後，文件內容將作為未加密數據保存到磁盤。

### 使用多個 Vault 密碼

Ansible 支持按不同保管庫 ID 分組的多個保管庫密碼。如果您想為不同的環境（例如開發、測試和生產環境）使用專用的保管庫密碼，這將非常有用。

要使用自定義保管庫 ID 創建新的加密文件，請包括 --vault-id 選項以及標籤和 ansible-vault 可以找到該保管庫密碼的位置。標籤可以是任何標識符，並且位置可以是提示符，這意味著該命令應提示您輸入密碼，或密碼文件的有效路徑。

```bash
$ ansible-vault create --vault-id dev@prompt credentials_dev.yml
```

這將創建一個名為 dev 的新保管庫 ID，它使用提示符作為密碼源。通過將此方法與組變量文件相結合，您將能夠為每個應用程序環境擁有單獨的 ansible 保險庫：

```bash
$ ansible-vault create --vault-id prod@prompt credentials_prod.yml
```

我們使用 dev 和 prod 作為文件庫 ID 來演示如何為每個環境創建單獨的文件庫，但您可以創建任意數量的文件庫，並且可以使用您選擇的任何標識符作為文件庫 ID。

現在要查看、編輯或解密這些文件，您需要提供相同的保管庫 ID 和密碼源以及 ansible-vault 命令：

```bash
$ ansible-vault edit credentials_dev.yml --vault-id dev@prompt
```

### 使用密碼文件

如果您需要使用第三方工具通過 Ansible 自動配置服務器的過程，您將需要一種方法來提供保管庫密碼而不被提示輸入。您可以通過使用帶有 ansible-vault 的密碼文件來做到這一點。

密碼文件可以是純文本文件或可執行腳本。如果文件是可執行腳本，則此腳本生成的輸出將用作保管庫密碼。否則，文件的原始內容將用作保管庫密碼。

要在 ansible-vault 中使用密碼文件，您需要在運行任何 vault 命令時提供密碼文件的路徑：

```bash
$ ansible-vault create --vault-id dev@path/to/passfile credentials_dev.yml
```

只要輸入的密碼相同，Ansible 不會區分使用提示加密的內容或密碼文件作為密碼源的內容。實際上，這意味著可以使用提示加密文件，然後使用密碼文件存儲與提示方法使用的相同密碼。反之亦然：您可以使用密碼文件加密內容，然後使用提示方法，在 Ansible 提示時提供相同的密碼。

為了提高靈活性和安全性，您可以使用 Python 腳本從其他來源獲取密碼，而不是將您的保管庫密碼存儲在純文本文件中。官方 Ansible 存儲庫包含一些 Vault 腳本示例，您可以在創建適合項目特定需求的自定義腳本時參考這些示例。

## 運行通過 Ansible Vault 加密數據的劇本

每當您運行使用先前通過 ansible-vault 加密的數據的 playbook 時，您都需要向 playbook 命令提供 vault 密碼。

如果您在加密此 playbook 中使用的數據時使用了默認選項和提示密碼源，則可以使用選項 --ask-vault-pass 使 Ansible 提示您輸入密碼：

```bash
$ ansible-playbook myplaybook.yml --ask-vault-pass
```

如果您使用密碼文件而不是提示輸入密碼，則應使用選項 --vault-password-file 代替：

```bash
$ ansible-playbook myplaybook.yml --vault-password-file my_vault_password.py
```

如果您使用在保管庫 ID 下加密的數據，則需要提供與首次加密數據時相同的保管庫 ID 和密碼源：

```bash
$ ansible-playbook myplaybook.yml --vault-id dev@prompt
```

如果使用帶有您的保管庫 ID 的密碼文件，您應該提供標籤，後跟密碼文件的完整路徑作為密碼源：

```bash
$ ansible-playbook myplaybook.yml --vault-id dev@vault_password.py
```

如果您的遊戲使用多個保險庫，您應該為每個保險庫提供一個 --vault-id 參數，沒有特定的順序：

```bash
$ ansible-playbook myplaybook.yml --vault-id dev@vault_password.py --vault-id test@prompt --vault-id ci@prompt
```

## 調試

如果您在執行 Ansible 命令和 playbook 時遇到錯誤，最好增加輸出詳細程度以獲取有關問題的更多信息。您可以通過在命令中包含 `-v` 選項來做到這一點：

```bash
$ ansible-playbook myplaybook.yml -v
```

如果您需要更多詳細信息，可以使用 -vvv，這將增加輸出的詳細程度。如果您無法通過 Ansible 連接到遠程節點，請使用 -vvvv 獲取連接調試信息：

```bash
$ ansible-playbook myplaybook.yml -vvvv
```

## 結論

本指南涵蓋了您在配置服務器時可能使用的一些最常見的 Ansible 命令，例如如何在節點上執行遠程命令以及如何使用各種自定義設置運行 playbook。

您可能會發現其他命令變體和標誌對您的 Ansible 工作流程很有用。要獲得所有可用選項的概覽，您可以使用幫助命令：

```bash
$ ansible  --help
```

如果您想更全面地了解 Ansible 及其所有可用命令和功能，請參閱 Ansible 官方文檔。

如果您想查看 Ansible 的另一個實際示例，請查看我們關於如何使用 Ansible 在 Ubuntu 20.04 上安裝和設置 Docker 的指南。
# 如何在 Ubuntu 20.04 上安裝和配置 Ansible

Ansible playbook 使用 YAML 格式來定義一個或多個劇本。劇本是一組有序的任務，這些任務以自動化流程的方式排列，例如設置 Web 服務器或將應用程序部署到生產環境。

在 playbook 文件中，plays 被定義為 YAML 列表。典型的 playbook 是從確定哪些主機是該特定設置的目標開始。這是通過 `hosts` 指令完成的。

將 `hosts` 指令設置為 `all` 是一種常見的選擇，因為你可以通過運行帶有 `-l` 參數的 `ansible-playbook` 命令來在執行時指定自動設定的目標。這允許你在不同的服務器或組上運行相同的 playbook，而無需每次都更改 playbook 文件。

## 前置準備

首先在你的主文件夾上創建一個新目錄，你可以在其中保存練習手冊。首先，確保你位於 Ubuntu 用戶的主目錄中。從那裡，創建一個名為 `ansible-practice` 的目錄，然後使用 cd 命令導航到該目錄：

```bash
$ mkdir ansible-practice
$ cd ansible-practice
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
  IdentityFile /home/dxlab/opt/ws_ansible/ansible-practice/.vagrant/machines/default/virtualbox/private_key
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

## Ansible Playbook 劇本

接下來，創建一個新的 playbook 文件：

```
$ nano playbook-01.yml
```

以下劇本定義了針對給定清單中的所有主機的劇本。它包含一個打印調試消息的任務。將以下內容添加到你的 `playbook-01.yml` 文件中：

```yaml title="playbook-01.yml"
---
- hosts: all
  tasks:
    - name: Print message
      debug:
        msg: Hello Ansible World
```

完成後保存並關閉文件。如果你使用的是 nano，你可以通過鍵入 CTRL+X、Y 和 ENTER 來確認。

要在你在清單文件中設置的服務器上嘗試此 playbook，請使用你在本系列介紹中運行連接測試時使用的相同連接參數運行 ansible-playbook。在這裡，我們將使用名為 inventory 的庫存文件和 sammy 用戶連接到遠程服務器，但請務必更改這些詳細信息以與你自己的庫存文件和管理用戶保持一致：

```bash
$ ansible-playbook -i inventory playbook-01.yml
```

結果:

```
PLAY [all] ***************************************************************************

TASK [Gathering Facts] ***************************************************************
ok: [demo-vm]

TASK [Print message] *****************************************************************
ok: [demo-vm] => {
    "msg": "Hello Ansible World"
}

PLAY RECAP ***************************************************************************
demo-vm                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## 結論

你可能已經注意到，即使你在 playbook 中只定義了一個任務，play 輸出中也列出了兩個任務。在每次播放開始時，Ansible 默認執行一個額外的任務，該任務收集有關遠程節點的信息（稱為 `facts`）。因為可以在劇本上使用 `fact` 來更好地自定義任務的行為，所以事實收集任務必須在執行任何其他任務之前發生。

我們將在本系列的後面部分了解有關 Ansible `fact` 的更多信息。


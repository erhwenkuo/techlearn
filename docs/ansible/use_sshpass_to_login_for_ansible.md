# 在 Ansible 使用基於 SSH 密碼的登入

原文: https://linuxhint.com/how_to_use_sshpass_to_login_for_ansible/

在本文中，我將向您展示如何使用基於 SSH 密碼的登入和 `sshpass` 來運行 Ansible playbook。

## 先決條件

如果您想嘗試本文中討論的示例，

1. 你必須在您的機台上安裝 Ansible。
2. 你必須至少有一個可以從 Ansible 連接的 Ubuntu/Debian 主機。
3. 你還需要在你的計算機上安裝 `sshpass`

在本文中向您展示如何在 Ubuntu/Debian 和 CentOS/RHEL 上安裝 `sshpass`。如果您的系統上尚未安裝這些程序，請不要擔心。

## 在 Ubuntu/Debian 上安裝 sshpass

程序 **sshpass** 在 Ubuntu/Debian 的官方軟件包存儲庫中可用。您可以輕鬆地在您的計算機上安裝此程序。

=== "在 Ubuntu/Debian 上安裝 sshpass"

    首先，通過以下命令更新 APT 包存儲庫緩存：

    ```bash
    $ sudo apt update
    ```

    現在，通過以下命令安裝 sshpass：

    ```bash
    $ sudo apt install sshpass -y
    ```

=== "在 CentOS 8/RHEL 8 上安裝 sshpass"

    `sshpass` 在 CentOS 8/RHEL 8 的 EPEL 存儲庫中可用。您必須啟用 EPEL 存儲庫才能安裝 sshpass。

    首先，通過以下命令更新 DNF 包存儲庫緩存：

    ```bash
    $ sudo dnf makecache
    ```

    接下來，通過以下命令安裝 EPEL 存儲庫包：

    ```bash
    $ sudo dnf install epel-release -y
    ```

    現在應該安裝 EPEL 存儲庫包並且應該啟用 EPEL 存儲庫。

    再次更新 DNF 包倉庫緩存，如下：

    ```bash
    $ sudo dnf makecache
    ```

    通過以下命令安裝 sshpass：

    ```bash
    $ sudo dnf install sshpass -y
    ```

到這個步驟 **sshpass** 應該安裝好了，接下來讓我們開始進行有關 ansilbe 專案的配置。


## 設置 Ansible 專案目錄

創建專案目錄 `sshpass/` 和所有必需的子目錄（在您當前的工作目錄中），請運行以下命令：

```bash
$ mkdir -pv ~/sshpass/{files,playbooks}
```

導航到項目目錄，如下：

```bash
$ cd ~/sshpass/
```

創建主機清單文件，如下所示：

```bash
$ nano hosts
```

比如：

``` title="hosts"
vm01    ansible_host=10.16.10.11    ansible_user=my_vm_user
```

- `vm01` - 主機 hostname 或是主機別名 (alias)，如果沒有設定 `ansible_host` 的話，那麼就得設定的是 DNS 可解析的主機 FQDN。
- `ansible_host` - ansible 要連線的到的主機 IP address
- `ansible_user` - ansible 要使用 ssh 連線進去時所使用的用戶名
- `ansible_ssh_pass` (optional)- ansible  要使用 ssh 連線進去時所使用的用戶密碼
- `ansible_become` (optional)- ansible 需要切換成 `root` 角色的 flag ("true/yes")
- `ansible_become_method` (optional) - 可使用的切換方法 sudo/su/pbrun/pfexec/doas/dzdo/ksu
- `ansible_become_user` (optional) - ansible 需要切換角色用戶, 預設是 `root`
- `ansible_become_password` (optional) - ansible 需要切換角色用戶的用戶密碼

在清單文件中添加您的主機 IP 或 FQDN 名稱。

完成此步驟後，按 <Ctrl> + X，然後按 Y 和 <Enter> 保存文件。


在項目目錄下創建一個Ansible配置文件，如下：

```bash
$ nano ansible.cfg
```
Rancher Manage Cluster
現在，在 `ansible.cfg` 文件中鍵入以下內容。

``` title="ansible.cfg"
[defaults]
inventory = hosts
host_key_checking = False
```

完成此步驟後，按 <Ctrl> + X，然後按 Y 和 <Enter> 保存文件。

## 在 Ansible 中測試基於密碼的 SSH 登錄

接下來，嘗試 ping 清單文件中的主機，如下所示：

```bash
$ ansible all -u {user_name} -m ping
```

!!! info
    注意： 這裡，`-u` 選項用於告訴 ansible 以哪個用戶身份登錄。在這種情況下，它將是用戶 `shovon`。從現在開始，在整個演示過程中，將此用戶名替換為您的用戶名。

如您所見，我無法登錄主機並運行任何命令。

```
vm01 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: dxlab@192.168.56.8: Permission denied (publickey,password).",
    "unreachable": true
}
```

要強制 Ansible 詢問用戶密碼，請運行帶有 `--ask-pass` 參數的 ansible 命令，如下所示：

```bash
$ ansible all -u {user_name} --ask-pass -m ping
```

Ansible 會要求輸入用戶的 SSH 密碼。現在，輸入您的 SSH 密碼（用戶登錄密碼）並按 <Enter>。

現在我們可以 ping 通主機，如下：

```
vm01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}

```

## Playbook 基於 Ansible 密碼的 SSH 登錄

運行 Ansible playbook 時，可以使用基於密碼的 SSH 登錄。讓我們看一個例子。

首先在 `playbooks/` 目錄下新建一個新的 playbook `askpass1.yaml`，比輸入內容如下：

```yaml
- hosts: all
  user: shovon
  tasks:
    - name: Ping all hosts
      ping:
    - name: Print a message
      debug:
        msg: 'All set'
```

完成此步驟後，按 <Ctrl> + X，然後按 Y 和 <Enter> 保存文件。

運行 `askpass1.yaml` playbook，如下：

```bash
$ ansible-playbook playbooks/askpass1.yaml
```

如您所見，我無法連接到主機。您可以看到這是因為我沒有運行帶有 `--ask-pass` 選項的 ansible-playbook 命令。


結果如下:

```
$ ansible-playbook playbooks/askpass1.yaml 

PLAY [all] *************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************
fatal: [vm01]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: dxlab@192.168.56.8: Permission denied (publickey,password).", "unreachable": true}

PLAY RECAP *************************************************************************************************************************************
vm01                       : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

使用 `--ask-pass` 選項運行 askpass1.yaml playbook，如下所示：

```bash
$ ansible-playbook --ask-pass playbooks/askpass1.yaml 
SSH password: 

PLAY [all] *************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************
ok: [vm01]

TASK [Ping all hosts] **************************************************************************************************************************
ok: [vm01]

TASK [Print a message] *************************************************************************************************************************
ok: [vm01] => {
    "msg": "All set"
}

PLAY RECAP *************************************************************************************************************************************
vm01                       : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

如您所見，Ansible 要求輸入 SSH 密碼。輸入您的 SSH 密碼並按 <Enter>。playbook `askpass1.yaml` 現在應該可以成功運行。

## 用於 Playbook 的 Ansible sudo 密碼登錄

`--ask-pass` 選項將僅詢問 SSH 登錄密碼。如果您還想輸入 `sudo` 密碼怎麼辦？您將在接下來的步驟中看到如何執行此操作。

首先在 `playbooks/` 目錄下新建 playbook `askpass2.yaml`，如下：

```yaml title="askpass2.yaml" hl_lines="3"
- hosts: all
  user: shovon
  become: True
  tasks:
    - name: Install apache2 Package
      apt:
       name: apache2
       state: latest
    - name: Make sure apache2 service is running
      service:
       name: apache2
       state: started
       enabled: True
    - name: Copy index.html file to server
      copy:
       src: ../files/index.html
       dest: /var/www/html/index.html
       mode: 0644
       owner: www-data
       group: www-data
```

在這裡，我使用了命令 `become: True` 來告訴 Ansible 以 `sudo` 權限運行這個 playbook。完成此步驟後，按 <Ctrl> + X，然後按 Y 和 <Enter> 保存 `askpass2.yaml` 文件。

在 `files/` 目錄下創建一個 `index.html` 文件，如下：

```bash
$ nano files/index.html
```

在 `index.html` 文件中鍵入以下 HTML 代碼：

```html title="index.html"
<!DOCTYPE html>
<html>
  <head>
    <title>Homepage</title>
  </head>
  <body>
    <h1>Hello World</h1>
    <p>It works</p>
  </body>
</html>
```

完成此步驟後，按 <Ctrl> + X 然後按 Y 和 <Enter> 保存文件。

您可以使用 `--ask-pass` 選項運行 `askpass2.yaml` playbook，如下所示：

```bash
$ ansible-playbook --ask-pass playbooks/askpass2.yaml 

SSH password: 

PLAY [all] *************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************
fatal: [vm01]: FAILED! => {"msg": "Missing sudo password"}

PLAY RECAP *************************************************************************************************************************************
vm01                       : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   


```

但是即使您提供 SSH 密碼，playbook 仍然可能無法運行。這樣做的原因是因為您必須告訴 Ansible 提示輸入 sudo 密碼以及 SSH 密碼。

您可以在運行 playbook 時使用 `--ask-become-pass` 選項告訴 Ansible 詢問 sudo 密碼，如下所示：

```bash hl_lines="3 4"
$ ansible-playbook --ask-pass --ask-become-pass playbooks/askpass2.yaml

SSH password: 
BECOME password[defaults to SSH password]: 

PLAY [all] *************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************
fatal: [vm01]: FAILED! => {"ansible_facts": {}, "changed": false, "failed_modules": {"ansible.legacy.setup": {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"}, "failed": true, "module_stderr": "Shared connection to 192.168.56.8 closed.\r\n", "module_stdout": "\r\ndxlab is not in the sudoers file.  This incident will be reported.\r\n", "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error", "rc": 1}}, "msg": "The following modules failed to execute: ansible.legacy.setup\n"}

PLAY RECAP *************************************************************************************************************************************
vm01                       : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
```

Ansible 將提示您輸入 SSH 密碼。接下來，Ansible 將提示您輸入 sudo 密碼。如果您的 sudo 密碼與 SSH 密碼相同（很可能），請將其留空並按 <Enter>。


```bash
...
TASK [Gathering Facts] *************************************************************************************************************************
fatal: [vm01]: FAILED! => {"ansible_facts": {}, "changed": false, "failed_modules": {"ansible.legacy.setup": {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python3"}, "failed": true, "module_stderr": "Shared connection to 192.168.56.8 closed.\r\n", "module_stdout": "\r\ndxlab is not in the sudoers file.  This incident will be reported.\r\n", "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error", "rc": 1}}, "msg": "The following modules failed to execute: ansible.legacy.setup\n"}

PLAY RECAP *************************************************************************************************************************************
vm01                       : ok=0    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
...
```

上述的錯誤訊息 `xxx is not in the sudoers file` 代表者 ssh 登入的帳戶並沒有被設置進 suoers 檔案導致了問題。

Linux 把用戶和群組的 sudo 權限在 `/etc/sudoers` 文件中定義。將用戶添加到此文件允許您授予對命令的自定義訪問權限並配置自定義安全策略。

登入有問題的機台并且切換并且使用 `root` 帳戶來將 將 vagrant ssh-user 加入 Sudoer 的列表：

```bash
$ sudo usermod -aG sudo shovon
```

再重新執行一次 playbook:

```bash
$ ansible-playbook --ask-pass --ask-become-pass  playbooks/askpass2.yaml 

SSH password: 
BECOME password[defaults to SSH password]: 

PLAY [all] ***********************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************
ok: [vm01]

TASK [Install apache2 Package] ***************************************************************************************************************
changed: [vm01]

TASK [Make sure apache2 service is running] **************************************************************************************************
ok: [vm01]

TASK [Copy index.html file to server] ********************************************************************************************************
changed: [vm01]

PLAY RECAP ***********************************************************************************************************************************
vm01                       : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

如您所見，劇本運行成功。

## 配置自動基於密碼的 SSH 登錄和 sudo 密碼登錄

您可能希望使用基於密碼的 SSH 和 sudo 登錄，但不想在每次運行 playbook 時都輸入 SSH 密碼和 sudo 密碼。如果是這種情況，那麼本節適合您。

要在不提示輸入密碼的情況下使用基於密碼的 SSH 登錄和 sudo 登錄，您所要做的就是在清單文件中添加 `ansible_ssh_pass` 和 `ansible_become_pass` 變量。

首先，打開 `hosts` 的庫存文件，如下：

```bash
$ nano hosts
```

如果清單文件中有多個主機並且每個主機都有不同的密碼，則添加 `ansible_ssh_pass` 和 `ansible_become_pass` 變量作為主機變量（在每個主機之後），如下所示。

請務必將 secret 替換為您的 SSH 和 sudo 密碼。

```
vm01    ansible_host=192.168.56.9    ansible_port=22    ansible_user=dxlab ansible_ssh_pass=****** ansible_become_pass=*******
```

如果所有或部分主機的密碼相同，則可以將 `ansible_ssh_pass` 和 `ansible_become_pass` 變量添加為主機群組變量，如下例所示。

在這裡，我只有一個主機，所以我為 all 組（庫存文件中的所有主機）添加了 `ansible_ssh_pass` 和 `ansible_become_pass` 變量。但是，您也可以為其他特定組添加這些變量。

```
vm01    ansible_host=192.168.56.9    ansible_port=22    

[all:vars]
ansible_user=dxlab
ansible_ssh_pass=*****
ansible_become_pass=*****
```

在主機清單文件中添加完 `ansible_ssh_pass` 和 `ansible_become_pass` 變量後，按 <Ctrl> + X，然後按 Y 和 <Enter> 保存主機清單文件。

您現在可以運行 `askpass2.yaml` 劇本，如下所示：

```bash
$ ansible-playbook playbooks/askpass2.yaml 

PLAY [all] *****************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************
ok: [vm01]

TASK [Install apache2 Package] *********************************************************************************************************************************
ok: [vm01]

TASK [Make sure apache2 service is running] ********************************************************************************************************************
ok: [vm01]

TASK [Copy index.html file to server] **************************************************************************************************************************
ok: [vm01]

PLAY RECAP *****************************************************************************************************************************************************
vm01                       : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

如您所見，playbook 運行成功，儘管它沒有要求輸入 SSH 密碼或 sudo 密碼。

因此，這就是在 Ansible 中使用 sshpass 進行基於密碼的 SSH 和 sudo 登錄的方式。
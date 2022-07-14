# 如何在 Ansible Playbooks 中安裝和管理系統包

自動安裝所需的系統包是 Ansible playbook 中的一項常見操作任務，因為典型的應用程序堆棧需要來自不同來源的軟件。

`apt` 模塊管理基於 `Debian` 的 Linux 操作系統（例如 Ubuntu）上的系統包，這是我們在本指南中在遠程節點上使用的發行版。以下劇本將更新 apt 緩存，然後確保在遠程節點上安裝了 `Vim`。

在你的 ansible-practice 目錄中創建一個名為 playbook-09.yml 的新文件：

```yaml title="playbook-09.yml"
---
- hosts: all
  become: yes
  tasks:
    - name: Update apt cache and make sure Vim is installed
      apt:
        name: vim
        update_cache: yes
```

請注意，我們在 playbook 的開頭包含了 `become` 指令。這是必需的，因為安裝軟件包需要管理系統權限。

刪除包的方式與此類似，唯一的變化是您必須將包狀態定義為不存在。 `state` 指令的默認值是 `present`，這將確保軟件包安裝在系統上，無論版本如何。如果不存在，將安裝該軟件包。為確保您擁有最新版本的軟件包，您可以使用 `latest` 代替。如果該軟件包不在最新版本上，這將導致 apt 更新請求的軟件包。

請記住在運行此 playbook 時提供 -K 選項，因為它需要 sudo 權限：

```bash
ansible-playbook -i inventory playbook-09.yml -K
```

結果:

```
BECOME password: 

PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [demo-vm]

TASK [Update apt cache and make sure Vim is installed] *********************************************************************************************************************
ok: [demo-vm]

PLAY RECAP *****************************************************************************************************************************************************************
demo-vm                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

安裝多個包時，您可以使用 `loop` 並提供一個陣列，其中包含您要安裝的包的名稱。以下 playbook 將確保安裝包 vim、unzip 和 curl 並處於最新版本。

在 Ansible 控制節點的 ansible-practice 目錄中創建一個名為 playbook-10.yml 的新文件：

```yaml title="playbook-10.yml"
---
- hosts: all
  become: yes
  tasks:
    - name: Update apt cache and make sure Vim, Curl and Unzip are installed
      apt:
        name: "{{ item }}"
        update_cache: yes
      loop:
        - vim
        - curl
        - unzip
```

然後，使用與前面示例相同的連接參數運行 ansible-playbook，並且不要忘記包含 -K 選項，因為此 playbook 需要管理權限：

```bash
$ ansible-playbook -i inventory playbook-09.yml -u vagrant -K
```

結果:

```
BECOME password: 

PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [demo-vm]

TASK [Update apt cache and make sure Vim, Curl and Unzip are installed] ****************************************************************************************************
ok: [demo-vm] => (item=vim)
ok: [demo-vm] => (item=curl)
changed: [demo-vm] => (item=unzip)

PLAY RECAP *****************************************************************************************************************************************************************
demo-vm                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

關於如何管理系統包的更多細節，包括如何刪除包以及如何使用高級apt選項，可以參考[官方文檔](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)。


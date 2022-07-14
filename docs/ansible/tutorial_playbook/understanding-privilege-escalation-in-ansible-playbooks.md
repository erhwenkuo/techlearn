# 了解 Ansible Playbook 中的權限提升

就像您在終端上執行的常規命令一樣，某些任務需要特殊權限才能讓 Ansible 在您的遠程節點上成功執行它們。

了解權限提昇在 Ansible 中的工作原理非常重要，這樣您就可以使用適當的權限執行任務。默認情況下，任務將以連接用戶身份運行 - 這可能是 `root` 或任何具有 SSH 訪問清單文件中遠程節點的常規用戶。

要運行具有擴展權限的命令，例如需要 `sudo` 的命令，您需要在 playbook 中包含設定 `become` 指令。這可以作為對該 playbook 中所有任務有效的全局設置來完成，也可以作為每個任務應用的單獨指令來完成。根據您的 `sudo` 用戶在遠程節點中的設置方式，您可能還需要提供用戶的 `sudo` 密碼。以下示例更新 apt 緩存，這是一項需要 `root` 權限的任務。

在你的 ansible-practice 目錄中創建一個名為 playbook-07.yml 的新文件：

```yaml title="playbook-07.yml"
---
- hosts: all
  become: yes
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
```

要運行這個 playbook，您需要在 ansible-playbook 命令中包含 -K 選項。這將使 Ansible 提示您輸入指定用戶的 `sudo` 密碼。

```bash
$ ansible-playbook -i inventory playbook-07.yml -K
```

結果:

```
BECOME password: 

PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [demo-vm]

TASK [Update apt cache] ****************************************************************************************************************************************************
changed: [demo-vm]

PLAY RECAP *****************************************************************************************************************************************************************
demo-vm                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

您還可以在執行任務或播放時更改要切換到的用戶。為此，請將 `become_user` 指令設置為您要切換到的遠程用戶的名稱。當您在 playbook 中有多個依賴 `sudo` 的任務時，這很有用，但也有一些任務應該作為您的普通用戶運行。

下面的例子定義了本次播放中的所有任務默認使用 `sudo` 執行。這是在播放級別設置的，就在主機定義之後。第一個任務使用 `root` 權限在 `/tmp` 上創建一個文件，因為這是默認的 `become_user` 值。然而，最後一個任務定義了它自己的 `become_user`。

在你的 ansible-practice 目錄中創建一個名為 playbook-08.yml 的新文件：

```yaml title="playbook-08.yml"
---
- hosts: all
  become: yes
  vars:
    user: "{{ ansible_env.USER }}"
  tasks:
    - name: Create root file
      file:
        path: /tmp/my_file_root
        state: touch

    - name: Create user file
      become_user: "{{ user }}"
      file:
        path: /tmp/my_file_{{ user }}
        state: touch
```

`ansible_env.USER` 的事實(fact)包含連接用戶的用戶名，可以在使用 `-u` 選項運行 ansible-playbook 命令時在執行時定義。在本指南中，我們以 `vagrant` 的身份進行連接：

```bash
$ ansible-playbook -i inventory playbook-08.yml -u vagrant -K
```

結果:

```
BECOME password: 

PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [demo-vm]

TASK [Create root file] ****************************************************************************************************************************************************
changed: [demo-vm]

TASK [Create user file] ****************************************************************************************************************************************************
changed: [demo-vm]

PLAY RECAP *****************************************************************************************************************************************************************
demo-vm                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

讓我們登入到這台VM去驗證:

```bash
$ vagrant ssh

$ ls -la /tmp/my_file*

-rw-r--r-- 1 root root 0 Jun  9 23:37 /tmp/my_file_root
```

有關 Ansible 中權限提升的更多詳細信息，請參閱[官方文檔](https://docs.ansible.com/ansible/latest/user_guide/become.html)。
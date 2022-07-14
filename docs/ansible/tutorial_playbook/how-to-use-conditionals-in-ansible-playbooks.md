# 如何在 ansible-playbooks 中使用條件

在 Ansible，您可以定義在執行任務之前要滿足的條件。如果條件不滿足，則跳過該任務。這是通過 `when` 關鍵字完成的，它接受通常基於`變量`或`事實`的表達式。

以下示例定義了兩個變量：`create_user_file` 和 `user`。當 `create_user_file` 被評估為 `true` 時，將在 `user`變量定義的用戶主目錄中創建一個新文件：

在你的 ansible-practice 目錄中創建一個名為 playbook-04.yml 的新文件：

```yaml title="playbook-04.yml"
---
- hosts: all
  vars:
    - create_user_file: yes
    - user: vagrant  
  tasks:
    - name: create file for user
      file:
        path: /home/{{ user }}/myfile
        state: touch
      when: create_user_file
```

要從您的清單文件在服務器上執行此 playbook，請使用您之前在運行本系列中的其他 playbook 時使用的相同連接參數運行 ansible-playbook。同樣，我們將使用一個名為 inventory 的清單文件連接到遠程服務器：

```bash
$ ansible-playbook -i inventory playbook-04.yml
```

當滿足條件時 (create_user_file＝true)，您將在播放輸出中看到更改的狀態：

```hl_lines="7"
PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [demo-vm]

TASK [create file for user] ************************************************************************************************************************************************
changed: [demo-vm]

PLAY RECAP *****************************************************************************************************************************************************************
demo-vm                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

如果將 `create_user_file` 的值更改為 `no`，則條件將被評估為 `false`。在這種情況下，您會在播放輸出中看到跳過狀態，表示任務未執行：

```yaml title="playbook-04.yml" hl_lines="4"
---
- hosts: all
  vars:
    - create_user_file: no
    - user: vagrant  
  tasks:
    - name: create file for user
      file:
        path: /home/{{ user }}/myfile
        state: touch
      when: create_user_file
```

再執行一次 playbook:

```bash
$ ansible-playbook -i inventory playbook-04.yml
```

結果:

```hl_lines="7"
PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [demo-vm]

TASK [create file for user] ************************************************************************************************************************************************
skipping: [demo-vm]

PLAY RECAP *****************************************************************************************************************************************************************
demo-vm                    : ok=1    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
```

在 Ansible playbook 的上下文中，條件語句的一個常見用途是將它們與 `register` 結合，這是一個創建新變量並將其分配給從命令獲得的輸出的關鍵字。這樣，您可以使用任何外部命令來評估任務的執行。

需要注意的一件重要事情是，默認情況下，如果您用於評估條件的命令失敗，Ansible 將中斷播放。出於這個原因，您需要在所述任務中包含一個設置為 yes 的 `ignore_errors` 指令，這將使 Ansible 繼續執行下一個任務並繼續播放。

以下示例只會在用戶主目錄中創建一個新文件，以防該文件尚不存在，我們將使用 `ls` 命令對其進行測試。但是，如果文件存在，我們將使用 `debug` 模塊顯示一條消息。

在你的 ansible-practice 目錄中創建一個名為 playbook-05.yml 的新文件：

```yaml title="playbook-05.yml"
---
- hosts: all
  vars:
    - user: vagrant
  tasks:
    - name: Check if file already exists
      command: ls /home/{{ user }}/myfile
      register: file_exists
      ignore_errors: yes

    - name: create file for user
      file:
        path: /home/{{ user }}/myfile
        state: touch
      when: file_exists is failed

    - name: show message if file exists
      debug:
        msg: The user file already exists.
      when: file_exists is succeeded
```

然後，使用與前面示例相同的連接參數運行 ansible-playbook。在這裡，我們使用了一個名為 inventory 的庫存文件，但是您應該相應地更改這些值：

```bash
$ ansible-playbook -i inventory playbook-05.yml
```

第一次運行此 playbook 時，命令 `Check if file already exists` 將失敗，因為該文件不存在於該路徑中。然後將執行 `create file for user` 創建文件的任務，而最後一個任務 `show message if file exists` 將被跳過：

```
PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [demo-vm]

TASK [Check if file already exists] ****************************************************************************************************************************************
fatal: [demo-vm]: FAILED! => {"changed": true, "cmd": ["ls", "/home/vagrant/myfile"], "delta": "0:00:00.002623", "end": "2022-06-08 23:00:34.377372", "msg": "non-zero return code", "rc": 2, "start": "2022-06-08 23:00:34.374749", "stderr": "ls: cannot access '/home/vagrant/myfile': No such file or directory", "stderr_lines": ["ls: cannot access '/home/vagrant/myfile': No such file or directory"], "stdout": "", "stdout_lines": []}
...ignoring

TASK [create file for user] ************************************************************************************************************************************************
changed: [demo-vm]

TASK [show message if file exists] *****************************************************************************************************************************************
skipping: [demo-vm]

PLAY RECAP *****************************************************************************************************************************************************************
demo-vm                    : ok=3    changed=2    unreachable=0    failed=0    skipped=1    rescued=0    ignored=1  
```

從輸出中，您可以看到 `create file for user` 任務導致服務器發生變化，這意味著文件已創建。現在，再次運行 playbook，你會得到不同的結果：

```bash
$ ansible-playbook -i inventory playbook-05.yml
```

結果:

```
PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [demo-vm]

TASK [Check if file already exists] ****************************************************************************************************************************************
changed: [demo-vm]

TASK [create file for user] ************************************************************************************************************************************************
skipping: [demo-vm]

TASK [show message if file exists] *****************************************************************************************************************************************
ok: [demo-vm] => {
    "msg": "The user file already exists."
}

PLAY RECAP *****************************************************************************************************************************************************************
demo-vm                    : ok=3    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
```

如果您想了解更多關於在 Ansible playbook 中使用條件的信息，請參閱[官方文檔](https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html)。
# 如何在 Ansible Playbook 中使用循環

在自動化服務器設置時，有時您需要使用不同的值重複執行相同的任務。例如，您可能需要更改多個文件的權限，或創建多個用戶。為避免在 playbook 文件中多次重複該任務，最好使用 `loop`。

在編程中，`loop` 允許您重複指令，通常直到滿足某個條件。 Ansible 提供了不同的循環方法，其中 `loop` 關鍵字是長期兼容性最推薦的選項。

以下示例在 `/tmp` 目錄下要創建三個不同的文件。它在使用三個不同值實現循環的任務中使用文件模塊。

在你的 ansible-practice 目錄中創建一個名為 playbook-06.yml 的新文件：

```yaml title="playbook-06.yml"
---
- hosts: all
  tasks:
    - name: creates users files
      file:
        path: /tmp/ansible-{{ item }}
        state: touch
      loop:
        - sammy
        - erika
        - brian
```

然後，使用與前面示例相同的連接參數運行 ansible-playbook。在這裡，我們使用了一個名為 inventory 的庫存文件，但是您應該相應地更改這些值：

```bash
$ ansible-playbook -i inventory playbook-06.yml
```

你會得到這樣的輸出，顯示循環中使用的每個單獨的項目值：

```hl_lines="7-9"
PLAY [all] *****************************************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************************
ok: [demo-vm]

TASK [creates user files] **************************************************************************************************************************************************
changed: [demo-vm] => (item=sammy)
changed: [demo-vm] => (item=erika)
changed: [demo-vm] => (item=brain)

PLAY RECAP *****************************************************************************************************************************************************************
demo-vm                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

有關在編寫 Ansible playbook 時如何使用循環的更多詳細信息，請參閱[官方文檔](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html)。
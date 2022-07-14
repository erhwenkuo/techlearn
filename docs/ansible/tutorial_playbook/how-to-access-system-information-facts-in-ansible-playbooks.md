# 如何在 Ansible Playbooks 中訪問系統信息（Facts）

默認情況下，在執行 playbook 中定義的任務集之前，Ansible 將花一些時間來收集有關正在配置的系統的信息。這些信息（稱為Facts）包含諸如網絡接口和地址、遠程節點上運行的操作系統和可用內存等詳細信息。

Ansible 以 JSON 格式存儲`事實`，項目按節點分組。要檢查您正在配置的系統有哪些類型的信息可用，您可以使用 ad hoc 命令運行 `setup` 模塊：

```bash
$ ansible all -i inventory -m setup
```

結果:

```json
{
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.0.2.15"
        ],
        "ansible_all_ipv6_addresses": [
            "fe80::45:8eff:fe95:1bba"
        ],
        "ansible_apparmor": {
            "status": "enabled"
        },
        "ansible_architecture": "x86_64",
        "ansible_bios_date": "12/01/2006",
        "ansible_bios_vendor": "innotek GmbH",
        "ansible_bios_version": "VirtualBox",
        "ansible_board_asset_tag": "NA",
        "ansible_board_name": "VirtualBox",
        "ansible_board_serial": "NA",
        "ansible_board_vendor": "Oracle Corporation",
        "ansible_board_version": "1.2",
        "ansible_chassis_asset_tag": "NA",
        "ansible_chassis_serial": "NA",
        "ansible_chassis_vendor": "Oracle Corporation",
        "ansible_chassis_version": "NA",
        "ansible_cmdline": {
            "BOOT_IMAGE": "/boot/vmlinuz-5.4.0-113-generic",
            "console": "ttyS0",
            "ro": true,
            "root": "UUID=e42c9539-b31f-497a-98f5-47111033eed3"
        },
        "ansible_date_time": {
            "date": "2022-06-08",
            "day": "08",
            "epoch": "1654679603",
            "epoch_int": "1654679603",
            "hour": "09",
            "iso8601": "2022-06-08T09:13:23Z",
            "iso8601_basic": "20220608T091323507293",
            "iso8601_basic_short": "20220608T091323",
            "iso8601_micro": "2022-06-08T09:13:23.507293Z",
            "minute": "13",
            "month": "06",
            "second": "23",
            "time": "09:13:23",
            "tz": "UTC",
            "tz_dst": "UTC",
            "tz_offset": "+0000",
            "weekday": "Wednesday",
            "weekday_number": "3",
            "weeknumber": "23",
            "year": "2022"
        },
        "ansible_default_ipv4": {
            "address": "10.0.2.15",
            "alias": "enp0s3",
            "broadcast": "10.0.2.255",
            "gateway": "10.0.2.2",
            "interface": "enp0s3",
            "macaddress": "02:45:8e:95:1b:ba",
            "mtu": 1500,
            "netmask": "255.255.255.0",
            "network": "10.0.2.0",
            "type": "ether"
        },
        "ansible_default_ipv6": {}
    },
    ...
    ...
}
```

此命令將輸出包含有關您的服務器信息的廣泛 JSON。要獲取該數據的子集，您可以使用 `filter` 參數並提供模式。例如，如果您想獲取有關遠程節點中所有 IPv4 地址的信息，可以使用以下命令：

```bash
$ ansible all -i inventory -m setup -a "filter=*ipv4*"
```

結果:

```bash
demo-vm | SUCCESS => {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "10.0.2.15"
        ],
        "ansible_default_ipv4": {
            "address": "10.0.2.15",
            "alias": "enp0s3",
            "broadcast": "10.0.2.255",
            "gateway": "10.0.2.2",
            "interface": "enp0s3",
            "macaddress": "02:45:8e:95:1b:ba",
            "mtu": 1500,
            "netmask": "255.255.255.0",
            "network": "10.0.2.0",
            "type": "ether"
        },
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false
}
```

一旦你找到了對你的自動化有用的 `fact`，你就可以相應地更新你的劇本。例如，以下 playbook 將打印出默認網絡接口的 IPv4 地址。從前面的命令輸出中，我們可以看到通過 Ansible 提供的 JSON 中的 `ansible_default_ipv4.address` 可以得到這個值。

在 ansible-practice 目錄中創建一個名為 playbook-03.yml 的新文件：

```yaml title="playbook-03.yml"
---
- hosts: all
  tasks:
    - name: print facts
      debug:
        msg: "IPv4 address: {{ ansible_default_ipv4.address }}"
```

要在您的清單文件中的服務器上嘗試此 playbook，請使用您之前在運行我們的第一個示例時使用的相同連接參數運行 `ansible-playbook`。同樣，我們將使用一個名為 inventory 的清單文件連接到遠程服務器：

```
$ ansible-playbook -i inventory playbook-03.yml
```

當您運行 playbook 時，您將在輸出中看到您的遠程服務器的 IPv4 地址，如預期的那樣：

```
PLAY [all] *******************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************
ok: [demo-vm]

TASK [print facts] ***********************************************************************************************************
ok: [demo-vm] => {
    "msg": "IPv4 address: 10.0.2.15"
}

PLAY RECAP *******************************************************************************************************************
demo-vm                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Facts 封裝了重要的元數據，您可以利用這些元數據更好地自定義您的劇本。要詳細了解您可以通過事實 facts 獲得的所有信息，請參閱 [Ansible 官方文檔](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html)。

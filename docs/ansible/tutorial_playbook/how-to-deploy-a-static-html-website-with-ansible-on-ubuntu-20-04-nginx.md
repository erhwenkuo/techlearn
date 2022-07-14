# 如何在 Ubuntu 20.04 (Nginx) 上使用 Ansible 部署靜態 HTML 網站

參考: https://www.digitalocean.com/community/tutorials/how-to-deploy-a-static-html-website-with-ansible-on-ubuntu-20-04-nginx

如果你正在閱讀本系列的所有部分，那麼此時你應該熟悉安裝系統包、應用模板和在 Ansible playbook 中使用處理程序。在本系列的這一部分中，你將使用到目前為止所見的內容來創建一個 playbook，該 playbook 可以自動設置 Nginx 服務器以在 Ubuntu 20.04 上託管靜態 HTML 網站。

## 先決條件

首先在 Ansible 控制節點上創建一個新目錄，你將在其中設置 Ansible 文件和一個演示靜態 HTML 網站以部署到遠程服務器。這可以在你的主文件夾中你選擇的任何位置。在本例中，我們將使用 ~/ansible-nginx-demo。

讓我們創建一個新目錄并且切換到新目錄裡。

```
$ mkdir ansible-nginx-demo
$ cd ansible-nginx-demo
```

使用 Vagrant 與 VirtualBox 來在本機啟動一台 Ubuntu 20.04 的虛擬機。

```
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

## 取得演示用的靜態網站

對於這個演示，我們將使用一個靜態 HTML 網站，這是[如何在 HTML 中編碼系列](https://www.digitalocean.com/community/tutorial_series/how-to-build-a-website-with-html)的教學中所構建的網站。首先通過運行以下命令下載演示網站文件：

```bash
$ curl -L https://github.com/do-community/html_demo_site/archive/refs/heads/main.zip -o html_demo.zip

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100  385k    0  385k    0     0   316k      0 --:--:--  0:00:01 --:--:--  316k

```

你需要解壓縮才能解壓縮此下載的內容。為確保你已安裝此工具，請運行：

```
$ sudo apt install unzip
```

然後，解壓縮演示網站文件：

```
unzip html_demo.zip
```

這將在當前的工作目錄上創建一個名為 `html_demo_site-main` 的新目錄。現在工作目錄的結構如下：

``` hl_lines="4-12"
$ tree .
.
├── ansible.cfg
├── html_demo_site-main
│   ├── about.html
│   ├── images
│   │   ├── background.jpg
│   │   ├── large-profile.jpg
│   │   └── small-profile.jpeg
│   ├── index.html
│   ├── LICENSE
│   └── README.md
├── html_demo.zip
├── inventory
└── Vagrantfile
```

## 為 Nginx 的配置創建模板

現在我們將設置配置遠程 Web 服務器所需的 Nginx 設定模板。在工作目錄中創建一個新文件夾來保存其它非 playbook 的文件：

```
$ mkdir files
```

然後，創建一個名為 `nginx.conf.j2` 的新文件：

```bash
$ nano files/nginx.conf.j2
```

此模板文件包含靜態 HTML 網站的 Nginx 服務器塊配置。它使用三個變量：`document_root`、`app_root` 和 `server_name`。我們稍後會在創建 playbook 時定義這些變量。將以下內容複製到你的模板文件中：

```title="nginx.conf.j2"
server {
  listen 80;

  root {{ document_root }}/{{ app_root }};
  index index.html index.htm;

  server_name {{ server_name }};
  
  location / {
   default_type "text/html";
   try_files $uri.html $uri $uri/ =404;
  }
}
```

完成後保存並關閉文件。

## 創建一個新的 Ansible playbook

接下來，我們將創建一個新的 Ansible 劇本並設置我們在本教程上一部分中使用的變量。打開一個名為 `playbook.yml` 的新文件：

```bash
$ nano playbook.yml
```

這個 playbook 將 `hosts` 定義設置為 `all` 和一個 `become` 指令開始，該指令告訴 Ansible 默認以 `root` 用戶身份運行所有任務（與使用 `sudo` 手動運行命令相同）。

在本劇本的 `var` 部分中，我們將創建三個變量：`server_name`、`document_root` 和 `app_root`。這些變量在 Nginx 配置模板中用於設置此 Web 服務器將響應的域名或 IP 地址，以及網站文件在服務器上的完整路徑。對於這個演示，我們將使用 ansible_default_ipv4.address 事實變量，因為它包含遠程服務器的公共 IP 地址，但是如果它在 DNS 服務中正確配置了一個域名以指向這個服務器：

```yaml title="ansible-nginx-demo/playbook.yml"
---
- hosts: all
  become: yes
  vars:
    server_name: "{{ ansible_default_ipv4.address }}"
    document_root: /var/www/html
    app_root: html_demo_site-main
  tasks:
```

你可以暫時保持此文件打開。接下來的部分將引導你完成所有需要包含在本教程中以使其完全正常運行的任務。

### 安裝所需的軟件包

以下任務將更新 `apt` 緩存，然後在遠程節點上安裝 nginx 包：

```yaml title="ansible-nginx-demo/playbook.yml"
    - name: Update apt cache and install Nginx
      apt:
        name: nginx
        state: latest
        update_cache: yes
```

### 將網站文件上傳到遠程節點

下一個任務將使用ansible `copy`內置模塊將網站文件上傳到遠程文檔根目錄。我們將使用 `document_root` 變量在服務器上設置應創建應用程序文件夾的目標。

```yaml title="ansible-nginx-demo/playbook.yml"
    - name: Copy website files to the server's document root
      copy:
        src: "{{ app_root }}"
        dest: "{{ document_root }}"
        mode: preserve
```

### 應用和啟用自定義 Nginx 配置

我們現在將應用 Nginx 模板，該模板將配置 Web 服務器以託管你的靜態 HTML 文件。在 `/etc/nginx/sites-available` 設置配置文件後，我們將在 `/etc/nginx-sites-enabled` 中創建指向該文件的符號鏈接，並通知 Nginx 服務以進行後續重啟。整個過程將需要兩個單獨的任務：

```yaml title="ansible-nginx-demo/playbook.yml"
    - name: Apply Nginx template
      template:
        src: files/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Restart Nginx

    - name: Enable new site
      file:
        src: /etc/nginx/sites-available/default
        dest: /etc/nginx/sites-enabled/default
        state: link
      notify: Restart Nginx
```

### 在 `UFW` 上允許端口 80

ufw 的全名是 Uncomplicated Firewall，意思是不複雜的防火牆。它的指令不但好記，寫好的規則也淺顯易懂，不會像 iptables 的裹腳布又臭又長。

```yaml title="ansible-nginx-demo/playbook.yml"
    - name: Allow all access to tcp port 80
      ufw:
        rule: allow
        port: '80'
        proto: tcp
```

### 為 Nginx 服務創建 Handler

要完成這個劇本，剩下要做的就是設置 Restart Nginx 處理程序：

```yaml title="ansible-nginx-demo/playbook.yml"
  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

### 運行完成的劇本

在 playbook 文件中包含所有必需的任務後，它將如下所示：

```yaml title="ansible-nginx-demo/playbook.yml"
---
- hosts: all
  become: yes
  vars:
    server_name: "{{ ansible_default_ipv4.address }}"
    document_root: /var/www
    app_root: html_demo_site-main
  tasks:
    - name: Update apt cache and install Nginx
      apt:
        name: nginx
        state: latest
        update_cache: yes

    - name: Copy website files to the server's document root
      copy:
        src: "{{ app_root }}"
        dest: "{{ document_root }}"
        mode: preserve

    - name: Apply Nginx template
      template:
        src: files/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Restart Nginx

    - name: Enable new site
      file:
        src: /etc/nginx/sites-available/default
        dest: /etc/nginx/sites-enabled/default
        state: link
      notify: Restart Nginx

    - name: Allow all access to tcp port 80
      ufw:
        rule: allow
        port: '80'
        proto: tcp

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

要在你在清單文件中設置的服務器上執行此 playbook，請使用你在本系列介紹中運行連接測試時使用的相同連接參數運行 `ansible-playbook`。在這裡，我們將使用一個名為 `inventory` 的庫存文件和 sammy 用戶連接到遠程服務器。因為 playbook 需要 sudo 才能運行，所以我們還包括 -K 參數，以在 Ansible 提示時提供遠程用戶的 sudo 密碼：

```bash
$ ansible-playbook playbook.yml
```

結果:

```
$ ansible-playbook playbook.yml 

PLAY [all] ***********************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************
ok: [demo-vm]

TASK [Update apt cache and install Nginx] ****************************************************************************************************************************************************
changed: [demo-vm]

TASK [Copy website files to the server's document root] **************************************************************************************************************************************
changed: [demo-vm]

TASK [Apply Nginx template] ******************************************************************************************************************************************************************
changed: [demo-vm]

TASK [Enable new site] ***********************************************************************************************************************************************************************
ok: [demo-vm]

TASK [Allow all access to tcp port 80] *******************************************************************************************************************************************************
changed: [demo-vm]

RUNNING HANDLER [Restart Nginx] **************************************************************************************************************************************************************
changed: [demo-vm]

PLAY RECAP ***********************************************************************************************************************************************************************************
demo-vm                    : ok=7    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

由於我們使用了Vagrant與VirtualBox來啟動VM，需要在 `Vagrantfile` 裡設定 port-forward 的參數:

```title="Vagrantfile"
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.network "forwarded_port", guest: 80, host: 8080
end
```

重新啟動虛擬機:

```
$ vagrant reload
```

完成後，如果你轉到瀏覽器並訪問服務器的主機名或 IP 地址(我們把網站映射到本機的port: 8080):

- http://localhost:8080

你現在應該會看到以下頁面：

![](./assets/html-demo-site.gif)

## 結論

恭喜，你已使用 Ansible 成功地將靜態 HTML 網站部署到遠程 Nginx 服務器。

如果你對演示網站中的任何文件進行了更改，你可以再次運行 playbook，複製任務將確保任何文件更改都反映在遠程主機中。因為 Ansible 具有冪等行為，所以多次運行 playbook 不會觸發已經對系統進行的更改。





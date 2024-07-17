---
draft: false 
date: 2024-07-17 
categories:
  - Jupyterlab
  - Python
---

# 如何設定 JupyterLab 環境

原文: https://docs.vultr.com/how-to-set-up-a-jupyterlab-environment-on-ubuntu-22-04

## 介紹

JupyterLab 為資​​料科學和人工智慧專案提供基於 Web 的開發環境。它是一個靈活的介面，允許開發人員配置和管理機器學習和科學計算工作流程。 JupyterLab 是下一代 Jupyter Notebook。它旨在解決 Notebook 的許多可用性問題，並透過支援 40 多種程式語言（包括 R、Python、Scala 和 Julia）大大擴展了其使用的範圍。

## 先決條件

在開始之前，您應該：

- 部署 Ubuntu 22.04 伺服器。
- 建立一個非 root sudo 使用者。

<!-- more -->

## 安裝所需的軟體套件

JupyterLab 是個包羅萬象的軟體包；除了用於隔離 Jupyter 核心的 `virtualenv` 套件之外，不需要手動安裝任何額外的套件。

1. 安裝 `virtualenv` 套件。

```bash
apt update

apt install python3-virtualenv
```

2. 建立一個新目錄並導航到其中。

```bash
mkdir /home/$USER/jupyter-env

cd /home/$USER/jupyter-env
```

3. 建立並啟動新的 Python 虛擬環境。

```bash
virtualenv env

source env/bin/activate
```

上述指令在伺服器上建立一個名為 `env` 的新目錄，其中包含 Python 虛擬環境檔案。您可以使用 bin 目錄中的啟動二進位進入虛擬環境。

4. 更新 `pip` 套件管理器。

```bash
(env) $ pip install --upgrade pip
```

5. 安裝 `JupyterLab` 套件。

```bash
(env) $ sudo pip install -U jupyterlab
```

上述指令將安裝最新版本的 JupyterLab。

## 配置 JupyterLab 伺服器

預設情況下，每當您產生 JupyterLab 伺服器時，它都會產生一個唯一的令牌，用於授予對介面的存取權。本部分示範將 JupyterLab 伺服器配置為使用靜態密碼並允許遠端存取的步驟。

1. 產生密碼 Hash。

```bash
(env) $ python3 -c "from jupyter_server.auth import passwd; print(passwd('YOUR_PASSWORD'))"
```

上述指令產生所提供輸入的雜湊值。它使用 Argon2 密碼雜湊函數來產生雜湊值。

!!! info
    記得替換上述的 'YOUR_PASSWORD' 的值。

複製產生的哈希值以供以後使用。

2. 建立一個新目錄來組織 Jupyter Notebooks。

```bash
(env) $ mkdir /home/$USER/jupyter-env/notebooks
```

3. 建立 JupyterLab 設定檔。

```bash
(env) $ sudo nano /home/$USER/jupyter-env/config.py
```

上面的命令產生一個用於持久設定的設定檔。

4. 複製並貼上以下配置。

```python title="config.py"
from jupyter_server.auth.identity import PasswordIdentityProvider

c.ServerApp.allow_remote_access = True
c.ServerApp.allow_origin = '*'
c.ServerApp.root_dir = "/home/USER/jupyter-env/notebooks"

PasswordIdentityProvider.hashed_password = ''
```

上述配置允許 JupyterLab 接受遠端連接並設定用於與介面互動的靜態密碼。

確保提供先前在配置中產生的雜湊值。

儲存並退出文件。

5.禁用防火牆。

```bash
(env) $ sudo ufw disable
```

上述命令停用允許所有連接埠入站連線的防火牆。您在後面的步驟中啟用防火牆以保護您的伺服器。

6. 運行 JupyterLab 伺服器。

```bash
(env) $ jupyter lab --ip 0.0.0.0 --config config.py
```

上述命令在預設連接埠 8888 上產生 JupyterLab 伺服器。確認存取後，使用 `CTRL + C` 可停止伺服器。
# JupyterLab Git 擴展

原文: [JupyterLab Git Extension](https://blogs.oracle.com/ai-and-datascience/post/jupyterlab-git-extension)

## 什麼是 Git?

Git 是一個版本控制系統，允許您跟踪對一組文件所做的更改，並允許您根據需要恢復到文件的先前版本。對程式碼進行版本控制非常重要，這樣您就可以跟踪在完成數據科學項目時所做的更改。通過使用版本控制系統，團隊可以通過擁有一個文件存儲庫來進行協作，團隊成員可以通過合併請求貢獻並添加其更改。

##　什麼是 Git 擴展?

Git 擴展是 JupyterLab 的附加組件，允許用戶通過點擊界面執行基本的 git 操作。它可與 Git 的託管服務（例如 GitHub 和 GitLab）配合使用，讓您可以管理要跟踪的文件的存儲庫。您可以通過登錄 JupyterLab Notebook 並單擊 Notebook 左側的 Git 圖標來訪問 Git 擴展。此外，您還會看到 “Git” 選項卡。如果您沒有看到它們，則您可能使用的是帶有較舊映像的筆記本。您可以停用並重新激活筆記本會話以訪問 Git 擴展。

![](./assets/Medium.avif)

讓我們了解如何創建新的本地 git 存儲庫並將文件提交到存儲庫以及 clone 遠程存儲庫。

## Git 設定

當您在 Notebook 會話中首次使用 Git 時，需要執行一些配置步驟。以下是配置 Git 和驗證 GitHub (or Gitlab) 帳戶的步驟：

轉到 File，選擇 "New Launcher"。啟動一個新的 Terminal 視窗。

![](./assets/git-config.avif)

在終端中，您可以使用以下命令輸入與您的 Git 帳戶關聯的 Git 用戶名和電子郵件。

```bash
$ git config --global user.name "Jane Smith"

$ git config --global user.email jane.smith@oracle.com
```

您可以隨時使用以下命令檢查您的設定：

```bash
$ git config --list
```

您需要設置 ssh 密鑰以用於 Git 網站的身份驗證。如果您還沒有可以使用的 ssh 密鑰，則需要生成一個。轉到終端並粘貼以下命令，將電子郵件替換為您的 GitHub 用戶電子郵件

```bash
$ ssh-keygen -t ed25519 -C "jane.smith@oracle.com" 
```

![](./assets/git-gen-ssh-key.avif)

您可以按 “Enter” 鍵接受保存密鑰的默認位置。

在下一步中，系統將提示您輸入安全密碼。

然後，您需要將 SSH 密鑰添加到 `ssh-agent``

在終端中輸入以下命令:

```bash
$ eval "$(ssh-agent -s)"
```

然後，您可以使用以下命令將 SSH 私鑰添加到 `ssh-agent`` 中。如果您使用不同名稱創建密鑰或使用現有密鑰，請將以下命令中的 `id_ed25519`` 替換為您的私鑰文件。

```bash
$ ssh-add -K ~/.ssh/id_ed25519
```

接下來，顯示 `id_ed25519.pub` (公鑰) 文件的內容

```bash
$ cat ~/.ssh/id_ed25519.pub
```

將內容複製到剪貼板。您將其添加到您的 Github 帳戶中。

轉到您的 GitHub 帳戶並單擊您的頭像。前往 `Settings`:


![](./assets/git-public-key-setup.avif)

然後單擊左側面板上的 “SSH and GPG keys” ，然後單擊頂部的 “New SSH key”:

![](./assets/git-public-key-new-ssh-key.avif)


![](./assets/git-public-key-new-ssh-key2.avif)

您可以從終端複製的 SSH 密鑰貼到此處。

有關設置 ssh 密鑰以對 GitHub 帳戶進行身份驗證的過程的更多信息，您可以訪問 [Adding a new SSH key to your GitHub account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)。有關 GitLab 帳戶身份驗證的信息，您可以訪問 [Add an SSH key to your GitLab account](https://docs.gitlab.com/ee/user/ssh.html#add-an-ssh-key-to-your-gitlab-account)。

## 創建新的本地 Git 存儲庫

1. 點擊 Folder 圖標創建一個新文件夾並將其重命名為您自己選擇的名稱。

    ![](./assets/jupyterlab-create-folder.avif)

2. 將您想要與此新存儲庫關聯的所有文件放入該文件夾中

3. 雙擊您的文件夾即可移至該文件夾。然後，點擊 `Git` 選項卡並選擇 `Initialize a Repository`，如下所示：

    ![](./assets/jupyterlab-init-repo.avif)

4. 然後，會彈出一個窗口要求確認；點擊 “YES”

    ![](./assets/jupyterlab-confirm-init-repo.avif)

現在，您的本地存儲庫已創建，您可以開始將文件提交到存儲庫。

## 將文件提交到存儲庫

當您更改文件時，將文件添加到存儲庫非常重要。讓我們引導您了解如何將更改提交到存儲庫：

1. 如果您還沒有這樣做，請創建一個要將文件推送到的 git 存儲庫。

    ![](./assets/jupyterlab-create-remote-repo.avif)

2. 轉到控制台中的 `Git` 選項卡，然後點擊 `Add Remote Repository`

    ![](./assets/jupyterlab-add-remote-repo.avif)

    將彈出一個屏幕供您添加遠程存儲庫:

    ![](./assets/jupyterlab-add-remote-repo2.avif)

3. 轉到要提交文件的存儲庫頂部，複製遠程存儲庫 URL 並粘貼到彈出窗口中

    ![](./assets/jupyterlab-add-remote-repo3.avif)

4. 返回控制台並單擊要提交的文件旁邊的 “+” 號。這將暫存要提交的文件。

    ![](./assets/jupyterlab-add-remote-repo4.avif)

5. 該文件將被移至 “Staged”。您可以添加提交消息並點擊 “Commit”

    ![](./assets/jupyterlab-add-remote-repo5.avif)

6. 返回 "Git" 選項卡並點擊 "Push to Remote"

    ![](./assets/jupyterlab-add-remote-repo6.avif)

7. 您可以檢查以確保您的文件位於 git 存儲庫中

    ![](./assets/jupyterlab-add-remote-repo7.avif)

## Clone 存儲庫

如果您想將存儲庫從 GitHub 帳戶克隆到筆記本，可以按照以下步驟操作

1. 轉到您的筆記本，單擊 "Git" 選項卡，然後點擊 "Clone a Repository"

    ![](./assets/jupyterlab-clone-remote-repo.avif)

2. 您將看到一個彈出窗口，要求您輸入要 clone 的存儲庫的 URL

    ![](./assets/jupyterlab-clone-remote-repo2.avif)

3. 轉到您要 clone 的 Git 存儲庫，複製存儲庫 URL 並將其粘貼到彈出窗口中

    ![](./assets/jupyterlab-clone-remote-repo3.avif)

4. 驗證您是否已 clone 存儲庫。

    ![](./assets/jupyterlab-clone-remote-repo4.avif)

您可以在[此處](https://github.com/jupyterlab/jupyterlab-git)的文檔中閱讀有關 JupyterLab Git 擴展的更多信息。
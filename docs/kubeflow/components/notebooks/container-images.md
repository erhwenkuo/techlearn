# 容器鏡像

Kubeflow Notebooks 原生支持三種類型的筆記本，`JupyterLab`、`RStudio` 和 `Visual Studio Code`（代碼服務器），但任何基於 Web 的 IDE 都應該可以使用。 Notebook 服務器作為 Kubernetes Pod 內的容器運行，這意味著 IDE 的類型（以及安裝的包）由您為服務器選擇的 Docker 鏡像決定。

## 鏡像

Kubeflow 提供了許多範例容器鏡像來幫助使用者入門。

詳細的配置與修改手法可參考: 

- [Kubeflow Notebook Repo](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers)

### Base Images

這些容器鏡像為 Kubeflow Notebook 容器提供了一個共同的起點。查看本文的 **自定義容器鏡像**　章節以了解如何使用您自己的套件來擴展它們。

|Dockerfile 	|Registry 	|Notes|
|---------------|-----------|-----|
|[base](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/base) 	|[kubeflownotebookswg/base:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/base) 	|common base image|
|[codeserver](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/codeserver) 	|[kubeflownotebookswg/codeserver:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/codeserver) 	|base [code-server](https://github.com/cdr/code-server) (Visual Studio Code) image|
|[jupyter](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter) 	|[kubeflownotebookswg/jupyter:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/jupyter) 	|base [JupyterLab](https://github.com/jupyterlab/jupyterlab) image|
|[rstudio](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/rstudio) 	|[kubeflownotebookswg/rstudio:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/rstudio) 	|base [RStudio](https://github.com/rstudio/rstudio) image|

### Full Images

下列容器鏡像使用數據科學家和 ML 工程師使用的常用套件來擴展了上述的 base images。

|Dockerfile 	|Registry 	|Notes|
|---------------|-----------|-----|
|[codeserver-python](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/codeserver-python) 	|[kubeflownotebookswg/codeserver-python:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/codeserver-python) 	|code-server (Visual Studio Code) + Conda Python|
|[jupyter-pytorch (CPU)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-pytorch) 	| 此版只有源碼但沒有 push 到 dockerhub|JupyterLab + PyTorch (CPU)|
|[jupyter-pytorch (CUDA)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-pytorch) 	| 此版只有源碼但沒有 push 到 dockerhub 	|JupyterLab + PyTorch (CUDA)|
|[jupyter-pytorch-full (CPU)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-pytorch-full) 	|[kubeflownotebookswg/jupyter-pytorch-full:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/jupyter-pytorch-full) 	|JupyterLab + PyTorch (CPU) + [common packages](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-pytorch-full/requirements.txt)|
|[jupyter-pytorch-full (CUDA)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-pytorch-full) 	|[kubeflownotebookswg/jupyter-pytorch-cuda-full:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/jupyter-pytorch-cuda-full) 	|JupyterLab + PyTorch (CUDA) + [common packages](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-pytorch-full/requirements.txt)|
|[jupyter-scipy](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-scipy) 	|[kubeflownotebookswg/jupyter-scipy:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/jupyter-scipy) 	|JupyterLab + [SciPy packages](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-scipy/requirements.txt)|
|[jupyter-tensorflow (CPU)](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-tensorflow) 	| 此版只有源碼但沒有 push 到 dockerhub|JupyterLab + TensorFlow (CPU)|
|[jupyter-tensorflow (CUDA)](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-tensorflow) 	| 此版只有源碼但沒有 push 到 dockerhub 	|JupyterLab + TensorFlow (CUDA)|
|[jupyter-tensorflow-full (CPU)](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-tensorflow-full) 	|[kubeflownotebookswg/jupyter-tensorflow-full:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/jupyter-tensorflow-full) 	|JupyterLab + TensorFlow (CPU) + [common packages](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-tensorflow-full/requirements.txt)|
|[jupyter-tensorflow-full (CUDA)](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-tensorflow-full) 	|[kubeflownotebookswg/jupyter-tensorflow-cuda-full:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/jupyter-tensorflow-cuda-full) 	|JupyterLab + TensorFlow (CUDA) + [common packages](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/jupyter-tensorflow-full/requirements.txt)|
|[rstudio-tidyverse](https://github.com/kubeflow/kubeflow/blob/master/components/example-notebook-servers/rstudio-tidyverse) 	|[kubeflownotebookswg/rstudio-tidyverse:{TAG}](https://hub.docker.com/r/kubeflownotebookswg/rstudio-tidyverse) 	|RStudio + [Tidyverse packages](https://www.tidyverse.org/)|


!!! tip
    當根據 Kubeflow 自定義容器鏡像的要求構建了客制容器鏡像後想要把這個容器鏡像加入到 Notebook 配置的下拉選單中, 需要去修改下列的配置檔案:

    ```yaml title="apps/jupyter/jupyter-web-app/upstream/base/configs/spawner_ui_config.yaml" hl_lines="7-11"
    spawnerFormDefaults:
    image:
        # The container Image for the user's Jupyter Notebook
        value: kubeflownotebookswg/jupyter-scipy:v1.7.0
        # The list of available standard container Images
        options:
        - kubeflownotebookswg/jupyter-scipy:v1.7.0
        - kubeflownotebookswg/jupyter-pytorch-full:v1.7.0
        - kubeflownotebookswg/jupyter-pytorch-cuda-full:v1.7.0
        - kubeflownotebookswg/jupyter-tensorflow-full:v1.7.0
        - kubeflownotebookswg/jupyter-tensorflow-cuda-full:v1.7.0
    ...
    ...
    ```



## 容器鏡像依存關係圖

下列的流程圖顯示了我們的筆記本容器鏡像的相互依賴關係。

![](./assets/notebook-container-image-chart.png)

## 自定義容器鏡像

用戶在生成 Kubeflow Notebook 後所安裝的套件只會持續在 pod 的生命週期（除非安裝到 PVC 支持的目錄中）。

為確保套件在整個 Pod 重啟過程中得到保留，用戶有兩種選擇：

1. 構建包含所需套件的自定義容器鏡像(Custom Image)
2. 確保套件安裝在 PVC 支持的目錄中


!!! info

    1. 通過擴展 `base image`` 之一來製作自定義容器鏡像
    2. 容器鏡像以 `jovyan`` 用戶身份運行
    3. 容器鏡像使用 `s6-overlay init` 系統來管理進程
      - 參考下列文章來了解 [s6-overlay](https://blog.chobon.top/posts/694278be/)

### 容器鏡像要求

要讓 Kubeflow Notebooks 使用的容器鏡像，容器鏡像必須符合下列的要求：

- 在端口 `8888` 上公開一個 HTTP 接口：

    - kubeflow 在運行時使用我們期望容器監聽的 URL 路徑設置環境變量 `NB_PREFIX`
    - kubeflow 使用 IFrame，因此請確保您的應用程序在 HTTP 響應標頭中設置了 `Access-Control-Allow-Origin: *`

- 以名為 `jovyan` 的用戶身份運行：

    - jovyan 的主目錄應該是 `/home/jovyan`
    - jovyan 的 UID 應該是 `1000`

- 使用掛載在 `/home/jovyan` 的空 PVC 成功啟動：

    - kubeflow 會在 `/home/jovyan` 掛載一個 PVC 以在 Pod 重啟時保持狀態


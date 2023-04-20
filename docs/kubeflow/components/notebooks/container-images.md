# 容器鏡像

Kubeflow Notebooks 原生支持三種類型的筆記本，`JupyterLab`、`RStudio` 和 `Visual Studio Code`（代碼服務器），但任何基於 Web 的 IDE 都應該可以使用。 Notebook 服務器作為 Kubernetes Pod 內的容器運行，這意味著 IDE 的類型（以及安裝的包）由您為服務器選擇的 Docker 鏡像決定。

## 鏡像

Kubeflow 提供了許多範例容器鏡像來幫助使用者入門。

### Base Images

這些鏡像為 Kubeflow Notebook 容器提供了一個共同的起點。查看 custom images 以了解如何使用您自己的包或套件來擴展它們。

|Dockerfile 	|Registry 	|Notes|
|===============|===========|=====|
|[base](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/base) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/base:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/base) 	|common base image|
|[codeserver](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/codeserver) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/codeserver:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/codeserver) 	|base [code-server](https://github.com/cdr/code-server) (Visual Studio Code) image|
|[jupyter](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter) 	|base [JupyterLab](https://github.com/jupyterlab/jupyterlab) image|
|[rstudio](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/rstudio) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/rstudio:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/rstudio) 	|base [RStudio](https://github.com/rstudio/rstudio) image|

### Full Images

下列鏡像使用數據科學家和 ML 工程師使用的常用包來擴展了 base images。

|Dockerfile 	|Registry 	|Notes|
|===============|===========|=====|
|[codeserver-python](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/codeserver-python) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/codeserver-python:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/codeserver-python) 	|code-server (Visual Studio Code) + Conda Python|
|[jupyter-pytorch (CPU)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-pytorch) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-pytorch:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-pytorch) 	|JupyterLab + PyTorch (CPU)|
|[jupyter-pytorch (CUDA)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-pytorch) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-pytorch-cuda:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-pytorch-cuda) 	|JupyterLab + PyTorch (CUDA)|
|[jupyter-pytorch-full (CPU)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-pytorch-full) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-pytorch-full:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-pytorch-full) 	|JupyterLab + PyTorch (CPU) + common packages|
|[jupyter-pytorch-full (CUDA)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-pytorch-full) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-pytorch-cuda-full:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-pytorch-cuda-full) 	|JupyterLab + PyTorch (CUDA) + common packages|
|[jupyter-scipy](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-scipy) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-scipy:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-scipy) 	|JupyterLab + [SciPy](https://www.scipy.org/) packages|
|[jupyter-tensorflow (CPU)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-tensorflow) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-tensorflow:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-tensorflow) 	|JupyterLab + TensorFlow (CPU)|
|[jupyter-tensorflow (CUDA)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-tensorflow) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-tensorflow-cuda:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-tensorflow-cuda) 	|JupyterLab + TensorFlow (CUDA)|
|[jupyter-tensorflow-full (CPU)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-tensorflow-full) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-tensorflow-full:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-tensorflow-full) 	|JupyterLab + TensorFlow (CPU) + [common](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-tensorflow-full/requirements.txt) packages|
|[jupyter-tensorflow-full (CUDA)](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-tensorflow-full) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-tensorflow-cuda-full:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/jupyter-tensorflow-cuda-full) 	|JupyterLab + TensorFlow (CUDA) + [common](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/jupyter-tensorflow-full/requirements.txt) packages|
|[rstudio-tidyverse](https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers/rstudio-tidyverse) 	|[public.ecr.aws/j1r0q0g6/notebooks/notebook-servers/rstudio-tidyverse:{TAG}](https://gallery.ecr.aws/j1r0q0g6/notebooks/notebook-servers/rstudio-tidyverse) 	|RStudio + [Tidyverse](https://www.tidyverse.org/) packages|

!!! tip
    當根據 Kubeflow 自定義鏡像的要求構建了客制鏡像後想要把這個鏡像加入到 Notebook 配置的下拉選單中, 需要去修改下列的配置檔案:

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



## 鏡像依存關係圖

下列的流程圖顯示了我們的筆記本容器鏡像的相互依賴關係。

![](./assets/notebook-container-image-chart.png)

## 自定義鏡像

用戶在生成 Kubeflow Notebook 後所安裝的套件只會持續在 pod 的生命週期（除非安裝到 PVC 支持的目錄中）。

為確保套件在整個 Pod 重啟過程中得到保留，用戶有兩種選擇：

1. 構建包含所需套件的自定義鏡像(Custom Image)
2. 確保套件安裝在 PVC 支持的目錄中


### 鏡像要求

要讓 Kubeflow Notebooks 使用的容器鏡像，鏡像必須符合下列的要求：

- 在端口 `8888` 上公開一個 HTTP 接口：

    - kubeflow 在運行時使用我們期望容器監聽的 URL 路徑設置環境變量 `NB_PREFIX`
    - kubeflow 使用 IFrame，因此請確保您的應用程序在 HTTP 響應標頭中設置了 `Access-Control-Allow-Origin: *`

- 以名為 `jovyan` 的用戶身份運行：

    - jovyan 的主目錄應該是 `/home/jovyan`
    - jovyan 的 UID 應該是 `1000`

- 使用掛載在 `/home/jovyan` 的空 PVC 成功啟動：

    - kubeflow 會在 `/home/jovyan` 掛載一個 PVC 以在 Pod 重啟時保持狀態


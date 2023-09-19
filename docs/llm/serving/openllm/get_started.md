# 使用 OpenLLM

## 先決條件

您已安裝 Python 3.8（或更高版本）和 pip。我們強烈建議使用虛擬環境來防止套件衝突。

```bash
conda create --name openllm

conda activate openllm

conda install pip
```

## 安裝 OpenLLM

使用 `pip` 安裝 OpenLLM，如下所示：

```bash
pip install openllm
```

要驗證安裝結果，請運行：

```bash
openllm -h
```

結果:

```bash
Usage: openllm [OPTIONS] COMMAND [ARGS]...

   ██████╗ ██████╗ ███████╗███╗   ██╗██╗     ██╗     ███╗   ███╗
  ██╔═══██╗██╔══██╗██╔════╝████╗  ██║██║     ██║     ████╗ ████║
  ██║   ██║██████╔╝█████╗  ██╔██╗ ██║██║     ██║     ██╔████╔██║
  ██║   ██║██╔═══╝ ██╔══╝  ██║╚██╗██║██║     ██║     ██║╚██╔╝██║
  ╚██████╔╝██║     ███████╗██║ ╚████║███████╗███████╗██║ ╚═╝ ██║
   ╚═════╝ ╚═╝     ╚══════╝╚═╝  ╚═══╝╚══════╝╚══════╝╚═╝     ╚═╝.

  An open platform for operating large language models in production.
  Fine-tune, serve, deploy, and monitor any LLMs with ease.

Options:
  -v, --version  Show the version and exit.
  -h, --help     Show this message and exit.

Commands:
  build       Package a given models into a Bento.
  embed       Get embeddings interactively, from a terminal.
  import      Setup LLM interactively.
  instruct    Instruct agents interactively for given tasks, from a...
  models      List all supported models.
  prune       Remove all saved models, (and optionally bentos) built with...
  query       Ask a LLM interactively, from a terminal.
  start       Start any LLM as a REST server.
  start-grpc  Start any LLM as a gRPC server.

Extensions:
  build-base-container  Base image builder for BentoLLM.
  dive-bentos           Dive into a BentoLLM.
  get-containerfile     Return Containerfile of any given Bento.
  get-prompt            Get the default prompt used by OpenLLM.
  list-bentos           List available bentos built by OpenLLM.
  list-models           This is equivalent to openllm models...
  playground            OpenLLM Playground.
```

## 啟動 LLM 服務器

OpenLLM 允許您使用 `openllm start` 快速啟動 LLM 服務器實例。例如，要啟動 [OPT](https://huggingface.co/docs/transformers/model_doc/opt) 服務器，請運行以下命令：

```bash
# install opt model dependency
pip install openllm[opt]

openllm start opt --model-id facebook/opt-125m
```

這將在 `http://0.0.0.0:3000/` 處啟動服務器。如果模型之前尚未註冊，OpenLLM 會將模型下載到 BentoML 本地模型商店。要查看本地模型，請運行 `bentoml models list` 或 `openllm list-models`。

要與服務器交互，您可以訪問 `http://0.0.0.0:3000/` 的 Web UI 或使用 `curl` 發送請求。您還可以使用 OpenLLM 的內置 Python 客戶端與服務器交互：

```python
import openllm

client = openllm.client.HTTPClient('http://localhost:3000')

client.query('Explain to me the difference between "further" and "farther"')
```

OpenLLM 無縫支持許多模型及其變體。您可以通過提供 `--model-id` 選項來指定要提供服務的模型的不同變體。例如：

```python
openllm start opt --model-id facebook/opt-2.7b
```

!!! info
    OpenLLM 支持為任何支持的模型指定微調權重和量化權重，只要它們可以加載模型架構即可。使用 `openllm models` 命令查看支持的模型、其架構及其變體的完整列表。


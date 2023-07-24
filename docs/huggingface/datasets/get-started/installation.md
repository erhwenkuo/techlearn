# Installation

在開始之前，您需要設置環境並安裝適當的套件。 🤗 數據集在 Python 3.7+ 上進行了測試。

!!! quote

    如果您想將 🤗 數據集與 TensorFlow 或 PyTorch 一起使用，則需要單獨安裝它們。請參閱 [TensorFlow 安裝頁面](https://www.tensorflow.org/install/pip#tensorflow-2-packages-are-available)或 [PyTorch 安裝頁面](https://pytorch.org/get-started/locally/#start-locally)，了解適用於您的框架的特定安裝命令。

## Virtual environment

您應該在虛擬環境中安裝🤗數據集，以保持整潔並避免依賴衝突。

1. 創建並導航到您的 project 目錄：

    ```bash
    mkdir ~/my-project
    cd ~/my-project
    ```

2. 在您的目錄中啟動虛擬環境：

    ```bash
    python -m venv .env
    ```

3. 使用以下命令激活和停用虛擬環境：

    ```bash
    # Activate the virtual environment
    source .env/bin/activate

    # Deactivate the virtual environment
    source .env/bin/deactivate
    ```

創建虛擬環境後，您可以在其中安裝 🤗 數據集。

## pip

安裝 🤗 數據集最直接的方法是使用 pip：

```bash
pip install datasets
```

運行以下命令檢查🤗數據集是否已正確安裝：

```bash
python -c "from datasets import load_dataset; print(load_dataset('squad', split='train')[0])"
```

此命令下載 Stanford 問答數據集 (SQuAD) 的 version 1、加載訓練分割並打印第一個訓練示例。你應該看到：

```bash
{'answers': {'answer_start': [515], 'text': ['Saint Bernadette Soubirous']}, 'context': 'Architecturally, the school has a Catholic character. Atop the Main Building\'s gold dome is a golden statue of the Virgin Mary. Immediately in front of the Main Building and facing it, is a copper statue of Christ with arms upraised with the legend "Venite Ad Me Omnes". Next to the Main Building is the Basilica of the Sacred Heart. Immediately behind the basilica is the Grotto, a Marian place of prayer and reflection. It is a replica of the grotto at Lourdes, France where the Virgin Mary reputedly appeared to Saint Bernadette Soubirous in 1858. At the end of the main drive (and in a direct line that connects through 3 statues and the Gold Dome), is a simple, modern stone statue of Mary.', 'id': '5733be284776f41900661182', 'question': 'To whom did the Virgin Mary allegedly appear in 1858 in Lourdes France?', 'title': 'University_of_Notre_Dame'}
```

## Audio

要使用音頻數據集，您需要安裝音頻功能作為額外的依賴項：

```bash
pip install datasets[audio]
```

!!! info
    要解碼 mp3 文件，您至少需要擁有 1.1.0 版的 `libsndfile` 系統庫。通常，它與 `python soundfile` 套件捆綁在一起，該套件作為 🤗 數據集的額外音頻依賴項安裝。對於 Linux，從版本 0.12.0 開始，所需版本的 `libsndfile` 與聲音文件捆綁在一起。您可以運行以下命令來確定 `soundfile` 使用的 `libsndfile` 版本：

    ```bash
    python -c "import soundfile; print(soundfile.__libsndfile_version__)"
    ```

## Vision

要使用圖像數據集，您需要安裝圖像功能作為額外的依賴項：

```bash
pip install datasets[vision]
```

## source

從源代碼構建 `datasets` 套件可以讓您對代碼庫進行更改。要從源代碼安裝，請 clone 存儲庫並使用以下命令進行安裝：

```bash
git clone https://github.com/huggingface/datasets.git
cd datasets
pip install -e .
```

同樣，您可以使用以下命令檢查🤗數據集是否已正確安裝：

```bash
python -c "from datasets import load_dataset; print(load_dataset('squad', split='train')[0])"
```

## conda

🤗 數據集也可以從套件管理系統 `conda` 安裝：

```bash
conda install -c huggingface -c conda-forge datasets
```
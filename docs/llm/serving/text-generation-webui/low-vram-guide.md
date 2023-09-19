# 低顯存指引

如果您的 GPU 的顯存不夠大，無法載入 16-bit 模型，請按以下順序嘗試這些操作：

## 以 8-bit 加載模型

```bash
python server.py --load-in-8bit
```

## 以 4-bit 加載模型

```bash
python server.py --load-in-4bit
```

## 將模型拆分到 GPU 和 CPU

```bash
python server.py --auto-devices
```

如果您可以使用此命令加載模型，但在嘗試生成文本時內存不足，請嘗試逐漸限制分配給 GPU 的內存量，直到錯誤停止發生：

```bash
python server.py --auto-devices --gpu-memory 10
python server.py --auto-devices --gpu-memory 9
python server.py --auto-devices --gpu-memory 8
...
```

其中數字以 GiB 為單位。

為了更好地控制，您還可以顯式指定 MiB 單位：

```bash
python server.py --auto-devices --gpu-memory 8722MiB
python server.py --auto-devices --gpu-memory 4725MiB
python server.py --auto-devices --gpu-memory 3500MiB
...
```

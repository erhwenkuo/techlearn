# Content Type 解碼

MLServer 透過新增對 `content_type` 註解的支援來擴展 V2 inference protocol。此註釋可以透過模型元資料參數或 `parameters` 提供。透過利用 `content_type` 註釋，我們可以向 MLServer 提供必要的訊息，以便它可以將來自 V2 inference protocol 的輸入 payload 解碼為對模型/使用者有意義的內容（例如 NumPy 陣列）。

本範例將引導您完成一些範例，說明其工作原理以及如何擴展。

## Echo Inference Runtime

首先，我們將編寫一個 dummy runtime，它只會列印輸入、解碼後的輸入並返回它。這將作為展示 `content_type` 支援如何運作的測試平台。

接下來，我們將透過新增自訂編解碼器來擴展此 dummy runtime，這些編解碼器會將我們的  V2 inference protocol 的輸入 payload 解碼為自訂類型。

```python title="runtime.py"
import json

from mlserver import MLModel
from mlserver.types import InferenceRequest, InferenceResponse, ResponseOutput
from mlserver.codecs import DecodedParameterName

_to_exclude = {
    "parameters": {DecodedParameterName, "headers"},
    'inputs': {"__all__": {"parameters": {DecodedParameterName, "headers"}}}
}


class EchoRuntime(MLModel):
    async def predict(self, payload: InferenceRequest) -> InferenceResponse:
        outputs = []
        for request_input in payload.inputs:
            decoded_input = self.decode(request_input)

            print(f"------ Encoded Input ({request_input.name}) ------")
            as_dict = request_input.dict(exclude=_to_exclude)  # type: ignore
            print(json.dumps(as_dict, indent=2))
            print(f"------ Decoded input ({request_input.name}) ------")
            print(decoded_input)

            outputs.append(
                ResponseOutput(
                    name=request_input.name,
                    datatype=request_input.datatype,
                    shape=request_input.shape,
                    data=request_input.data
                )
            )

        return InferenceResponse(model_name=self.name, outputs=outputs)
```

如您在上面看到的，該運行時將透過呼叫 `self.decode()` 輔助方法來解碼傳入的有效負載。此方法將按以下順序檢查每個輸入的正確內容類型：

1. **request payload** 中的 `input[].parameters.content_type` 欄位中是否定義了任何內容類型？
2. **model metadata** 中的 `input[].parameters.content_type` 欄位中是否定義了任何內容類型？
3. 是否有任何應該假定的預設內容類型？

### Model Settings

為了啟用此運行時，我們還將建立一個 `model-settings.json` 檔案。該檔案應該存在於（或可存取）我們執行 `mlserver start` 的資料夾中。

```json title="model-settings.json"
{
    "name": "content-type-example",
    "implementation": "runtime.EchoRuntime"
}
```

### 開始模型推論服務

現在我們已經有了配置，我們可以透過執行 `mlserver start` 來啟動伺服器。

```bash
mlserver start .
```

由於此命令將啟動伺服器並封鎖終端，等待請求，因此需要在單獨的終端機上在背景執行。

## Request Inputs

我們的第一步是根據傳入的 `input[].parameters` 欄位決定內容類型。為此，我們將在背景啟動 MLServer（例如執行 `mlserver start` 。）

```python title="test_infer.py"
import requests

payload = {
    "inputs": [
        {
            "name": "parameters-np",
            "datatype": "INT32",
            "shape": [2, 2],
            "data": [1, 2, 3, 4],
            "parameters": {
                "content_type": "np"
            }
        },
        {
            "name": "parameters-str",
            "datatype": "BYTES",
            "shape": [1],
            "data": "hello world 😁",
            "parameters": {
                "content_type": "str"
            }
        }
    ]
}

response = requests.post(
    "http://localhost:8080/v2/models/content-type-example/infer",
    json=payload
)

# 打印結果
print(json.dumps(response.json(), indent=2))
```

結果:

```json
{
  "model_name": "content-type-example",
  "id": "32dd0d7f-36c6-4b0e-a391-a39f1a5c351f",
  "parameters": {},
  "outputs": [
    {
      "name": "parameters-np",
      "shape": [
        2,
        2
      ],
      "datatype": "INT32",
      "data": [
        1,
        2,
        3,
        4
      ]
    },
    {
      "name": "parameters-str",
      "shape": [
        1
      ],
      "datatype": "BYTES",
      "data": "hello world 😁"
    }
  ]
}
```

### Codecs

您可能已經注意到，編寫符合 V2 Inference Protocol 的請求有效負載需要對 V2 規範和每種內容類型所需的結構有一定的了解。為了解決這個問題並簡化使用，MLServer 套件公開了一組實用程序，可協助您透過 V2 協定與模型進行互動。

這些工具主要被塑造為 "codecs"。也就是說，知道如何將任意 Python 資料類型與 V2 推理協定「編碼」和「解碼」的抽象化。

通常，我們建議使用現有的編解碼器集來產生 V2 有效負載。這將確保請求和回應遵循正確的結構，並應提供更無縫的體驗。

按照我們先前的範例，可以使用編解碼器將相同的程式碼重寫為：

```python
import requests
import numpy as np

from mlserver.types import InferenceRequest, InferenceResponse
from mlserver.codecs import NumpyCodec, StringCodec

parameters_np = np.array([[1, 2], [3, 4]])
parameters_str = ["hello world 😁"]

payload = InferenceRequest(
    inputs = [
        NumpyCodec.encode_input("parameters_np", parameters_np),
        # The `use_bytes=False` flag will ensure that the encoded payload is JSON-compatible
        StringCodec.encode_input("parameters-str", parameters_str, use_bytes=False),
    ]
)

response = requests.post(
    "http://localhost:8080/v2/models/content-type-example/infer",
    json=payload.model_dump()
)


# 重新構建成 InferenceResponse 物件
raw_response = response.json()

response_payload = InferenceResponse(**raw_response)

print(NumpyCodec.decode_output(response_payload.outputs[0]))
print(StringCodec.decode_output(response_payload.outputs[1]))
```

請注意，重寫的程式碼片段現在使用內建的 `InferenceRequest` 類別，它表示 V2 推理請求。最重要的是，它還使用 `NumpyCodec` 和 `StringCodec` 實現，它們知道如何將 Numpy array 和 string 編碼為 V2 相容的請求輸入。

### Model Metadata

我們的下一步將是透過模型元資料定義預期的內容類型。這可以透過擴展 `model-settings.json` 檔案並添加有關輸入的部分來完成。

```json title="model-settings.json"
{
    "name": "content-type-example",
    "implementation": "runtime.EchoRuntime",
    "inputs": [
        {
            "name": "metadata-np",
            "datatype": "INT32",
            "shape": [2, 2],
            "parameters": {
                "content_type": "np"
            }
        },
        {
            "name": "metadata-str",
            "datatype": "BYTES",
            "shape": [11],
            "parameters": {
                "content_type": "str"
            }
        }
    ]
}
```

新增此元資料後，我們將重新啟動 MLServer（例如 `mlserver start` ），並且我們將發送一個不帶任何明確參數的新請求。

```python
import requests

payload = {
    "inputs": [
        {
            "name": "metadata-np",
            "datatype": "INT32",
            "shape": [2, 2],
            "data": [1, 2, 3, 4],
        },
        {
            "name": "metadata-str",
            "datatype": "BYTES",
            "shape": [11],
            "data": "hello world 😁",
        }
    ]
}

response = requests.post(
    "http://localhost:8080/v2/models/content-type-example/infer",
    json=payload
)
```

正如您應該能夠在伺服器日誌中看到的那樣，MLServer 將根據模型元資料交叉引用輸入名稱以找到正確的內容類型。

### 客制 Codecs

在某些情況下，自訂推理運行時可能需要 編碼/解碼 為自訂資料類型。作為一個例子，我們可以想到只能對 `pillow` 圖像物件進行操作的電腦視覺模型。

在這些場景中，可以擴展 `Codec` 介面來編寫我們的自訂編碼邏輯。編解碼器只是一個定義了 `decode()` 和 `encode()` 方法的物件。為了說明這是如何工作的，我們將擴展自訂運行時以添加自訂 `PillowCodec`。

```python title="runtime.py"
import io
import json

from PIL import Image

from mlserver import MLModel
from mlserver.types import (
    InferenceRequest,
    InferenceResponse,
    RequestInput,
    ResponseOutput,
)
from mlserver.codecs import NumpyCodec, register_input_codec, DecodedParameterName
from mlserver.codecs.utils import InputOrOutput

_to_exclude = {
    "parameters": {DecodedParameterName},
    "inputs": {"__all__": {"parameters": {DecodedParameterName}}},
}

@register_input_codec
class PillowCodec(NumpyCodec):
    ContentType = "img"
    DefaultMode = "L"

    @classmethod
    def can_encode(cls, payload: Image) -> bool:
        return isinstance(payload, Image)

    @classmethod
    def _decode(cls, input_or_output: InputOrOutput) -> Image:
        if input_or_output.datatype != "BYTES":
            # If not bytes, assume it's an array
            image_array = super().decode_input(input_or_output)  # type: ignore
            return Image.fromarray(image_array, mode=cls.DefaultMode)

        encoded = input_or_output.data
        if isinstance(encoded, str):
            encoded = encoded.encode()

        return Image.frombytes(
            mode=cls.DefaultMode, size=input_or_output.shape, data=encoded
        )

    @classmethod
    def encode_output(cls, name: str, payload: Image) -> ResponseOutput:  # type: ignore
        byte_array = io.BytesIO()
        payload.save(byte_array, mode=cls.DefaultMode)

        return ResponseOutput(
            name=name, shape=payload.size, datatype="BYTES", data=byte_array.getvalue()
        )

    @classmethod
    def decode_output(cls, response_output: ResponseOutput) -> Image:
        return cls._decode(response_output)

    @classmethod
    def encode_input(cls, name: str, payload: Image) -> RequestInput:  # type: ignore
        output = cls.encode_output(name, payload)
        return RequestInput(
            name=output.name,
            shape=output.shape,
            datatype=output.datatype,
            data=output.data,
        )

    @classmethod
    def decode_input(cls, request_input: RequestInput) -> Image:
        return cls._decode(request_input)

class EchoRuntime(MLModel):
    async def predict(self, payload: InferenceRequest) -> InferenceResponse:
        outputs = []
        for request_input in payload.inputs:
            decoded_input = self.decode(request_input)
            print(f"------ Encoded Input ({request_input.name}) ------")
            as_dict = request_input.dict(exclude=_to_exclude)  # type: ignore
            print(json.dumps(as_dict, indent=2))
            print(f"------ Decoded input ({request_input.name}) ------")
            print(decoded_input)

            outputs.append(
                ResponseOutput(
                    name=request_input.name,
                    datatype=request_input.datatype,
                    shape=request_input.shape,
                    data=request_input.data,
                )
            )

        return InferenceResponse(model_name=self.name, outputs=outputs)
```

我們現在應該能夠重新啟動 MLServer 實例（即使用 `mlserver start .` 命令），以傳送一些測試請求。


```python
import requests

payload = {
    "inputs": [
        {
            "name": "image-int32",
            "datatype": "INT32",
            "shape": [8, 8],
            "data": [
                1, 0, 1, 0, 1, 0, 1, 0,
                1, 0, 1, 0, 1, 0, 1, 0,
                1, 0, 1, 0, 1, 0, 1, 0,
                1, 0, 1, 0, 1, 0, 1, 0,
                1, 0, 1, 0, 1, 0, 1, 0,
                1, 0, 1, 0, 1, 0, 1, 0,
                1, 0, 1, 0, 1, 0, 1, 0,
                1, 0, 1, 0, 1, 0, 1, 0
            ],
            "parameters": {
                "content_type": "img"
            }
        },
        {
            "name": "image-bytes",
            "datatype": "BYTES",
            "shape": [8, 8],
            "data": (
                "10101010"
                "10101010"
                "10101010"
                "10101010"
                "10101010"
                "10101010"
                "10101010"
                "10101010"
            ),
            "parameters": {
                "content_type": "img"
            }
        }
    ]
}

response = requests.post(
    "http://localhost:8080/v2/models/content-type-example/infer",
    json=payload
)
```

正如您應該能夠在 MLServer 日誌中看到的那樣，伺服器現在能夠將有效負載解碼為 `Pillow` image。此範例也說明了 Codec 物件如何與多種資料類型值（例如本例中的張量和 BYTES）相容。

## Request Codecs

到目前為止，我們已經了解如何指定編解碼器，以便將它們套用在 input level。但是，也可以使用請求範圍的編解碼器來聚合多個輸入來解碼有效負載。這通常與模型需要多列輸入類型的情況相關，例如 Pandas DataFrame。

為了說明這一點，我們將首先調整 EchoRuntime，以便它在請求層級列印解碼的內容。

```python title="runtime.py"
import json

from mlserver import MLModel
from mlserver.types import InferenceRequest, InferenceResponse, ResponseOutput
from mlserver.codecs import DecodedParameterName

_to_exclude = {
    "parameters": {DecodedParameterName},
    'inputs': {"__all__": {"parameters": {DecodedParameterName}}}
}

class EchoRuntime(MLModel):
    async def predict(self, payload: InferenceRequest) -> InferenceResponse:
        print("------ Encoded Input (request) ------")
        as_dict = payload.dict(exclude=_to_exclude)  # type: ignore
        print(json.dumps(as_dict, indent=2))
        print("------ Decoded input (request) ------")
        decoded_request = None
        if payload.parameters:
            decoded_request = getattr(payload.parameters, DecodedParameterName)
        print(decoded_request)

        outputs = []
        for request_input in payload.inputs:
            outputs.append(
                ResponseOutput(
                    name=request_input.name,
                    datatype=request_input.datatype,
                    shape=request_input.shape,
                    data=request_input.data
                )
            )

        return InferenceResponse(model_name=self.name, outputs=outputs)
```

我們現在應該能夠重新啟動 MLServer 實例（即使用 `mlserver start .` 命令），以傳送一些測試請求。

```python
import requests

payload = {
    "inputs": [
        {
            "name": "parameters-np",
            "datatype": "INT32",
            "shape": [2, 2],
            "data": [1, 2, 3, 4],
            "parameters": {
                "content_type": "np"
            }
        },
        {
            "name": "parameters-str",
            "datatype": "BYTES",
            "shape": [2, 11],
            "data": ["hello world 😁", "bye bye 😁"],
            "parameters": {
                "content_type": "str"
            }
        }
    ],
    "parameters": {
        "content_type": "pd"
    }
}

response = requests.post(
    "http://localhost:8080/v2/models/content-type-example/infer",
    json=payload
)
```


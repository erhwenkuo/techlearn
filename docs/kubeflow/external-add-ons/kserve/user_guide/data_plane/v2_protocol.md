# Data Plane (V2)

**Open Inference Protocol (V2 Inference Protocol)**

對於符合此協議的推理服務器，服務器必須實現健康、元數據和推理 V2 API。不需要明確註明的可選功能。合規推理服務器可以選擇實現 [HTTP/REST API](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#httprest) 和/或 [GRPC API](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#grpc)。

檢查 在 [model serving runtime](https://kserve.github.io/website/0.10/modelserving/v1beta1/serving_runtime/) 中的 `protocolVersion` 欄位，以確保您正在使用的模型服務運行 runtime 支持 `V2` 協議。

注意：對於此頁面上的所有 API 描述，所有上下文中的所有字符串都區分大小寫。 `V2` 協議支持擴展機製作為 API 的必需部分，但本文檔沒有提出任何具體的擴展。任何具體的 extensions 將單獨提出。

!!! info
  V2 協議目前不像 V1 協議那樣支持 `explain` 端點。如果這是您希望在 V2 協議中擁有的功能，請提交一個 [github 問題](https://github.com/kserve/kserve/issues)。

## HTTP/REST

HTTP/REST API 使用 JSON，因為它得到廣泛支持並且與語言無關。在本文檔中顯示的所有 JSON 模式中，`$number`、`$string`、`$boolean`、`$object` 和 `$array` 指的是基本的 JSON 類型。 `#optional` 表示可選的 JSON 欄位。

|API	|Verb	|Path	|Request Payload	|Response Payload|
|-----|-----|-----|-----------------|----------------|
|Inference	|POST	|`v2/models/[/versions/<model_version>]/infer`	|$inference_request|$inference_response|
|Model Metadata	|GET	|`v2/models/<model_name>[/versions/<model_version>]`		||$metadata_model_response|
|Server Ready	|GET	|`v2/health/ready`		||$ready_server_response|
|Server Live	|GET	|`v2/health/live`		||$live_server_response|
|Server Metadata	|GET	|`v2`		||$metadata_server_response|
|Model Ready	|GET	|`v2/models/<model_name>[/versions/]/ready`		||$ready_model_response|

有關負載內容的更多信息，請參閱 Payload Contents。

### API 定義

|API	|Definition|
|-----|----------|
|Inference	|`/infer` 端點對模型執行推理。響應是預測結果。|
|Model Metadata	|"model metadata" API 是每個模型的端點，它返回有關在路徑中傳遞的模型的詳細信息。|
|Server Ready	|"server ready" 健康 API 指示是否所有模型都已準備好進行推理。可以直接使用 "server ready" health API 來實現 Kubernetes readinessProbe|
|Server Live	|"server live" 健康 API 指示推理服務器是否能夠接收和響應元數據和推理請求。可以直接使用 "server live" API 來實現 Kubernetes livenessProbe。|
|Server Metadata	|"server metadata" API 返回描述服務器的詳細信息。|
|Model Ready	|"model ready" 健康 API 指示特定模型是否已準備好進行推理。模型名稱和（optional）版本必須在 URL 中可用。|

### Health/Readiness/Liveness 探針

“Model Readiness” 探測主要是要用來回答 “模型是否下載並且能夠開始服務推理請求？” 這類型的問題。 而 "Server Readiness/Liveness" 探測是要回答 “我的服務及其基礎設施是否運行、健康並且能夠接收和處理請求？” 的問題。

### Payload 說明

#### Inference

使用 HTTP POST 向 `/infer` 端點發出推理請求。在請求中，HTTP 正文包含推理請求 JSON 對象。在相應的響應中，HTTP 正文包含推理響應 JSON 對像或推理響應 JSON 錯誤對象。有關一些示例 HTTP/REST 請求和響應，請參閱[推理請求示例](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#inference-request-examples)。

**Inference Request JSON Object**

POST 請求的 HTTP 正文中需要推理請求對象，標識為 `$inference_request`。模型名稱和（可選）版本必須在 URL 中可用。如果未提供版本，服務器可能會根據自己的策略選擇版本或返回錯誤。

```python
$inference_request =
{
  "id" : $string #optional,
  "parameters" : $parameters #optional,
  "inputs" : [ $request_input, ... ],
  "outputs" : [ $request_output, ... ] #optional
}
```

- `id`：此請求的標識符。可選，但如果指定，則必須在響應中返回此標識符。
- `parameters`：包含此推理請求的零個或多個參數的對象，表示為鍵/值對。有關詳細信息，請參閱[參數](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#parameters)。
- `inputs`：輸入張量。每個輸入都使用請求輸入中定義的 `$request_input` 模式進行描述。
- `outputs`：為該推理請求的輸出張量。每個請求的輸出都使用請求輸出中定義的 `$request_output` 模式進行描述。可選，如果未指定，模型產生的所有輸出將使用默認的 `$request_output` 設置返回。


`$inference_request_input` JSON 描述模型的輸入。如果輸入是批次(batch)處理的，則 `shape` 和 `data` 必須代表整個批次的完整 shape 和 content。

```python
$inference_request_input =
{
  "name" : $string,
  "shape" : [ $number, ... ],
  "datatype"  : $string,
  "parameters" : $parameters #optional,
  "data" : $tensor_data
}
```

- `name`: 輸入張量的名稱。
- `shape`: 輸入張量的形狀。每個維度都必須是可表示為無符號 64 位整數值的整數。
- `datatype`: 張量數據類型中定義的輸入 [Tensor Data Types](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#tensor-data-types)。
- `parameters`: 包含此輸入的零個或多個參數的對象，表示為鍵/值對。有關詳細信息，請參閱 [Parameters](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#parameters)。
- `data`: 張量的內容。有關詳細信息，請參閱 [Tensor Data](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#tensor-data)。

**Inference Response JSON Object**

成功的推理請求由 200 HTTP 狀態代碼指示。推理響應對象，標識為 `$inference_response`，在 HTTP 正文中返回。

```python
$inference_response =
{
  "model_name" : $string,
  "model_version" : $string #optional,
  "id" : $string,
  "parameters" : $parameters #optional,
  "outputs" : [ $response_output, ... ]
}
```

- `id`：請求中給出的 `id` 標識符，如果有的話。
- `model_name`: 用於推理的模型的名稱。
- `model_version`: 用於推理的特定模型版本。未實現版本控制的推理服務器不應在響應中提供此字段。
- `parameters`：包含此響應的零個或多個參數的對象，表示為鍵/值對。有關詳細信息，請參閱[參數](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#parameters)。
- `outputs`：為該推理請求的輸出張量。每個請求的輸出都使用請求輸出中定義的 `$request_output` 模式進行描述。

### Tensor Data

張量數據必須以張量元素的行優先順序呈現。元素值必須以“線性”順序給出，元素之間沒有任何步幅或填充。張量元素可以以其本質的多維表示形式呈現，或作為扁平的一維表示形式呈現。

明確給出的張量數據在 JSON array 中提供。Array 的每個元素可以是整數、浮點數、字符串或布爾值。服務器可以決定將每個元素強制轉換為所需的類型，或者在收到意外值時返回錯誤。請注意，`fp16` 和 `bf16` 顯式通信存在問題，因為沒有跨後端的標準 `fp16/bf16` 表示，通常也沒有為 JSON 數字創建 `fp16/bf16` 表示的編程支持。

例如二維矩陣：

```json
[ 1 2
  4 5 ]
```

可以表示為：

```json
"data" : [ [ 1, 2 ], [ 4, 5 ] ]
```

或者在扁平化的一維表示中：

```json
"data" : [ 1, 2, 4, 5 ]
```

**Tensor Data Types**

下表顯示了張量數據類型以及每種類型的大小（以 byte 為單位）。

| Data Type	|Size (bytes)|
|-----------|------------|
|BOOL	|1|
|UINT8	|1|
|UINT16	|2|
|UINT32	|4|
|UINT64	|8|
|INT8	|1|
|INT16	|2|
|INT32	|4|
|INT64	|8|
|FP16	|2|
|FP32	|4|
|FP64	|8|
|BYTES	|Variable (max 232)|

**推理請求示例**

以下示例顯示了對具有兩個輸入和一個輸出的模型的推理請求。 HTTP Content-Length 標頭給出了 JSON 對象的大小。

```bash
POST /v2/models/mymodel/infer HTTP/1.1
Host: localhost:8000
Content-Type: application/json
Content-Length: <xx>
{
  "id" : "42",
  "inputs" : [
    {
      "name" : "input0",
      "shape" : [ 2, 2 ],
      "datatype" : "UINT32",
      "data" : [ 1, 2, 3, 4 ]
    },
    {
      "name" : "input1",
      "shape" : [ 3 ],
      "datatype" : "BOOL",
      "data" : [ true ]
    }
  ],
  "outputs" : [
    {
      "name" : "output0"
    }
  ]
}
```

對於上述請求，推理服務器必須返回 “output0” 輸出張量。假設模型返回數據類型 FP32 的 `[ 3, 2 ]` 張量，將返回以下響應。

```bash
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: <yy>
{
  "id" : "42"
  "outputs" : [
    {
      "name" : "output0",
      "shape" : [ 3, 2 ],
      "datatype"  : "FP32",
      "data" : [ 1.0, 1.1, 2.0, 2.1, 3.0, 3.1 ]
    }
  ]
}
```

## gRPC

GRPC API 嚴格遵循 HTTP/REST API 中定義的概念。合規服務器必須實施本節中描述的健康、元數據和推理 API。

|API	|rpc Endpoint	|Request Message	|Response Message|
|-----|-------------|-----------------|----------------|
|Inference	|[ModelInfer](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#inf)	|ModelInferRequest	|ModelInferResponse|
|Model Ready	|[ModelReady](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#model-ready)	|[ModelReadyRequest]	|ModelReadyResponse|
|Model Metadata	|[ModelMetadata](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#model-metadata)	|ModelMetadataRequest	|ModelMetadataResponse|
|Server Ready	|[ServerReady](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#server-ready)	|ServerReadyRequest	|ServerReadyResponse|
|Server Live	|[ServerLive](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#server-live)	|ServerLiveRequest	|ServerLiveResponse|

有關每個端點及其內容的更多詳細信息，請參閱 API 定義和消息內容。

另請參閱：gRPC 端點、請求/響應消息和內容在 [grpc_predict_v2.proto](https://github.com/kserve/kserve/blob/master/docs/predict-api/v2/grpc_predict_v2.proto) 中定義

### API 定義

服務的 GRPC 定義是：

```grpc
//
// Inference Server GRPC endpoints.
//
service GRPCInferenceService
{
  // Check liveness of the inference server.
  rpc ServerLive(ServerLiveRequest) returns (ServerLiveResponse) {}

  // Check readiness of the inference server.
  rpc ServerReady(ServerReadyRequest) returns (ServerReadyResponse) {}

  // Check readiness of a model in the inference server.
  rpc ModelReady(ModelReadyRequest) returns (ModelReadyResponse) {}

  // Get server metadata.
  rpc ServerMetadata(ServerMetadataRequest) returns (ServerMetadataResponse) {}

  // Get model metadata.
  rpc ModelMetadata(ModelMetadataRequest) returns (ModelMetadataResponse) {}

  // Perform inference using a specific model.
  rpc ModelInfer(ModelInferRequest) returns (ModelInferResponse) {}
}
```

### Message 內容

#### Health

使用 `ServerLive`、`ServerReady` 或 `ModelReady` 端點發出健康請求。對於這些端點中的每一個，錯誤都由為請求返回的 `google.rpc.Status` 指示。 OK 代碼表示成功，其他代碼表示失敗。

**Server Live**

ServerLive API 指示推理服務器是否能夠接收和響應元數據和推理請求。 ServerLive 的請求和響應消息是：

```grpc
message ServerLiveRequest {}

message ServerLiveResponse
{
  // True if the inference server is live, false if not live.
  bool live = 1;
}
```

**Server Ready**

ServerReady API 指示服務器是否已準備好進行推理。 ServerReady 的請求和響應消息是：

```grpc
message ServerReadyRequest {}

message ServerReadyResponse
{
  // True if the inference server is ready, false if not ready.
  bool ready = 1;
}
```

**Model Ready**

ModelReady API 指示特定模型是否已準備好進行推理。 ModelReady 的請求和響應消息是：

```grpc
message ModelReadyRequest
{
  // The name of the model to check for readiness.
  string name = 1;

  // The version of the model to check for readiness. If not given the
  // server will choose a version based on the model and internal policy.
  string version = 2;
}

message ModelReadyResponse
{
  // True if the model is ready, false if not ready.
  bool ready = 1;
}
```

#### Metadata

**Server Metadata**

ServerMetadata API 提供有關服務器的信息。錯誤由為請求返回的 `google.rpc.Status` 指示。 OK 代碼表示成功，其他代碼表示失敗。 ServerMetadata 的請求和響應消息是：

```grpc
message ServerMetadataRequest {}

message ServerMetadataResponse
{
  // The server name.
  string name = 1;

  // The server version.
  string version = 2;

  // The extensions supported by the server.
  repeated string extensions = 3;
}
```

**Model Metadata**

每個模型的元數據 API 提供有關模型的信息。錯誤由為請求返回的 `google.rpc.Status` 指示。 OK 代碼表示成功，其他代碼表示失敗。 ModelMetadata 的請求和響應消息是：

```grpc
message ModelMetadataRequest
{
  // The name of the model.
  string name = 1;

  // The version of the model to check for readiness. If not given the
  // server will choose a version based on the model and internal policy.
  string version = 2;
}

message ModelMetadataResponse
{
  // Metadata for a tensor.
  message TensorMetadata
  {
    // The tensor name.
    string name = 1;

    // The tensor data type.
    string datatype = 2;

    // The tensor shape. A variable-size dimension is represented
    // by a -1 value.
    repeated int64 shape = 3;
  }

  // The model name.
  string name = 1;

  // The versions of the model available on the server.
  repeated string versions = 2;

  // The model's platform. See Platforms.
  string platform = 3;

  // The model's inputs.
  repeated TensorMetadata inputs = 4;

  // The model's outputs.
  repeated TensorMetadata outputs = 5;
}
```

**Platforms**

平台是指示 DL/ML 框架或後端的字符串。平台作為對模型元數據請求的響應的一部分返回，但僅供參考。提議的推理 API 相對於模型使用的 DL/ML 框架是通用的，因此客戶端不需要知道給定模型的平台即可使用 API。平台名稱使用格式“_”。允許使用以下平台名稱：

- `tensorrt_plan` : A TensorRT model encoded as a serialized engine or “plan”.
- `tensorflow_graphdef` : A TensorFlow model encoded as a GraphDef.
- `tensorflow_savedmodel` : A TensorFlow model encoded as a SavedModel.
- `onnx_onnxv1` : A ONNX model encoded for ONNX Runtime.
- `pytorch_torchscript` : A PyTorch model encoded as TorchScript.
- `mxnet_mxnet`: An MXNet model
- `caffe2_netdef` : A Caffe2 model encoded as a NetDef.

#### Inference

ModelInfer API 使用指定的模型執行推理。錯誤由為請求返回的 `google.rpc.Status` 指示。 OK 代碼表示成功，其他代碼表示失敗。 ModelInfer 的請求和響應消息是：

```grpc
message ModelInferRequest
{
  // An input tensor for an inference request.
  message InferInputTensor
  {
    // The tensor name.
    string name = 1;

    // The tensor data type.
    string datatype = 2;

    // The tensor shape.
    repeated int64 shape = 3;

    // Optional inference input tensor parameters.
    map<string, InferParameter> parameters = 4;

    // The tensor contents using a data-type format. This field must
    // not be specified if "raw" tensor contents are being used for
    // the inference request.
    InferTensorContents contents = 5;
  }

  // An output tensor requested for an inference request.
  message InferRequestedOutputTensor
  {
    // The tensor name.
    string name = 1;

    // Optional requested output tensor parameters.
    map<string, InferParameter> parameters = 2;
  }

  // The name of the model to use for inferencing.
  string model_name = 1;

  // The version of the model to use for inference. If not given the
  // server will choose a version based on the model and internal policy.
  string model_version = 2;

  // Optional identifier for the request. If specified will be
  // returned in the response.
  string id = 3;

  // Optional inference parameters.
  map<string, InferParameter> parameters = 4;

  // The input tensors for the inference.
  repeated InferInputTensor inputs = 5;

  // The requested output tensors for the inference. Optional, if not
  // specified all outputs produced by the model will be returned.
  repeated InferRequestedOutputTensor outputs = 6;

  // The data contained in an input tensor can be represented in "raw"
  // bytes form or in the repeated type that matches the tensor's data
  // type. To use the raw representation 'raw_input_contents' must be
  // initialized with data for each tensor in the same order as
  // 'inputs'. For each tensor, the size of this content must match
  // what is expected by the tensor's shape and data type. The raw
  // data must be the flattened, one-dimensional, row-major order of
  // the tensor elements without any stride or padding between the
  // elements. Note that the FP16 data type must be represented as raw
  // content as there is no specific data type for a 16-bit float
  // type.
  //
  // If this field is specified then InferInputTensor::contents must
  // not be specified for any input tensor.
  repeated bytes raw_input_contents = 7;
}

message ModelInferResponse
{
  // An output tensor returned for an inference request.
  message InferOutputTensor
  {
    // The tensor name.
    string name = 1;

    // The tensor data type.
    string datatype = 2;

    // The tensor shape.
    repeated int64 shape = 3;

    // Optional output tensor parameters.
    map<string, InferParameter> parameters = 4;

    // The tensor contents using a data-type format. This field must
    // not be specified if "raw" tensor contents are being used for
    // the inference response.
    InferTensorContents contents = 5;
  }

  // The name of the model used for inference.
  string model_name = 1;

  // The version of the model used for inference.
  string model_version = 2;

  // The id of the inference request if one was specified.
  string id = 3;

  // Optional inference response parameters.
  map<string, InferParameter> parameters = 4;

  // The output tensors holding inference results.
  repeated InferOutputTensor outputs = 5;

  // The data contained in an output tensor can be represented in
  // "raw" bytes form or in the repeated type that matches the
  // tensor's data type. To use the raw representation 'raw_output_contents'
  // must be initialized with data for each tensor in the same order as
  // 'outputs'. For each tensor, the size of this content must match
  // what is expected by the tensor's shape and data type. The raw
  // data must be the flattened, one-dimensional, row-major order of
  // the tensor elements without any stride or padding between the
  // elements. Note that the FP16 data type must be represented as raw
  // content as there is no specific data type for a 16-bit float
  // type.
  //
  // If this field is specified then InferOutputTensor::contents must
  // not be specified for any output tensor.
  repeated bytes raw_output_contents = 6;
}
```

**Parameters**

參數消息描述了一個 “名稱/值” 對，其中 “名稱” 是參數的名稱，“值” 是與參數對應的布爾值、整數或字符串。

當前，未定義任何參數。根據需要，未來的提案可能會定義一個或多個標準參數，以允許跨不同推理服務器的可移植功能。服務器可以實現特定於服務器的參數以提供非標準功能。

```grpc
//
// An inference parameter value.
//
message InferParameter
{
  // The parameter value can be a string, an int64, a boolean
  // or a message specific to a predefined parameter.
  oneof parameter_choice
  {
    // A boolean parameter value.
    bool bool_param = 1;

    // An int64 parameter value.
    int64 int64_param = 2;

    // A string parameter value.
    string string_param = 3;
  }
}
```

#### Tensor Data

在所有表示中，張量數據必須展平為張量元素的一維、行主序。元素值必須以“線性”順序給出，元素之間沒有任何步幅或填充。

由於 protobuf 分配和重用與 GRPC 交互的方式，將張量的“原始”表示與 `ModelInferRequest::raw_input_contents` 和 `ModelInferResponse::raw_output_contents` 使用通常會允許更高的性能。例如，請參閱 https://github.com/grpc/grpc/issues/23231。

“raw”表示的替代方法是使用 `InferTensorContents` 以與張量數據類型匹配的格式表示張量數據。

```grpc
//
// The data contained in a tensor represented by the repeated type
// that matches the tensor's data type. Protobuf oneof is not used
// because oneofs cannot contain repeated fields.
//
message InferTensorContents
{
  // Representation for BOOL data type. The size must match what is
  // expected by the tensor's shape. The contents must be the flattened,
  // one-dimensional, row-major order of the tensor elements.
  repeated bool bool_contents = 1;

  // Representation for INT8, INT16, and INT32 data types. The size
  // must match what is expected by the tensor's shape. The contents
  // must be the flattened, one-dimensional, row-major order of the
  // tensor elements.
  repeated int32 int_contents = 2;

  // Representation for INT64 data types. The size must match what
  // is expected by the tensor's shape. The contents must be the
  // flattened, one-dimensional, row-major order of the tensor elements.
  repeated int64 int64_contents = 3;

  // Representation for UINT8, UINT16, and UINT32 data types. The size
  // must match what is expected by the tensor's shape. The contents
  // must be the flattened, one-dimensional, row-major order of the
  // tensor elements.
  repeated uint32 uint_contents = 4;

  // Representation for UINT64 data types. The size must match what
  // is expected by the tensor's shape. The contents must be the
  // flattened, one-dimensional, row-major order of the tensor elements.
  repeated uint64 uint64_contents = 5;

  // Representation for FP32 data type. The size must match what is
  // expected by the tensor's shape. The contents must be the flattened,
  // one-dimensional, row-major order of the tensor elements.
  repeated float fp32_contents = 6;

  // Representation for FP64 data type. The size must match what is
  // expected by the tensor's shape. The contents must be the flattened,
  // one-dimensional, row-major order of the tensor elements.
  repeated double fp64_contents = 7;

  // Representation for BYTES data type. The size must match what is
  // expected by the tensor's shape. The contents must be the flattened,
  // one-dimensional, row-major order of the tensor elements.
  repeated bytes bytes_contents = 8;
}
```

**Tensor Data Types**

下表顯示了張量數據類型以及每種類型的大小（以 `byte` 為單位）。

|Data Type	|Size (bytes)|
|-----------|------------|
|BOOL	|1|
|UINT8	|1|
|UINT16	|2|
|UINT32	|4|
|UINT64	|8|
|INT8	|1|
|INT16	|2|
|INT32	|4|
|INT64	|8|
|FP16	|2|
|FP32	|4|
|FP64	|8|
|BYTES	|Variable (max 2的32次方)|

